# Módulo 2.2 — Anatomia da criação de um container

Nesta aula vamos investigar o que realmente acontece internamente quando executamos:

```bash
docker run -d --name web nginx
```

Nosso objetivo é entender como uma imagem armazenada no host se transforma em um processo Linux isolado.

Vamos relacionar cada etapa com os mecanismos estudados no Módulo 1 e, principalmente, entender como o runtime OCI representa essas configurações.

Ao final, você deverá conseguir olhar para um `config.json` e identificar:

- Qual filesystem será utilizado.
- Qual processo será iniciado.
- Quais namespaces serão criados.
- Quais limites de recursos serão aplicados.
- Quais capabilities estarão disponíveis.
- Qual perfil Seccomp será utilizado.
- Quais restrições de segurança serão aplicadas.

---

# 1. Visão geral do processo

Quando executamos:

```bash
docker run -d --name web nginx
```

O Docker realiza uma sequência de operações.

Simplificando:

```text
                 docker run
                     |
                     v
              Docker Engine
                     |
                     v
                 containerd
                     |
                     v
              Download da imagem
                     |
                     v
             Preparação do snapshot
                     |
                     v
               Root Filesystem
                     |
                     v
             Preparação do container
                     |
                     v
                OCI Runtime
                     |
                     v
          Configuração do ambiente
                     |
          +----------+----------+
          |          |          |
          v          v          v
      Namespaces   Cgroups   Security
          |          |          |
          +----------+----------+
                     |
                     v
              Execução do init
                     |
                     v
              Container ativo
```

Essa sequência é conceitual. Na implementação real, algumas operações acontecem em ordens diferentes ou parcialmente em paralelo. O importante é compreender as responsabilidades de cada componente.

Vamos detalhar.

---

# 2. Etapa 1 — Imagem

A primeira etapa é obter a imagem que será utilizada pelo container.

Por exemplo:

```bash
docker pull nginx:latest
```

O Docker consulta o registry e obtém os metadados e as camadas da imagem.

Uma imagem OCI é composta principalmente por:

1. Manifest.
2. Config.
3. Layers.

Visualmente:

```text
             nginx:latest
                  |
                  v
           +-------------+
           |  Manifest   |
           +-------------+
                  |
          +-------+-------+
          |               |
          v               v
      Config JSON      Layers
                          |
              +-----------+-----------+
              |           |           |
              v           v           v
           Layer 1     Layer 2     Layer 3
```

## 2.1 O que existe dentro das layers?

Imagine que uma imagem tenha sido construída assim:

```dockerfile
FROM debian:13

RUN apt-get update

RUN apt-get install -y nginx

COPY index.html /var/www/html/

CMD ["nginx", "-g", "daemon off;"]
```

A imagem resultante possui uma configuração e camadas que representam alterações no filesystem.

Conceitualmente:

```text
+----------------------------------+
| Layer 3                          |
|                                  |
| /var/www/html/index.html         |
+----------------------------------+
| Layer 2                          |
|                                  |
| nginx                            |
| bibliotecas                      |
| configurações                    |
+----------------------------------+
| Layer 1                          |
|                                  |
| Debian base                      |
|                                  |
+----------------------------------+
```

As layers são imutáveis. Quando um container precisa modificar arquivos, essas alterações são armazenadas em uma camada gravável associada ao container, dependendo do mecanismo de armazenamento utilizado.

## 2.2 Inspecionando as camadas

Execute:

```bash
docker pull nginx:latest
```

Depois:

```bash
docker image inspect nginx:latest
```

Para visualizar somente as layers:

```bash
docker image inspect nginx:latest \
  --format '{{json .RootFS.Layers}}' | jq
```

Você verá identificadores SHA256.

Também podemos visualizar o histórico:

```bash
docker history nginx:latest
```

**Uma distinção importante:** `docker history` apresenta o histórico de construção da imagem, não necessariamente uma representação fiel de cada operação que ocorreu durante a construção.

---

# 3. Etapa 2 — Snapshotter

Agora chegamos a um componente importante do containerd.

O **snapshotter** é responsável por gerenciar os snapshots de filesystem utilizados pelos containers.

Ele trabalha em conjunto com o armazenamento das imagens.

Para entender, precisamos conhecer uma característica importante das imagens de containers: o compartilhamento de camadas.

Imagine três containers utilizando a mesma imagem:

```text
             Imagem Debian
                  |
          +-------+-------+
          |       |       |
          v       v       v
       Container A     Container B     Container C
          |               |               |
          v               v               v
       RW Layer         RW Layer         RW Layer
```

As camadas originais da imagem podem ser compartilhadas.

Entretanto, cada container possui sua própria camada gravável, quando utiliza um snapshotter com esse modelo.

Isso evita duplicar desnecessariamente todo o conteúdo da imagem.

## 3.1 Copy-on-Write

O conceito utilizado nesse modelo é conhecido como **Copy-on-Write (CoW)**.

Imagine que a imagem possui:

```text
/etc/app.conf
```

O arquivo pertence a uma camada somente leitura.

Quando o container tenta modificar esse arquivo, o mecanismo de armazenamento pode copiar o arquivo para a camada gravável do container.

A alteração acontece nessa cópia.

```text
          Image Layer
          (Read Only)
               |
               v
          /etc/app.conf
               |
               |
       Container modifica
               |
               v
          Copy-on-Write
               |
               v
        Container RW Layer
          /etc/app.conf
```

Assim, a imagem original permanece intacta.

### Observação importante

O comportamento exato depende do snapshotter e do driver de armazenamento.

No Docker com `overlay2`, esse mecanismo utiliza OverlayFS.

No containerd, podem existir diferentes snapshotters, como:

- OverlayFS.
- Btrfs.
- ZFS.
- Devmapper.
- Stargz.
- Outros implementados por plugins.

Isso faz uma conexão interessante com nosso estudo anterior de Btrfs: ambos podem trabalhar com snapshots e compartilhamento de dados, embora sejam mecanismos diferentes.

## 3.2 Inspecionando o Storage Driver

Execute:

```bash
docker info | grep -A5 "Storage Driver"
```

Exemplo:

```text
Storage Driver: overlay2
Backing Filesystem: extfs
```

Em instalações com OverlayFS, você poderá encontrar estruturas como:

```text
/var/lib/docker/overlay2/
```

**Não altere manualmente o conteúdo desse diretório.** Ele é gerenciado pelo Docker.

---

# 4. Etapa 3 — Root Filesystem

Agora que temos as camadas da imagem e o snapshot correspondente ao container, precisamos disponibilizar o filesystem para o processo.

Esse filesystem será a raiz `/` vista pelo container.

Imagine:

```text
Host:

/
├── etc
├── usr
├── var
├── home
└── opt
```

Dentro do container, queremos que a aplicação enxergue:

```text
Container:

/
├── etc
├── usr
├── var
├── tmp
└── app
```

O container não precisa enxergar diretamente a árvore de diretórios do host.

O runtime configura o root filesystem para o processo, utilizando operações de montagem e isolamento apropriadas.

## 4.1 chroot não é suficiente

Talvez você já conheça o comando:

```bash
chroot /algum/diretorio
```

Ele altera a raiz de diretórios vista por um processo.

Entretanto, `chroot` sozinho não constitui uma sandbox segura.

Ele não cria automaticamente:

- PID namespace.
- Network namespace.
- Mount namespace.
- User namespace.
- Cgroups.
- Seccomp.
- Restrições de capabilities.

O runtime OCI combina esses mecanismos para produzir o isolamento esperado.

## 4.2 Root filesystem e pivot_root

O Linux possui a operação `pivot_root()`, utilizada para substituir a raiz do filesystem de um processo.

De maneira simplificada:

```text
Antes:

Host filesystem
      |
      v
      /

Depois:

Container filesystem
      |
      v
      /
```

O `pivot_root()` permite substituir a raiz e deslocar a raiz anterior para outro ponto de montagem.

Isso é especialmente útil na construção de ambientes isolados.

Não confunda `pivot_root()` com `chroot()`: são operações diferentes, com semânticas diferentes.

---

# 5. Etapa 4 — Mounts

O filesystem do container normalmente precisa de recursos adicionais além das camadas da imagem.

Por exemplo:

- `/proc`.
- `/sys`.
- `/dev`.
- `/dev/pts`.
- `/tmp`.
- Volumes persistentes.
- Arquivos de configuração montados.

Esses recursos são configurados por meio de mounts.

## 5.1 O que são mounts no container?

Imagine:

```text
Container
   |
   +-- /
   |
   +-- /proc
   |
   +-- /sys
   |
   +-- /dev
   |
   +-- /tmp
   |
   +-- /dados
```

Cada ponto pode representar um filesystem ou uma montagem específica.

Por exemplo, `/proc` normalmente será um procfs associado ao ambiente de processos do container.

Isso explica por que, dentro do container:

```bash
ps aux
```

pode mostrar apenas os processos visíveis no PID namespace.

## 5.2 Inspecionando mounts

Crie um container:

```bash
docker run -d --name mount-lab alpine sleep 3600
```

Entre nele:

```bash
docker exec -it mount-lab sh
```

Execute:

```bash
mount
```

Ou:

```bash
cat /proc/mounts
```

Observe:

- Filesystem.
- Origem.
- Ponto de montagem.
- Opções de montagem.

Saia do container:

```bash
exit
```

Agora consulte:

```bash
docker inspect mount-lab \
  --format '{{json .Mounts}}' | jq
```

Observe como o Docker apresenta os mounts associados ao container.

## 5.3 Montando um volume

Vamos criar um diretório no host:

```bash
mkdir -p /tmp/container-dados
echo "Arquivo criado no host" > /tmp/container-dados/teste.txt
```

Execute:

```bash
docker run -d \
  --name volume-lab \
  -v /tmp/container-dados:/dados \
  alpine sleep 3600
```

Agora:

```bash
docker exec volume-lab cat /dados/teste.txt
```

O container consegue acessar o arquivo porque o diretório do host foi disponibilizado por meio de um mount.

### Implicação de segurança

Montar diretórios do host dentro de containers pode comprometer o isolamento.

Por exemplo:

```bash
-v /:/host
```

Isso disponibiliza a árvore de diretórios do host dentro do container.

Se o container tiver permissões de escrita, o risco é ainda maior.

Por isso, mounts devem ser tratados como parte importante da superfície de ataque.

---

# 6. Etapa 5 — Namespaces

Agora chegamos aos mecanismos estudados na Aula 1.2.

O runtime precisa configurar os namespaces do container.

Os principais são:

| Namespace | Isolamento |
|---|---|
| PID | Identificadores e visibilidade dos processos |
| NET | Interfaces e pilha de rede |
| MNT | Pontos de montagem |
| IPC | Recursos IPC |
| UTS | Hostname e domínio NIS |
| USER | Identidades e mapeamento de usuários |
| CGROUP | Visão da hierarquia de cgroups |
| TIME | Determinadas visões de tempo |

Imagine:

```text
                 Host Linux
                     |
        +------------+------------+
        |                         |
        v                         v
    Processo A                Container B
                                  |
                         +--------+--------+
                         |        |        |
                         v        v        v
                       PID      NET      MNT
                      Namespace Namespace Namespace
```

O runtime pode criar esses namespaces por meio de chamadas como:

- `clone()`.
- `unshare()`.
- `setns()`.

O importante é entender que o container não possui um mecanismo mágico de isolamento. Ele utiliza funcionalidades do próprio kernel.

## 6.1 Investigando os namespaces do container

Vamos utilizar o container criado anteriormente:

```bash
PID=$(docker inspect \
  --format '{{.State.Pid}}' mount-lab)
```

Execute:

```bash
sudo lsns -p "$PID"
```

Compare com:

```bash
sudo lsns -p 1
```

Agora compare os identificadores:

```bash
sudo readlink /proc/1/ns/pid
sudo readlink /proc/"$PID"/ns/pid
```

Os identificadores deverão ser diferentes se o container estiver utilizando um PID namespace próprio.

---

# 7. Etapa 6 — Cgroups

Os namespaces controlam o isolamento da visão dos processos.

Os cgroups controlam o consumo de recursos.

Quando executamos:

```bash
docker run -d \
  --memory=512m \
  --cpus=1 \
  --pids-limit=100 \
  --name cgroup-lab \
  alpine sleep 3600
```

O Docker configura limites de recursos para o container.

Podemos verificar:

```bash
docker inspect cgroup-lab \
  --format '{{json .HostConfig}}' | jq
```

E consultar o cgroup do processo:

```bash
PID=$(docker inspect \
  --format '{{.State.Pid}}' cgroup-lab)

cat /proc/"$PID"/cgroup
```

Em sistemas com cgroups v2, podemos localizar os arquivos correspondentes em `/sys/fs/cgroup`.

Por exemplo:

```bash
cat /sys/fs/cgroup/cgroup.controllers
```

E, dependendo da localização do container:

```bash
cat /sys/fs/cgroup/caminho-do-container/memory.max
```

O resultado deve refletir o limite configurado.

### Relação com Kubernetes

É aqui que entram os conceitos de Requests e Limits estudados em Kubernetes.

O kubelet e o runtime configuram os cgroups dos containers de acordo com as especificações do Pod e do container.

---

# 8. Etapa 7 — Capabilities

Agora vamos estudar como o runtime aplica os privilégios do processo.

Por padrão, o Docker remove diversas capabilities do conjunto completo disponível ao root.

Isso significa que um processo executando como UID 0 dentro de um container não necessariamente possui todos os privilégios tradicionais de root.

Podemos consultar:

```bash
docker run --rm alpine sh -c \
  'grep Cap /proc/self/status'
```

Para visualizar as capabilities:

```bash
docker run --rm alpine sh -c \
  'capsh --decode=$(awk "/CapEff/ {print \$2}" /proc/self/status)'
```

O comando `capsh` precisa estar disponível na imagem. Caso contrário, podemos utilizar uma imagem que contenha `libcap2-bin`.

## 8.1 Removendo todas as capabilities

Execute:

```bash
docker run --rm \
  --cap-drop=ALL \
  alpine sh -c \
  'grep Cap /proc/self/status'
```

Agora o processo não terá capabilities efetivas concedidas pelo Docker.

Podemos adicionar somente uma:

```bash
docker run --rm \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  alpine sh -c \
  'grep Cap /proc/self/status'
```

Esse é um exemplo de princípio do menor privilégio.

---

# 9. Etapa 8 — Seccomp e LSM

O runtime também precisa configurar os mecanismos de segurança associados às chamadas de sistema e ao controle obrigatório de acesso.

## 9.1 Seccomp

O Docker aplica um perfil Seccomp padrão, salvo configuração em contrário.

Podemos verificar:

```bash
docker run --rm alpine \
  grep Seccomp /proc/self/status
```

Resultado esperado:

```text
Seccomp:        2
```

Isso indica Filter Mode.

Para desabilitar o filtro:

```bash
docker run --rm \
  --security-opt seccomp=unconfined \
  alpine sh
```

Essa configuração não é recomendada em produção.

## 9.2 AppArmor

Em hosts com AppArmor habilitado, o Docker pode associar um perfil ao container.

Verifique:

```bash
docker info | grep -i apparmor
```

E dentro do container:

```bash
docker run --rm alpine cat /proc/self/attr/current
```

O resultado depende do perfil efetivamente aplicado.

## 9.3 SELinux

Em hosts com SELinux, o runtime pode configurar os rótulos de segurança dos processos e arquivos.

Podemos investigar com:

```bash
ps -eZ
```

E:

```bash
ls -Z /var/lib/containers
```

No contexto de containers, o SELinux é particularmente importante para controlar o acesso entre workloads e recursos do host.

---

# 10. Etapa 9 — O config.json da especificação OCI

Agora vamos juntar os conceitos.

A especificação OCI Runtime define um formato de configuração utilizado para executar containers.

O arquivo central é:

```text
config.json
```

Ele descreve como o container deve ser executado.

Uma estrutura simplificada:

```json
{
  "ociVersion": "1.2.0",

  "process": {
    "terminal": false,
    "user": {
      "uid": 1000,
      "gid": 1000
    },
    "args": [
      "/bin/sh"
    ],
    "env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "cwd": "/"
  },

  "root": {
    "path": "rootfs",
    "readonly": true
  },

  "hostname": "container-lab",

  "mounts": [],

  "linux": {
    "namespaces": [
      {
        "type": "pid"
      },
      {
        "type": "network"
      },
      {
        "type": "mount"
      },
      {
        "type": "ipc"
      },
      {
        "type": "uts"
      }
    ],

    "resources": {
      "memory": {
        "limit": 536870912
      }
    },

    "capabilities": {
      "bounding": [],
      "effective": [],
      "permitted": [],
      "inheritable": [],
      "ambient": []
    },

    "seccomp": {
      "defaultAction": "SCMP_ACT_ERRNO",
      "syscalls": []
    }
  }
}
```

Esse exemplo é ilustrativo, não um arquivo pronto para executar. Um container real exige mounts, configurações de processo, permissões, namespaces e outros campos adequados ao ambiente.

Vamos analisar os principais elementos.

## 10.1 process

Define o processo que será iniciado.

```json
"process": {
  "user": {
    "uid": 1000,
    "gid": 1000
  },
  "args": [
    "/bin/sh"
  ],
  "cwd": "/"
}
```

Relaciona-se à configuração de usuário, argumentos e diretório de trabalho.

## 10.2 root

Define o root filesystem:

```json
"root": {
  "path": "rootfs",
  "readonly": true
}
```

O caminho `rootfs` é relativo ao bundle OCI.

## 10.3 mounts

Define os pontos de montagem:

```json
"mounts": [
  {
    "destination": "/proc",
    "type": "proc",
    "source": "proc",
    "options": [
      "nosuid",
      "noexec",
      "nodev"
    ]
  }
]
```

## 10.4 namespaces

Define os namespaces utilizados:

```json
"namespaces": [
  {
    "type": "pid"
  },
  {
    "type": "network"
  },
  {
    "type": "mount"
  }
]
```

É importante observar que o arquivo OCI permite também associar um container a um namespace já existente, por meio do campo `path`.

## 10.5 resources

Representa as configurações de recursos:

```json
"resources": {
  "memory": {
    "limit": 536870912
  },
  "cpu": {
    "quota": 100000,
    "period": 100000
  }
}
```

Nesse exemplo:

- Memória: 512 MiB.
- CPU: limite equivalente a uma CPU.

## 10.6 capabilities

Representa os conjuntos de capabilities do processo:

```json
"capabilities": {
  "bounding": [],
  "effective": [],
  "permitted": [],
  "inheritable": [],
  "ambient": []
}
```

Os conjuntos devem ser configurados de acordo com os privilégios necessários.

## 10.7 seccomp

Representa a política de filtragem das chamadas de sistema.

Exemplo simplificado:

```json
"seccomp": {
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": [
        "read",
        "write",
        "close",
        "exit",
        "exit_group"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

Observe a relação direta com nossa aula anterior.

---

# 11. Laboratório: criando um container diretamente com runc

Agora vamos fazer algo interessante: executar um container sem utilizar `docker run`.

Vamos utilizar diretamente o runtime OCI.

Esse laboratório permite visualizar a separação entre o gerenciador de containers e o runtime de baixo nível.

**Pré-requisitos:** Linux com `runc`, `curl`, `jq` e privilégios administrativos. Faça o laboratório em uma VM Debian 13, não em um servidor de produção.

## 11.1 Verificando o runc

```bash
runc --version
```

Caso não esteja instalado:

```bash
sudo apt install runc
```

## 11.2 Criando o bundle OCI

O bundle OCI contém:

```text
bundle/
├── config.json
└── rootfs/
```

Vamos criar o diretório:

```bash
mkdir -p ~/oci-lab/rootfs
cd ~/oci-lab
```

## 11.3 Obtendo um root filesystem

Para simplificar, vamos utilizar um rootfs mínimo do Alpine.

A imagem oficial do Alpine é distribuída como uma imagem OCI/Docker. Para obter um rootfs, podemos usar ferramentas como `skopeo` e `umoci`.

Instale:

```bash
sudo apt install skopeo umoci
```

Copie a imagem para um layout OCI:

```bash
skopeo copy \
  docker://docker.io/library/alpine:latest \
  oci:alpine-oci:latest
```

Extraia o rootfs:

```bash
umoci unpack \
  --image alpine-oci:latest \
  alpine-bundle
```

O diretório `alpine-bundle/rootfs` conterá o filesystem.

Copie ou mova para nosso laboratório:

```bash
rm -rf rootfs

mv alpine-bundle/rootfs ./rootfs
```

## 11.4 Gerando config.json

Execute:

```bash
runc spec
```

O comando gera um `config.json` de exemplo.

Abra:

```bash
vim config.json
```

Localize a seção `process` e ajuste:

```json
"args": [
  "sh"
]
```

Confira também o diretório de trabalho e as configurações de namespaces.

Para um primeiro teste, mantenha a configuração gerada pelo `runc spec`.

## 11.5 Executando o container

Crie um identificador:

```bash
sudo runc run oci-lab
```

Você deverá entrar no shell do container.

Experimente:

```bash
hostname
id
ps
mount
```

Depois saia:

```bash
exit
```

Verifique:

```bash
sudo runc list
```

O container deverá aparecer como encerrado ou não estar mais listado como execução ativa, dependendo do estado apresentado pela versão do runtime.

Para remover o estado:

```bash
sudo runc delete oci-lab
```

Esse laboratório mostra algo muito importante:

**O Docker não é necessário para executar um container OCI.**

O `runc` consegue executar um container diretamente, desde que receba um bundle válido e um root filesystem apropriado.

---

# 12. Consolidando a arquitetura

Vamos relacionar cada componente à sua responsabilidade.

| Etapa | Componente ou mecanismo | Responsabilidade |
|---|---|---|
| Obtenção da imagem | Docker/containerd | Buscar imagem |
| Armazenamento | Snapshotter | Gerenciar snapshots |
| Root filesystem | Runtime/storage | Disponibilizar a raiz |
| Mounts | Kernel/runtime | Montar recursos |
| Namespaces | Kernel/runtime | Isolar visão dos recursos |
| Cgroups | Kernel/runtime | Controlar recursos |
| Capabilities | Kernel/runtime | Restringir privilégios |
| Seccomp | Kernel/runtime | Filtrar System Calls |
| AppArmor/SELinux | Kernel/LSM | Aplicar políticas MAC |
| Execução | OCI Runtime | Iniciar processo |

Uma observação: a tabela apresenta responsabilidades conceituais. O containerd e o runtime trabalham em conjunto, e a implementação pode delegar algumas operações entre componentes auxiliares.

---

# 13. Exercícios de fixação

Agora gostaria que você realizasse algumas investigações no Debian 13.

### Exercício 1 — Inspeção de imagem

Execute:

```bash
docker image inspect nginx:latest
```

Identifique:

- Arquitetura.
- Sistema operacional.
- Entrypoint.
- Cmd.
- Layers.

### Exercício 2 — Snapshotter e armazenamento

Execute:

```bash
docker info
```

Identifique o Storage Driver e o filesystem utilizado.

Explique como o mecanismo de Copy-on-Write evita duplicar toda a imagem para cada container.

### Exercício 3 — Processo e namespaces

Crie um container:

```bash
docker run -d --name aula22 alpine sleep 3600
```

Identifique:

- PID no host.
- PID dentro do container.
- Namespaces utilizados.
- Cgroup associado.

### Exercício 4 — Segurança

Execute:

```bash
docker run -d \
  --name aula22-secure \
  --cap-drop=ALL \
  --security-opt no-new-privileges \
  --memory=256m \
  --pids-limit=50 \
  alpine sleep 3600
```

Investigue:

```bash
docker inspect aula22-secure
```

E:

```bash
PID=$(docker inspect \
  --format '{{.State.Pid}}' aula22-secure)

grep -E 'Cap|NoNewPrivs|Seccomp' /proc/"$PID"/status
```

Identifique as configurações aplicadas.

### Exercício 5 — OCI

Examine o `config.json` gerado pelo:

```bash
runc spec
```

Identifique onde estão representados:

- Processo.
- Root filesystem.
- Namespaces.
- Mounts.
- Cgroups.
- Capabilities.
- Seccomp.

---

## Próxima aula — 2.3: OCI Runtime na prática — runc versus crun

Na próxima aula vamos aprofundar a execução de containers diretamente pelo runtime.

Vamos comparar `runc` e `crun`, investigar os processos criados, entender o ciclo de vida OCI e observar como os runtimes aplicam as configurações de segurança no kernel.

Essa etapa será especialmente útil para compreender o funcionamento do containerd no Kubernetes e do CRI-O no OpenShift.
