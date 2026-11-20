# Módulo 2 — Docker e OCI Runtime Security
## Aula 2.7 — Segurança do Runtime: Privileged Containers, Host Namespaces e Container Escape

Esta é uma das aulas mais importantes do módulo porque vamos estudar o que acontece quando um container deixa de ter um isolamento adequado e passa a ter acesso a recursos que deveriam pertencer exclusivamente ao host.

Até aqui, estudamos os mecanismos que permitem ao Linux isolar containers. Agora vamos analisar como configurações inadequadas podem enfraquecer ou até comprometer esse isolamento.

A ideia central desta aula é:

> Um container não é uma fronteira de segurança absoluta. Sua segurança depende da combinação de isolamento, restrições de privilégios, controles de acesso e configuração do runtime.

---

# 1. O que significa executar um container privilegiado?

No Docker, podemos iniciar um container com:

```bash
docker run --privileged -it alpine sh
```

O parâmetro `--privileged` solicita ao Docker que execute o container com um conjunto muito mais amplo de permissões e acessos ao host.

Isso não significa simplesmente executar o processo como `root`.

São conceitos diferentes:

| Configuração | Significado |
|---|---|
| `--user root` | Executa o processo com UID 0 dentro do container. |
| `--cap-add` | Adiciona capabilities específicas. |
| `--cap-drop` | Remove capabilities específicas. |
| `--privileged` | Concede um conjunto muito amplo de privilégios e acessos a recursos do host. |

Um container executado como root, mas com capabilities limitadas, Seccomp e MAC habilitados, ainda pode ter um isolamento razoável.

Já um container privilegiado pode ter acesso a recursos que permitem interferir diretamente no host.

## 1.1 O que muda com `--privileged`?

Em linhas gerais, o Docker:

- Concede todas as capabilities disponíveis ao container, conforme o contexto do kernel e do runtime.
- Desabilita ou relaxa o perfil Seccomp padrão.
- Desabilita ou relaxa o perfil AppArmor aplicado pelo Docker.
- Configura acesso ampliado aos dispositivos do host.
- Ajusta restrições relacionadas a dispositivos e outras proteções do runtime.

Atenção: o comportamento exato depende da implementação do runtime, da versão do Docker e das políticas de segurança do host.

O importante é entender que `--privileged` não é apenas uma forma de adicionar permissões. Ele altera significativamente o modelo de isolamento.

### Laboratório 1 — Comparando containers

Vamos criar dois containers:

**Container normal:**

```bash
docker run -d \
  --name normal \
  alpine \
  sleep 3600
```

**Container privilegiado:**

```bash
docker run -d \
  --name privileged \
  --privileged \
  alpine \
  sleep 3600
```

Agora vamos comparar as capabilities:

```bash
docker exec normal sh -c 'grep Cap /proc/1/status'
```

```bash
docker exec privileged sh -c 'grep Cap /proc/1/status'
```

O resultado apresenta cinco conjuntos:

```text
CapInh
CapPrm
CapEff
CapBnd
CapAmb
```

Converta os valores para nomes legíveis:

```bash
docker exec normal sh -c \
  'apk add --no-cache libcap >/dev/null 2>&1; capsh --decode=$(awk "/CapEff/ {print \$2}" /proc/1/status)'
```

Faça o mesmo no container privilegiado:

```bash
docker exec privileged sh -c \
  'apk add --no-cache libcap >/dev/null 2>&1; capsh --decode=$(awk "/CapEff/ {print \$2}" /proc/1/status)'
```

Você deverá observar uma diferença expressiva.

Para verificar o modo Seccomp:

```bash
docker exec normal grep Seccomp /proc/1/status
docker exec privileged grep Seccomp /proc/1/status
```

Normalmente, o container comum apresenta:

```text
Seccomp: 2
```

O valor `2` indica que existe um filtro Seccomp ativo.

No privilegiado, é esperado encontrar:

```text
Seccomp: 0
```

O valor `0` indica ausência de filtro Seccomp nesse processo.

Também podemos verificar o AppArmor no host:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' normal)

sudo cat /proc/$PID/attr/current
```

Depois:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' privileged)

sudo cat /proc/$PID/attr/current
```

O resultado depende da configuração AppArmor do host e do perfil utilizado pelo Docker.

**Conclusão:** `--privileged` enfraquece simultaneamente várias camadas de proteção.

---

# 2. Diferenças entre `--privileged`, `--cap-add` e `--cap-drop`

Agora vamos aprofundar um conceito importante do Módulo 1: Linux Capabilities.

Lembre-se:

Tradicionalmente, o Linux concentrava muitos privilégios administrativos no usuário root.

O mecanismo de capabilities divide esses privilégios em unidades menores.

Por exemplo:

| Capability | Permissão |
|---|---|
| `CAP_NET_ADMIN` | Administrar interfaces, rotas e configurações de rede. |
| `CAP_NET_RAW` | Criar sockets raw e packet sockets. |
| `CAP_SYS_ADMIN` | Conjunto muito amplo de operações administrativas. |
| `CAP_SYS_PTRACE` | Permitir operações de rastreamento de processos. |
| `CAP_DAC_OVERRIDE` | Contornar verificações tradicionais de permissões DAC. |
| `CAP_SYS_CHROOT` | Utilizar determinadas operações de `chroot`. |
| `CAP_MKNOD` | Criar determinados arquivos especiais de dispositivo. |

## 2.1 `--cap-add`

Adiciona uma capability específica.

Exemplo:

```bash
docker run -d \
  --name netadmin \
  --cap-add=NET_ADMIN \
  alpine \
  sleep 3600
```

Nesse caso, o container recebe a capability `CAP_NET_ADMIN`.

Isso pode ser necessário para aplicações que precisam configurar interfaces de rede, rotas ou regras de firewall.

Mas não significa que o container ganhou todas as permissões administrativas do host.

## 2.2 `--cap-drop`

Remove capabilities.

Exemplo:

```bash
docker run -d \
  --name restricted \
  --cap-drop=ALL \
  alpine \
  sleep 3600
```

Esse container começa sem capabilities adicionais concedidas pelo runtime.

Podemos verificar:

```bash
docker exec restricted grep Cap /proc/1/status
```

Agora podemos conceder somente uma capability necessária:

```bash
docker run -d \
  --name restricted-net \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  alpine \
  sleep 3600
```

Esse modelo é muito interessante para segurança.

Em vez de conceder permissões amplas e tentar restringi-las posteriormente, removemos tudo e adicionamos somente o necessário.

É o princípio do menor privilégio.

### 2.3 Comparação prática

| Recurso | Container normal | `--cap-add` | `--cap-drop=ALL` | `--privileged` |
|---|---|---|---|---|
| Capabilities | Conjunto padrão | Adiciona específicas | Remove todas as adicionais concedidas | Conjunto amplo |
| Seccomp | Perfil padrão | Geralmente mantido | Geralmente mantido | Desabilitado ou relaxado |
| AppArmor | Perfil configurado | Geralmente mantido | Geralmente mantido | Desabilitado ou relaxado |
| Dispositivos | Acesso restrito | Não concede automaticamente acesso a dispositivos | Acesso restrito | Acesso ampliado |
| Isolamento | Padrão | Em geral preservado | Em geral preservado | Significativamente enfraquecido |

Uma observação: capabilities e acesso a dispositivos são controles distintos. Adicionar `CAP_MKNOD`, por exemplo, não significa automaticamente que o processo poderá acessar qualquer dispositivo do host.

---

# 3. Riscos de utilizar namespaces do host

Lembre-se do que estudamos na Aula 1.2.

Namespaces são responsáveis por isolar a visão que os processos possuem dos recursos do sistema.

O problema aparece quando permitimos que um container compartilhe namespaces do host.

O Docker oferece opções para isso:

```bash
--pid=host
--network=host
--ipc=host
```

Vamos estudar cada uma.

## 3.1 `--pid=host`

Normalmente, um container possui seu próprio PID namespace.

Assim, o processo principal pode aparecer como PID 1 dentro do container, mesmo tendo outro PID no host.

Quando utilizamos:

```bash
docker run -it --pid=host alpine sh
```

O container passa a compartilhar o PID namespace do host.

Consequências:

- O container consegue visualizar processos do host.
- Pode conseguir enviar sinais a processos do host, conforme as permissões.
- Pode expor informações de processos e aplicações.
- Ferramentas de diagnóstico podem visualizar processos que normalmente estariam isolados.

### Laboratório 2 — Comparando PID namespaces

Primeiro, crie um container normal:

```bash
docker run -d --name pid-normal alpine sleep 3600
```

Agora crie outro compartilhando o PID namespace:

```bash
docker run -d \
  --name pid-host \
  --pid=host \
  alpine \
  sleep 3600
```

Instale ferramentas de processo:

```bash
docker exec pid-normal apk add --no-cache procps
docker exec pid-host apk add --no-cache procps
```

Compare:

```bash
docker exec pid-normal ps -ef
```

```bash
docker exec pid-host ps -ef
```

Observe que o segundo container consegue visualizar processos do host.

Isso não significa que ele pode controlar todos esses processos irrestritamente. As permissões continuam dependendo de UID, capabilities, ptrace restrictions, LSM e outros controles.

Mas a superfície de exposição aumentou consideravelmente.

## 3.2 `--network=host`

Normalmente, um container possui seu próprio network namespace.

Isso significa que ele possui uma visão isolada de:

- Interfaces de rede.
- Endereços IP.
- Tabelas de roteamento.
- Regras de firewall associadas ao namespace.
- Portas e sockets de rede.

Quando utilizamos:

```bash
docker run -it --network=host alpine sh
```

O container compartilha o network namespace do host.

Consequências:

1. Não existe o isolamento habitual de interfaces de rede.
2. O container utiliza as interfaces e endereços do host.
3. As portas utilizadas pelo container são portas do próprio host.
4. A aplicação pode observar informações da rede do host.
5. Com permissões suficientes, pode modificar configurações de rede.

Um exemplo:

```bash
docker run -d \
  --name nginx-host \
  --network=host \
  nginx
```

O Nginx passa a utilizar diretamente a rede do host.

Não precisamos fazer o mapeamento:

```bash
-p 8080:80
```

Isso acontece porque não existe a separação habitual entre as pilhas de rede.

**Atenção:** `--network=host` não significa automaticamente que o container pode administrar o firewall do host. Para isso, entram em jogo capabilities como `CAP_NET_ADMIN`.

## 3.3 `--ipc=host`

O namespace IPC controla determinados mecanismos de comunicação entre processos.

Entre eles:

- System V IPC.
- Filas de mensagens.
- Semáforos.
- Memória compartilhada.

Com:

```bash
docker run -it --ipc=host alpine sh
```

O container compartilha o namespace IPC do host.

Isso pode permitir interação com mecanismos IPC existentes no host, conforme as permissões e configurações aplicáveis.

O risco aumenta quando aplicações trabalham com dados sensíveis em memória compartilhada ou utilizam mecanismos IPC sem isolamento adequado.

### Resumo dos namespaces do host

| Opção | Namespace compartilhado | Principal risco |
|---|---|---|
| `--pid=host` | PID | Exposição e possível interação com processos do host. |
| `--network=host` | NET | Acesso direto à rede do host. |
| `--ipc=host` | IPC | Interação com mecanismos IPC do host. |
| `--uts=host` | UTS | Compartilhamento de hostname e domínio NIS. |
| `--userns=host` | USER | Ausência do isolamento por user namespace. |

A recomendação é evitar compartilhar namespaces do host sem uma necessidade operacional muito bem justificada.

---

# 4. Acesso a dispositivos do host

Este é outro ponto importante.

O Linux representa diversos dispositivos por arquivos especiais, normalmente encontrados em:

```bash
/dev
```

Exemplos:

| Dispositivo | Finalidade |
|---|---|
| `/dev/sda` | Disco ou dispositivo de bloco. |
| `/dev/nvme0n1` | Disco NVMe. |
| `/dev/mem` | Interface de acesso à memória física, sujeita a restrições. |
| `/dev/kvm` | Interface de virtualização KVM. |
| `/dev/dri` | Dispositivos gráficos. |
| `/dev/net/tun` | Dispositivo TUN/TAP. |
| `/dev/fuse` | Interface FUSE. |

Um container normalmente recebe apenas um conjunto limitado de dispositivos.

O runtime utiliza mecanismos como:

- Bind mounts.
- Device cgroup controller.
- Permissões Unix.
- Capabilities.
- LSM.

Esses mecanismos atuam em conjunto.

## 4.1 Por que dispositivos são perigosos?

Imagine que um container tenha acesso de escrita a um dispositivo de bloco do host.

Se ele conseguir acessar diretamente o dispositivo que contém o filesystem do host, poderá potencialmente modificar dados fora do seu próprio root filesystem.

Isso compromete o isolamento.

Outro exemplo é o dispositivo:

```text
/dev/kvm
```

Ele permite que aplicações utilizem a infraestrutura de virtualização KVM, quando os demais requisitos e permissões estão presentes.

O acesso pode ser necessário para aplicações específicas, mas deve ser concedido deliberadamente.

## 4.2 Docker e `--device`

O Docker permite conceder acesso a dispositivos específicos:

```bash
docker run \
  --device=/dev/fuse:/dev/fuse \
  alpine
```

O container recebe acesso ao dispositivo indicado, sujeito às permissões e restrições aplicáveis.

Isso é muito diferente de:

```bash
docker run --privileged alpine
```

No primeiro caso, concedemos acesso a um dispositivo específico.

No segundo, ampliamos significativamente o acesso a dispositivos e privilégios.

**Princípio de segurança:** conceda somente os dispositivos que a aplicação realmente precisa.

---

# 5. Como uma configuração inadequada pode permitir acesso ao host?

Vamos conectar os conceitos.

Imagine um container com a seguinte configuração:

```bash
docker run -d \
  --name insecure \
  --privileged \
  --pid=host \
  --network=host \
  -v /:/host \
  alpine \
  sleep 3600
```

Essa configuração combina vários problemas:

| Configuração | Problema |
|---|---|
| `--privileged` | Amplia privilégios e acesso a dispositivos. |
| `--pid=host` | Compartilha os processos do host. |
| `--network=host` | Compartilha a rede do host. |
| `-v /:/host` | Expõe o filesystem do host dentro do container. |

Observe o último ponto.

O diretório `/host` dentro do container representa o filesystem raiz do host.

Podemos verificar:

```bash
docker exec insecure ls /host
```

Dependendo das permissões, poderíamos encontrar diretórios como:

```text
/host/etc
/host/root
/host/var
/host/usr
/host/home
```

Se o bind mount estiver em modo leitura e escrita, o container poderá modificar arquivos do host, conforme as permissões e os demais controles de segurança.

**Esse exemplo é intencionalmente inseguro. Não o execute em uma máquina de produção.**

Para experimentar, utilize uma VM descartável.

Uma alternativa menos perigosa para estudar o conceito é montar somente um diretório temporário:

```bash
mkdir -p /tmp/container-lab

echo "arquivo do host" > /tmp/container-lab/host.txt

docker run --rm -it \
  -v /tmp/container-lab:/host-lab \
  alpine sh
```

Dentro do container:

```bash
cat /host-lab/host.txt
```

Crie um arquivo:

```bash
echo "alterado pelo container" > /host-lab/container.txt
```

Saia do container e verifique:

```bash
cat /tmp/container-lab/container.txt
```

Você acabou de demonstrar que um bind mount permite compartilhar dados entre host e container.

O isolamento do root filesystem não impede o acesso aos diretórios explicitamente compartilhados.

---

# 6. O que é Container Escape?

Agora chegamos ao conceito central desta aula.

Container escape é uma situação em que um processo que deveria estar restrito ao ambiente do container consegue ultrapassar a fronteira de isolamento e acessar recursos do host ou de outros ambientes que deveriam estar protegidos.

Podemos dividir os cenários em algumas categorias.

## 6.1 Escape por configuração inadequada

Ocorre quando o administrador concede permissões excessivas.

Exemplos:

- Container privilegiado.
- Bind mount da raiz do host.
- Acesso irrestrito ao Docker socket.
- Compartilhamento de namespaces do host.
- Acesso a dispositivos sensíveis.

Esse tipo de problema não exige necessariamente uma vulnerabilidade no kernel.

A configuração já enfraqueceu a fronteira de segurança.

## 6.2 Escape por vulnerabilidade do runtime

Pode ocorrer quando existe uma falha em componentes como:

- `runc`.
- `containerd`.
- `crun`.
- Docker Engine.
- Bibliotecas utilizadas pelo runtime.

Uma vulnerabilidade pode permitir que uma operação realizada dentro de um container afete recursos do host.

Por isso é importante manter o runtime atualizado.

## 6.3 Escape por vulnerabilidade do kernel

Esse é um cenário especialmente importante.

Lembre-se:

> Containers compartilham o kernel do host.

Se existir uma vulnerabilidade explorável no kernel Linux, um processo dentro de um container pode, dependendo das condições, explorar essa falha e obter acesso a recursos fora do seu ambiente isolado.

Namespaces não criam um kernel independente.

Cgroups também não criam um kernel independente.

Capabilities não corrigem vulnerabilidades do kernel.

Seccomp e MAC reduzem a superfície de ataque, mas não garantem proteção contra todas as vulnerabilidades.

## 6.4 Docker socket: uma exposição particularmente perigosa

Considere:

```bash
-v /var/run/docker.sock:/var/run/docker.sock
```

Esse socket permite que o container se comunique com o Docker Engine.

Se o processo dentro do container tiver acesso irrestrito ao socket, poderá enviar comandos administrativos ao daemon.

Por exemplo, poderá solicitar a criação de outros containers com acesso ao host.

Na prática, em uma instalação Docker convencional, conceder acesso irrestrito ao Docker socket frequentemente equivale a conceder controle administrativo sobre o host.

Isso é extremamente perigoso.

Evite montar esse socket em aplicações que não precisam administrá-lo.

---

# 7. Como os mecanismos estudados no Módulo 1 ajudam a prevenir escapes?

Vamos conectar os conceitos.

Nenhum mecanismo isolado oferece proteção completa.

A segurança depende de camadas complementares.

| Mecanismo | O que protege | Limitação |
|---|---|---|
| Namespaces | Isolamento da visão dos recursos | Não impedem, sozinhos, acesso a recursos explicitamente compartilhados. |
| Cgroups | Limites e contabilização de recursos | Não são mecanismos gerais de controle de acesso. |
| Capabilities | Restringem privilégios administrativos | Não corrigem vulnerabilidades do kernel. |
| Seccomp | Restringe chamadas de sistema | Não controla sozinho todos os acessos a arquivos e recursos. |
| AppArmor | Restringe operações de processos sobre recursos | Depende de políticas adequadas e corretamente aplicadas. |
| SELinux | Controle de acesso obrigatório baseado em políticas e rótulos | Depende da política, dos rótulos e da configuração. |

Vamos entender algumas combinações.

### 7.1 Namespaces + Capabilities

Namespaces isolam recursos.

Capabilities restringem operações privilegiadas.

Por exemplo, um container pode ter seu próprio network namespace, mas sem `CAP_NET_ADMIN` não poderá executar várias operações administrativas sobre sua rede.

### 7.2 Capabilities + Seccomp

Imagine que um processo não possui `CAP_SYS_ADMIN`.

Isso já restringe muitas operações administrativas.

Mas podemos também restringir chamadas de sistema que não são necessárias para a aplicação.

Assim, mesmo que uma aplicação tenha uma vulnerabilidade, a superfície de operações disponíveis ao invasor será menor.

### 7.3 Seccomp + AppArmor/SELinux

Seccomp atua no nível das chamadas de sistema.

AppArmor e SELinux atuam no controle de acesso obrigatório.

Exemplo conceitual:

Uma aplicação pode conseguir executar uma chamada `openat()` porque ela não foi bloqueada pelo Seccomp.

Entretanto, o AppArmor ou SELinux pode impedir que ela abra determinado arquivo.

São camadas diferentes.

### 7.4 Defesa em profundidade

Uma configuração razoavelmente restritiva para uma aplicação comum poderia ser:

```bash
docker run -d \
  --name secure-app \
  --user 10001:10001 \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --read-only \
  --memory=512m \
  --cpus=1 \
  --pids-limit=100 \
  alpine \
  sleep 3600
```

Observe as proteções:

- Usuário não root.
- Nenhuma capability adicional concedida.
- Impede ganho de privilégios por mecanismos como setuid.
- Root filesystem somente leitura.
- Limite de memória.
- Limite de CPU.
- Limite de processos.

Em produção, a aplicação ainda pode precisar de diretórios graváveis, como `/tmp`, que podem ser disponibilizados por tmpfs ou volumes específicos.

Também é importante verificar se o perfil Seccomp padrão e o perfil MAC estão efetivamente aplicados.

---

# 8. Investigando configurações perigosas no Docker

Agora vamos construir um pequeno procedimento de auditoria.

## 8.1 Identificar containers privilegiados

Liste os containers:

```bash
docker ps -q
```

Inspecione cada container:

```bash
for container in $(docker ps -q); do
    docker inspect "$container" \
      --format '{{.Name}} Privileged={{.HostConfig.Privileged}}'
done
```

Saída ilustrativa:

```text
/normal Privileged=false
/privileged Privileged=true
```

## 8.2 Identificar compartilhamento de namespaces

```bash
for container in $(docker ps -q); do
    docker inspect "$container" \
      --format '{{.Name}} PID={{.HostConfig.PidMode}} NET={{.HostConfig.NetworkMode}} IPC={{.HostConfig.IpcMode}}'
done
```

Procure configurações como:

```text
PID=host
NET=host
IPC=host
```

## 8.3 Identificar mounts do host

```bash
for container in $(docker ps -q); do
    echo "Container: $(docker inspect -f '{{.Name}}' "$container")"

    docker inspect "$container" \
      --format '{{range .Mounts}}{{println .Source "->" .Destination "RW=" .RW}}{{end}}'
done
```

Fique atento a:

```text
/ -> /host
/etc -> /host-etc
/var/run/docker.sock -> /var/run/docker.sock
/proc -> /host-proc
/sys -> /host-sys
```

Nem todo mount desses caminhos é necessariamente vulnerável, mas todos merecem investigação.

## 8.4 Identificar capabilities

```bash
docker inspect container-name \
  --format 'Add={{json .HostConfig.CapAdd}} Drop={{json .HostConfig.CapDrop}}'
```

Investigue especialmente:

```text
SYS_ADMIN
SYS_PTRACE
NET_ADMIN
SYS_MODULE
DAC_OVERRIDE
```

A presença de uma capability não significa automaticamente comprometimento, mas deve existir uma justificativa operacional.

## 8.5 Criando uma função de auditoria simples

Podemos reunir algumas verificações em um script:

```bash
#!/usr/bin/env bash

for container in $(docker ps -q); do

    docker inspect "$container" | jq -r '
      .[0] |
      {
        name: .Name,
        privileged: .HostConfig.Privileged,
        pid_mode: .HostConfig.PidMode,
        network_mode: .HostConfig.NetworkMode,
        ipc_mode: .HostConfig.IpcMode,
        cap_add: .HostConfig.CapAdd,
        cap_drop: .HostConfig.CapDrop,
        readonly_rootfs: .HostConfig.ReadonlyRootfs,
        security_opt: .HostConfig.SecurityOpt,
        mounts: [
          .Mounts[]? |
          {
            source: .Source,
            destination: .Destination,
            rw: .RW
          }
        ]
      }'
done
```

Esse script não é um scanner completo, mas já permite identificar configurações que merecem atenção.

---

# 9. Segurança no Kubernetes

Agora vamos transportar os conceitos para Kubernetes.

No Kubernetes, as configurações de segurança de containers são definidas principalmente por meio de `securityContext`, configurações do Pod e políticas de admissão.

## 9.1 Container privilegiado

Exemplo de configuração perigosa:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-lab
spec:
  containers:
    - name: app
      image: alpine
      command: ["sleep", "3600"]
      securityContext:
        privileged: true
```

O campo:

```yaml
privileged: true
```

solicita execução privilegiada do container.

No Kubernetes, isso possui efeitos semelhantes à execução de um container Docker privilegiado, mas o comportamento final também depende do CRI, runtime e políticas do cluster.

## 9.2 Capabilities

Podemos definir capabilities no `securityContext`:

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
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
```

Esse modelo segue o princípio do menor privilégio.

Removemos todas as capabilities e concedemos somente a necessária.

## 9.3 Host namespaces

O Kubernetes também permite compartilhar namespaces do host.

### PID

```yaml
spec:
  hostPID: true
```

### Network

```yaml
spec:
  hostNetwork: true
```

### IPC

```yaml
spec:
  hostIPC: true
```

Essas opções devem ser avaliadas cuidadosamente.

Uma aplicação que precisa observar processos do host, por exemplo, pode exigir `hostPID`.

Mas isso não significa que todos os containers devam receber essa permissão.

## 9.4 hostPath

O Kubernetes oferece `hostPath` para montar diretórios do nó dentro do Pod.

Exemplo:

```yaml
volumes:
  - name: host-data
    hostPath:
      path: /var/log
      type: Directory
```

O container pode acessar esse diretório por meio de um volume.

O risco depende de:

- Caminho montado.
- Permissões.
- Modo de montagem.
- UID/GID do processo.
- Política SELinux.
- Outras proteções do host.

Uma montagem somente leitura reduz os riscos de modificação:

```yaml
volumeMounts:
  - name: host-data
    mountPath: /host-logs
    readOnly: true
```

Mas somente leitura não elimina todos os riscos de exposição de informações.

---

# 10. Hardening de um Pod Kubernetes

Vamos construir um exemplo mais restritivo.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: app
      image: alpine
      command: ["sleep", "3600"]

      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001

        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true

        capabilities:
          drop:
            - ALL
```

Observe que não precisamos declarar:

```yaml
privileged: false
```

O valor padrão já é `false`.

Também não precisamos declarar:

```yaml
hostPID: false
hostNetwork: false
hostIPC: false
```

Esses campos também são falsos por padrão.

A configuração acima combina:

- Usuário não root.
- Seccomp RuntimeDefault.
- Remoção de capabilities.
- Impedimento de escalada de privilégios.
- Root filesystem somente leitura.

Dependendo da imagem, será necessário disponibilizar diretórios temporários graváveis.

Por exemplo:

```yaml
volumes:
  - name: tmp
    emptyDir: {}

containers:
  - name: app
    volumeMounts:
      - name: tmp
        mountPath: /tmp
```

Esse padrão é muito mais adequado para aplicações que não precisam modificar seu próprio root filesystem.

---

# 11. OpenShift e Security Context Constraints (SCC)

Agora chegamos a um assunto especialmente importante no seu ambiente de trabalho.

O OpenShift utiliza Security Context Constraints (SCC) para controlar quais permissões os Pods podem receber.

A SCC funciona como uma política de segurança aplicada à criação dos Pods.

Ela pode controlar:

- Execução privilegiada.
- Capabilities permitidas.
- Tipos de volumes.
- Uso de host namespaces.
- UID e grupos.
- SELinux context.
- Perfis Seccomp.
- Escalação de privilégios.

## 11.1 SCCs mais conhecidas

No OpenShift, podemos encontrar SCCs como:

| SCC | Finalidade geral |
|---|---|
| `restricted-v2` | Política restritiva padrão para workloads comuns. |
| `restricted` | Política restritiva legada. |
| `nonroot` | Permite execução não root, com outras restrições. |
| `anyuid` | Permite utilizar qualquer UID. |
| `hostmount-anyuid` | Permite determinadas montagens do host e execução com qualquer UID. |
| `hostnetwork` | Permite compartilhamento de rede do host. |
| `privileged` | Permite configurações altamente privilegiadas. |

A disponibilidade e os detalhes podem variar conforme a versão e a configuração do OpenShift.

## 11.2 Inspecionando as SCCs

Liste:

```bash
oc get scc
```

Examine a SCC padrão:

```bash
oc describe scc restricted-v2
```

Procure campos como:

```yaml
allowPrivilegedContainer: false
allowHostDirVolumePlugin: false
allowHostNetwork: false
allowHostPID: false
allowHostIPC: false
```

Também investigue:

```yaml
requiredDropCapabilities:
allowedCapabilities:
defaultAddCapabilities:
allowPrivilegeEscalation:
seccompProfiles:
```

Esses campos mostram como a SCC restringe os recursos que os containers podem utilizar.

## 11.3 Verificando qual SCC foi utilizada

Depois que um Pod é criado:

```bash
oc get pod meu-pod -o yaml
```

Procure a anotação:

```yaml
metadata:
  annotations:
    openshift.io/scc: restricted-v2
```

Essa informação é muito útil durante troubleshooting.

Você pode ter um Deployment com um `securityContext` aparentemente correto, mas o Pod não ser criado porque a SCC não permite determinada configuração.

## 11.4 Identificando usuários com acesso a uma SCC

Por exemplo:

```bash
oc describe scc privileged
```

Você poderá encontrar informações sobre usuários e grupos autorizados.

Também pode consultar:

```bash
oc adm policy who-can use scc privileged
```

Esse comando é especialmente útil em auditorias.

Ele ajuda a identificar quem possui autorização para utilizar uma SCC privilegiada.

**Recomendação:** evite conceder a SCC `privileged` diretamente a uma ServiceAccount de aplicação sem uma necessidade operacional muito bem justificada.

A autorização para usar uma SCC é uma permissão de segurança importante.

---

# 12. Comparação: Docker, Kubernetes e OpenShift

Vamos consolidar os conceitos:

| Recurso | Docker | Kubernetes | OpenShift |
|---|---|---|---|
| Container privilegiado | `--privileged` | `privileged: true` | Controlado pela SCC |
| Capabilities | `--cap-add`, `--cap-drop` | `securityContext.capabilities` | SCC + `securityContext` |
| PID do host | `--pid=host` | `hostPID: true` | SCC controla permissão |
| Rede do host | `--network=host` | `hostNetwork: true` | SCC controla permissão |
| IPC do host | `--ipc=host` | `hostIPC: true` | SCC controla permissão |
| Mount do host | `-v` / `--mount` | `hostPath` | SCC controla tipo de volume |
| Seccomp | `--security-opt seccomp=...` | `seccompProfile` | SCC pode restringir perfis |
| AppArmor | `--security-opt apparmor=...` | `appArmorProfile` em versões compatíveis | Depende da integração e configuração do cluster |
| SELinux | Configuração do host/runtime | Configuração do nó/runtime | Integração nativa importante |

Uma observação: a tabela apresenta os mecanismos de configuração. A disponibilidade efetiva depende da versão do Kubernetes/OpenShift, do runtime e das políticas instaladas.

---

# 13. Laboratório de auditoria — Docker e Kubernetes

Vamos fechar a aula com um exercício prático.

## Parte A — Docker

Crie um container normal:

```bash
docker run -d \
  --name audit-normal \
  alpine sleep 3600
```

Crie outro com algumas configurações perigosas, mas sem utilizar `--privileged`:

```bash
docker run -d \
  --name audit-risk \
  --pid=host \
  --network=host \
  alpine sleep 3600
```

Agora execute o script de auditoria apresentado na seção 8.

Identifique:

1. Qual container compartilha o PID namespace do host?
2. Qual container compartilha a rede do host?
3. Algum container está privilegiado?
4. Existem capabilities adicionais?
5. Existem mounts do host?

## Parte B — Kubernetes

Crie um arquivo `pod-risk.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-risk
spec:
  hostPID: true
  hostNetwork: true

  containers:
    - name: app
      image: alpine
      command: ["sleep", "3600"]

      securityContext:
        privileged: false
```

Em um cluster que permita essas configurações:

```bash
kubectl apply -f pod-risk.yaml
```

Inspecione:

```bash
kubectl get pod pod-risk -o yaml
```

Investigue:

```bash
kubectl describe pod pod-risk
```

Se estiver utilizando OpenShift, observe se a SCC aplicada permite a configuração.

Em um cluster com política restritiva, a criação poderá ser rejeitada ou o Pod poderá não ser admitido.

Esse comportamento é justamente o que queremos de uma política de segurança bem configurada.

---

# 14. Checklist de segurança para containers

Este checklist resume os principais pontos da aula.

| Verificação | Recomendação |
|---|---|
| `privileged` | Evitar. |
| `CAP_SYS_ADMIN` | Evitar sempre que possível. |
| `CAP_SYS_MODULE` | Evitar em workloads comuns. |
| `--pid=host` | Evitar. |
| `--network=host` | Evitar quando não for necessário. |
| `--ipc=host` | Evitar. |
| Bind mount `/` | Não utilizar em aplicações comuns. |
| Docker socket | Não expor a aplicações sem necessidade administrativa. |
| `hostPath` | Evitar ou restringir rigorosamente. |
| Root filesystem | Preferir somente leitura. |
| Usuário | Executar como não root. |
| Capabilities | Remover todas e adicionar somente as necessárias. |
| Seccomp | Utilizar `RuntimeDefault` ou perfil restritivo apropriado. |
| AppArmor/SELinux | Manter políticas ativas e adequadas. |
| Recursos | Definir limits de CPU, memória e PIDs quando aplicável. |
| Runtime | Manter atualizado. |

---

# 15. Exercícios de fixação

Tente responder antes de consultar suas anotações.

**1.** Qual é a diferença entre executar um container como `root` e executá-lo com `--privileged`?

**2.** Por que `CAP_SYS_ADMIN` merece atenção especial em ambientes de containers?

**3.** Se um container utiliza `--pid=host`, ele automaticamente consegue enviar SIGKILL para qualquer processo do host? Justifique.

**4.** Qual é a diferença entre conceder `CAP_NET_ADMIN` e utilizar `--network=host`?

**5.** Por que montar `/var/run/docker.sock` dentro de um container pode comprometer o host?

**6.** Um container possui namespaces isolados, capabilities removidas e Seccomp ativo. Isso garante proteção contra vulnerabilidades do kernel? Explique.

**7.** Qual é a diferença entre um container privilegiado e um container que recebeu acesso a um único dispositivo com `--device`?

**8.** No Kubernetes, qual é a finalidade de `hostPID`, `hostNetwork` e `hostIPC`?

**9.** Qual é o papel da SCC `restricted-v2` no OpenShift?

**10.** Por que a combinação de namespaces, capabilities, Seccomp e AppArmor/SELinux é mais eficiente do que depender de apenas um desses mecanismos?

---

## Conclusão da aula

O principal aprendizado desta aula é que o isolamento de containers não depende de um único mecanismo.

Ele é construído por várias camadas:

```text
                 APLICAÇÃO
                     |
             Security Context
                     |
       +-------------+-------------+
       |             |             |
   Capabilities    Seccomp      LSM/MAC
       |             |             |
       +-------------+-------------+
                     |
                 Namespaces
                     |
                   Cgroups
                     |
                OCI Runtime
                     |
                 Linux Kernel
                     |
                    HOST
```

Se uma dessas camadas for configurada inadequadamente, o risco aumenta.

Se várias forem enfraquecidas simultaneamente, podemos criar um ambiente em que o container possui acesso significativo ao host.

**Minha recomendação para o seu estudo:** dê atenção especial à relação entre `CAP_SYS_ADMIN`, user namespaces, mounts, `hostPath` e Docker socket. Esses conceitos aparecem frequentemente em análises de segurança de containers e ajudam a compreender por que determinados workloads são considerados perigosos.

### Próxima aula — 2.8: Runtime Security no Kubernetes

Na próxima aula, podemos aprofundar a aplicação desses conceitos no ambiente Kubernetes:

1. Como o kubelet utiliza o CRI.
2. Como containerd e CRI-O aplicam as políticas de segurança.
3. Como investigar o runtime efetivamente utilizado por um Pod.
4. Como identificar configurações inseguras em workloads.
5. Como funcionam Pod Security Standards e Pod Security Admission.
6. Como aplicar políticas de segurança em namespaces.
7. Relação entre Pod Security Admission e SCC no OpenShift.

Essa será a transição entre a segurança do runtime individual e a segurança do ambiente de orquestração.
