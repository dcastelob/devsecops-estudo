# Módulo 1 — Linux Security
# Aula 1.4 — Linux Capabilities

Até agora estudamos dois mecanismos fundamentais do Linux:

- **Namespaces:** isolam a visão que um processo possui dos recursos do sistema.
- **Cgroups:** controlam quanto de determinado recurso um processo pode consumir.

Agora vamos estudar um terceiro mecanismo: **Linux Capabilities**, responsável por dividir os privilégios tradicionalmente associados ao usuário `root`.

A ideia central desta aula é entender por que um processo pode executar como UID 0 e, mesmo assim, não ter autorização para realizar determinadas operações administrativas.

---

# 1. Como surgiu o modelo de Capabilities?

Historicamente, o Linux utilizava um modelo relativamente simples de privilégios:

```text
UID 0 (root)
    |
    +-- Pode realizar operações administrativas
    |
UID diferente de 0
    |
    +-- Não possui privilégios administrativos
```

O usuário root possuía praticamente todos os privilégios administrativos do sistema.

Isso criava um problema de segurança.

Imagine uma aplicação Web executada como root. Se um atacante explorasse uma vulnerabilidade nessa aplicação e conseguisse executar comandos arbitrários, também obteria os privilégios administrativos do sistema.

O problema era o excesso de privilégios.

O Linux precisava de uma maneira de conceder somente os privilégios necessários para cada aplicação.

Foi nesse contexto que surgiu o modelo de **Linux Capabilities**.

A partir dele, os privilégios administrativos foram divididos em capacidades independentes.

Em vez de conceder todos os privilégios de root, podemos conceder apenas alguns.

Por exemplo:

| Capability | Permissão |
|---|---|
| `CAP_NET_ADMIN` | Administrar configurações de rede |
| `CAP_NET_BIND_SERVICE` | Abrir portas privilegiadas (< 1024) |
| `CAP_SYS_ADMIN` | Diversas operações administrativas |
| `CAP_SYS_PTRACE` | Inspecionar e rastrear processos |
| `CAP_CHOWN` | Alterar proprietário de arquivos |
| `CAP_DAC_OVERRIDE` | Ignorar verificações tradicionais de permissões de arquivos |
| `CAP_SETUID` | Alterar UID de processos |
| `CAP_SETGID` | Alterar GID de processos |
| `CAP_KILL` | Enviar sinais ignorando verificações normais de propriedade |

A lista completa pode ser consultada em:

```bash
man 7 capabilities
```

Também podemos consultar os nomes disponíveis no sistema:

```bash
cat /usr/include/linux/capability.h
```

## 1.1. A principal mudança

Antes:

```text
ROOT
 |
 +-- Todos os privilégios administrativos
```

Depois:

```text
ROOT
 |
 +-- CAP_CHOWN
 +-- CAP_DAC_OVERRIDE
 +-- CAP_NET_ADMIN
 +-- CAP_SYS_ADMIN
 +-- CAP_SYS_PTRACE
 +-- CAP_SETUID
 +-- CAP_SETGID
 +-- ...
```

Isso permite construir aplicações com privilégios mínimos.

---

# 2. UID 0 não significa necessariamente privilégios totais

Essa é uma das partes mais importantes da aula.

No Linux, precisamos separar dois conceitos:

1. Identidade do processo.
2. Privilégios disponíveis para o processo.

A identidade é representada principalmente por UID e GID.

Os privilégios são controlados, entre outros mecanismos, pelas Capabilities.

Portanto:

```text
UID = 0
```

não significa necessariamente:

```text
Todas as Capabilities disponíveis
```

Um processo pode ser root, mas possuir somente:

```text
CAP_CHOWN
CAP_DAC_OVERRIDE
CAP_NET_BIND_SERVICE
```

Nesse cenário, ele não terá automaticamente permissão para realizar operações que exigem `CAP_SYS_ADMIN`, por exemplo.

## 2.1. Um exemplo prático

Vamos executar um container Alpine:

```bash
docker run --rm -it alpine sh
```

Dentro dele:

```bash
id
```

Resultado:

```text
uid=0(root) gid=0(root)
```

Agora consulte as Capabilities:

```bash
grep Cap /proc/self/status
```

Você verá algo semelhante a:

```text
CapInh: 0000000000000000
CapPrm: 00000000a80425fb
CapEff: 00000000a80425fb
CapBnd: 00000000a80425fb
CapAmb: 0000000000000000
```

Os valores dependem da configuração do runtime e da versão do kernel.

O importante é perceber que, mesmo sendo root, o processo possui um conjunto limitado de Capabilities.

Isso é uma das proteções implementadas pelos runtimes de containers.

---

# 3. Os cinco conjuntos de Capabilities

O Linux trabalha com cinco conjuntos principais de Capabilities por thread.

São eles:

| Conjunto | Significado |
|---|---|
| Effective | Capabilities efetivamente utilizadas nas verificações de permissão |
| Permitted | Capabilities que o processo pode tornar efetivas |
| Inheritable | Capabilities que podem participar da herança durante execuções |
| Bounding | Limite superior das Capabilities que podem ser adquiridas por mecanismos de herança |
| Ambient | Capabilities preservadas em determinadas execuções de programas não privilegiados |

Vamos entender cada um.

## 3.1. Effective — CapEff

Representa as Capabilities que o processo pode utilizar naquele momento para passar pelas verificações de autorização do kernel.

Exemplo:

```text
CapEff: 0000000000000000
```

Significa que o processo não possui nenhuma Capability efetiva.

Mesmo que tenha outras Capabilities no conjunto Permitted, elas não estão necessariamente disponíveis para uso naquele instante.

## 3.2. Permitted — CapPrm

Representa o conjunto de Capabilities que o processo pode tornar efetivas.

Imagine:

```text
CapPrm: CAP_NET_ADMIN
CapEff: vazio
```

O processo possui `CAP_NET_ADMIN` em seu conjunto permitido, mas ela não está efetiva.

Ele pode ativá-la por meio das interfaces apropriadas do kernel, respeitando as regras de segurança.

## 3.3. Inheritable — CapInh

Controla quais Capabilities podem participar da herança quando um processo executa outro programa por meio de `execve()`.

A herança de Capabilities envolve regras adicionais, inclusive os conjuntos de Capabilities do arquivo executado.

É especialmente importante compreender esse conjunto quando estudamos binários com file capabilities.

## 3.4. Bounding — CapBnd

É o conjunto que estabelece um limite para as Capabilities que podem ser adquiridas por determinados mecanismos de herança.

Uma aplicação não consegue simplesmente adquirir uma Capability que foi retirada de seu bounding set.

Exemplo:

```bash
capsh --drop=cap_sys_admin -- -c 'capsh --print'
```

Esse comando remove `CAP_SYS_ADMIN` do bounding set do processo e executa um shell.

A remoção é irreversível para o processo e seus descendentes, salvo situações especiais relacionadas a privilégios e namespaces.

## 3.5. Ambient — CapAmb

Foi introduzido no Linux 4.3.

Permite preservar determinadas Capabilities durante a execução de programas não privilegiados.

É útil em cenários nos quais queremos executar um programa sem utilizar binários com SUID ou file capabilities.

Exemplo conceitual:

```text
Processo pai
    |
    +-- Ambient: CAP_NET_BIND_SERVICE
    |
    +-- execve()
            |
            v
       Processo filho
            |
            +-- CAP_NET_BIND_SERVICE disponível
```

A herança depende de regras específicas, incluindo a presença da Capability nos conjuntos Permitted e Inheritable.

---

# 4. Como visualizar as Capabilities de um processo

O kernel disponibiliza essas informações em:

```text
/proc/<PID>/status
```

Por exemplo:

```bash
grep Cap /proc/self/status
```

Ou:

```bash
PID=$(pgrep -n firefox)

grep Cap /proc/"$PID"/status
```

O resultado apresenta valores hexadecimais.

Para interpretá-los de forma legível, podemos utilizar o pacote `libcap2-bin`.

No Debian:

```bash
sudo apt install libcap2-bin
```

Agora:

```bash
capsh --decode=00000000a80425fb
```

Exemplo de resultado:

```text
0x00000000a80425fb=
cap_chown,
cap_dac_override,
cap_fowner,
...
```

Também podemos consultar as Capabilities do próprio shell:

```bash
capsh --print
```

Esse comando é muito útil para diagnosticar ambientes de containers.

---

# 5. Capabilities perigosas em containers

Agora chegamos a uma parte especialmente importante para quem trabalha com Kubernetes e OpenShift.

Nem todas as Capabilities representam o mesmo nível de risco.

Algumas merecem atenção especial.

## 5.1. CAP_SYS_ADMIN

É frequentemente considerada uma das Capabilities mais perigosas.

Ela permite diversas operações administrativas, incluindo determinadas operações de montagem e gerenciamento de recursos do kernel.

Exemplo:

```bash
mount -t tmpfs tmpfs /mnt
```

Uma operação de montagem pode exigir `CAP_SYS_ADMIN` no User Namespace que governa o filesystem envolvido.

O alcance dessa Capability depende dos namespaces e dos recursos envolvidos.

Por isso, não devemos interpretá-la como uma autorização universal para administrar o host, mas sua concessão amplia significativamente a superfície de ataque.

**Regra prática:** evite conceder `CAP_SYS_ADMIN` a containers comuns.

## 5.2. CAP_NET_ADMIN

Permite administrar diversos aspectos da rede.

Por exemplo:

- Configurar interfaces.
- Alterar rotas.
- Configurar regras de firewall.
- Modificar determinados parâmetros de rede.

Imagine um container que recebeu:

```text
CAP_NET_ADMIN
```

Ele pode manipular sua configuração de rede dentro do namespace de rede em que opera.

Se estiver compartilhando o namespace de rede do host, o risco é muito maior.

## 5.3. CAP_SYS_PTRACE

Permite operações de rastreamento e inspeção de processos, sujeitas a outras verificações do kernel.

Pode ser utilizada por ferramentas como:

```bash
strace
gdb
```

Se concedida indevidamente, pode facilitar a inspeção de outros processos, inclusive a exposição de dados sensíveis presentes em sua memória.

## 5.4. CAP_DAC_OVERRIDE

Permite ignorar verificações tradicionais de permissões de acesso a arquivos.

Imagine:

```bash
chmod 000 segredo.txt
```

Um processo sem privilégios não conseguirá ler o arquivo normalmente.

Entretanto, um processo com `CAP_DAC_OVERRIDE` pode ignorar determinadas verificações de permissões discricionárias.

Isso não elimina todas as verificações de segurança: ACLs, LSMs e outras restrições ainda podem se aplicar.

## 5.5. CAP_NET_RAW

Permite utilizar sockets RAW e PACKET, sujeitos às restrições aplicáveis.

É utilizada por ferramentas como:

```bash
ping
tcpdump
```

Em containers, conceder essa Capability sem necessidade aumenta a superfície de ataque.

---

# 6. Laboratório: remover Capabilities de um container

Vamos realizar um experimento.

Primeiro, execute um container Alpine normalmente:

```bash
docker run --rm -it alpine sh
```

Dentro dele:

```bash
id
grep Cap /proc/self/status
```

Agora saia:

```bash
exit
```

Execute novamente, removendo todas as Capabilities:

```bash
docker run --rm -it \
  --cap-drop=ALL \
  alpine sh
```

Dentro:

```bash
id
grep Cap /proc/self/status
```

Observe:

- O processo continua executando como root.
- O UID continua sendo zero.
- As Capabilities efetivas foram removidas.

Isso demonstra na prática que identidade e privilégios são conceitos diferentes.

## 6.1. Testando CAP_NET_RAW

Primeiro, sem conceder a Capability:

```bash
docker run --rm -it \
  --cap-drop=ALL \
  alpine sh
```

Dentro do container:

```bash
ping -c 1 8.8.8.8
```

Dependendo da implementação do `ping`, da versão do Alpine e das permissões do binário, o resultado pode variar. Algumas implementações utilizam sockets RAW, enquanto outras podem utilizar sockets ICMP datagram.

Agora execute:

```bash
docker run --rm -it \
  --cap-drop=ALL \
  --cap-add=NET_RAW \
  alpine sh
```

Teste novamente.

A comparação ajuda a compreender o papel da Capability, mas não esqueça que outras permissões também participam da decisão do kernel.

---

# 7. Como remover Capabilities com Docker

O Docker disponibiliza duas opções importantes:

```bash
--cap-drop
--cap-add
```

A recomendação é partir do princípio de menor privilégio.

Por exemplo:

```bash
docker run -d \
  --name nginx-lab \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  nginx
```

Nesse exemplo, removemos todas as Capabilities e concedemos apenas `NET_BIND_SERVICE`.

Entretanto, essa configuração pode não ser suficiente para todas as imagens. O Nginx, dependendo de como está configurado, pode precisar de outras permissões para iniciar, alterar arquivos, criar diretórios ou gerenciar processos.

Por isso, a abordagem correta é identificar as necessidades reais da aplicação.

Uma configuração mais restritiva também pode ser construída com:

```bash
docker run -d \
  --name nginx-lab \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  nginx
```

Aqui, nenhuma Capability adicional é concedida e o processo não pode adquirir novos privilégios por mecanismos como SUID ou file capabilities.

---

# 8. Linux Capabilities no Kubernetes

No Kubernetes, as Capabilities são configuradas por meio do `securityContext`.

Exemplo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: capabilities-lab
spec:
  containers:
    - name: app
      image: alpine
      command: ["sleep", "3600"]

      securityContext:
        runAsNonRoot: false
        runAsUser: 0

        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
```

Observe:

```yaml
capabilities:
  drop:
    - ALL
  add:
    - NET_BIND_SERVICE
```

O Kubernetes solicita ao runtime que remova todas as Capabilities e conceda somente `NET_BIND_SERVICE`.

O processo continua executando com UID 0, mas possui um conjunto reduzido de privilégios.

Uma configuração ainda mais restritiva seria:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false

  capabilities:
    drop:
      - ALL

  seccompProfile:
    type: RuntimeDefault
```

Aqui combinamos:

- Usuário não privilegiado.
- Remoção de todas as Capabilities.
- Bloqueio de escalada de privilégios.
- Perfil seccomp padrão do runtime.

Esse é um excelente ponto de partida para aplicações que não necessitam de privilégios especiais.

---

# 9. Relação com OpenShift SCC

No OpenShift, precisamos considerar também as Security Context Constraints (SCC).

As SCC controlam quais configurações de segurança os Pods podem utilizar.

Entre outras coisas, elas podem controlar:

- UID utilizado pelo container.
- Capabilities permitidas.
- Uso de containers privilegiados.
- Host namespaces.
- Volumes permitidos.
- Escalada de privilégios.

Por exemplo, uma SCC pode restringir as Capabilities permitidas:

```yaml
allowedCapabilities:
  - NET_BIND_SERVICE

requiredDropCapabilities:
  - ALL
```

Nesse cenário, o Pod poderá solicitar `NET_BIND_SERVICE`, mas deverá remover todas as demais.

Atenção: as SCC são aplicadas pelo OpenShift e precisam permitir a configuração solicitada pelo Pod. O `securityContext` não pode simplesmente ignorar as restrições da SCC.

Uma característica importante do OpenShift é o uso frequente de UIDs aleatórios para executar containers. Isso reduz a dependência de UID 0 dentro das aplicações.

Você pode inspecionar as SCC disponíveis:

```bash
oc get scc
```

E consultar uma SCC específica:

```bash
oc describe scc restricted-v2
```

---

# 10. Exercício prático: descobrir as Capabilities do Firefox

Vamos conectar a aula ao nosso laboratório anterior com Bubblewrap e Firefox.

Inicie o Firefox isolado e obtenha seu PID:

```bash
pgrep -a firefox
```

Escolha um PID:

```bash
PID=<PID_DO_FIREFOX>
```

Consulte:

```bash
grep Cap /proc/"$PID"/status
```

Decodifique:

```bash
capsh --decode="$(awk '/CapEff/ {print $2}' /proc/"$PID"/status)"
```

Faça o mesmo para:

```bash
CapPrm
CapBnd
CapInh
CapAmb
```

Isso permitirá observar as permissões efetivas do processo.

**Importante:** se o Firefox estiver executando em User Namespace, as Capabilities devem ser interpretadas considerando o namespace de usuário no qual são válidas. Uma Capability dentro de um User Namespace não equivale automaticamente ao mesmo privilégio no namespace inicial do host.

---

# 11. Resumo da aula

| Mecanismo | Responsabilidade |
|---|---|
| UID/GID | Identidade do processo |
| Capabilities | Privilégios administrativos granulares |
| Namespaces | Isolamento da visão dos recursos |
| Cgroups | Controle de consumo de recursos |
| Seccomp | Restrição de chamadas de sistema |
| AppArmor/SELinux | Controle de acesso obrigatório |

Uma configuração segura de container normalmente combina esses mecanismos.

Por exemplo:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true

  capabilities:
    drop:
      - ALL

  seccompProfile:
    type: RuntimeDefault
```

Essa configuração é um excelente ponto de partida, desde que a aplicação seja compatível.

---

## 12. Exercícios de fixação

**Exercício 1 — Conceitual**

Se um processo está executando como:

```text
UID=0
CapEff=0000000000000000
```

Ele possui privilégios administrativos?

Explique a diferença entre identidade e privilégios.

**Exercício 2 — Prático**

Execute:

```bash
docker run --rm -it --cap-drop=ALL alpine sh
```

Verifique:

```bash
id
capsh --print
```

Explique por que o processo continua sendo root.

**Exercício 3 — Análise de segurança**

Imagine que você encontrou um Pod com:

```yaml
securityContext:
  privileged: true
```

Explique por que essa configuração é perigosa e como você começaria a substituí-la por uma configuração de menor privilégio.

**Exercício 4 — Kubernetes**

Crie um Pod que:

- Execute como UID 10001.
- Remova todas as Capabilities.
- Não permita escalada de privilégios.
- Utilize `RuntimeDefault` para seccomp.

---

### Referência para aprofundamento

A documentação oficial do kernel é a melhor referência para esta aula:

- [Linux man-pages — capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html?utm_source=chatgpt.com)
- [Linux Kernel — Capabilities](https://docs.kernel.org/security/credentials.html?utm_source=chatgpt.com)
- [Kubernetes — Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/?utm_source=chatgpt.com)

**Minha recomendação:** antes de avançarmos para Seccomp, faça os exercícios 1 e 2. Quero que fique muito clara a diferença entre ser root e possuir privilégios administrativos, pois essa distinção será fundamental quando estudarmos containers privilegiados e os mecanismos de segurança do Kubernetes.
