# Módulo 1 — Segurança no Linux e isolamento de processos
## Aula 1.3 — Cgroups (Control Groups)

Nesta aula vamos estudar um dos mecanismos mais importantes para o funcionamento de containers: **Control Groups (cgroups)**.

Enquanto os Namespaces controlam a visão que um processo possui dos recursos do sistema, os Cgroups controlam quanto desses recursos um conjunto de processos pode consumir.

Essa diferença é fundamental:

| Mecanismo | Pergunta que responde |
|---|---|
| Namespaces | O que o processo consegue enxergar? |
| Cgroups | Quanto de recurso o processo pode consumir? |

Por exemplo, imagine um servidor Linux com 16 GB de RAM executando quatro containers.

Sem mecanismos de controle de recursos, um container poderia consumir uma quantidade excessiva de memória e comprometer o funcionamento dos demais serviços, inclusive do próprio host.

Os Cgroups permitem estabelecer limites para evitar esse cenário.

---

# 1. O que são Cgroups?

Cgroups são mecanismos implementados pelo kernel Linux que permitem agrupar processos e controlar o consumo de recursos desses grupos.

O nome vem de *Control Groups*.

Eles foram desenvolvidos inicialmente pelo Google e incorporados ao kernel Linux em 2008.

O objetivo era permitir que o administrador controlasse recursos computacionais de maneira granular.

Entre os recursos que podem ser controlados estão:

| Recurso | O que pode ser controlado |
|---|---|
| CPU | Tempo de processamento |
| Memory | Consumo de memória |
| PIDs | Quantidade de processos |
| I/O | Operações de leitura e escrita |
| CPUSet | CPUs disponíveis para execução |
| HugeTLB | Uso de páginas de memória HugeTLB |

Uma característica importante é que os Cgroups trabalham com grupos de processos, e não apenas com processos individuais.

Imagine:

```text
HOST
 |
 +-- systemd
 |
 +-- Container A
 |     |
 |     +-- Processo 1
 |     +-- Processo 2
 |     +-- Processo 3
 |
 +-- Container B
       |
       +-- Processo 4
       +-- Processo 5
```

Podemos estabelecer limites independentes para cada container.

Por exemplo:

| Container | Limite de memória | CPU |
|---|---:|---:|
| A | 512 MB | 1 CPU |
| B | 1 GB | 2 CPUs |

O kernel passa a controlar o consumo de cada grupo.

**Importante:** os processos criados por um processo que já pertence a determinado Cgroup normalmente herdam a associação ao grupo, salvo movimentações explícitas.

Isso permite que os processos filhos permaneçam sujeitos aos mesmos controles.

---

# 2. Cgroups v1 versus Cgroups v2

O Linux possui duas implementações principais de Cgroups.

- Cgroups v1: implementação original.
- Cgroups v2: implementação mais recente, com arquitetura unificada.

Essa distinção é importante porque você encontrará ambas em ambientes Linux, Docker, Kubernetes e OpenShift.

## 2.1. Cgroups v1

No Cgroups v1, cada controlador de recurso podia possuir sua própria hierarquia.

Por exemplo:

```text
/sys/fs/cgroup/
 |
 +-- cpu/
 |
 +-- memory/
 |
 +-- pids/
 |
 +-- blkio/
 |
 +-- cpuset/
```

Cada diretório representava uma hierarquia de controle.

Isso significa que um processo poderia pertencer a grupos diferentes dependendo do controlador.

Exemplo conceitual:

```text
Processo 1000

CPU:
  /docker/container-a

MEMORY:
  /docker/container-b

PIDS:
  /docker/container-a
```

Essa arquitetura oferecia flexibilidade, mas também introduzia complexidade.

Entre os problemas estavam:

- Hierarquias independentes.
- Dificuldade de gerenciamento.
- Comportamentos inconsistentes entre controladores.
- Maior complexidade para integração com systemd.

## 2.2. Cgroups v2

O Cgroups v2 introduziu uma hierarquia unificada.

Em vez de cada controlador possuir sua própria árvore, todos trabalham sobre uma estrutura comum.

Exemplo:

```text
/sys/fs/cgroup/
 |
 +-- system.slice/
 |
 +-- user.slice/
 |
 +-- docker/
      |
      +-- container-a/
      |
      +-- container-b/
```

Os controladores são habilitados e configurados nessa hierarquia.

Algumas melhorias importantes:

1. Hierarquia unificada.
2. Interface de controle mais consistente.
3. Melhor integração com systemd.
4. Melhor gerenciamento de memória.
5. Controle de recursos mais previsível.
6. Melhor suporte a mecanismos como `memory.high` e `memory.max`.

### Comparação

| Característica | Cgroups v1 | Cgroups v2 |
|---|---|---|
| Arquitetura | Múltiplas hierarquias | Hierarquia unificada |
| Controladores | Independentes | Integrados |
| Gerenciamento | Mais complexo | Mais consistente |
| systemd | Integração mais complexa | Integração nativa |
| memory.max | Não existe | Existe |
| memory.high | Não existe | Existe |
| CPU | cpu.cfs_* | cpu.max |
| PID limit | pids.max | pids.max |

Observe que algumas interfaces possuem nomes diferentes entre as versões.

**Nossa recomendação:** concentre seus estudos em Cgroups v2. É a arquitetura que devemos priorizar para novos ambientes Linux.

---

# 3. Identificando a versão de Cgroups do Linux

Vamos começar a parte prática.

No seu Debian 13, execute:

```bash
stat -fc %T /sys/fs/cgroup
```

Se o resultado for:

```text
cgroup2fs
```

Você está utilizando Cgroups v2.

Se encontrar uma estrutura com diretórios separados para `cpu`, `memory`, `pids` etc., provavelmente está utilizando Cgroups v1 ou uma configuração híbrida.

Outra maneira:

```bash
findmnt -t cgroup,cgroup2
```

Exemplo de resultado em Cgroups v2:

```text
TARGET         SOURCE FSTYPE
/sys/fs/cgroup cgroup cgroup2
```

Você também pode consultar:

```bash
cat /proc/cgroups
```

Mas atenção: esse arquivo sozinho não é suficiente para determinar se o sistema está utilizando v1 ou v2.

---

# 4. Entendendo a hierarquia de Cgroups

Agora vamos compreender como os grupos são organizados.

Em Cgroups v2, a hierarquia começa normalmente em:

```text
/sys/fs/cgroup
```

Podemos visualizar:

```bash
find /sys/fs/cgroup -maxdepth 2 -type d
```

Exemplo:

```text
/sys/fs/cgroup
/sys/fs/cgroup/init.scope
/sys/fs/cgroup/system.slice
/sys/fs/cgroup/system.slice/ssh.service
/sys/fs/cgroup/user.slice
/sys/fs/cgroup/user.slice/user-1000.slice
```

O systemd utiliza essa hierarquia para organizar serviços e sessões de usuários.

Por exemplo:

```text
/sys/fs/cgroup/
 |
 +-- init.scope/
 |
 +-- system.slice/
 |     |
 |     +-- ssh.service/
 |     |
 |     +-- docker.service/
 |
 +-- user.slice/
       |
       +-- user-1000.slice/
```

Cada grupo pode possuir seus próprios parâmetros de controle.

Uma característica fundamental do Cgroups v2 é a organização hierárquica dos recursos.

Um processo associado a um grupo está sujeito aos limites estabelecidos nele e também aos limites dos grupos ancestrais.

Imagine:

```text
/
|
+-- grupo-pai (2 GB)
      |
      +-- container-a (512 MB)
      |
      +-- container-b (1 GB)
```

Mesmo que o container A tente consumir 1 GB, o limite de 512 MB estabelecido para seu grupo será aplicado.

Se o grupo pai possuir um limite de 2 GB, o consumo agregado dos descendentes também estará sujeito a esse limite.

---

# 5. Controladores de recursos no Cgroups v2

Agora vamos estudar os principais controladores.

## 5.1. Controlador de memória (memory)

Este é um dos controladores mais importantes para ambientes de containers.

Ele permite estabelecer limites para a quantidade de memória que um grupo de processos pode consumir.

No Cgroups v2, encontramos alguns arquivos importantes:

| Arquivo | Função |
|---|---|
| `memory.current` | Memória atualmente utilizada |
| `memory.max` | Limite máximo de memória |
| `memory.high` | Limite de pressão de memória |
| `memory.low` | Proteção de memória sob pressão |
| `memory.min` | Proteção mínima de memória |
| `memory.peak` | Pico de utilização |
| `memory.events` | Contadores de eventos de memória |
| `memory.stat` | Estatísticas detalhadas |
| `memory.swap.current` | Swap atualmente utilizado |
| `memory.swap.max` | Limite de swap |

Vamos entender os principais.

### memory.current

Apresenta o consumo atual de memória do Cgroup.

Exemplo:

```bash
cat /sys/fs/cgroup/memory.current
```

Resultado:

```text
268435456
```

O valor está em bytes.

Nesse exemplo:

```text
268435456 / 1024 / 1024 = 256 MB
```

### memory.max

Define o limite máximo de memória.

Exemplo:

```bash
echo 536870912 > /sys/fs/cgroup/memory.max
```

Estamos estabelecendo:

```text
512 MB
```

Se os processos associados ao Cgroup tentarem ultrapassar esse limite, o kernel tentará recuperar memória. Se não conseguir, poderá acionar o mecanismo de OOM.

O valor `max` representa ausência de limite explícito naquele Cgroup.

### memory.high

Este parâmetro merece atenção especial.

Diferentemente de `memory.max`, ele não representa um limite rígido.

Quando o consumo ultrapassa `memory.high`, o kernel começa a exercer pressão sobre os processos, tentando recuperar memória por meio de mecanismos de reclaim.

Na prática:

- `memory.high`: mecanismo de controle de pressão.
- `memory.max`: limite máximo de memória.

Essa distinção é importante para entender o comportamento de aplicações sob pressão de memória.

### memory.events

Permite verificar eventos relacionados ao consumo de memória.

Exemplo:

```bash
cat /sys/fs/cgroup/memory.events
```

Você poderá encontrar:

```text
low 0
high 0
max 0
oom 0
oom_kill 0
oom_group_kill 0
```

Os principais contadores são:

| Contador | Significado |
|---|---|
| high | Eventos de ultrapassagem de `memory.high` |
| max | Situações em que o uso tentou ultrapassar `memory.max` |
| oom | Situações em que o Cgroup entrou em condição de OOM |
| oom_kill | Processos encerrados pelo OOM Killer |
| oom_group_kill | Eventos de encerramento de grupos |

Esses indicadores são extremamente úteis para diagnosticar problemas em aplicações Kubernetes.

---

## 5.2. Controlador de CPU

O controlador CPU permite controlar quanto tempo de processamento um grupo pode utilizar.

No Cgroups v2, os principais arquivos são:

| Arquivo | Função |
|---|---|
| `cpu.max` | Limite máximo de utilização da CPU |
| `cpu.weight` | Peso relativo para distribuição de CPU |
| `cpu.stat` | Estatísticas de utilização |
| `cpu.pressure` | Indicadores de pressão de CPU |

### cpu.max

Este arquivo possui dois parâmetros:

```text
QUOTA PERIOD
```

Exemplo:

```bash
cat /sys/fs/cgroup/cpu.max
```

Resultado:

```text
50000 100000
```

Isso significa:

- Quota: 50.000 microssegundos.
- Período: 100.000 microssegundos.

Portanto, o grupo pode consumir o equivalente a 50% de uma CPU durante cada período.

A fórmula é:

```text
CPU disponível = QUOTA / PERIOD
```

Exemplos:

| cpu.max | Capacidade |
|---|---:|
| `100000 100000` | 1 CPU |
| `200000 100000` | 2 CPUs |
| `50000 100000` | 0,5 CPU |
| `25000 100000` | 0,25 CPU |
| `max 100000` | Sem limite de quota |

Uma observação importante: limitar a quota não significa reservar uma CPU física exclusivamente para o container.

O controlador limita o tempo de processamento disponível.

### cpu.weight

Diferentemente de `cpu.max`, esse parâmetro define o peso relativo do grupo na distribuição de CPU.

Seu valor padrão geralmente é 100, e o intervalo permitido é de 1 a 10.000.

Imagine:

```text
Container A: cpu.weight = 100
Container B: cpu.weight = 200
```

Se ambos estiverem competindo por CPU no mesmo nível hierárquico, o container B terá aproximadamente o dobro do peso do A na distribuição de tempo de processamento.

Isso não significa que B terá garantidamente o dobro de CPU em qualquer circunstância. A distribuição depende da competição pelos recursos e da hierarquia dos grupos.

---

## 5.3. Controlador PIDs

Esse controlador limita a quantidade de tarefas que podem existir dentro de um Cgroup.

Os principais arquivos são:

| Arquivo | Função |
|---|---|
| `pids.current` | Quantidade atual de tarefas |
| `pids.max` | Limite máximo |
| `pids.events` | Eventos relacionados ao limite |

Exemplo:

```bash
cat /sys/fs/cgroup/pids.current
cat /sys/fs/cgroup/pids.max
```

Imagine uma aplicação que começa a criar processos indiscriminadamente.

Um processo comprometido poderia tentar executar um ataque de exaustão de recursos, conhecido como *fork bomb*.

O controlador PIDs ajuda a limitar esse comportamento.

Exemplo:

```bash
echo 100 > /sys/fs/cgroup/pids.max
```

Agora o grupo não poderá ultrapassar 100 tarefas.

**Atenção:** o controlador contabiliza tarefas do kernel, incluindo threads, e não apenas processos tradicionais.

Esse detalhe é importante para aplicações Java, servidores web e aplicações multithread.

---

# 6. OOM Killer — O que acontece quando o container ultrapassa o limite?

Chegamos à questão central da aula.

Imagine:

```text
HOST
 |
 | Memória disponível: 16 GB
 |
 +-- Container A
 |     |
 |     +-- memory.max = 512 MB
 |
 +-- Container B
       |
       +-- memory.max = 2 GB
```

O container A começa a consumir memória.

Seu consumo evolui:

```text
100 MB
200 MB
300 MB
400 MB
500 MB
512 MB
```

A aplicação tenta alocar mais memória.

O que acontece?

## 6.1. Funcionamento do mecanismo

Quando o consumo se aproxima do limite, o kernel tenta recuperar memória.

Entre os mecanismos utilizados estão:

- Reclaim de páginas de memória.
- Recuperação de cache.
- Liberação de páginas recuperáveis.
- Tratamento de pressão de memória.

Se não houver memória suficiente para atender às novas alocações, o kernel poderá entrar em uma condição de OOM (*Out Of Memory*).

Nesse cenário, o OOM Killer poderá encerrar um ou mais processos.

No Cgroups v2, o mecanismo de OOM pode atuar considerando o grupo de processos que atingiu o limite.

O resultado pode ser:

```text
Container A
    |
    +-- Processo principal
    |
    +-- Processo secundário
    |
    +-- Processo encerrado pelo OOM Killer
```

Dependendo da configuração, o processo principal pode ser encerrado ou permanecer em execução.

Isso é especialmente importante para containers com múltiplos processos.

Existe também o parâmetro:

```text
memory.oom.group
```

Quando configurado como `1`, ele permite tratar o Cgroup como uma unidade de OOM, encerrando conjuntamente os processos do grupo, ressalvadas as exceções previstas pelo kernel.

## 6.2. OOM do container versus OOM do host

É importante não confundir dois cenários.

**Cenário A — OOM do Cgroup**

Um container ultrapassa seu limite de memória.

O kernel tenta resolver a situação dentro do contexto do Cgroup.

**Cenário B — OOM global do host**

O próprio servidor fica sem memória disponível.

Nesse caso, o OOM Killer pode precisar encerrar processos de diferentes serviços e containers para preservar o funcionamento do sistema.

São situações diferentes, com impactos diferentes.

Para investigar:

```bash
dmesg -T | grep -i -E 'oom|killed process'
```

Ou:

```bash
journalctl -k | grep -i -E 'oom|killed process'
```

Em ambientes Docker, você também pode verificar:

```bash
docker inspect \
  --format '{{.State.OOMKilled}}' \
  nome-do-container
```

Se retornar:

```text
true
```

O Docker registrou que o container foi encerrado por OOM.

---

# 7. Como o Docker utiliza Cgroups?

Agora vamos relacionar os conceitos com a prática.

Quando executamos:

```bash
docker run -d \
  --name memory-lab \
  --memory=512m \
  --cpus=1 \
  --pids-limit=100 \
  alpine sleep 3600
```

O Docker solicita ao runtime que configure os controles de recursos do container.

Nesse exemplo:

| Parâmetro Docker | Controle |
|---|---|
| `--memory=512m` | Limite de memória |
| `--cpus=1` | Limite de CPU equivalente a 1 CPU |
| `--pids-limit=100` | Limite de tarefas |

No Cgroups v2, podemos encontrar arquivos semelhantes a:

```text
memory.max
cpu.max
pids.max
```

Os valores efetivos são configurados pelo runtime, utilizando as interfaces disponibilizadas pelo kernel.

É importante entender que o Docker não implementa seu próprio mecanismo de limitação de memória. Ele utiliza os recursos de controle fornecidos pelo kernel Linux.

# 8. Como investigar os Cgroups de um container?

Esta é uma parte que considero especialmente importante para seu trabalho com Linux e Kubernetes.

Quero que você aprenda a identificar os limites diretamente no filesystem do kernel, sem depender exclusivamente de `docker inspect`.

## 8.1. Descobrindo o Cgroup de um processo

Vamos utilizar o container criado anteriormente:

```bash
docker run -d \
  --name cgroup-lab \
  --memory=512m \
  --cpus=1 \
  --pids-limit=100 \
  alpine sleep 3600
```

Obtenha o PID no host:

```bash
PID=$(docker inspect \
  --format '{{.State.Pid}}' cgroup-lab)
```

Agora examine:

```bash
cat /proc/$PID/cgroup
```

Em Cgroups v2, você deverá encontrar algo semelhante a:

```text
0::/system.slice/docker-<container-id>.scope
```

Esse resultado informa o caminho do processo na hierarquia unificada.

O formato:

```text
0::/
```

indica que estamos utilizando Cgroups v2.

## 8.2. Localizando os arquivos de controle

O caminho do Cgroup pode ser obtido assim:

```bash
CGROUP=$(awk -F: '$1 == "0" {print $3}' /proc/$PID/cgroup)

echo "$CGROUP"
```

Agora podemos consultar os arquivos:

```bash
CGROUP_DIR="/sys/fs/cgroup${CGROUP}"
```

Memória:

```bash
cat "$CGROUP_DIR/memory.current"
cat "$CGROUP_DIR/memory.max"
```

CPU:

```bash
cat "$CGROUP_DIR/cpu.max"
cat "$CGROUP_DIR/cpu.stat"
```

Processos:

```bash
cat "$CGROUP_DIR/pids.current"
cat "$CGROUP_DIR/pids.max"
```

Você estará consultando diretamente as informações disponibilizadas pelo kernel.

**Uma observação:** dependendo do driver de cgroups, da configuração do Docker e da hierarquia utilizada, o caminho pode ser diferente. Se não encontrar o diretório esperado, verifique a montagem com `findmnt -t cgroup2` e a hierarquia com `systemd-cgls`.

## 8.3. Consultando os controladores disponíveis

Execute:

```bash
cat /sys/fs/cgroup/cgroup.controllers
```

Exemplo:

```text
cpuset cpu io memory pids
```

Esse arquivo informa quais controladores estão disponíveis para utilização naquele nível da hierarquia.

Para verificar quais estão habilitados para os grupos filhos:

```bash
cat /sys/fs/cgroup/cgroup.subtree_control
```

Você poderá encontrar:

```text
cpuset cpu io memory pids
```

No Cgroups v2, a disponibilização dos controladores aos grupos filhos depende dessa hierarquia e das regras de habilitação.

---

# 9. Relação entre Cgroups e Kubernetes Requests/Limits

Agora vamos conectar tudo ao Kubernetes.

Quando você configura um Pod com:

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "500m"

  limits:
    memory: "512Mi"
    cpu: "1"
```

O Kubernetes utiliza os mecanismos de controle de recursos do Linux para implementar os limites dos containers.

Mas existe uma diferença muito importante entre Requests e Limits.

## 9.1. Memory Request

```yaml
requests:
  memory: "256Mi"
```

Representa a quantidade de memória que o Kubernetes considera para fins de agendamento do Pod.

O scheduler utiliza esse valor para decidir em qual Node o Pod poderá ser executado.

**Request não representa um limite rígido de consumo de memória.**

Uma aplicação pode consumir mais que seu Memory Request, desde que existam recursos disponíveis e não ultrapasse outras restrições.

## 9.2. Memory Limit

```yaml
limits:
  memory: "512Mi"
```

Representa o limite de memória do container.

Em um ambiente Linux com Cgroups v2, esse limite é normalmente implementado utilizando:

```text
memory.max
```

Portanto, o kernel consegue impor o limite independentemente de a aplicação ter conhecimento dele.

## 9.3. CPU Request

```yaml
requests:
  cpu: "500m"
```

Representa 0,5 CPU para fins de agendamento.

Também influencia a distribuição relativa de CPU por meio dos mecanismos de peso do Cgroup.

## 9.4. CPU Limit

```yaml
limits:
  cpu: "1"
```

Representa o limite de processamento equivalente a uma CPU.

Em Cgroups v2, normalmente encontramos a configuração correspondente em:

```text
cpu.max
```

Por exemplo:

```text
100000 100000
```

Isso representa uma CPU de quota.

### Resumo Kubernetes x Cgroups

| Kubernetes | Cgroups v2 | Finalidade |
|---|---|---|
| Memory Request | Indiretamente relacionado a políticas de alocação e prioridade | Scheduling e QoS |
| Memory Limit | `memory.max` | Limite de memória |
| CPU Request | `cpu.weight` | Peso relativo de CPU |
| CPU Limit | `cpu.max` | Limite de CPU |
| PID Limit | `pids.max` | Limite de tarefas, quando configurado pelo kubelet/runtime |

Uma observação: a relação entre Requests e os arquivos de Cgroups não é uma correspondência universal e direta. O Kubernetes, o kubelet, o runtime e a configuração do sistema determinam como esses parâmetros são traduzidos.

---

# 10. Laboratório prático — Provocando um OOM em um container

Agora vamos realizar um experimento para observar o comportamento do kernel.

### 10.1. Criando um container com limite de memória

```bash
docker run -d \
  --name oom-lab \
  --memory=512m \
  --memory-swap=512m \
  alpine sleep 3600
```

O parâmetro `--memory-swap=512m` estabelece que o limite combinado de memória e swap seja 512 MB. Assim, não estamos permitindo uma extensão adicional do limite por swap.

### 10.2. Instalando uma ferramenta de stress

Entre no container:

```bash
docker exec -it oom-lab sh
```

Instale o `stress-ng`:

```bash
apk add --no-cache stress-ng
```

### 10.3. Provocando o consumo excessivo

Execute:

```bash
stress-ng --vm 1 --vm-bytes 700M --vm-keep --timeout 60s
```

Estamos solicitando que um processo tente alocar aproximadamente 700 MB, embora o container tenha limite de 512 MB.

Observe o comportamento.

Em condições normais, o kernel deverá impedir que essa alocação seja sustentada e poderá acionar o OOM Killer.

### 10.4. Investigando o resultado

No host:

```bash
docker inspect \
  --format '{{.State.OOMKilled}}' oom-lab
```

Consulte os eventos do Cgroup:

```bash
PID=$(docker inspect --format '{{.State.Pid}}' oom-lab)

CGROUP=$(awk -F: '$1 == "0" {print $3}' /proc/$PID/cgroup)

cat "/sys/fs/cgroup${CGROUP}/memory.events"
```

Observe especialmente:

```text
oom
oom_kill
```

Os contadores permitem identificar se houve uma condição de OOM e se processos foram efetivamente encerrados.

O container pode continuar em execução se somente o processo responsável pelo consumo excessivo tiver sido encerrado.

Para limpar:

```bash
docker rm -f oom-lab
```

---

# 11. Investigando Cgroups diretamente no Kubernetes

Vamos fazer a mesma investigação em um Pod.

Crie o manifesto:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cgroup-lab
spec:
  containers:
    - name: alpine
      image: alpine:latest
      command: ["sh", "-c", "sleep 3600"]

      resources:
        requests:
          memory: 256Mi
          cpu: 500m

        limits:
          memory: 512Mi
          cpu: "1"
```

Aplique:

```bash
kubectl apply -f cgroup-lab.yaml
```

Verifique:

```bash
kubectl describe pod cgroup-lab
```

Agora entre no container:

```bash
kubectl exec -it cgroup-lab -- sh
```

Você pode consultar o próprio namespace:

```bash
cat /proc/self/cgroup
```

Entretanto, dependendo do runtime e da configuração de namespaces, a visão do filesystem `/sys/fs/cgroup` dentro do container pode ser limitada.

Para investigar os limites efetivamente aplicados, o método mais confiável é consultar o Cgroup correspondente diretamente no Node onde o Pod está executando.

No Node:

1. Identifique o PID do container.
2. Consulte `/proc/<PID>/cgroup`.
3. Localize o diretório correspondente em `/sys/fs/cgroup`.
4. Consulte `memory.max`, `cpu.max` e `pids.max`.

No Kubernetes, você pode descobrir o Node com:

```bash
kubectl get pod cgroup-lab -o wide
```

No OpenShift, os mesmos conceitos se aplicam, podendo utilizar `oc` no lugar de `kubectl`.

**Um detalhe importante:** o Kubernetes também trabalha com classes de QoS:

- Guaranteed.
- Burstable.
- BestEffort.

Elas influenciam o tratamento dos Pods sob pressão de recursos, inclusive a seleção de processos para encerramento em situações de OOM.

Isso será especialmente interessante quando estudarmos a relação entre `oom_score_adj`, Memory Requests e Memory Limits.

---

# 12. Resumo da aula

Ao concluir esta aula, você deve compreender:

| Conceito | O que precisa saber |
|---|---|
| Cgroups | Agrupamento e controle de recursos |
| Cgroups v1 | Múltiplas hierarquias |
| Cgroups v2 | Hierarquia unificada |
| memory.max | Limite rígido de memória |
| memory.high | Controle de pressão de memória |
| cpu.max | Limite de processamento |
| cpu.weight | Peso relativo na distribuição de CPU |
| pids.max | Limite de tarefas |
| OOM Killer | Encerramento de processos por falta de memória |
| Kubernetes Requests | Recursos considerados pelo scheduler |
| Kubernetes Limits | Limites aplicados aos workloads |

## 13. Exercícios de fixação

Gostaria que você tentasse responder estas questões antes de avançarmos.

**Questão 1.** Qual é a principal diferença arquitetural entre Cgroups v1 e v2?

**Questão 2.** Se um container possui:

```text
memory.max = 536870912
```

Qual é seu limite de memória em MB?

**Questão 3.** Qual é a diferença entre `memory.high` e `memory.max`?

**Questão 4.** Se configurarmos:

```text
cpu.max = 50000 100000
```

Qual é a capacidade máxima de processamento disponível para o grupo?

**Questão 5.** Um container possui:

```yaml
resources:
  requests:
    memory: 256Mi
  limits:
    memory: 512Mi
```

Se a aplicação tentar consumir 400 MiB, ela será encerrada pelo OOM Killer?

Explique.

**Questão 6.** Qual é a diferença entre um OOM provocado pelo limite de memória de um Cgroup e um OOM global do host?

**Questão 7.** Por que limitar a quantidade de processos de um container é uma medida de segurança?

---

## Próxima aula — 1.4. Linux Capabilities

Depois de compreender como o Linux isola processos e controla seus recursos, vamos estudar outro mecanismo fundamental: as Capabilities.

A pergunta que vamos investigar é:

**Se um processo está executando como root dentro de um container, por que ele não necessariamente possui todos os privilégios de root no Linux?**

Vamos estudar:

- Como surgiu o modelo de Linux Capabilities.
- Diferença entre UID 0 e privilégios administrativos.
- Effective, Permitted, Inheritable, Bounding e Ambient Capabilities.
- Capabilities perigosas em containers.
- `CAP_SYS_ADMIN`, `CAP_NET_ADMIN`, `CAP_SYS_PTRACE` e outras.
- Como remover privilégios com `--cap-drop`.
- Como configurar Security Context no Kubernetes.
- Relação entre Capabilities e OpenShift SCC.

**Minha recomendação:** dê atenção especial às Capabilities. É um dos mecanismos mais importantes para reduzir o impacto de um eventual comprometimento de aplicações em containers.
