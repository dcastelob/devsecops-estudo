# Módulo 2 — Docker e OCI Runtime Security
## Aula 2.10 — Segurança de Imagens de Containers: Vulnerabilidades, SBOM e Supply Chain

Até aqui, estudamos como o Linux e o runtime isolam e controlam os processos de containers. Agora vamos analisar outra parte fundamental da segurança: **a imagem que fornece o filesystem, as bibliotecas, os executáveis e a aplicação que serão executados**.

Uma imagem pode estar sendo executada com todas as proteções corretas — usuário não privilegiado, capabilities removidas, Seccomp, AppArmor e limites de recursos — e ainda assim conter vulnerabilidades exploráveis.

Por isso, precisamos tratar a imagem como um artefato de software que deve ser inspecionado, rastreado, assinado e verificado antes de chegar ao ambiente de produção.

### Objetivos da aula

Ao final, você deverá conseguir:

- Inspecionar a estrutura de uma imagem OCI.
- Identificar vulnerabilidades em pacotes do sistema operacional e dependências da aplicação.
- Interpretar CVE, CVSS e EPSS.
- Gerar e analisar um SBOM.
- Assinar imagens e verificar suas assinaturas com Cosign.
- Entender proveniência e cadeia de confiança.
- Integrar verificações de segurança ao GitLab CI.
- Aplicar controles de segurança em um registry Harbor.
- Entender como GitLab, Harbor e ArgoCD participam da cadeia de fornecimento de software.

---

# 1. Anatomia de uma imagem OCI

Antes de procurar vulnerabilidades, precisamos entender exatamente o que estamos analisando.

Uma imagem de container não é uma máquina virtual compactada. É um conjunto de metadados e camadas de filesystem que, combinados, fornecem o ambiente necessário para iniciar um processo.

## 1.1 Os principais componentes

Uma imagem OCI possui alguns elementos importantes:

| Componente | Função |
|---|---|
| Manifest | Descreve a imagem e referencia seus componentes |
| Config | Contém configurações da imagem, como entrypoint, usuário e variáveis |
| Layers | Camadas de filesystem que compõem a imagem |
| Digest | Identificador criptográfico do conteúdo |
| Annotations | Metadados adicionais |

Uma representação simplificada:

```text
Imagem OCI
│
├── Manifest
│   │
│   ├── Config
│   │   ├── Entrypoint
│   │   ├── Cmd
│   │   ├── Env
│   │   ├── User
│   │   └── WorkingDir
│   │
│   ├── Layer 1
│   ├── Layer 2
│   ├── Layer 3
│   └── Layer N
│
└── Digest
```

O manifest referencia o objeto de configuração e as camadas. Esses objetos são identificados por digests.

## 1.2 O que é um digest?

Um digest é um identificador calculado a partir do conteúdo de um objeto.

Exemplo:

```text
sha256:8a1c...f7d2
```

É comum encontrarmos imagens referenciadas por tags:

```bash
docker pull nginx:latest
```

O problema é que uma tag é mutável.

Hoje:

```text
nginx:latest → sha256:AAAA
```

Amanhã:

```text
nginx:latest → sha256:BBBB
```

O nome continua igual, mas o conteúdo pode ter mudado.

Por isso, em ambientes de produção, é preferível utilizar imagens referenciadas por digest:

```bash
docker pull nginx@sha256:SEU_DIGEST
```

Isso permite identificar precisamente o artefato utilizado.

**Importante:** digest garante identificação e integridade do conteúdo referenciado, mas não garante que a imagem seja confiável. Uma imagem maliciosa também possui um digest válido.

## 1.3 Inspecionando uma imagem

Vamos utilizar uma imagem pequena para nossos primeiros experimentos:

```bash
docker pull alpine:3.22
```

Inspecione seus metadados:

```bash
docker image inspect alpine:3.22
```

Alguns campos interessantes:

```bash
docker image inspect alpine:3.22 \
  --format '{{.Id}}'

docker image inspect alpine:3.22 \
  --format '{{json .RepoDigests}}'

docker image inspect alpine:3.22 \
  --format '{{json .Config}}'
```

Observe:

- ID da imagem.
- Digest do registry.
- Usuário configurado.
- Entrypoint.
- Cmd.
- Variáveis de ambiente.
- Working directory.

Para visualizar as camadas:

```bash
docker history alpine:3.22
```

O comando mostra o histórico de construção da imagem.

Entretanto, existe uma distinção importante:

**`docker history` não é um inventário completo dos arquivos existentes na imagem.** Ele apresenta o histórico de camadas e instruções de construção disponíveis nos metadados.

Para uma análise mais detalhada, podemos utilizar ferramentas OCI, como `skopeo` e `umoci`.

### 1.4 Inspecionando o conteúdo das camadas

Instale as ferramentas auxiliares, caso ainda não estejam disponíveis:

```bash
sudo apt update
sudo apt install skopeo umoci jq
```

Copie a imagem para um layout OCI local:

```bash
mkdir -p ~/image-security-lab
cd ~/image-security-lab

skopeo copy \
  docker://docker.io/library/alpine:3.22 \
  oci:alpine-oci:3.22
```

O diretório `alpine-oci` conterá a representação OCI da imagem.

Inspecione:

```bash
find alpine-oci -maxdepth 3 -type f
```

Você encontrará arquivos como:

```text
alpine-oci/
├── blobs/
│   └── sha256/
│       ├── ...
├── index.json
└── oci-layout
```

O arquivo `index.json` aponta para os manifests disponíveis no layout. Os objetos são armazenados em `blobs/sha256/`.

Para desempacotar a imagem:

```bash
umoci unpack \
  --image alpine-oci:3.22 \
  alpine-bundle
```

O resultado terá uma estrutura semelhante a:

```text
alpine-bundle/
├── config.json
└── rootfs/
    ├── bin/
    ├── etc/
    ├── lib/
    ├── sbin/
    └── usr/
```

Agora você consegue examinar o filesystem sem iniciar um container:

```bash
find alpine-bundle/rootfs -maxdepth 2 -type d
```

Isso é útil para análises de segurança porque permite inspecionar o conteúdo de uma imagem sem executar seus binários.

**Uma observação importante:** uma imagem OCI pode conter várias plataformas, como `linux/amd64` e `linux/arm64`. Um índice OCI pode apontar para manifests diferentes. Portanto, ao analisar uma imagem, precisamos saber qual plataforma será utilizada.

---

# 2. Vulnerabilidades em imagens de containers

Uma imagem pode conter vulnerabilidades em diferentes componentes.

Considere uma aplicação Java construída sobre uma imagem Debian:

```dockerfile
FROM debian:13-slim

RUN apt-get update && \
    apt-get install -y openjdk-21-jre-headless

COPY app.jar /app/app.jar

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Podemos separar os riscos em três categorias.

## 2.1 Vulnerabilidades do sistema operacional

São vulnerabilidades em pacotes instalados na imagem:

- OpenSSL;
- glibc;
- curl;
- zlib;
- bash;
- OpenSSH;
- bibliotecas e utilitários do sistema.

Exemplo:

```text
Imagem: debian:13-slim

Pacote: openssl
Versão: 3.x
CVE: CVE-XXXX-YYYY
Severidade: HIGH
```

Mesmo que sua aplicação não utilize diretamente o OpenSSL, ele pode estar presente na imagem e ser utilizado por outros componentes.

## 2.2 Vulnerabilidades das dependências da aplicação

Uma aplicação Java pode ter dependências vulneráveis mesmo que a imagem base esteja atualizada.

Exemplo:

```text
/app/app.jar
    |
    +-- spring-core
    +-- spring-web
    +-- logback
    +-- jackson-databind
    +-- commons-io
```

Essas dependências podem apresentar CVEs próprias.

O scanner precisa ser capaz de identificar os componentes da aplicação, e não apenas os pacotes do sistema operacional.

Para aplicações Java, ferramentas de análise podem identificar dependências em arquivos JAR, inclusive quando estão empacotadas em um JAR executável.

## 2.3 Vulnerabilidades introduzidas durante a construção

Considere:

```dockerfile
FROM debian:13

RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get install -y net-tools
```

Essa imagem pode conter ferramentas desnecessárias para executar a aplicação.

Cada pacote adicional aumenta a superfície de ataque.

Imagine um container de aplicação web que não precisa de:

- Compilador;
- Editor de texto;
- Cliente SSH;
- Ferramentas de diagnóstico;
- Utilitários de rede desnecessários.

Se essas ferramentas estiverem presentes, um invasor que conseguir executar comandos dentro do container poderá encontrar mais recursos disponíveis.

### 2.4 O problema de atualizar pacotes em camadas diferentes

Observe:

```dockerfile
FROM debian:13

RUN apt-get update
RUN apt-get install -y curl

RUN apt-get update
RUN apt-get upgrade -y
```

O Docker constrói camadas. Uma atualização posterior pode corrigir um pacote no filesystem final, mas o conteúdo antigo ainda pode permanecer armazenado em camadas anteriores.

O scanner normalmente analisa o filesystem resultante, mas ferramentas que inspecionam o histórico ou as camadas também podem identificar dados antigos que não aparecem no filesystem final.

Uma construção melhor seria:

```dockerfile
FROM debian:13-slim

RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

Isso reduz camadas desnecessárias e evita manter o cache do APT na imagem final.

**Atenção:** `apt-get upgrade` não deve ser utilizado indiscriminadamente em toda construção. É importante definir uma estratégia de atualização e reconstrução da imagem base, testar compatibilidade e manter builds reproduzíveis.

---

# 3. CVE, CVSS e EPSS

Para entender os resultados dos scanners, precisamos conhecer três conceitos.

## 3.1 CVE — Common Vulnerabilities and Exposures

O CVE é um identificador público utilizado para referenciar uma vulnerabilidade conhecida.

Exemplo fictício:

```text
CVE-2026-12345
```

A estrutura é:

```text
CVE-ANO-NÚMERO
```

O identificador permite que fabricantes, pesquisadores, scanners e equipes de segurança falem sobre a mesma vulnerabilidade.

Uma CVE não significa necessariamente que todos os sistemas que possuem determinado pacote estejam vulneráveis.

É necessário considerar:

- Versão afetada;
- Distribuição Linux;
- Correções aplicadas pelo fabricante;
- Configuração;
- Possibilidade de exploração;
- Exposição do componente.

Um detalhe muito importante em distribuições Linux:

**A versão do pacote nem sempre indica corretamente se a vulnerabilidade foi corrigida.**

Distribuições como Debian podem aplicar patches de segurança sem atualizar o pacote para a mesma versão upstream mais recente.

Por isso, scanners precisam considerar os metadados de segurança da distribuição.

## 3.2 CVSS — Common Vulnerability Scoring System

O CVSS é uma metodologia para atribuir uma pontuação de severidade a uma vulnerabilidade.

A escala vai de 0 a 10.

| Pontuação | Classificação |
|---:|---|
| 0.0 | None |
| 0.1 – 3.9 | Low |
| 4.0 – 6.9 | Medium |
| 7.0 – 8.9 | High |
| 9.0 – 10.0 | Critical |

O CVSS considera características como:

- Vetor de ataque;
- Complexidade do ataque;
- Privilégios necessários;
- Interação do usuário;
- Impacto sobre confidencialidade;
- Impacto sobre integridade;
- Impacto sobre disponibilidade.

Exemplo conceitual:

```text
Vulnerabilidade A
CVSS: 9.8
Severidade: CRITICAL

Vulnerabilidade B
CVSS: 5.3
Severidade: MEDIUM
```

Isso não significa necessariamente que a vulnerabilidade A seja a primeira que devemos corrigir em todos os cenários.

O CVSS mede severidade técnica, mas não representa sozinho a probabilidade de exploração no ambiente.

## 3.3 EPSS — Exploit Prediction Scoring System

O EPSS procura estimar a probabilidade de uma vulnerabilidade ser explorada na prática nos próximos 30 dias.

Exemplo:

```text
CVE-XXXX-YYYY

CVSS: 8.8
EPSS: 0.92
```

O EPSS de `0.92` representa uma estimativa de 92% de probabilidade de exploração nos próximos 30 dias, segundo o modelo EPSS.

Outro exemplo:

```text
CVE-AAAA-BBBB

CVSS: 9.8
EPSS: 0.01
```

Essa vulnerabilidade possui severidade técnica muito alta, mas uma probabilidade estimada de exploração muito menor.

O EPSS não garante que uma vulnerabilidade será ou não explorada. É uma estimativa probabilística.

### 3.4 Como combinar CVE, CVSS e EPSS?

Uma estratégia de priorização pode considerar:

| Critério | Pergunta |
|---|---|
| CVE | Qual vulnerabilidade foi identificada? |
| CVSS | Qual é a severidade técnica? |
| EPSS | Qual a probabilidade estimada de exploração? |
| Exposição | O componente está acessível por uma rede não confiável? |
| Contexto | O componente vulnerável é realmente utilizado? |
| Correção | Existe versão corrigida ou mitigação? |

Por exemplo:

Uma biblioteca vulnerável em um componente que processa requisições HTTP externas merece atenção diferente de uma biblioteca presente em uma ferramenta de build que não existe na imagem de produção.

**Não devemos priorizar correções somente pela pontuação CVSS.** O contexto de execução e a exposição real são fundamentais.

---

# 4. Trivy — Scanner de vulnerabilidades

O Trivy é uma ferramenta de segurança desenvolvida pela Aqua Security.

Ela consegue analisar:

- Imagens de containers;
- Filesystems;
- Repositórios Git;
- Configurações de infraestrutura como código;
- Kubernetes;
- SBOMs;
- Dependências de aplicações.

É uma ferramenta bastante útil para incorporar verificações de segurança ao pipeline.

## 4.1 Instalando o Trivy no Debian 13

Uma opção é utilizar o repositório oficial da ferramenta.

Consulte as instruções atualizadas de instalação na documentação oficial:

[Documentação de instalação do Trivy](https://trivy.dev/latest/getting-started/installation/?utm_source=chatgpt.com)

Depois de instalado:

```bash
trivy --version
```

## 4.2 Analisando uma imagem

Vamos analisar a imagem Alpine:

```bash
trivy image alpine:3.22
```

O Trivy irá:

1. Obter a imagem;
2. Identificar o sistema operacional;
3. Identificar os pacotes instalados;
4. Comparar as versões com suas bases de vulnerabilidades;
5. Apresentar os resultados.

Exemplo ilustrativo:

```text
alpine:3.22

Total: 12

┌────────────┬──────────┬─────────────────┐
│ Library    │ Severity │ Vulnerability   │
├────────────┼──────────┼─────────────────┤
│ openssl    │ HIGH     │ CVE-XXXX-YYYY   │
│ busybox    │ MEDIUM   │ CVE-AAAA-BBBB   │
│ libcrypto  │ LOW      │ CVE-CCCC-DDDD   │
└────────────┴──────────┴─────────────────┘
```

Os números acima são apenas ilustrativos. O resultado real depende da versão da imagem e da base de vulnerabilidades consultada.

## 4.3 Filtrando por severidade

Para mostrar somente vulnerabilidades importantes:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  alpine:3.22
```

Para gerar JSON:

```bash
trivy image \
  --format json \
  --output trivy-report.json \
  alpine:3.22
```

Podemos extrair os resultados com `jq`:

```bash
jq '.Results[]? | {
  Target,
  Type,
  Vulnerabilities
}' trivy-report.json
```

## 4.4 Fazendo o scanner retornar erro no pipeline

Por padrão, encontrar vulnerabilidades não significa necessariamente que o comando terminará com código diferente de zero.

Podemos definir um limite:

```bash
trivy image \
  --exit-code 1 \
  --severity HIGH,CRITICAL \
  alpine:3.22
```

Nesse caso, o Trivy retorna código de erro quando encontra vulnerabilidades das severidades especificadas.

Isso é fundamental para transformar o scanner em um controle de qualidade do pipeline.

## 4.5 Ignorando vulnerabilidades justificadas

Em alguns casos, uma vulnerabilidade não pode ser corrigida imediatamente.

O Trivy permite trabalhar com arquivos de exceção, como `.trivyignore`.

Exemplo conceitual:

```text
CVE-2026-12345
```

A exceção deve ser documentada e ter justificativa, responsável e prazo de revisão.

Evite ignorar vulnerabilidades indiscriminadamente ou criar exceções permanentes sem avaliação.

---

# 5. Grype — Outro scanner de vulnerabilidades

O Grype é um scanner open source desenvolvido pela Anchore.

Ele analisa:

- Imagens de containers;
- Filesystems;
- SBOMs;
- Pacotes e dependências.

Documentação oficial:

[Documentação do Grype](https://oss.anchore.com/docs/?utm_source=chatgpt.com)

## 5.1 Instalando

Consulte o procedimento oficial de instalação para Debian/Linux:

[Instalação do Grype](https://oss.anchore.com/docs/installation/grype/?utm_source=chatgpt.com)

Depois:

```bash
grype version
```

## 5.2 Analisando uma imagem

```bash
grype alpine:3.22
```

Para gerar JSON:

```bash
grype alpine:3.22 \
  -o json \
  > grype-report.json
```

Para filtrar severidades:

```bash
grype alpine:3.22 \
  --only-fixed \
  --fail-on high
```

O parâmetro `--only-fixed` restringe a apresentação às vulnerabilidades para as quais existe correção conhecida.

O `--fail-on high` define o limite de severidade para o código de saída.

## 5.3 Trivy versus Grype

| Característica | Trivy | Grype |
|---|---|---|
| Scanner de imagens | Sim | Sim |
| Pacotes de sistema operacional | Sim | Sim |
| Dependências de aplicação | Sim | Sim |
| SBOM | Gera e consome | Consome e integra-se à geração |
| IaC | Sim | Não é seu foco principal |
| Kubernetes | Sim | Não é seu foco principal |
| Integração CI/CD | Sim | Sim |
| Ecossistema | Aqua Security | Anchore |

Minha sugestão para o laboratório é utilizar o Trivy como scanner principal e o Grype como ferramenta complementar.

Isso permite comparar resultados e compreender que scanners diferentes podem apresentar divergências por causa de:

- Bases de vulnerabilidades;
- Identificação de pacotes;
- Tratamento de versões;
- Regras de correspondência;
- Metadados disponíveis.

Uma divergência não significa automaticamente que um dos scanners esteja errado. É necessário investigar o pacote e a origem da informação.

---

# 6. SBOM — Software Bill of Materials

Agora chegamos a um dos conceitos mais importantes da segurança da cadeia de fornecimento de software.

SBOM significa *Software Bill of Materials*, ou lista de materiais de software.

É um inventário dos componentes que constituem determinado software.

Pense em uma imagem de container como um produto industrial. Para avaliar sua segurança, precisamos saber quais componentes foram utilizados para construí-la.

Um SBOM pode conter:

- Nome dos componentes;
- Versões;
- Fabricantes;
- Licenças;
- Relacionamentos entre dependências;
- Identificadores de pacotes;
- Hashes;
- Informações sobre origem.

## 6.1 Por que precisamos de SBOM?

Imagine que uma vulnerabilidade crítica seja divulgada em uma biblioteca Java.

Sua equipe possui 150 imagens distribuídas entre desenvolvimento, homologação e produção.

Sem um inventário de componentes, você precisará descobrir manualmente quais aplicações utilizam aquela biblioteca.

Com SBOMs armazenados e associados às imagens, podemos pesquisar quais artefatos contêm o componente vulnerável.

O SBOM também auxilia em:

- Auditorias;
- Gestão de licenças;
- Resposta a incidentes;
- Rastreabilidade;
- Análise de dependências;
- Gestão de risco de fornecedores.

## 6.2 Formatos de SBOM

Existem dois formatos bastante conhecidos:

**CycloneDX**

Desenvolvido pela OWASP, possui foco em segurança da cadeia de fornecimento e inventário de componentes.

Documentação: [CycloneDX](https://cyclonedx.org/?utm_source=chatgpt.com)

**SPDX**

É um padrão para representar informações sobre componentes de software, licenças e proveniência.

Documentação: [SPDX](https://spdx.dev/?utm_source=chatgpt.com)

Ambos são formatos relevantes. A escolha depende das ferramentas e dos processos da organização.

## 6.3 Syft — Gerando um SBOM

O Syft é uma ferramenta open source da Anchore que identifica componentes e gera SBOMs.

Documentação:

[Documentação do Syft](https://oss.anchore.com/docs/installation/syft/?utm_source=chatgpt.com)

Instale conforme as instruções oficiais e confira:

```bash
syft version
```

Gere um SBOM da imagem:

```bash
syft alpine:3.22
```

O Syft irá identificar os pacotes presentes na imagem.

Para gerar CycloneDX:

```bash
syft alpine:3.22 \
  -o cyclonedx-json \
  > alpine-sbom.cdx.json
```

Para gerar SPDX:

```bash
syft alpine:3.22 \
  -o spdx-json \
  > alpine-sbom.spdx.json
```

Inspecione o arquivo:

```bash
jq '.components[] | {
  name,
  version,
  type,
  purl
}' alpine-sbom.cdx.json
```

O campo `purl` é o *Package URL*. Ele identifica um pacote de forma padronizada.

Exemplo ilustrativo:

```text
pkg:deb/debian/openssl@3.x
```

Isso facilita a identificação de um componente independentemente do nome utilizado em determinada ferramenta.

## 6.4 Utilizando o SBOM com Grype

Uma das vantagens de gerar SBOM é poder reutilizá-lo em outras ferramentas.

Por exemplo:

```bash
grype sbom:alpine-sbom.cdx.json
```

Assim, podemos separar duas atividades:

1. Identificar os componentes e gerar o inventário.
2. Correlacionar o inventário com as bases de vulnerabilidades.

Essa separação é especialmente interessante em pipelines e ambientes corporativos.

## 6.5 SBOM não é uma garantia de segurança

Um SBOM não significa que o software é seguro.

Ele informa quais componentes foram identificados.

Não garante, por si só:

- Que todos os componentes foram identificados;
- Que não existem vulnerabilidades;
- Que o software foi construído de maneira confiável;
- Que o código não contém comportamento malicioso;
- Que o artefato implantado corresponde ao SBOM.

Por isso, precisamos associar o SBOM a um artefato específico, preferencialmente identificado por digest, e protegê-lo contra adulteração.

---

# 7. Cosign — Assinatura de imagens

Até agora, aprendemos a identificar componentes e vulnerabilidades.

Mas existe outra pergunta:

**Como saber se a imagem que estamos executando foi realmente produzida e aprovada pelo nosso processo de build?**

É aqui que entra a assinatura de imagens.

O Cosign faz parte do projeto Sigstore.

Documentação:

[Documentação do Cosign](https://docs.sigstore.dev/cosign/overview/?utm_source=chatgpt.com)

O Cosign permite assinar imagens OCI e verificar posteriormente suas assinaturas.

## 7.1 Por que assinar uma imagem?

Imagine o seguinte cenário:

```text
GitLab CI
    |
    v
Build da imagem
    |
    v
Harbor
    |
    v
Kubernetes
```

A equipe de DevOps constrói uma imagem e envia para o Harbor.

Entretanto, alguém com acesso ao registry pode publicar outra imagem utilizando a mesma tag.

Exemplo:

```text
aplicacao:1.0
```

A tag é a mesma, mas o conteúdo pode ser diferente.

Com assinatura, podemos verificar se o artefato corresponde a uma identidade confiável.

A assinatura não impede que alguém publique uma imagem adulterada. Ela permite detectar que a imagem não possui uma assinatura válida de uma identidade autorizada.

## 7.2 Instalação do Cosign

Consulte as instruções oficiais:

[Instalação do Cosign](https://docs.sigstore.dev/cosign/system_config/installation/?utm_source=chatgpt.com)

Depois:

```bash
cosign version
```

## 7.3 Gerando um par de chaves

Para nosso primeiro laboratório, vamos utilizar assinatura baseada em chave.

```bash
cosign generate-key-pair
```

Serão criados arquivos semelhantes a:

```text
cosign.key
cosign.pub
```

A chave privada deve ser protegida. Não a coloque no Git, na imagem ou em um diretório público.

## 7.4 Assinando uma imagem

Para facilitar o laboratório, vamos utilizar uma imagem publicada em um registry.

Primeiro, descubra seu digest:

```bash
docker pull alpine:3.22

docker image inspect alpine:3.22 \
  --format '{{json .RepoDigests}}'
```

O resultado terá uma referência semelhante a:

```text
docker.io/library/alpine@sha256:...
```

Defina a referência exata:

```bash
IMAGE='docker.io/library/alpine@sha256:SEU_DIGEST'
```

Assine:

```bash
cosign sign --key cosign.key "$IMAGE"
```

Atenção: a assinatura de uma imagem em registry exige que o usuário tenha permissão para publicar os objetos de assinatura. Para este laboratório, utilize um registry e uma imagem sobre os quais você tenha autorização.

## 7.5 Verificando a assinatura

```bash
cosign verify \
  --key cosign.pub \
  "$IMAGE"
```

O Cosign verificará a assinatura e a associação com o digest do artefato.

Se o conteúdo da imagem mudar, o digest também mudará, e a assinatura anterior não será válida para o novo digest.

## 7.6 Assinatura com chave versus keyless

O exemplo anterior utiliza uma chave privada armazenada pelo usuário.

Existe também o modelo *keyless signing*, no qual a identidade do assinante é comprovada por meio de um provedor de identidade e a assinatura é registrada em uma infraestrutura de transparência.

O ecossistema Sigstore utiliza componentes como:

- Fulcio: autoridade certificadora para certificados de curta duração;
- Rekor: log de transparência;
- Cosign: assinatura e verificação de artefatos.

A vantagem é reduzir a necessidade de distribuir e armazenar chaves privadas de longa duração.

Entretanto, é necessário configurar corretamente a identidade que poderá assinar e a identidade esperada na verificação.

**Ponto de segurança:** verificar apenas que existe uma assinatura não é suficiente. Precisamos verificar quem assinou, qual artefato foi assinado e se aquela identidade está autorizada.

---

# 8. Proveniência — De onde veio a imagem?

A assinatura responde a uma pergunta:

> Quem assinou este artefato?

A proveniência (*provenance*) procura responder:

> Como, onde e a partir de quais fontes este artefato foi construído?

Imagine uma imagem:

```text
harbor.exemplo.local/sistema/api:1.0
```

Queremos saber:

- Qual repositório Git originou o código?
- Qual commit foi utilizado?
- Qual pipeline executou o build?
- Qual ferramenta construiu a imagem?
- Qual Dockerfile foi utilizado?
- Quais dependências foram utilizadas?
- Qual identidade executou o processo?
- O artefato corresponde ao build aprovado?

Essas informações compõem a proveniência.

## 8.1 SLSA

O SLSA (*Supply-chain Levels for Software Artifacts*) é um framework para melhorar a segurança da cadeia de fornecimento de software.

Documentação:

[SLSA](https://slsa.dev/?utm_source=chatgpt.com)

O SLSA estabelece requisitos para fortalecer a integridade do processo de construção.

Entre os aspectos importantes estão:

- Proveniência;
- Identidade do processo de build;
- Integridade dos artefatos;
- Isolamento do processo de construção;
- Proteção contra adulteração.

Não basta gerar um arquivo dizendo que uma imagem foi construída por determinado pipeline. Precisamos confiar na origem e na integridade das informações.

## 8.2 Attestation

Uma attestation é uma declaração estruturada sobre determinado artefato.

Ela pode conter informações como:

```text
Subject:
  imagem: api
  digest: sha256:...

Predicate:
  source:
    repository: gitlab.exemplo.local/projeto/api
    commit: abc123

  builder:
    pipeline: build-1234

  build:
    dockerfile: Dockerfile
    result: success
```

O conteúdo acima é ilustrativo.

Uma attestation pode ser assinada e armazenada junto ao artefato.

O Cosign oferece suporte a attestations:

```bash
cosign attest \
  --key cosign.key \
  --type cyclonedx \
  --predicate alpine-sbom.cdx.json \
  "$IMAGE"
```

A ideia é associar o SBOM ao digest da imagem.

Posteriormente, podemos verificar a attestation:

```bash
cosign verify-attestation \
  --key cosign.pub \
  --type cyclonedx \
  "$IMAGE"
```

O formato e o tipo de predicate precisam corresponder ao conteúdo produzido. Em um ambiente real, também devemos validar a identidade do assinante e a estrutura da declaração.

## 8.3 Assinatura e proveniência são coisas diferentes

| Recurso | O que comprova |
|---|---|
| Digest | Identidade do conteúdo |
| Assinatura | Integridade e identidade do assinante |
| SBOM | Componentes identificados |
| Attestation | Declaração verificável sobre o artefato |
| Proveniência | Origem e processo de construção |

Esses mecanismos são complementares.

Uma imagem pode estar assinada, mas ter sido construída a partir de código comprometido. Uma imagem pode possuir SBOM, mas conter vulnerabilidades. Uma imagem pode ter proveniência, mas ter sido produzida por um pipeline mal configurado.

A confiança depende da combinação dos controles.

---

# 9. Integração de segurança ao GitLab CI

Agora vamos conectar os conceitos com uma ferramenta que você já utiliza no dia a dia: GitLab CI.

A ideia é construir um pipeline no qual uma imagem somente avance para publicação quando passar pelos controles definidos pela equipe.

## 9.1 Arquitetura proposta

```text
              GitLab Repository
                     |
                     v
                GitLab CI
                     |
          +----------+----------+
          |                     |
          v                     v
       Build                 Testes
          |                     |
          +----------+----------+
                     |
                     v
                Build Image
                     |
                     v
              Vulnerability Scan
                     |
                     v
                  SBOM
                     |
                     v
               Security Gate
                     |
              +------+------+
              |             |
            Falha          OK
              |             |
              v             v
           Bloqueia       Assinatura
                            |
                            v
                         Harbor
                            |
                            v
                       ArgoCD
                            |
                            v
                       Kubernetes
```

O pipeline pode implementar controles diferentes em cada etapa.

## 9.2 Exemplo de `.gitlab-ci.yml`

Vamos criar um exemplo didático com:

1. Build da imagem;
2. Scan de vulnerabilidades;
3. Geração de SBOM;
4. Publicação no Harbor.

Para simplificar, vamos utilizar BuildKit para construção da imagem e Trivy para análise.

```yaml
stages:
  - build
  - security
  - publish

variables:
  IMAGE_NAME: "$CI_REGISTRY_IMAGE"
  IMAGE_TAG: "$CI_COMMIT_SHA"
  IMAGE_REF: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"

build:
  stage: build

  image:
    name: moby/buildkit:rootless
    entrypoint: [""]

  script:
    - mkdir -p ~/.docker

    - |
      cat > ~/.docker/config.json <<EOF
      {
        "auths": {
          "$CI_REGISTRY": {
            "username": "$CI_REGISTRY_USER",
            "password": "$CI_REGISTRY_PASSWORD"
          }
        }
      }
      EOF

    - |
      buildctl-daemonless.sh build \
        --frontend dockerfile.v0 \
        --local context=. \
        --local dockerfile=. \
        --output type=image,name=$IMAGE_REF,push=true

  rules:
    - if: '$CI_COMMIT_BRANCH'

scan:
  stage: security

  image:
    name: aquasec/trivy:latest
    entrypoint: [""]

  needs:
    - build

  script:
    - trivy image
        --severity HIGH,CRITICAL
        --exit-code 1
        "$IMAGE_REF"

    - trivy image
        --format cyclonedx
        --output sbom.cdx.json
        "$IMAGE_REF"

  artifacts:
    when: always
    paths:
      - sbom.cdx.json
    expire_in: 30 days

  rules:
    - if: '$CI_COMMIT_BRANCH'

publish:
  stage: publish

  image: alpine:3.22

  needs:
    - scan

  script:
    - echo "A imagem passou pelo controle de segurança."
    - echo "Imagem: $IMAGE_REF"

  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

Esse exemplo demonstra o fluxo, mas há alguns detalhes importantes para uma implementação real:

- `CI_REGISTRY`, `CI_REGISTRY_USER` e `CI_REGISTRY_PASSWORD` são variáveis padrão do GitLab quando o registry integrado está habilitado. Para Harbor externo, configure credenciais específicas.
- O exemplo de build utiliza BuildKit rootless; o runner precisa oferecer as condições necessárias para sua execução.
- A etapa `publish` é apenas ilustrativa: a imagem já foi enviada ao registry pelo BuildKit. Em um pipeline real, essa etapa pode representar promoção, assinatura ou autorização para deploy.
- Não utilize `latest` em imagens de ferramentas de segurança em produção. Fixe versões e, preferencialmente, digests.
- Para análise de imagens privadas, configure autenticação do Trivy com credenciais protegidas.
- Para reprodutibilidade, considere fixar também a imagem base e as dependências por versões ou digests.

**Uma observação sobre o controle:** se o scan falhar, o job `scan` falha e o job `publish` não será executado, porque depende dele por meio de `needs`.

## 9.3 Como adicionar assinatura ao pipeline?

Depois da aprovação do scan, podemos adicionar uma etapa de assinatura com Cosign.

O ideal é que a assinatura seja realizada utilizando uma identidade controlada pelo pipeline ou uma chave armazenada em um gerenciador de segredos.

Exemplo conceitual:

```yaml
sign:
  stage: publish

  image:
    name: cgr.dev/chainguard/cosign:latest
    entrypoint: [""]

  needs:
    - scan

  script:
    - cosign sign
        --key "$COSIGN_PRIVATE_KEY"
        "$IMAGE_REF"
```

Aqui, `COSIGN_PRIVATE_KEY` deve ser uma variável protegida e mascarada do GitLab, com os controles de acesso apropriados.

O exemplo pressupõe que a chave esteja disponível no formato esperado pelo Cosign e que o pipeline tenha autorização para publicar a assinatura no registry.

Uma evolução mais robusta seria utilizar assinatura keyless ou um serviço de gerenciamento de chaves, evitando manter uma chave privada de longa duração como variável de CI.

---

# 10. Segurança do Harbor

Agora vamos tratar do registry.

O Harbor é um registry open source que oferece recursos importantes para segurança e governança de imagens.

Documentação oficial:

[Documentação do Harbor](https://goharbor.io/docs/?utm_source=chatgpt.com)

Ele oferece recursos como:

- Controle de acesso por projeto;
- Autenticação;
- Replicação;
- Vulnerability Scanning;
- Assinatura e verificação de artefatos, conforme configuração e versão;
- Políticas de retenção;
- Auditoria;
- Quotas;
- Imagens imutáveis;
- Integração com scanners.

## 10.1 Controle de acesso

Imagine que você tenha estes projetos:

```text
harbor.exemplo.local/
│
├── desenvolvimento/
├── homologacao/
└── producao/
```

Uma política de acesso poderia ser:

| Perfil | Desenvolvimento | Homologação | Produção |
|---|---|---|---|
| Desenvolvedor | Push/Pull | Pull | Sem acesso |
| CI/CD | Push/Pull | Push/Pull | Push/Pull controlado |
| ArgoCD | Pull | Pull | Pull |
| Administrador | Administrador | Administrador | Administrador |

O objetivo é impedir que qualquer desenvolvedor publique diretamente uma imagem em produção.

## 10.2 Immutability

Imagine que sua aplicação tenha uma imagem:

```text
harbor.exemplo.local/producao/api:1.0
```

Se essa tag puder ser sobrescrita, alguém poderá publicar outro conteúdo utilizando o mesmo identificador.

O Harbor permite configurar regras de imutabilidade de tags.

Uma política interessante é proteger tags de releases:

```text
v1.0.0
v1.1.0
v2.0.0
```

Assim, depois que uma versão for publicada, ela não poderá ser substituída por outro conteúdo por meio da mesma tag, conforme a regra configurada.

Ainda assim, prefira referenciar o digest no manifesto de deploy.

## 10.3 Vulnerability Scanning no Harbor

O Harbor pode integrar scanners de vulnerabilidades.

O fluxo fica assim:

```text
GitLab CI
    |
    v
Build Image
    |
    v
Harbor
    |
    v
Vulnerability Scanner
    |
    v
Relatório de vulnerabilidades
```

A vantagem é que o registry pode manter informações de segurança sobre os artefatos armazenados.

Isso permite que a equipe consulte o estado de segurança das imagens sem depender exclusivamente do pipeline que as construiu.

Dependendo da versão e da configuração, o Harbor pode realizar scans automáticos ou sob demanda e aplicar políticas relacionadas ao resultado.

**Atenção:** o fato de uma imagem estar armazenada no Harbor não significa que ela foi aprovada. O registry precisa estar configurado para aplicar as políticas desejadas.

## 10.4 Política de imagens confiáveis

Uma política corporativa pode estabelecer:

1. Imagens de produção somente podem vir do Harbor corporativo.
2. Imagens devem ser referenciadas por digest.
3. Imagens precisam possuir SBOM.
4. Imagens devem passar pelo scanner.
5. Imagens devem possuir assinatura válida.
6. Assinaturas devem pertencer a identidades autorizadas.
7. Imagens com vulnerabilidades acima do limite definido não podem ser implantadas.

Essas regras precisam ser aplicadas também no cluster. Não basta documentá-las ou validá-las somente no GitLab.

---

# 11. Relação entre Harbor, GitLab e ArgoCD

Agora vamos conectar tudo ao fluxo de GitOps.

Considere uma arquitetura semelhante à que você utiliza:

- GitLab para repositórios e CI;
- Harbor para armazenamento de imagens;
- ArgoCD para implantação;
- Kubernetes ou OpenShift para execução.

## 11.1 Fluxo completo

```text
                   Desenvolvedor
                        |
                        v
                  GitLab Repository
                        |
                        v
                    GitLab CI
                        |
             +----------+----------+
             |          |          |
             v          v          v
           Build       Trivy      Syft
             |          |          |
             |          v          |
             |       Security      |
             |        Gate         |
             |          |          |
             +----------+----------+
                        |
                        v
                     Cosign
                        |
                        v
                      Harbor
                        |
                        v
                Repositório GitOps
                        |
                        v
                     ArgoCD
                        |
                        v
                  Kubernetes/OCP
                        |
                        v
                 Container Runtime
```

Existem duas cadeias distintas que precisam estar protegidas:

**Cadeia de construção:** garante que o artefato foi construído, analisado e assinado por um processo confiável.

**Cadeia de implantação:** garante que o cluster implante o artefato autorizado, e não uma imagem diferente ou adulterada.

## 11.2 O ArgoCD verifica automaticamente a assinatura?

Não.

O ArgoCD é uma ferramenta de GitOps. Ele sincroniza os recursos declarados no Git com o estado desejado do cluster.

Ele não deve ser tratado como um verificador automático de assinaturas de imagens.

Por exemplo, um manifesto pode conter:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2

  selector:
    matchLabels:
      app: api

  template:
    metadata:
      labels:
        app: api

    spec:
      containers:
        - name: api
          image: harbor.exemplo.local/producao/api@sha256:SEU_DIGEST
```

O ArgoCD aplica esse Deployment.

Mas a validação de assinatura precisa ser implementada por outro mecanismo.

## 11.3 Como impedir imagens não assinadas no Kubernetes?

Podemos utilizar ferramentas de admission policy que verificam imagens antes de permitir a criação dos Pods.

O Kyverno, por exemplo, possui suporte a verificação de assinaturas.

O fluxo seria:

```text
ArgoCD
   |
   v
Kubernetes API Server
   |
   v
Kyverno Admission Controller
   |
   +---- Imagem assinada por identidade confiável?
   |
   +---- Sim: permite
   |
   +---- Não: rejeita
```

Exemplo didático de política Kyverno:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy

metadata:
  name: verify-image-signature

spec:
  validationFailureAction: Enforce

  rules:
    - name: verify-api-image

      match:
        any:
          - resources:
              kinds:
                - Pod

      verifyImages:
        - imageReferences:
            - "harbor.exemplo.local/producao/*"

          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      COLOQUE_A_CHAVE_PUBLICA_AQUI
                      -----END PUBLIC KEY-----
```

Esse exemplo apresenta a estrutura básica, não uma política pronta para produção. A sintaxe e os requisitos devem ser conferidos para a versão instalada do Kyverno.

Em uma implementação real, também precisamos considerar:

- Todas as imagens dos containers;
- `initContainers`;
- `ephemeralContainers`;
- Identidade do assinante;
- Verificação do digest;
- Configuração de acesso ao registry;
- Cache de verificações;
- Política de falha quando o registry estiver indisponível.

O ponto mais importante é que a verificação ocorre no admission do Kubernetes, antes da criação do Pod.

Isso cria uma segunda barreira de segurança, independente do GitLab CI.

---

# 12. Laboratório integrado — Image Security

Vamos consolidar os conceitos em um laboratório no Debian 13.

## 12.1 Preparação

Instale ou disponibilize:

```text
Docker
Skopeo
Umoci
Trivy
Syft
Grype
Cosign
jq
```

Verifique:

```bash
docker version
skopeo --version
umoci --version
trivy --version
syft version
grype version
cosign version
```

## 12.2 Baixar e inspecionar uma imagem

```bash
docker pull alpine:3.22

docker image inspect alpine:3.22

docker history alpine:3.22
```

Responda:

1. Qual é o digest da imagem?
2. Qual usuário está configurado?
3. Qual é o entrypoint?
4. Quantas camadas existem?
5. Qual é o tamanho da imagem?

## 12.3 Gerar SBOM

```bash
syft alpine:3.22 \
  -o cyclonedx-json \
  > alpine-sbom.json
```

Consulte os componentes:

```bash
jq '.components[] | {
  name,
  version,
  type
}' alpine-sbom.json
```

## 12.4 Analisar vulnerabilidades

Com Trivy:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  alpine:3.22
```

Com Grype:

```bash
grype alpine:3.22
```

Compare os resultados.

Tente explicar eventuais diferenças entre os scanners.

## 12.5 Analisar o SBOM

```bash
grype sbom:alpine-sbom.json
```

Observe que não precisamos necessariamente analisar novamente a imagem para realizar essa consulta: estamos utilizando o inventário gerado anteriormente.

## 12.6 Assinar uma imagem

Gere as chaves:

```bash
cosign generate-key-pair
```

Depois, escolha uma imagem em um registry sobre o qual você tenha permissão de publicação.

Descubra seu digest, assine e verifique:

```bash
cosign sign --key cosign.key "$IMAGE"

cosign verify --key cosign.pub "$IMAGE"
```

Observe o resultado da verificação.

Não compartilhe nem armazene a chave privada junto ao código-fonte.

## 12.7 Experimento de integridade

Faça o seguinte exercício conceitual:

1. Obtenha uma imagem identificada por digest.
2. Assine essa imagem.
3. Verifique a assinatura.
4. Publique outra imagem, com conteúdo diferente, usando a mesma tag.
5. Obtenha o novo digest.
6. Tente verificar a assinatura antiga contra o novo digest.

A assinatura não deverá validar o novo conteúdo.

Esse experimento demonstra por que tags não são suficientes para garantir a identidade de um artefato.

---

# 13. Boas práticas para ambientes corporativos

Para fechar a aula, vamos organizar os controles em uma matriz.

| Controle | Objetivo | Ferramenta |
|---|---|---|
| Scanner de vulnerabilidades | Identificar CVEs | Trivy / Grype |
| Inventário de componentes | Gerar SBOM | Syft |
| Assinatura | Identificar artefato e assinante | Cosign |
| Proveniência | Rastrear processo de build | SLSA / attestations |
| Armazenamento | Centralizar imagens | Harbor |
| Imutabilidade | Evitar substituição de tags protegidas | Harbor |
| CI Security Gate | Bloquear imagens reprovadas | GitLab CI |
| Admission Verification | Bloquear imagens não autorizadas | Kyverno |
| GitOps | Declarar e sincronizar implantação | ArgoCD |

## Uma arquitetura de referência

Minha recomendação para um ambiente Kubernetes/OpenShift corporativo seria:

1. **GitLab CI:** construir a imagem e executar testes.
2. **Trivy:** analisar vulnerabilidades.
3. **Syft:** gerar SBOM.
4. **GitLab CI:** bloquear a publicação conforme a política de risco.
5. **Cosign:** assinar o artefato aprovado.
6. **Harbor:** armazenar imagens, assinaturas e artefatos associados, com controle de acesso e imutabilidade.
7. **GitOps:** atualizar o manifesto com o digest da imagem aprovada.
8. **ArgoCD:** sincronizar o manifesto.
9. **Kyverno:** verificar assinatura e identidade autorizada durante o admission.
10. **Kubernetes/OpenShift:** executar o container com os controles de runtime estudados nas aulas anteriores.

O resultado é uma cadeia na qual cada etapa possui uma responsabilidade de segurança.

---

# 14. Exercícios de fixação

Responda às perguntas abaixo para consolidar a aula.

**1. Imagens OCI**

Qual é a diferença entre uma tag e um digest? Por que utilizar digest em produção?

**2. Vulnerabilidades**

Uma imagem apresenta uma vulnerabilidade CVSS 9.8, mas o pacote vulnerável não está acessível pela aplicação. Isso significa que podemos ignorá-la?

**3. Scanners**

Por que Trivy e Grype podem apresentar resultados diferentes ao analisar a mesma imagem?

**4. SBOM**

Qual é a diferença entre gerar um SBOM e executar um scanner de vulnerabilidades?

**5. Assinatura**

Uma imagem possui assinatura válida. Isso significa que ela é necessariamente segura?

**6. Proveniência**

Qual é a diferença entre assinatura e proveniência?

**7. GitLab CI**

Em que etapa do pipeline você colocaria o controle que impede a publicação de imagens com vulnerabilidades críticas?

**8. Harbor**

Por que a imutabilidade de tags é importante? Ela substitui o uso de digest?

**9. ArgoCD**

O ArgoCD verifica automaticamente se uma imagem está assinada?

**10. Kubernetes**

Por que é interessante verificar a assinatura da imagem também no admission do cluster, mesmo que o GitLab CI já faça essa validação?

---

# 15. Conclusão da aula

A principal mudança de mentalidade desta aula é entender que **a segurança de um container começa antes de ele ser iniciado**.

Não adianta configurar corretamente:

- Namespaces;
- Cgroups;
- Capabilities;
- Seccomp;
- AppArmor;
- SELinux;
- SecurityContext;
- Pod Security Standards;

se estamos executando uma imagem comprometida ou contendo componentes vulneráveis.

Da mesma forma, uma imagem sem vulnerabilidades conhecidas não é automaticamente confiável. Precisamos saber de onde veio, quem a construiu, quais componentes contém e se o artefato implantado corresponde ao que foi aprovado.

A combinação de scanners, SBOM, assinatura, proveniência, registry protegido e admission policies permite construir uma cadeia de confiança desde o código-fonte até o container em execução.

## Próxima aula — 2.11: Container Supply Chain Security — Hardening de Builds e Imagens

Na próxima aula, podemos aprofundar:

1. Dockerfile seguro e redução da superfície de ataque.
2. Multi-stage builds.
3. Imagens distroless e minimalistas.
4. Execução de builds sem privilégios.
5. BuildKit e segurança do processo de build.
6. Secrets durante o build, sem gravá-los nas camadas.
7. Pinning de imagens base por digest.
8. Atualização automatizada de dependências e imagens.
9. Políticas de aprovação de imagens no Harbor.
10. Integração completa GitLab CI + Harbor + Cosign + ArgoCD + Kyverno.
