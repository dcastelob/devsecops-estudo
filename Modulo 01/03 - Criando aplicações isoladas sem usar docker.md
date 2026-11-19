Sim! E essa é uma excelente experiência para aprofundar o estudo de segurança de containers.

Você pode executar uma aplicação diretamente no host, criando manualmente seus próprios Namespaces e Cgroups, **sem Docker, containerd ou qualquer outro runtime de containers**.

Na verdade, essa experiência permite entender o que acontece internamente quando um runtime cria um container.

O Linux fornece os mecanismos necessários por meio de chamadas de sistema e interfaces expostas em `/proc` e `/sys/fs/cgroup`.

Existem três ferramentas particularmente interessantes:

| Ferramenta | Função |
|---|---|
| `unshare` | Cria novos namespaces para um processo |
| `nsenter` | Entra nos namespaces de outro processo |
| `systemd-run` | Executa processos em unidades systemd com controles de recursos |

Também utilizaremos:

- `mount`: para configurar o filesystem.
- `ip`: para configurar namespaces de rede.
- `cgcreate` e `cgexec`: para administrar Cgroups, caso utilize ferramentas auxiliares do pacote `cgroup-tools`.
- `/sys/fs/cgroup`: interface nativa de administração do Cgroups v2.

Minha sugestão é começar com `unshare` e a interface nativa de Cgroups v2. Assim você não cria dependência de ferramentas externas.

# 1. Criando um ambiente isolado com unshare

Vamos começar criando um processo que terá seus próprios namespaces.

Execute:

```bash
sudo unshare \
  --pid \
  --fork \
  --mount \
  --uts \
  --ipc \
  --net \
  --mount-proc \
  bash
```

O que estamos fazendo?

| Parâmetro | Namespace criado |
|---|---|
| `--pid` | PID |
| `--fork` | Cria um processo filho no novo namespace PID |
| `--mount` | MNT |
| `--uts` | UTS |
| `--ipc` | IPC |
| `--net` | NET |
| `--mount-proc` | Monta um `/proc` adequado ao novo namespace |

Você agora está executando um shell com namespaces próprios.

Experimente:

```bash
hostname
```

Altere o hostname:

```bash
hostname laboratorio
```

Verifique:

```bash
hostname
```

O hostname do host não deverá ser alterado.

Agora:

```bash
ps aux
```

Você deverá observar somente os processos visíveis no novo namespace PID.

E:

```bash
ip addr
```

O namespace de rede será independente do host. Inicialmente, normalmente você verá apenas a interface loopback, que pode estar desativada.

Para sair:

```bash
exit
```

**Observe:** nesse primeiro experimento ainda não configuramos um filesystem independente nem um limite de recursos. Criamos apenas o isolamento de namespaces.

# 2. Criando um ambiente com User Namespace

Agora vamos acrescentar um mecanismo importante: o User Namespace.

Ele permite que o processo seja root dentro do seu ambiente isolado, mas tenha um UID sem privilégios administrativos no host.

Experimente:

```bash
unshare \
  --user \
  --map-root-user \
  --pid \
  --fork \
  --mount \
  --uts \
  --ipc \
  --net \
  --mount-proc \
  bash
```

Dentro do ambiente:

```bash
id
```

Resultado esperado:

```text
uid=0(root) gid=0(root)
```

Agora, em outro terminal do host, identifique o processo:

```bash
ps -eo pid,uid,gid,cmd | grep unshare
```

E examine o mapeamento:

```bash
cat /proc/SEU_PID/uid_map
cat /proc/SEU_PID/gid_map
```

O resultado deverá mostrar que o UID 0 do namespace está associado ao seu UID não privilegiado no host.

Isso demonstra um conceito fundamental:

**É possível ser root dentro de um namespace sem ser root no host.**

Esse é um dos fundamentos dos containers rootless.

---

# 3. Aplicando limites de Cgroups sem Docker

Agora vamos controlar os recursos da aplicação.

Considerando que seu Debian 13 utiliza Cgroups v2, podemos administrar diretamente os arquivos do kernel.

Verifique:

```bash
stat -fc %T /sys/fs/cgroup
```

Esperamos:

```text
cgroup2fs
```

## 3.1. Criando um Cgroup

Vamos criar um grupo chamado `laboratorio`:

```bash
sudo mkdir /sys/fs/cgroup/laboratorio
```

Configure os limites:

### Memória: 512 MB

```bash
echo 536870912 | sudo tee \
  /sys/fs/cgroup/laboratorio/memory.max
```

### CPU: 50% de uma CPU

```bash
echo "50000 100000" | sudo tee \
  /sys/fs/cgroup/laboratorio/cpu.max
```

### Processos: máximo de 100 tarefas

```bash
echo 100 | sudo tee \
  /sys/fs/cgroup/laboratorio/pids.max
```

Agora temos:

```text
/sys/fs/cgroup/laboratorio/
    |
    +-- memory.max
    +-- cpu.max
    +-- pids.max
    +-- cgroup.procs
    +-- memory.current
    +-- cpu.stat
    +-- pids.current
```

Atenção: a criação de grupos diretamente na raiz de `/sys/fs/cgroup` pode ser impedida pela organização da hierarquia do systemd e pelas regras de habilitação de controladores do Cgroups v2. Para um laboratório persistente, prefira uma hierarquia delegada pelo systemd. Vamos apresentar essa alternativa também.

## 3.2. Associando um processo ao Cgroup

Vamos executar uma aplicação:

```bash
sleep 3600 &
PID=$!
```

Associe o processo:

```bash
echo "$PID" | sudo tee \
  /sys/fs/cgroup/laboratorio/cgroup.procs
```

Pronto!

O processo agora está associado ao Cgroup `laboratorio`.

Verifique:

```bash
cat /sys/fs/cgroup/laboratorio/cgroup.procs
```

E consulte o consumo:

```bash
cat /sys/fs/cgroup/laboratorio/memory.current
cat /sys/fs/cgroup/laboratorio/cpu.stat
cat /sys/fs/cgroup/laboratorio/pids.current
```

Uma propriedade importante: se você iniciar processos filhos a partir desse processo, eles herdarão a associação ao Cgroup.

Portanto, podemos iniciar uma aplicação já dentro de um Cgroup e fazer com que seus processos filhos permaneçam sujeitos aos limites.

---

# 4. Combinando Namespaces e Cgroups

Agora vamos juntar os dois mecanismos.

Imagine que você queira executar uma aplicação com:

| Recurso | Configuração |
|---|---|
| PID Namespace | Isolado |
| Network Namespace | Isolado |
| Mount Namespace | Isolado |
| UTS Namespace | Isolado |
| IPC Namespace | Isolado |
| User Namespace | Isolado |
| Memória | 512 MB |
| CPU | 50% |
| Processos | 100 |

Uma maneira bastante interessante de fazer isso é utilizando o `systemd-run`.

O systemd já possui integração nativa com Cgroups v2 e consegue administrar os limites sem que você precise manipular diretamente os arquivos de controle.

Exemplo:

```bash
sudo systemd-run \
  --scope \
  --unit=laboratorio-container \
  -p MemoryMax=512M \
  -p CPUQuota=50% \
  -p TasksMax=100 \
  unshare \
    --pid \
    --fork \
    --mount \
    --uts \
    --ipc \
    --net \
    --mount-proc \
    bash
```

Aqui temos duas operações distintas:

**systemd-run:**
- Cria uma unidade de execução.
- Cria um Cgroup.
- Configura os limites de recursos.
- Associa os processos à unidade.

**unshare:**
- Cria os namespaces.
- Executa o shell dentro do ambiente isolado.

Você pode investigar a unidade com:

```bash
systemctl status laboratorio-container.scope
```

E consultar a árvore de Cgroups:

```bash
systemd-cgls
```

Essa abordagem é bastante interessante porque aproveita a infraestrutura de gerenciamento de recursos que já existe no Debian.

---

# 5. Administrando o ambiente

Você não precisa de Docker para administrar esses processos.

O próprio Linux e o systemd oferecem ferramentas para isso.

### Consultar os processos

```bash
ps -ef
```

```bash
systemd-cgls
```

### Consultar os recursos

```bash
systemctl show laboratorio-container.scope \
  -p MemoryCurrent \
  -p MemoryMax \
  -p CPUUsageNSec \
  -p TasksCurrent \
  -p TasksMax
```

### Consultar os namespaces

Identifique o PID:

```bash
PID=$(systemctl show \
  laboratorio-container.scope \
  -p MainPID --value)
```

Depois:

```bash
lsns -p "$PID"
```

### Encerrar o ambiente

Se estiver utilizando `systemd-run --scope`:

```bash
sudo systemctl stop laboratorio-container.scope
```

Se estiver utilizando um Cgroup criado manualmente, primeiro encerre os processos associados e depois remova o diretório:

```bash
sudo rmdir /sys/fs/cgroup/laboratorio
```

O diretório só poderá ser removido quando estiver vazio e não possuir processos associados.

---

# 6. Mas ainda falta um componente: o filesystem

Até aqui conseguimos isolar:

- Processos.
- Rede.
- Hostname.
- IPC.
- Usuários.
- Recursos computacionais.

Entretanto, ainda não criamos um filesystem independente.

Isso significa que nosso shell ainda pode enxergar o filesystem do host, dependendo dos mounts disponíveis.

Para construir um ambiente mais próximo de um container, precisamos trabalhar com:

- Mount Namespace.
- Root Filesystem independente.
- `chroot`.
- `pivot_root`.
- Bind mounts.
- OverlayFS.
- `/proc`.
- `/sys`.
- `/dev`.

Por exemplo, podemos criar um root filesystem Debian:

```text
/var/lib/labs/container01/
    |
    +-- bin/
    +-- etc/
    +-- lib/
    +-- proc/
    +-- sys/
    +-- dev/
    +-- usr/
    +-- var/
    +-- tmp/
```

E executar uma aplicação utilizando esse diretório como sua raiz.

O mecanismo mais interessante para isso é o `pivot_root`, que permite trocar a raiz do filesystem dentro de um Mount Namespace.

**Importante:** `chroot` e `pivot_root` não são, isoladamente, mecanismos de segurança. Precisamos combinar permissões, namespaces, capabilities, seccomp e controles de acesso.

---

# 7. Ferramentas que recomendo conhecer

Se seu objetivo é administrar containers sem Docker, existem algumas alternativas muito interessantes.

| Ferramenta | Característica |
|---|---|
| `unshare` | Manipulação direta de namespaces |
| `nsenter` | Acesso aos namespaces existentes |
| `systemd-run` | Execução com gerenciamento de Cgroups |
| `systemd-nspawn` | Execução de ambientes Linux isolados |
| `bubblewrap` | Construção de sandboxes utilizando namespaces |
| `chroot` | Alteração da raiz aparente do filesystem |
| `runc` | Implementação de runtime OCI |
| `crun` | Runtime OCI escrito em C |

Eu daria atenção especial a três delas:

**1. unshare:** para compreender os mecanismos internos do kernel.

**2. systemd-nspawn:** para administrar ambientes Linux isolados sem Docker.

**3. runc:** para compreender como um runtime de containers implementa a especificação OCI.

Documentação:
- [unshare — Linux man pages](https://man7.org/linux/man-pages/man1/unshare.1.html?utm_source=chatgpt.com)
- [systemd-nspawn](https://www.freedesktop.org/software/systemd/man/latest/systemd-nspawn.html?utm_source=chatgpt.com)
- [runc](https://github.com/opencontainers/runc?utm_source=chatgpt.com)
- [bubblewrap](https://github.com/containers/bubblewrap?utm_source=chatgpt.com)

---

# 8. Minha proposta para nosso laboratório

Eu sugiro que não avancemos imediatamente para Capabilities.

Podemos construir um pequeno runtime manual, progressivamente, utilizando somente recursos do Linux.

O projeto seria:

```text
HOST DEBIAN 13
    |
    +-- /var/lib/labs/container01
    |       |
    |       +-- Root Filesystem
    |       +-- /proc
    |       +-- /sys
    |       +-- /dev
    |
    +-- Namespaces
    |       |
    |       +-- PID
    |       +-- NET
    |       +-- MNT
    |       +-- UTS
    |       +-- IPC
    |       +-- USER
    |
    +-- Cgroups v2
    |       |
    |       +-- memory.max
    |       +-- cpu.max
    |       +-- pids.max
    |
    +-- Processo da aplicação
```

A sequência que proponho:

1. Criar um root filesystem Debian utilizando `debootstrap`.
2. Criar os namespaces utilizando `unshare`.
3. Configurar o filesystem utilizando `mount` e `pivot_root`.
4. Configurar os Cgroups v2.
5. Executar uma aplicação dentro do ambiente.
6. Investigar o isolamento utilizando `/proc`, `/sys` e `lsns`.
7. Introduzir restrições de segurança.

**O resultado será um container construído manualmente, sem Docker e sem Kubernetes.**

Isso será particularmente útil quando chegarmos ao estudo de Capabilities, Seccomp e AppArmor, porque conseguiremos observar diretamente o efeito de cada mecanismo sobre um processo real.

Minha recomendação é seguirmos por esse laboratório antes de encerrar o Módulo 1.
