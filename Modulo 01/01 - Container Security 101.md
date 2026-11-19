# Módulo 1 — Segurança no Linux e isolamento de processos
## Container Security 101

Vamos iniciar estudando os mecanismos que tornam os containers possíveis do ponto de vista de segurança.

A ideia é não apenas aprender a executar comandos, mas entender o que acontece internamente no kernel Linux quando um container é criado.

Este módulo será dividido em aulas progressivas, com explicação teórica, demonstrações, laboratórios e exercícios.

## 1.1. Primeiro conceito: containers não são máquinas virtuais

Antes de estudar namespaces, capabilities e cgroups, precisamos estabelecer uma diferença fundamental.

Uma máquina virtual possui, normalmente, seu próprio kernel virtualizado. Um container, por outro lado, compartilha o kernel do sistema operacional hospedeiro (*host*).

Observe:

### Arquitetura de máquinas virtuais

```text
+--------------------------------------+
|              HOST                    |
|                                      |
|  +-------------+  +-------------+    |
|  |     VM 1    |  |     VM 2    |    |
|  |             |  |             |    |
|  | Aplicações  |  | Aplicações  |    |
|  | Bibliotecas |  | Bibliotecas |    |
|  | Kernel 1    |  | Kernel 2    |    |
|  +-------------+  +-------------+    |
|          Hypervisor                  |
|                                      |
|          Hardware                    |
+--------------------------------------+
```

Cada máquina virtual possui seu próprio kernel.

### Arquitetura de containers

```text
+--------------------------------------+
|                 HOST                 |
|                                      |
|  +-------------+  +-------------+    |
|  | Container 1 |  | Container 2 |    |
|  |             |  |             |    |
|  | Aplicações  |  | Aplicações  |    |
|  | Bibliotecas |  | Bibliotecas |    |
|  +-------------+  +-------------+    |
|                                      |
|          Linux Kernel                |
|                                      |
|          Hardware                    |
+--------------------------------------+
```

Os containers compartilham o kernel do host.

Isso significa que o isolamento não é fornecido por um hypervisor, mas por mecanismos implementados pelo próprio kernel Linux.

Os principais mecanismos são:

| Mecanismo | Responsabilidade |
|---|---|
| Namespaces | Isolar a visão que um processo possui dos recursos do sistema |
| Cgroups | Controlar e limitar o consumo de recursos |
| Capabilities | Dividir os privilégios tradicionalmente associados ao root |
| Seccomp | Filtrar chamadas de sistema |
| AppArmor | Restringir o acesso a recursos por meio de perfis |
| SELinux | Aplicar políticas de controle de acesso obrigatório |
| LSM | Infraestrutura do kernel para implementar controles de segurança |

Uma observação importante: **nenhum desses mecanismos, isoladamente, transforma um container em uma fronteira de segurança perfeita.** A proteção resulta da combinação deles.

---

# 1.2. Namespaces — O isolamento da visão do sistema

Vamos começar pelo mecanismo mais importante para entender containers.

O Linux Namespaces permite que um processo tenha uma visão isolada de determinados recursos do sistema operacional.

Imagine dois processos executados no mesmo servidor:

- Processo A: executado diretamente no host.
- Processo B: executado dentro de um container.

Mesmo compartilhando o mesmo kernel, eles podem ter visões diferentes de:

- Processos em execução.
- Interfaces de rede.
- Endereços IP.
- Pontos de montagem.
- Identidade de usuários.
- Hostname.
- Filas de comunicação IPC.

Isso é possível porque o kernel associa os processos a determinados namespaces.

## 1.2.1. Os tipos de namespaces

O Linux disponibiliza os seguintes namespaces principais:

| Namespace | Isolamento |
|---|---|
| PID | Identificadores e árvore de processos |
| NET | Interfaces, endereços IP, rotas e portas |
| MNT | Pontos de montagem e visão do filesystem |
| IPC | Comunicação entre processos |
| UTS | Hostname e NIS domain name |
| USER | Identificadores de usuários e grupos |
| CGROUP | Visão dos caminhos de cgroups |
| TIME | Relógios monotônicos e de boot |

O namespace TIME também é relevante, embora não apareça na lista inicial do nosso roadmap.

Vamos entender cada um.

### A. PID Namespace

O PID Namespace is responsável por isolar os identificadores de processos.

Considere o seguinte cenário:

No host:

```text
PID 1    systemd
PID 850  dockerd
PID 1200 containerd
PID 1500 processo do container
```

Dentro do container, o mesmo processo pode aparecer como:

```text
PID 1    sleep 3600
```

Observe algo importante: o processo possui dois identificadores diferentes, dependendo do namespace a partir do qual estamos observando.

O kernel mantém a associação entre esses identificadores.

**Implicações de segurança:**

- Um container não visualiza automaticamente todos os processos do host.
- Um processo dentro do container não consegue enviar sinais livremente para processos de outros namespaces.
- O isolamento de PID reduz a exposição dos processos do host.

Entretanto, isso não significa que o isolamento seja absoluto. Configurações como `--pid=host` permitem compartilhar o namespace PID do host.

### B. NET Namespace

O NET Namespace isola os recursos relacionados à rede.

Cada namespace de rede pode possuir:

- Interfaces de rede próprias.
- Endereços IP próprios.
- Tabelas de roteamento próprias.
- Regras de firewall próprias.
- Portas TCP e UDP próprias.
- Tabelas de vizinhos próprias.

Por exemplo:

```text
HOST
 |
 +-- Interface eth0
 |      IP: 192.168.10.10
 |
 +-- Container A
 |      eth0: 172.17.0.2
 |
 +-- Container B
        eth0: 172.17.0.3
```

Os containers possuem suas próprias interfaces de rede, mesmo compartilhando o kernel.

**Implicações de segurança:**

Um processo dentro de um container não deve conseguir manipular diretamente as interfaces e rotas do host.

Entretanto, existem exceções importantes, como:

```bash
docker run --network host alpine
```

Nesse caso, o container compartilha o namespace de rede do host.

Isso aumenta a superfície de ataque e exige cuidados adicionais.

### C. MNT Namespace

O Mount Namespace isola a visão dos pontos de montagem do sistema de arquivos.

Imagine que o host possua:

```text
/
├── etc
├── home
├── var
├── proc
├── sys
└── dev
```

Dentro de um container, podemos apresentar uma estrutura diferente:

```text
/
├── etc
├── bin
├── lib
├── proc
├── sys
└── dev
```

Isso não significa necessariamente que os arquivos foram copiados para outro disco.

O runtime prepara uma visão específica do filesystem utilizando mecanismos como:

- Mount namespaces.
- Bind mounts.
- OverlayFS.
- tmpfs.
- Mount propagation.

**Implicações de segurança:**

Um container não deve ter acesso automático ao filesystem completo do host.

Mas atenção a este exemplo:

```bash
docker run -v /:/host alpine
```

Aqui estamos disponibilizando o filesystem do host dentro do container.

Se o container possuir permissões suficientes, isso pode permitir alterações extremamente perigosas no host.

Portanto, compartilhar diretórios do host é uma decisão de segurança, não apenas uma questão operacional.

### D. USER Namespace

Esse é um dos namespaces mais interessantes do ponto de vista de segurança.

O User Namespace permite mapear identificadores de usuários e grupos entre o container e o host.

Imagine:

Dentro do container:

```bash
id
```

Resultado:

```text
uid=0(root) gid=0(root)
```

No host, esse mesmo processo pode estar associado a um UID sem privilégios administrativos.

Por exemplo:

```text
Container UID: 0
Host UID:      100000
```

Isso é possível por meio do mapeamento de IDs.

**Por que isso é importante?**

Sem User Namespace, um processo executado como root dentro de um container convencional pode ter UID 0 também no host.

Embora namespaces, capabilities e outros mecanismos ainda limitem seu acesso, isso aumenta o impacto potencial de uma falha de isolamento.

Com User Namespace, podemos reduzir esse risco.

Conceito importante:

**Root dentro do container não precisa significar root no host.**

### E. UTS Namespace

Isola informações relacionadas à identificação do sistema.

Principalmente:

- Hostname.
- NIS domain name.

Exemplo:

Host:

```bash
hostname
```

Resultado:

```text
servidor-linux
```

Container:

```bash
hostname
```

Resultado:

```text
container-a
```

Um container pode ter seu próprio hostname sem alterar o hostname do host.

### F. IPC Namespace

Isola mecanismos de comunicação entre processos.

Entre os recursos envolvidos estão:

- System V IPC.
- POSIX message queues.

Isso impede que processos de namespaces IPC diferentes compartilhem automaticamente esses mecanismos.

É importante para evitar interferências entre aplicações.

### G. CGROUP Namespace

Isola a visão que um processo possui da hierarquia de cgroups.

É importante distinguir duas coisas:

- Cgroup Namespace: controla a visão dos caminhos de cgroups.
- Cgroups: controlam a alocação e limitação de recursos.

Vamos estudar cgroups em detalhes posteriormente.

### H. TIME Namespace

Permite isolar determinados valores de relógios do kernel, como:

- CLOCK_MONOTONIC.
- CLOCK_BOOTTIME.

É um recurso mais específico e menos utilizado na configuração convencional de containers.

---

# 1.3. Laboratório prático — Investigando Namespaces

Agora vamos sair da teoria.

O objetivo é observar os namespaces de um container e compará-los com os namespaces do host.

### Pré-requisitos

- Linux.
- Docker instalado.
- Utilitário `util-linux` (fornece o comando `lsns`).

No Debian:

```bash
sudo apt install util-linux
```

### Experimento 1 — Criando um container

Execute:

```bash
docker run -d \
  --name ns-lab \
  alpine \
  sleep 3600
```

Verifique se está em execução:

```bash
docker ps
```

Obtenha o PID do processo principal no host:

```bash
PID=$(docker inspect \
  --format '{{.State.Pid}}' ns-lab)

echo "$PID"
```

Observe que esse é o PID visto pelo host, não necessariamente o PID visto dentro do container.

### Experimento 2 — Identificando os namespaces

Primeiro, examine os namespaces do host:

```bash
lsns
```

Agora examine os namespaces associados ao processo do container:

```bash
lsns -p "$PID"
```

Você deverá encontrar namespaces associados a PID, NET, MNT, IPC, UTS, USER e CGROUP, entre outros.

A presença de namespaces separados demonstra que o container possui uma visão isolada desses recursos.

### Experimento 3 — Comparando os identificadores

Execute:

```bash
ls -l /proc/1/ns/
```

Agora:

```bash
ls -l /proc/"$PID"/ns/
```

Você verá resultados semelhantes a:

```text
pid:[4026531836]
mnt:[4026531841]
net:[4026531840]
uts:[4026531838]
ipc:[4026531839]
user:[4026531837]
```

Os números são identificadores de namespaces, conhecidos como *namespace inode numbers*.

Faça a comparação entre o processo 1 do host e o processo principal do container.

Por exemplo:

```bash
readlink /proc/1/ns/pid
readlink /proc/"$PID"/ns/pid
```

Se os valores forem diferentes, significa que os processos pertencem a namespaces PID diferentes.

Repita para:

```bash
for ns in pid net mnt ipc uts user cgroup time; do
    printf "%-8s HOST: " "$ns"
    readlink "/proc/1/ns/$ns"

    printf "%-8s CONTAINER: " "$ns"
    readlink "/proc/$PID/ns/$ns"

    echo
done
```

**Atenção:** se você estiver utilizando Docker rootless ou alguma configuração especial, alguns resultados podem variar.

### Experimento 4 — Comparando a visão dos processos

Execute dentro do container:

```bash
docker exec ns-lab sh -c '
    echo "Hostname:"
    hostname

    echo "PID 1:"
    cat /proc/1/status | grep -E "Name|Pid|PPid"

    echo "Namespaces:"
    ls -l /proc/1/ns/
'
```

Agora compare com o host:

```bash
cat /proc/"$PID"/status | grep -E "Name|Pid|PPid"
```

Observe:

- Dentro do container, o processo `sleep` aparece como PID 1.
- No host, ele possui outro PID.
- Os dois processos referenciam o mesmo processo, mas a identificação depende do namespace.

Uma propriedade importante do PID Namespace é sua hierarquia: um processo em um namespace ancestral pode visualizar processos de namespaces descendentes, mas o inverso não é permitido.

### Experimento 5 — Entrando nos namespaces de um container

O Linux disponibiliza o comando `nsenter`, que permite executar um processo dentro dos namespaces de outro processo.

Experimente:

```bash
sudo nsenter -t "$PID" -p -m -u -i -n sh
```

Aqui:

| Parâmetro | Namespace |
|---|---|
| `-t` | Processo de destino |
| `-p` | PID |
| `-m` | MNT |
| `-u` | UTS |
| `-i` | IPC |
| `-n` | NET |

Você estará executando um shell utilizando os namespaces indicados do container.

Verifique:

```bash
hostname
ip addr
lsns -p $$
```

Para sair:

```bash
exit
```

O `nsenter` é particularmente interessante para administração e investigação de incidentes em containers.

---

# 1.4. O que você precisa compreender antes de avançar

Quero destacar quatro conceitos que considero fundamentais:

**1. Namespace não é uma máquina virtual.**

Ele altera a visão que um processo possui dos recursos do kernel. Não cria um kernel independente.

**2. Namespaces não controlam diretamente o consumo de recursos.**

Esse é o papel dos cgroups.

**3. Namespaces não substituem controle de privilégios.**

Um processo isolado pode continuar possuindo privilégios excessivos dentro do seu namespace.

**4. O isolamento depende da configuração.**

Opções como `--privileged`, `--pid=host`, `--network=host` e compartilhamento de diretórios do host podem enfraquecer significativamente o isolamento.

---

# 1.5. Exercícios de fixação

Antes de avançarmos para Cgroups, gostaria que você conseguisse responder às seguintes questões:

1. Qual é a diferença entre Namespace PID e Namespace CGROUP?
2. Por que um processo pode ser PID 1 dentro de um container e possuir PID 15000 no host?
3. Qual é o risco de executar um container utilizando `--network=host`?
4. O que acontece quando utilizamos `-v /:/host`?
5. Qual é a relação entre User Namespace e o conceito de rootless containers?
6. Se dois containers possuem namespaces PID diferentes, eles necessariamente possuem namespaces NET diferentes?

### Desafio prático

Execute:

```bash
docker run -d --name ns-lab-2 \
  --pid=host \
  alpine sleep 3600
```

Descubra:

1. Qual namespace foi compartilhado com o host?
2. O que acontece quando você executa `ps` dentro desse container?
3. Quais namespaces continuam isolados?
4. Qual é o impacto dessa configuração para a segurança?

Para limpar os laboratórios:

```bash
docker rm -f ns-lab ns-lab-2
```

---

## Próxima aula: 1.3 — Cgroups (Control Groups)

Vamos estudar:

- O que são cgroups e por que foram criados.
- Diferenças entre cgroups v1 e v2.
- Hierarquia de cgroups.
- Controle de CPU.
- Controle de memória.
- Limites de processos (`pids.max`).
- OOM Killer e containers.
- Relação entre cgroups e Kubernetes Requests/Limits.
- Como identificar os limites efetivamente aplicados a um container.

**Uma questão que vamos investigar:** se um container possui um limite de memória de 512 MB, como o kernel Linux consegue impedir que seus processos consumam 2 GB?

Essa pergunta nos levará diretamente ao funcionamento interno dos cgroups.
