# Módulo 2 — Docker e OCI Runtime Security
## Aula 2.4 — Filesystem de Containers: OverlayFS, Copy-on-Write e Snapshotters

Nesta aula vamos estudar uma das partes mais importantes da arquitetura de containers: **como uma imagem, composta por camadas aparentemente imutáveis, transforma-se em um filesystem utilizável por um processo**.

O objetivo é entender o que acontece por baixo de `docker run`, como o runtime monta o filesystem do container e quais consequências isso tem para desempenho, persistência e segurança.

Como você já está estudando Btrfs, vamos aproveitar esse conhecimento para comparar os modelos de armazenamento.

---

# 1. Primeiro: por que containers precisam de um filesystem especial?

Imagine que você execute:

```bash
docker run -d --name web nginx
```

O Docker precisa disponibilizar ao processo Nginx uma árvore de diretórios semelhante a esta:

```text
/
├── bin
├── etc
├── lib
├── usr
├── var
├── tmp
└── ...
```

Mas existe um detalhe: a imagem `nginx` não é um disco completo tradicional. Ela é composta por várias camadas (*layers*).

Uma representação simplificada:

```text
             Container nginx
                    |
                    v
          +-------------------+
          |   Writable Layer  |  <-- alterações do container
          +-------------------+
          |     Layer 4       |
          +-------------------+
          |     Layer 3       |
          +-------------------+
          |     Layer 2       |
          +-------------------+
          |     Layer 1       |
          +-------------------+
```

As camadas da imagem são reutilizáveis e compartilháveis entre containers. A camada gravável é específica de cada container.

Isso permite, por exemplo, que dez containers criados a partir da mesma imagem compartilhem as camadas inferiores sem precisar manter dez cópias independentes de todo o conteúdo.

**A tecnologia que permite apresentar essas camadas como um único filesystem é chamada de Union Filesystem.**

O OverlayFS é uma implementação desse conceito no kernel Linux.

---

# 2. Como funciona o OverlayFS?



O **OverlayFS** é um filesystem do kernel Linux que permite combinar dois ou mais diretórios em uma única árvore de diretórios.

Em vez de copiar todos os arquivos de diferentes diretórios para um novo local, ele apresenta uma visão unificada.

O OverlayFS trabalha principalmente com quatro componentes:

| Componente | Função |
|---|---|
| `lowerdir` | Camadas inferiores, normalmente somente leitura |
| `upperdir` | Camada superior, onde são armazenadas as alterações |
| `workdir` | Diretório auxiliar utilizado internamente pelo OverlayFS |
| `merged` | Visão unificada apresentada ao processo |

Vamos entender cada um.

## 2.1. lowerdir

O `lowerdir` representa uma ou mais camadas inferiores.

Em containers, normalmente corresponde às camadas da imagem.

Exemplo:

```text
lowerdir/
├── bin/
├── etc/
├── usr/
└── var/
```

O processo consegue acessar os arquivos como se estivessem em um filesystem convencional, mas não pode modificar diretamente essa camada.

O `lowerdir` é tratado como somente leitura pelo OverlayFS.

É possível combinar vários diretórios inferiores:

```text
lowerdir=/layer4:/layer3:/layer2:/layer1
```

Quando existem múltiplas camadas inferiores, a primeira da lista tem precedência sobre as seguintes.

Assim, se `layer4` e `layer2` possuírem um arquivo chamado `/etc/app.conf`, a versão de `layer4` será apresentada.

## 2.2. upperdir

O `upperdir` é a camada gravável.

É nela que o OverlayFS registra as modificações realizadas no filesystem unificado.

Imagine que a imagem contenha:

```text
/etc/nginx/nginx.conf
```

Quando o container modifica esse arquivo, o OverlayFS não altera diretamente o arquivo original da imagem.

Em vez disso, ele realiza um processo chamado **Copy-up**:

1. Identifica que o arquivo está em uma camada inferior.
2. Copia o arquivo para o `upperdir`.
3. Aplica a modificação na cópia.
4. Passa a apresentar a versão do `upperdir` ao processo.

O arquivo original permanece intacto.

Exemplo conceitual:

```text
lowerdir/
└── etc/
    └── nginx/
        └── nginx.conf       <-- original

upperdir/
└── etc/
    └── nginx/
        └── nginx.conf       <-- cópia modificada
```

O processo continua enxergando apenas:

```text
/etc/nginx/nginx.conf
```

Ele não precisa saber que o arquivo está fisicamente armazenado no `upperdir`.

## 2.3. workdir

O `workdir` é um diretório interno utilizado pelo OverlayFS para operações relacionadas à camada superior.

Ele precisa:

- Existir antes da montagem.
- Estar vazio no momento da montagem.
- Residir no mesmo filesystem que o `upperdir`.

O OverlayFS utiliza esse diretório para operações internas, incluindo procedimentos relacionados à consistência de alterações e renomeações.

**Importante:** o `workdir` não é uma camada visível para o processo do container. Ele é uma estrutura de implementação do OverlayFS.

## 2.4. merged

O `merged` é a visão unificada das camadas.

É o diretório que será utilizado como ponto de montagem do filesystem OverlayFS.

Imagine:

```text
lowerdir/
├── etc/
│   └── app.conf
├── usr/
│   └── bin/
│       └── app
└── var/
    └── log/


upperdir/
└── etc/
    └── app.conf


merged/
├── etc/
│   └── app.conf       <-- versão do upperdir
├── usr/
│   └── bin/
│       └── app
└── var/
    └── log/
```

O diretório `merged` apresenta uma visão combinada:

- Arquivos existentes somente no `lowerdir` aparecem normalmente.
- Arquivos existentes no `upperdir` têm precedência.
- Arquivos novos criados pelo processo aparecem no `upperdir`.
- Arquivos removidos são ocultados por mecanismos próprios do OverlayFS.

Essa visão unificada é o filesystem que o container utiliza.

---

# 3. Copy-on-Write (CoW)

O Copy-on-Write é um mecanismo no qual os dados originais são preservados e uma cópia é criada somente quando ocorre uma alteração.

Esse conceito não é exclusivo do OverlayFS. Ele também aparece em Btrfs, ZFS e outras tecnologias.

Vamos considerar uma imagem contendo:

```text
/etc/app.conf
```

O arquivo está na camada inferior.

Inicialmente:

```text
lowerdir/
└── etc/
    └── app.conf
```

O container executa:

```bash
echo "debug=true" >> /etc/app.conf
```

O OverlayFS precisa disponibilizar uma cópia gravável do arquivo.

Depois da operação:

```text
lowerdir/
└── etc/
    └── app.conf              <-- original preservado

upperdir/
└── etc/
    └── app.conf              <-- cópia modificada
```

O `merged` apresenta o arquivo modificado.

## 3.1. O que acontece quando um arquivo é criado?

Se o container executar:

```bash
touch /tmp/novo-arquivo
```

E esse arquivo ainda não existir nas camadas inferiores, não existe nada para copiar.

O arquivo será criado diretamente no `upperdir`.

## 3.2. O que acontece quando um arquivo é removido?

Aqui existe uma particularidade interessante.

Imagine que a imagem possua:

```text
lowerdir/
└── etc/
    └── app.conf
```

O container executa:

```bash
rm /etc/app.conf
```

O OverlayFS não pode simplesmente remover o arquivo da camada inferior, pois ela é compartilhada e somente leitura.

Então ele utiliza um mecanismo chamado **whiteout**.

O whiteout é uma marcação que indica que determinado arquivo de uma camada inferior deve ser considerado removido na visão unificada.

Conceitualmente:

```text
lowerdir/
└── etc/
    └── app.conf

upperdir/
└── etc/
    └── .wh.app.conf
```

A representação `.wh.app.conf` é uma forma didática de visualizar o conceito. A implementação real pode utilizar um dispositivo de caractere especial ou outro mecanismo compatível com as opções do OverlayFS.

O resultado no `merged`:

```text
merged/
└── etc/
    └── (app.conf não aparece)
```

Isso é importante para entender por que remover arquivos de uma imagem não necessariamente reduz o tamanho das camadas originais.

## 3.3. Uma consequência importante: apagar arquivos não libera espaço da imagem

Imagine:

1. Uma camada adiciona um arquivo de 500 MB.
2. Uma camada posterior remove esse arquivo.
3. A imagem final aparentemente não contém mais o arquivo.

Mesmo assim, a camada original continua armazenando os 500 MB.

A remoção apenas adiciona uma marcação de exclusão.

Por isso, em Dockerfiles, é interessante executar a instalação e a limpeza de arquivos temporários na mesma instrução `RUN`, quando apropriado:

```dockerfile
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

Se a instalação e a remoção forem feitas em instruções `RUN` separadas, a camada anterior continuará contendo os arquivos removidos pela camada posterior.

Esse é um dos motivos pelos quais a organização das camadas influencia o tamanho das imagens.

---

# 4. Laboratório: montando um OverlayFS manualmente

Agora vamos reproduzir o funcionamento do OverlayFS diretamente no Debian 13, sem Docker.

Esse laboratório é importante porque permite enxergar o mecanismo que normalmente fica escondido pelo container engine.

## 4.1. Preparar os diretórios

```bash
mkdir -p ~/overlay-lab/{lower,upper,work,merged}
```

Criar arquivos na camada inferior:

```bash
mkdir -p ~/overlay-lab/lower/etc
mkdir -p ~/overlay-lab/lower/usr/bin

echo "configuracao original" > ~/overlay-lab/lower/etc/app.conf
echo "Aplicacao de exemplo" > ~/overlay-lab/lower/usr/bin/app
```

Conferir:

```bash
tree ~/overlay-lab
```

Se não tiver o `tree`:

```bash
find ~/overlay-lab -type f
```

## 4.2. Montar o OverlayFS

Execute:

```bash
sudo mount -t overlay overlay \
  -o lowerdir="$HOME/overlay-lab/lower",\
upperdir="$HOME/overlay-lab/upper",\
workdir="$HOME/overlay-lab/work" \
  "$HOME/overlay-lab/merged"
```

Verifique a montagem:

```bash
findmnt ~/overlay-lab/merged
```

Ou:

```bash
mount | grep overlay
```

Você deverá encontrar uma montagem do tipo `overlay`.

## 4.3. Visualizar a união das camadas

```bash
ls -l ~/overlay-lab/merged/etc
ls -l ~/overlay-lab/merged/usr/bin
```

O conteúdo da camada inferior está disponível pela visão unificada:

```bash
cat ~/overlay-lab/merged/etc/app.conf
```

Compare:

```bash
ls ~/overlay-lab/lower
ls ~/overlay-lab/upper
ls ~/overlay-lab/merged
```

Observe que o arquivo aparece em `merged`, embora ainda esteja fisicamente armazenado em `lower`.

## 4.4. Modificar um arquivo existente

Execute:

```bash
echo "nova configuracao" >> ~/overlay-lab/merged/etc/app.conf
```

Agora confira:

```bash
cat ~/overlay-lab/merged/etc/app.conf
```

E compare os diretórios:

```bash
echo "LOWER:"
cat ~/overlay-lab/lower/etc/app.conf

echo "UPPER:"
cat ~/overlay-lab/upper/etc/app.conf
```

Você deverá observar:

- O arquivo original continua intacto em `lower`.
- Uma cópia modificada foi criada em `upper`.
- A visão `merged` apresenta a versão modificada.

Esse é o mecanismo Copy-up do OverlayFS.

## 4.5. Criar um arquivo novo

```bash
echo "arquivo criado pelo container" \
  > ~/overlay-lab/merged/etc/novo.conf
```

Confira:

```bash
find ~/overlay-lab/upper -type f
```

O novo arquivo deverá estar no `upperdir`.

## 4.6. Remover um arquivo da camada inferior

Agora execute:

```bash
rm ~/overlay-lab/merged/usr/bin/app
```

Verifique:

```bash
ls ~/overlay-lab/merged/usr/bin
ls ~/overlay-lab/lower/usr/bin
ls -la ~/overlay-lab/upper/usr/bin
```

O arquivo continua existindo na camada inferior, mas não aparece mais em `merged`.

Dependendo da configuração e da versão do OverlayFS, você poderá observar uma entrada de whiteout no `upperdir`.

Para verificar os atributos:

```bash
sudo stat ~/overlay-lab/upper/usr/bin/*
```

## 4.7. Desmontar o filesystem

Ao terminar:

```bash
sudo umount ~/overlay-lab/merged
```

Se quiser remover o laboratório:

```bash
rm -rf ~/overlay-lab
```

**Atenção:** não remova os diretórios enquanto o OverlayFS estiver montado.

---

# 5. Como o Docker utiliza OverlayFS?



O Docker utiliza um mecanismo de armazenamento baseado em camadas.

Em instalações Linux que utilizam o driver `overlay2`, o Docker usa o OverlayFS do kernel para combinar as camadas da imagem com a camada gravável do container.

Vamos observar isso na prática.

## 5.1. Identificar o storage driver

Execute:

```bash
docker info | grep -i -A 5 "Storage Driver"
```

Em uma instalação que utiliza OverlayFS, você deverá encontrar algo semelhante a:

```text
Storage Driver: overlay2
```

O nome `overlay2` é o nome do driver de armazenamento do Docker. Ele utiliza recursos do OverlayFS, incluindo suporte a múltiplas camadas inferiores.

O driver efetivamente disponível depende do kernel, da distribuição e da configuração do Docker.

## 5.2. Criar um container para investigação

```bash
docker run -d \
  --name overlay-lab \
  alpine \
  sleep 3600
```

Identifique o diretório de armazenamento:

```bash
docker inspect overlay-lab \
  --format '{{json .GraphDriver.Data}}' | jq
```

Em uma instalação com `overlay2`, a saída poderá conter campos como:

```json
{
  "LowerDir": "...",
  "MergedDir": "...",
  "UpperDir": "...",
  "WorkDir": "..."
}
```

Esses campos correspondem diretamente aos conceitos estudados:

| Docker | OverlayFS |
|---|---|
| `LowerDir` | `lowerdir` |
| `UpperDir` | `upperdir` |
| `WorkDir` | `workdir` |
| `MergedDir` | `merged` |

O caminho exato depende da configuração do Docker e do seu diretório de dados.

Para descobrir esse diretório:

```bash
docker info --format '{{.DockerRootDir}}'
```

## 5.3. Investigar a camada gravável

Entre no container:

```bash
docker exec -it overlay-lab sh
```

Crie um arquivo:

```bash
echo "alteracao realizada no container" > /tmp/teste.txt
exit
```

Agora consulte novamente:

```bash
docker inspect overlay-lab \
  --format '{{.GraphDriver.Data.UpperDir}}'
```

O arquivo deverá estar fisicamente armazenado na camada gravável:

```text
.../diff/tmp/teste.txt
```

Você também pode investigar o filesystem do container:

```bash
docker exec overlay-lab df -h /
```

E comparar com:

```bash
docker inspect overlay-lab \
  --format '{{.GraphDriver.Name}}'
```

## 5.4. O que acontece quando o container é removido?

Execute:

```bash
docker rm -f overlay-lab
```

A camada gravável daquele container deixa de fazer parte do armazenamento ativo do Docker e seus dados são descartados, conforme o gerenciamento do driver.

As camadas da imagem continuam disponíveis.

Essa é uma característica importante:

**A camada gravável do container não deve ser considerada armazenamento persistente.**

Para dados que precisam sobreviver à recriação do container, utilize volumes ou outros mecanismos de armazenamento persistente.

Por exemplo:

```bash
docker volume create dados-app
```

```bash
docker run -d \
  --name app \
  -v dados-app:/var/lib/app \
  alpine \
  sleep 3600
```

O conteúdo de `/var/lib/app` será armazenado no volume, não na camada gravável comum do container.

---

# 6. Como o containerd utiliza Snapshotters?

Até aqui, falamos sobre o Docker e seu driver `overlay2`.

Agora vamos observar o modelo utilizado pelo containerd.

O containerd é especialmente importante porque aparece na arquitetura de Kubernetes.

## 6.1. O que é um snapshotter?

Um snapshotter é um componente responsável por disponibilizar e gerenciar snapshots do filesystem das imagens.

Ele fornece uma abstração para operações como:

- Preparar uma camada gravável.
- Criar uma visão somente leitura.
- Compor camadas.
- Confirmar um snapshot.
- Remover snapshots.

O containerd não precisa implementar diretamente cada tecnologia de armazenamento. Ele utiliza implementações de snapshotters.

Exemplos:

| Snapshotter | Tecnologia |
|---|---|
| `overlayfs` | OverlayFS do kernel Linux |
| `btrfs` | Subvolumes e snapshots Btrfs |
| `zfs` | Datasets e snapshots ZFS |
| `devmapper` | Device Mapper |
| `native` | Cópias convencionais de diretórios |

A disponibilidade depende da compilação, configuração e versão do containerd.

## 6.2. Content Store e Snapshotter são coisas diferentes

Essa distinção é fundamental.

O containerd possui, entre outros componentes, um Content Store e um Snapshotter.

### Content Store

O Content Store armazena os conteúdos identificados por digest.

Por exemplo:

```text
sha256:abc123...
sha256:def456...
sha256:789abc...
```

Esses conteúdos podem incluir:

- Manifestos.
- Configurações de imagens.
- Camadas comprimidas.

O Content Store não é, por si só, a visão de diretórios que será apresentada ao processo do container.

### Snapshotter

O Snapshotter utiliza os conteúdos das camadas para construir a estrutura de filesystem.

Podemos representar assim:

```text
             Registry
                |
                v
          Image Manifest
                |
                v
           Content Store
                |
                v
            Snapshotter
                |
                v
       Filesystem do container
                |
                v
             OCI Runtime
                |
                v
             Processo
```

O Content Store armazena os conteúdos. O Snapshotter organiza esses conteúdos em snapshots utilizáveis.

## 6.3. Tipos de snapshots

O containerd trabalha com diferentes estados de snapshots.

De forma simplificada:

| Tipo | Característica |
|---|---|
| Committed | Snapshot confirmado, normalmente usado como camada somente leitura |
| Active | Snapshot gravável em uso ou em preparação |
| View | Visão somente leitura de um snapshot |

Um fluxo conceitual:

```text
Camadas da imagem
       |
       v
Snapshots committed
       |
       v
Snapshot active
       |
       v
Container
```

Quando um container precisa de uma camada gravável, o snapshotter prepara um snapshot ativo a partir das camadas inferiores.

No caso do snapshotter OverlayFS, isso é implementado usando os recursos do OverlayFS.

## 6.4. Investigando o snapshotter no containerd

Em um ambiente com containerd instalado:

```bash
containerd --version
```

Para consultar a configuração:

```bash
sudo containerd config dump
```

Procure a configuração de snapshotter:

```bash
sudo containerd config dump | grep -i snapshotter
```

Se estiver utilizando o `ctr`:

```bash
sudo ctr plugins ls | grep snapshot
```

Uma saída poderá apresentar algo semelhante a:

```text
io.containerd.snapshotter.v1    overlayfs    linux/amd64    ok
```

Para listar snapshots:

```bash
sudo ctr snapshots ls
```

O namespace utilizado pelo `ctr` pode influenciar os resultados. Para containers gerenciados pelo Kubernetes, por exemplo, normalmente será necessário consultar o namespace `k8s.io`:

```bash
sudo ctr -n k8s.io snapshots ls
```

**Atenção:** não altere ou remova snapshots manualmente enquanto estiverem sendo utilizados por containers. O containerd gerencia dependências e ciclo de vida desses objetos.

---

# 7. OverlayFS versus Btrfs versus ZFS

Essa comparação é particularmente interessante porque as três tecnologias podem ser utilizadas para implementar armazenamento baseado em snapshots e Copy-on-Write, mas trabalham de maneiras diferentes.



## 7.1. OverlayFS

OverlayFS é um filesystem de união (*union filesystem*).

Seu objetivo principal é combinar diretórios de diferentes camadas em uma única visão.

Características:

- Utiliza diretórios inferiores e superiores.
- Possui mecanismo de Copy-up.
- Utiliza whiteouts para representar remoções.
- Não exige que o armazenamento inferior seja Btrfs ou ZFS.
- É bastante utilizado em containers.

Uma representação:

```text
lowerdir  ─────┐
lowerdir  ─────┤
lowerdir  ─────┼──> OverlayFS ───> merged
upperdir  ─────┤
workdir   ─────┘
```

O OverlayFS não é, por si só, um gerenciador de volumes com snapshots no mesmo sentido do Btrfs ou ZFS.

## 7.2. Btrfs

O Btrfs é um filesystem que possui suporte nativo a Copy-on-Write, subvolumes e snapshots.

Você já conhece alguns desses conceitos do nosso curso Btrfs 101.

Imagine:

```text
Btrfs filesystem
│
├── @
├── @home
├── @containers
│   ├── snapshot-001
│   ├── snapshot-002
│   └── snapshot-003
└── @snapshots
```

No Btrfs, um snapshot é criado a partir de um subvolume:

```bash
btrfs subvolume snapshot \
  /mnt/btrfs/@containers \
  /mnt/btrfs/@containers-snapshot
```

O snapshot inicialmente compartilha os extents existentes com a origem. Quando ocorre uma alteração, novos extents são alocados.

Isso é diferente do mecanismo de Copy-up do OverlayFS.

### Uma diferença importante

No OverlayFS:

- O arquivo de uma camada inferior é copiado para a camada superior quando precisa ser modificado.

No Btrfs:

- Os dados são compartilhados por referência.
- A alteração gera novos extents.
- Os dados originais continuam disponíveis enquanto houver referências para eles.

Portanto, ambos utilizam conceitos relacionados a CoW, mas implementam o mecanismo de maneiras diferentes.

O containerd possui um snapshotter Btrfs que utiliza as funcionalidades nativas do filesystem.

## 7.3. ZFS

O ZFS também oferece Copy-on-Write, snapshots e clones.

Sua arquitetura utiliza datasets e snapshots.

Exemplo conceitual:

```text
pool/containers
│
├── container-001
├── container-002
└── container-003
```

Um snapshot pode ser criado com:

```bash
zfs snapshot pool/containers@baseline
```

Um clone pode ser criado a partir dele:

```bash
zfs clone \
  pool/containers@baseline \
  pool/containers/container-001
```

O clone compartilha inicialmente os dados com o snapshot de origem.

As modificações posteriores são armazenadas separadamente.

O containerd também possui snapshotter ZFS.

## 7.4. Comparação geral

| Característica | OverlayFS | Btrfs | ZFS |
|---|---|---|---|
| Tipo | Union filesystem | Filesystem CoW | Filesystem CoW |
| Camadas de diretórios | Sim | Não como recurso principal | Não como recurso principal |
| Copy-up | Sim | Não | Não |
| Snapshots nativos | Não no sentido de snapshots de subvolumes | Sim | Sim |
| Subvolumes/datasets | Não | Sim | Sim |
| Whiteouts | Sim | Não como mecanismo principal | Não como mecanismo principal |
| Compressão nativa | Não | Sim | Sim |
| Checksums de dados | Não como recurso geral do OverlayFS | Sim | Sim |
| Uso em containers | Muito comum | Suportado | Suportado |

### Qual é melhor para containers?

Não existe uma resposta universal.

**OverlayFS:**
- Simples de integrar.
- Amplamente utilizado.
- Não exige um filesystem especializado.
- É uma opção comum em distribuições Linux.

**Btrfs:**
- Interessante quando você deseja aproveitar snapshots nativos.
- Pode facilitar operações de armazenamento.
- Possui funcionalidades que vão além do armazenamento de camadas de containers.

**ZFS:**
- Oferece recursos avançados de integridade e gerenciamento de armazenamento.
- Pode ser interessante em infraestruturas que já utilizam ZFS.
- Exige considerar a integração e a operação do módulo ZFS no sistema.

Uma observação: utilizar Btrfs ou ZFS como armazenamento subjacente não significa necessariamente que o OverlayFS esteja sendo utilizado. São abordagens diferentes de snapshotter.

---

# 8. Implicações de segurança: mounts e filesystem

Agora chegamos à parte mais importante para o nosso curso de segurança de containers.

O filesystem não é apenas uma questão de desempenho e armazenamento. A maneira como ele é montado pode comprometer o isolamento do container.

## 8.1. O perigo dos bind mounts

Considere:

```bash
docker run -it \
  -v /etc:/host-etc \
  alpine sh
```

Dentro do container:

```bash
ls /host-etc
```

O processo consegue visualizar o diretório `/etc` do host.

Isso acontece porque o Docker disponibilizou um diretório do host dentro do container.

O isolamento de namespaces não significa que o processo esteja impedido de acessar arquivos do host que foram explicitamente montados.

Imagine agora:

```bash
docker run -it \
  -v /:/host \
  alpine sh
```

Você disponibilizou praticamente todo o filesystem do host ao container.

Dependendo das permissões e dos controles adicionais, isso pode permitir modificações extremamente perigosas.

**Regra de segurança:** nunca monte diretórios sensíveis do host dentro de containers sem uma justificativa operacional clara.

## 8.2. Mount somente leitura

Se o container precisa apenas consultar os arquivos:

```bash
docker run -it \
  --mount type=bind,src=/etc,dst=/host-etc,readonly \
  alpine sh
```

Ou:

```bash
docker run -it \
  -v /etc:/host-etc:ro \
  alpine sh
```

Isso reduz a possibilidade de o processo modificar os arquivos disponibilizados.

Mas atenção: somente leitura não significa isolamento completo.

Um container ainda pode obter informações sensíveis dos arquivos montados, e a configuração de submontagens também precisa ser considerada.

## 8.3. Root filesystem somente leitura

Outra medida importante é impedir alterações na camada gravável convencional do container.

No Docker:

```bash
docker run -d \
  --name readonly-lab \
  --read-only \
  alpine \
  sleep 3600
```

Agora tente:

```bash
docker exec readonly-lab sh -c 'touch /teste.txt'
```

A operação deverá falhar porque o filesystem raiz está montado como somente leitura.

Entretanto, algumas aplicações precisam gravar em diretórios temporários.

Você pode disponibilizar um filesystem temporário:

```bash
docker run -d \
  --name readonly-lab \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=64m \
  alpine \
  sleep 3600
```

Teste:

```bash
docker exec readonly-lab sh -c 'touch /tmp/teste.txt'
```

Agora a operação deverá funcionar.

Esse modelo é especialmente útil para aplicações que não precisam alterar seu próprio filesystem durante a execução.

## 8.4. Opções de segurança de montagem

Algumas opções importantes:

| Opção | Finalidade |
|---|---|
| `ro` | Somente leitura |
| `rw` | Leitura e escrita |
| `noexec` | Impede execução direta de arquivos naquele mount |
| `nosuid` | Ignora efeitos de bits SUID e SGID |
| `nodev` | Impede que dispositivos especiais sejam utilizados por aquele mount |

Exemplo:

```bash
docker run -d \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=64m \
  alpine sleep 3600
```

Essas opções são camadas adicionais de proteção.

Não substituem namespaces, capabilities, Seccomp ou mecanismos MAC.

Uma ressalva: `noexec` não impede todas as formas de executar código. Por exemplo, um interpretador pode ler um script como dado e executá-lo. É uma restrição sobre a execução direta pelo kernel, não uma garantia de que nenhum código será executado a partir daquele diretório.

## 8.5. Mount propagation

Outro assunto importante é o *mount propagation*.

No Linux, uma montagem pode propagar eventos de montagem e desmontagem entre diferentes namespaces de montagem.

Os modos mais relevantes são:

| Modo | Comportamento |
|---|---|
| `private` | Não propaga eventos de montagem |
| `shared` | Propaga eventos entre mounts compartilhados |
| `slave` | Recebe eventos de um mount compartilhado, mas não os propaga de volta |
| `rprivate` | Modo privado recursivo |
| `rshared` | Modo compartilhado recursivo |
| `rslave` | Modo slave recursivo |

Isso é relevante quando um container recebe um bind mount que contém outros pontos de montagem.

Uma configuração inadequada pode fazer com que eventos de montagem atravessem fronteiras que deveriam permanecer isoladas.

Em ambientes Kubernetes, esse assunto também aparece na configuração de volumes e no funcionamento do kubelet.

## 8.6. A camada gravável não é armazenamento confiável

Imagine que uma aplicação escreva:

```bash
echo "dados importantes" > /var/lib/app/database.txt
```

Se `/var/lib/app` estiver apenas na camada gravável do container, esses dados podem desaparecer quando o container for removido.

Além disso, não é adequado usar a camada gravável como substituto de um mecanismo de backup.

Para dados persistentes, prefira volumes, PVCs ou soluções de armazenamento apropriadas.

---

# 9. Relação com Kubernetes e OpenShift

Agora vamos conectar tudo à arquitetura que você utiliza no dia a dia.

## 9.1. Caminho de uma imagem até o container

Em um Kubernetes que utiliza containerd:

```text
              Image Registry
                    |
                    v
                  kubelet
                    |
                    v
                 CRI API
                    |
                    v
                containerd
                    |
                    v
              Content Store
                    |
                    v
                Snapshotter
                    |
                    v
              OCI Runtime
                    |
                    v
                  runc
                    |
                    v
                  Kernel
                    |
                    v
               Container
```

O snapshotter prepara o filesystem. O runtime OCI utiliza a estrutura preparada para criar e iniciar o processo.

Em OpenShift, uma arquitetura comum é:

```text
                 kubelet
                    |
                    v
                   CRI
                    |
                    v
                  CRI-O
                    |
                    v
               Containers
                    |
                    v
                  crun
                    |
                    v
                  Kernel
```

A configuração exata depende da versão e do ambiente, mas CRI-O e containerd são implementações diferentes da camada de gerenciamento de containers.

## 9.2. Kubernetes: filesystem somente leitura

No Kubernetes, você pode configurar o filesystem raiz de um container como somente leitura:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-example
spec:
  containers:
    - name: app
      image: nginx:latest
      securityContext:
        readOnlyRootFilesystem: true
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir:
        medium: Memory
        sizeLimit: 64Mi
```

Observe a combinação:

- `readOnlyRootFilesystem`: protege o filesystem raiz.
- `emptyDir`: fornece armazenamento temporário.
- `medium: Memory`: utiliza memória como armazenamento temporário.

Isso é uma prática importante de hardening.

No OpenShift, você também deve considerar as restrições de UID, SCC, permissões dos volumes e compatibilidade da imagem com execução não privilegiada.

---

# 10. Laboratório de segurança: testar as proteções

Vamos juntar os conceitos estudados.

Crie um container com:

- Filesystem raiz somente leitura.
- `/tmp` gravável.
- `/tmp` com `noexec`, `nosuid` e `nodev`.
- Limite de memória.
- Limite de processos.
- Sem capabilities adicionais.

Execute:

```bash
docker run -d \
  --name secure-fs-lab \
  --read-only \
  --memory=256m \
  --pids-limit=100 \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=32m \
  alpine \
  sleep 3600
```

Entre no container:

```bash
docker exec -it secure-fs-lab sh
```

### Teste 1 — Escrever no filesystem raiz

```bash
touch /teste.txt
```

Resultado esperado: falha.

### Teste 2 — Escrever no `/tmp`

```bash
touch /tmp/teste.txt
```

Resultado esperado: sucesso.

### Teste 3 — Verificar a montagem

```bash
mount | grep -E ' /tmp | / '
```

### Teste 4 — Verificar as capabilities

```bash
grep '^Cap' /proc/self/status
```

### Teste 5 — Verificar Seccomp

```bash
grep '^Seccomp' /proc/self/status
```

### Teste 6 — Verificar os limites de processos

```bash
cat /sys/fs/cgroup/pids.max
```

O valor esperado é:

```text
100
```

O caminho pode variar conforme a organização dos cgroups e o runtime.

Saia do container e remova-o:

```bash
exit
docker rm -f secure-fs-lab
```

---

# 11. O que você deve guardar desta aula

Vamos consolidar os conceitos:

| Conceito | O que você precisa lembrar |
|---|---|
| OverlayFS | Combina diretórios em uma única visão |
| `lowerdir` | Camadas inferiores, normalmente somente leitura |
| `upperdir` | Armazena alterações |
| `workdir` | Diretório auxiliar do OverlayFS |
| `merged` | Visão unificada apresentada ao processo |
| Copy-up | Copia um arquivo da camada inferior para a superior quando precisa modificá-lo |
| Whiteout | Representa a remoção de um arquivo de uma camada inferior |
| Snapshotter | Gerencia snapshots utilizados para compor filesystems de containers |
| Content Store | Armazena conteúdos identificados por digest |
| Btrfs | Utiliza CoW, subvolumes e snapshots nativos |
| ZFS | Utiliza CoW, datasets, snapshots e clones |

## 12. Exercícios de fixação

Sugiro tentar responder antes de consultar o material.

**1. OverlayFS**

Se um container modificar um arquivo existente no `lowerdir`, o que acontece com o arquivo original?

**2. Copy-on-Write**

Qual é a diferença entre o Copy-up do OverlayFS e o Copy-on-Write do Btrfs?

**3. Whiteout**

Por que o OverlayFS precisa de whiteouts quando um container remove arquivos da imagem?

**4. Docker**

Qual é a diferença entre a camada gravável de um container e um volume Docker?

**5. containerd**

Qual é a diferença entre Content Store e Snapshotter?

**6. Segurança**

Por que utilizar `--read-only` não impede que uma aplicação grave dados em um volume montado dentro do container?

**7. Kubernetes**

Qual é a finalidade de `readOnlyRootFilesystem: true` no `securityContext`?

**8. Arquitetura**

No Kubernetes com containerd, quem prepara o filesystem e quem efetivamente inicia o processo do container?

---

## Próxima aula — 2.5: Volumes, Bind Mounts e Isolamento de Filesystem

Na próxima aula vamos aprofundar a relação entre o filesystem do container e o filesystem do host.

Estudaremos:

1. Diferenças entre volumes Docker, bind mounts e tmpfs.
2. Como funcionam os mounts no Linux.
3. `mount namespaces` e isolamento de filesystem.
4. `pivot_root` e construção do root filesystem.
5. Mount propagation: `private`, `shared` e `slave`.
6. Riscos de expor diretórios do host.
7. Volumes no Kubernetes: `emptyDir`, `hostPath`, PVC e CSI.
8. Técnicas de hardening para evitar exposição indevida do filesystem do host.

Essa próxima etapa será especialmente importante para compreender os riscos de segurança de workloads no Kubernetes e no OpenShift.
