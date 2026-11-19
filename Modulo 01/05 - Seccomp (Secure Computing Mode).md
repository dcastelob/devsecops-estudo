Exatamente! O próximo assunto do nosso **Módulo 1 — Linux Security** será:

# Aula 1.5 — Seccomp (Secure Computing Mode)

Depois de estudar Namespaces, Cgroups e Linux Capabilities, vamos conhecer outro mecanismo fundamental de segurança do kernel Linux: **Seccomp**.

A pergunta central será:

**Se um processo possui permissões para executar determinadas operações, como podemos impedir que ele sequer invoque determinadas chamadas de sistema do kernel?**

Vamos estudar:

1. **Como funciona a comunicação entre aplicações e o kernel Linux:** entender o papel das System Calls.
2. **O que é Seccomp:** como o kernel filtra chamadas de sistema.
3. **Modos de operação:** Strict Mode e Filter Mode.
4. **Seccomp BPF:** como filtros são construídos utilizando Berkeley Packet Filter.
5. **Ações dos filtros:** `ALLOW`, `ERRNO`, `KILL`, `TRAP`, `LOG` e `NOTIFY`.
6. **Identificação de Syscalls:** utilizar `strace` para descobrir quais chamadas uma aplicação executa.
7. **Laboratório com seccomp:** criar filtros e restringir chamadas de sistema de processos Linux.
8. **Seccomp no Docker:** entender o perfil padrão e utilizar `--security-opt seccomp`.
9. **Seccomp no Kubernetes:** configurar `seccompProfile` no `securityContext`.
10. **Seccomp no OpenShift:** compreender sua utilização em conjunto com SCC e Security Context.

### Como os mecanismos se complementam

| Mecanismo | O que controla? |
|---|---|
| Namespaces | O que o processo consegue enxergar |
| Cgroups | Quanto de recursos pode consumir |
| Capabilities | Quais operações privilegiadas pode realizar |
| Seccomp | Quais chamadas de sistema pode executar |
| AppArmor/SELinux | Quais recursos pode acessar |

### Laboratório que vamos construir

Quero utilizar novamente nosso Debian 13 e trabalhar com aplicações reais, incluindo o Firefox que estamos isolando.

A ideia será construir progressivamente um ambiente assim:

```text
DEBIAN 13 HOST
│
└── Aplicação isolada
    │
    ├── Namespaces
    │   ├── PID
    │   ├── NET
    │   ├── MNT
    │   └── USER
    │
    ├── Cgroups
    │   ├── MemoryMax
    │   ├── CPUQuota
    │   └── TasksMax
    │
    ├── Capabilities
    │   └── Privilégios mínimos
    │
    └── Seccomp
        └── Syscalls permitidas
```

**Um detalhe importante:** Seccomp não substitui Capabilities. Mesmo uma aplicação sem privilégios administrativos pode executar chamadas de sistema perigosas ou desnecessárias. O Seccomp permite reduzir essa superfície de ataque.

Esse será nosso próximo passo antes de avançarmos para AppArmor e SELinux.

# Módulo 1.5 — Seccomp: filtragem de chamadas de sistema

Vamos seguir a estrutura proposta. Esta aula é especialmente importante para segurança de containers porque o Seccomp permite reduzir as operações que um processo pode solicitar ao kernel Linux, mesmo quando o processo já está isolado por namespaces e limitado por capabilities.

A ideia central é:

> Um processo pode ter acesso ao kernel, mas não precisa necessariamente ter permissão para executar todas as chamadas de sistema que o kernel disponibiliza.

Vamos construir o conhecimento gradualmente, começando pela comunicação entre uma aplicação e o kernel.

---

# 1. Como funciona a comunicação entre aplicações e o kernel Linux

## 1.1 Aplicações não acessam diretamente o hardware

No Linux, as aplicações executam normalmente em **User Space**, enquanto o kernel opera em **Kernel Space**.

Essa separação existe para proteger o sistema operacional.

Imagine uma aplicação que precisa ler um arquivo:

```bash
cat /etc/hostname
```

O `cat` não acessa diretamente o disco rígido. Ele solicita ao kernel que realize essa operação.

Simplificando, temos:

```text
+--------------------------------------+
|              USER SPACE              |
|                                      |
|  Aplicação: cat                      |
|                                      |
|  Biblioteca C (glibc)                |
|                                      |
+------------------+-------------------+
                   |
                   | System Call
                   |
+------------------v-------------------+
|             KERNEL SPACE             |
|                                      |
|  VFS (Virtual File System)            |
|  Filesystem                          |
|  Driver de armazenamento             |
|                                      |
+------------------+-------------------+
                   |
                   v
               Hardware
```

A aplicação solicita uma operação, o kernel verifica se ela está autorizada e executa o trabalho.

## 1.2 O que são System Calls?

**System Calls**, ou chamadas de sistema, são a interface pela qual um processo solicita serviços ao kernel.

Alguns exemplos:

| System Call | Finalidade |
|---|---|
| `read()` | Ler dados de um arquivo ou descritor |
| `write()` | Escrever dados |
| `openat()` | Abrir arquivos |
| `close()` | Fechar descritores |
| `fork()` | Criar um processo |
| `execve()` | Executar um programa |
| `clone()` | Criar processos ou threads |
| `mmap()` | Mapear memória |
| `socket()` | Criar um socket |
| `connect()` | Estabelecer conexão |
| `mount()` | Montar filesystem |
| `ptrace()` | Inspecionar ou controlar outro processo |
| `reboot()` | Solicitar reinicialização do sistema |

Você pode consultar as chamadas disponíveis no seu Debian:

```bash
man 2 syscalls
```

Para consultar uma chamada específica:

```bash
man 2 openat
man 2 execve
man 2 clone
```

O número `2` indica a seção do manual dedicada às chamadas de sistema.

## 1.3 Exemplo prático: observando System Calls

Vamos utilizar o `strace`, que permite observar as chamadas de sistema realizadas por um processo.

Instale-o, caso ainda não esteja disponível:

```bash
sudo apt install strace
```

Execute:

```bash
strace ls /tmp
```

Você verá uma quantidade considerável de chamadas de sistema.

Para filtrar apenas algumas categorias:

```bash
strace -e trace=file ls /tmp
```

```bash
strace -e trace=process ls /tmp
```

```bash
strace -e trace=memory ls /tmp
```

Também podemos observar as chamadas de sistema de um processo em execução:

```bash
strace -p PID
```

Ou acompanhar a criação de processos filhos:

```bash
strace -f comando
```

A opção `-f` é particularmente importante porque muitas aplicações criam processos ou threads durante sua execução.

### Uma observação importante

Nem toda função chamada por uma aplicação representa uma System Call.

Por exemplo, uma aplicação pode chamar uma função da biblioteca C:

```c
printf("Olá mundo\n");
```

O `printf()` não é uma System Call. Ele é uma função de biblioteca que, em algum momento, pode utilizar `write()` para enviar os dados ao terminal.

Essa distinção será importante quando começarmos a construir filtros.

---

# 2. O que é Seccomp?

**Seccomp** significa *Secure Computing Mode*.

É um mecanismo de segurança implementado no kernel Linux que permite restringir as chamadas de sistema que um processo pode executar.

Imagine uma aplicação que precisa apenas:

- Ler arquivos.
- Escrever arquivos.
- Alocar memória.
- Criar threads.
- Realizar conexões de rede.

Ela provavelmente não precisa executar:

- `reboot()`.
- `mount()`.
- `kexec_load()`.
- `init_module()`.
- `delete_module()`.

O Seccomp permite estabelecer uma política para controlar essas chamadas.

Podemos representar assim:

```text
             Aplicação
                 |
                 |
           System Call
                 |
                 v
          +--------------+
          |   SECCOMP    |
          |              |
          | Avalia a     |
          | chamada      |
          +------+-------+
                 |
          +------+------+
          |             |
          v             v
       Permitida     Bloqueada
          |             |
          v             v
       Kernel        Retorno de
       executa       erro ou outra
                     ação definida
```

O Seccomp atua no momento em que o processo tenta realizar uma System Call.

Isso significa que não estamos simplesmente restringindo arquivos, diretórios ou permissões de usuários. Estamos controlando a interface de comunicação entre o processo e o kernel.

## 2.1 Seccomp não substitui outros mecanismos

É importante entender a responsabilidade de cada mecanismo:

| Mecanismo | O que controla |
|---|---|
| Namespaces | O que o processo consegue enxergar |
| Cgroups | Quanto de recursos pode consumir |
| Capabilities | Quais privilégios especiais possui |
| Seccomp | Quais System Calls pode executar |
| AppArmor/SELinux | Quais operações sobre recursos são permitidas |
| DAC | Permissões tradicionais de arquivos e diretórios |

Por exemplo:

Um processo pode ter a System Call `mount()` permitida pelo Seccomp, mas não possuir `CAP_SYS_ADMIN`.

Nesse caso, a chamada poderá ser rejeitada pelo mecanismo de controle de privilégios do kernel.

**Seccomp permitir uma chamada não significa que ela necessariamente será executada com sucesso.**

---

# 3. Modos de operação

O Seccomp possui dois modos principais:

- Strict Mode.
- Filter Mode.

## 3.1 Strict Mode

É o modo mais restritivo e antigo.

Nesse modo, o processo fica limitado a um conjunto muito pequeno de chamadas de sistema, tradicionalmente:

- `read()`
- `write()`
- `_exit()`
- `sigreturn()`

É um ambiente extremamente limitado.

Não é adequado para aplicações modernas em geral, pois praticamente qualquer aplicação real precisa de outras chamadas de sistema.

Podemos consultar o estado do Seccomp de um processo:

```bash
grep Seccomp /proc/self/status
```

Os valores possíveis são:

| Valor | Significado |
|---|---|
| 0 | Seccomp desabilitado |
| 1 | Strict Mode |
| 2 | Filter Mode |

## 3.2 Filter Mode

É o modo mais interessante para segurança de containers.

Permite criar filtros personalizados utilizando programas BPF.

Podemos definir regras como:

- Permitir `read()`.
- Permitir `write()`.
- Bloquear `mount()`.
- Bloquear `ptrace()`.
- Bloquear `reboot()`.

O processo pode continuar funcionando normalmente, desde que suas operações necessárias estejam permitidas.

É esse modo que normalmente encontramos em ambientes Docker e Kubernetes.

---

# 4. Seccomp BPF

Aqui chegamos a um conceito importante.

O Seccomp Filter utiliza uma tecnologia baseada em **BPF — Berkeley Packet Filter**.

Talvez você já tenha encontrado BPF em outros contextos, como:

- tcpdump;
- filtros de captura de pacotes;
- eBPF;
- observabilidade do kernel;
- ferramentas como Falco.

Mas precisamos distinguir duas coisas:

**Seccomp utiliza classic BPF (cBPF)** para avaliar chamadas de sistema. Isso não é a mesma coisa que executar programas eBPF modernos.

## 4.1 Como funciona?

Quando um processo tenta executar uma System Call, o kernel disponibiliza informações para avaliação do filtro.

Entre elas:

- Número da System Call.
- Arquitetura.
- Argumentos da chamada.
- Outras informações relacionadas à execução.

O filtro avalia essas informações e retorna uma ação.

Exemplo conceitual:

```text
Processo solicita:

mount("/dev/sdb1", "/mnt", "ext4", ...)

                |
                v

          Filtro Seccomp

                |
                v

       Número da System Call
              mount

                |
                v

       Regra encontrada?

                |
                v

       SCMP_ACT_ERRNO

                |
                v

       Operação retorna EPERM
```

O filtro não precisa analisar o conteúdo do arquivo, nem o destino do mount em si. Ele pode simplesmente bloquear a chamada `mount()`.

Isso é uma diferença importante em relação a mecanismos como AppArmor e SELinux, que possuem modelos de controle de acesso mais ricos sobre recursos.

## 4.2 Os filtros são aplicados ao processo

O Seccomp é associado às tarefas do Linux. Um filtro pode ser herdado por processos criados posteriormente.

Os filtros são cumulativos: adicionar outro filtro não permite remover as restrições anteriores.

Em aplicações multithread, existe ainda uma particularidade: filtros podem ser aplicados individualmente às threads. Para sincronizar a aplicação em todas elas, existe o mecanismo `SECCOMP_FILTER_FLAG_TSYNC`.

Os runtimes de containers cuidam desses detalhes ao configurar o ambiente de execução.

---

# 5. Ações dos filtros

O filtro Seccomp pode retornar diferentes ações.

As principais são:

| Ação | Comportamento |
|---|---|
| `ALLOW` | Permite a System Call |
| `ERRNO` | Bloqueia e retorna um erro |
| `KILL` | Encerra a tarefa ou processo |
| `TRAP` | Gera um sinal SIGSYS |
| `LOG` | Registra a tentativa e permite a execução |
| `NOTIFY` | Notifica um processo supervisor para decidir o que fazer |

Uma observação: o comportamento exato de `KILL` possui variantes, como `KILL_PROCESS` e `KILL_THREAD`. Além disso, a disponibilidade de algumas ações depende da versão do kernel.

## 5.1 ALLOW

Permite executar a System Call.

## 5.2 ERRNO

Bloqueia a chamada e retorna um código de erro.

Por exemplo:

```text
mount() = -1 EPERM (Operation not permitted)
```

Essa é uma ação bastante útil porque o processo recebe um erro em vez de ser encerrado.

## 5.3 KILL

Encerra a execução da tarefa ou do processo, conforme a variante utilizada.

É uma ação bastante restritiva.

## 5.4 TRAP

Gera um sinal `SIGSYS`.

A aplicação pode, dependendo da configuração, tratar esse sinal.

## 5.5 LOG

Registra a tentativa de execução da chamada, mas permite que ela prossiga.

É importante não confundir `LOG` com uma ação de bloqueio.

## 5.6 NOTIFY

Permite que um processo supervisor receba uma notificação sobre a tentativa de execução da System Call.

O supervisor pode analisar a solicitação e responder se ela deve prosseguir ou ser bloqueada.

Esse mecanismo é conhecido como **Seccomp User Notification**.

Ele permite implementar políticas mais elaboradas, mas exige uma aplicação supervisora preparada para esse trabalho.

---

# 6. Identificando System Calls com strace

Agora vamos começar a preparar nosso laboratório.

O objetivo é descobrir quais chamadas de sistema uma aplicação executa.

Vamos utilizar o próprio `ls`.

```bash
strace ls /tmp
```

Para obter um resumo das chamadas:

```bash
strace -c ls /tmp
```

Exemplo de saída:

```text
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- --------
...
read
write
openat
close
mmap
mprotect
munmap
brk
...
```

Para observar apenas chamadas relacionadas a arquivos:

```bash
strace -e trace=%file ls /tmp
```

Para observar chamadas de rede:

```bash
strace -e trace=%network curl https://example.com
```

Para observar chamadas de criação de processos:

```bash
strace -e trace=%process bash -c 'sleep 2 & wait'
```

Para descobrir chamadas utilizadas por uma aplicação real, utilize:

```bash
strace -f -c -o /tmp/syscalls.txt comando
```

O resultado será gravado em `/tmp/syscalls.txt`.

**Atenção:** o `strace` mostra somente as chamadas executadas durante o período observado. Ele não garante que você descobriu todas as chamadas que a aplicação poderá precisar em outras situações.

---

# 7. Laboratório: criando um filtro Seccomp

Vamos começar com um laboratório simples, sem Docker.

Primeiro, instale as ferramentas:

```bash
sudo apt install gcc libseccomp-dev seccomp
```

O pacote `libseccomp-dev` disponibiliza a biblioteca que facilita a criação de filtros Seccomp.

O pacote `seccomp` disponibiliza ferramentas adicionais, dependendo da versão e dos pacotes disponíveis na distribuição.

## 7.1 Criando uma aplicação de teste

Crie um arquivo:

```bash
vim seccomp-lab.c
```

Conteúdo:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/utsname.h>

int main(void)
{
    struct utsname info;

    printf("Aplicacao iniciada\n");

    if (uname(&info) == 0) {
        printf("Sistema: %s\n", info.sysname);
        printf("Hostname: %s\n", info.nodename);
    } else {
        perror("uname");
    }

    printf("PID: %d\n", getpid());

    return 0;
}
```

Compile:

```bash
gcc seccomp-lab.c -o seccomp-lab
```

Execute:

```bash
./seccomp-lab
```

A aplicação deverá apresentar informações do sistema.

Agora vamos observar suas chamadas:

```bash
strace ./seccomp-lab
```

Você deverá encontrar uma chamada semelhante a:

```text
uname({sysname="Linux", ...}) = 0
```

A chamada `uname()` é utilizada para obter informações sobre o sistema.

## 7.2 Criando um filtro que bloqueia uname()

Agora vamos utilizar a biblioteca libseccomp para bloquear especificamente essa chamada.

Crie:

```bash
vim seccomp-filter.c
```

Conteúdo:

```c
#include <stdio.h>
#include <unistd.h>
#include <seccomp.h>

int main(void)
{
    scmp_filter_ctx ctx;

    ctx = seccomp_init(SCMP_ACT_ALLOW);

    if (ctx == NULL) {
        perror("seccomp_init");
        return 1;
    }

    if (seccomp_rule_add(
            ctx,
            SCMP_ACT_ERRNO(1),
            SCMP_SYS(uname),
            0) < 0) {
        perror("seccomp_rule_add");
        seccomp_release(ctx);
        return 1;
    }

    if (seccomp_load(ctx) < 0) {
        perror("seccomp_load");
        seccomp_release(ctx);
        return 1;
    }

    seccomp_release(ctx);

    printf("Filtro Seccomp carregado\n");

    struct utsname info;

    if (uname(&info) == 0) {
        printf("Sistema: %s\n", info.sysname);
    } else {
        perror("uname");
    }

    printf("PID: %d\n", getpid());

    return 0;
}
```

Compile:

```bash
gcc seccomp-filter.c -o seccomp-filter -lseccomp
```

Execute:

```bash
./seccomp-filter
```

Resultado esperado:

```text
Filtro Seccomp carregado
uname: Operation not permitted
PID: 12345
```

Observe o que aconteceu:

1. O processo iniciou normalmente.
2. O filtro foi carregado.
3. A chamada `uname()` foi bloqueada.
4. A chamada `getpid()` continuou funcionando.

Esse é o princípio fundamental do Seccomp.

**Importante:** o filtro foi instalado pelo próprio processo e permanece associado a ele. A partir desse momento, ele não pode simplesmente remover a restrição para voltar a executar `uname()`.

---

# 8. Seccomp no Docker

Agora vamos relacionar o que aprendemos com containers.

O Docker utiliza filtros Seccomp para restringir chamadas de sistema dos containers.

Por padrão, ele aplica um perfil de segurança que bloqueia chamadas consideradas perigosas ou desnecessárias.

O perfil padrão funciona essencialmente como uma política de allowlist: chamadas permitidas são explicitamente autorizadas, enquanto as demais são rejeitadas, com exceções condicionais conforme as capabilities e a arquitetura.

Isso significa que um container Docker já nasce com uma camada de proteção Seccomp, salvo quando essa proteção é desabilitada ou substituída.

## 8.1 Verificando o Seccomp de um container

Execute:

```bash
docker run --rm alpine sh -c 'grep Seccomp /proc/self/status'
```

Resultado esperado:

```text
Seccomp:        2
Seccomp_filters: 1
```

O valor `2` indica Filter Mode.

## 8.2 Criando um perfil personalizado

Vamos criar um filtro que permite todas as chamadas, exceto `uname()`.

Crie:

```bash
vim deny-uname.json
```

Conteúdo:

```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "names": [
        "uname"
      ],
      "action": "SCMP_ACT_ERRNO",
      "errnoRet": 1
    }
  ]
}
```

Execute normalmente:

```bash
docker run --rm alpine uname -a
```

Agora execute com nosso perfil:

```bash
docker run --rm \
  --security-opt seccomp=deny-uname.json \
  alpine uname -a
```

Resultado esperado:

```text
uname: can't get system name: Operation not permitted
```

Observe que o container continua funcionando, mas a System Call `uname()` foi bloqueada.

Esse perfil é didático, não um perfil de produção: ele permite todas as chamadas por padrão e bloqueia apenas uma. Um perfil de produção deve partir de uma política mais restritiva e considerar as necessidades reais da aplicação.

## 8.3 Desabilitando o Seccomp no Docker

Também existe a possibilidade de desabilitar o filtro:

```bash
docker run --rm \
  --security-opt seccomp=unconfined \
  alpine sh
```

Isso remove a proteção Seccomp fornecida pelo Docker para aquele container.

Não recomendo essa configuração em ambientes de produção.

---

# 9. Seccomp no Kubernetes

No Kubernetes, o Seccomp é configurado por meio do `securityContext`.

Temos duas possibilidades principais:

- `RuntimeDefault`
- `Localhost`

## 9.1 RuntimeDefault

Utiliza o perfil padrão disponibilizado pelo runtime de containers.

Exemplo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-lab
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: alpine
      image: alpine:latest
      command:
        - sleep
        - "3600"
```

Nesse exemplo, todos os containers do Pod herdam a configuração definida no `securityContext` do Pod, salvo quando um container possui configuração própria.

## 9.2 Localhost

Permite utilizar um perfil personalizado disponibilizado no nó onde o container será executado.

Exemplo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-localhost
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/deny-uname.json

  containers:
    - name: alpine
      image: alpine:latest
      command:
        - sleep
        - "3600"
```

Nesse caso, o arquivo precisa estar disponível no caminho de perfis Seccomp configurado no runtime do nó.

Um detalhe importante: o arquivo precisa existir em todos os nós nos quais o Pod poderá ser agendado.

## 9.3 Verificando dentro do Pod

```bash
kubectl exec seccomp-lab -- cat /proc/1/status | grep Seccomp
```

Esperamos encontrar:

```text
Seccomp:        2
```

Também podemos verificar:

```bash
kubectl describe pod seccomp-lab
```

---

# 10. Seccomp no OpenShift

No OpenShift, o Seccomp também pode ser configurado por meio do `securityContext`.

Exemplo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-openshift
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: alpine
      image: registry.access.redhat.com/ubi9/ubi-minimal
      command:
        - sleep
        - "3600"
      securityContext:
        allowPrivilegeEscalation: false
```

O OpenShift utiliza o runtime de containers e os mecanismos de segurança do Linux para aplicar essas restrições.

## 10.1 Relação entre Seccomp e SCC

Aqui existe uma distinção importante.

A **SCC (Security Context Constraints)** controla quais configurações de segurança um Pod pode solicitar.

Entre suas responsabilidades estão:

- Usuários e grupos permitidos.
- Capabilities.
- Execução privilegiada.
- Volumes permitidos.
- Estratégias de SELinux.
- Perfis Seccomp permitidos.

O campo `seccompProfiles`, quando configurado na SCC, restringe quais perfis Seccomp podem ser utilizados pelos Pods associados àquela SCC.

Podemos consultar as SCCs:

```bash
oc get scc
```

Consultar uma SCC específica:

```bash
oc describe scc restricted-v2
```

E consultar o Pod:

```bash
oc get pod seccomp-openshift -o yaml
```

**Atenção:** não confunda SCC com Seccomp. A SCC é um mecanismo de autorização e validação das configurações de segurança do Pod; o Seccomp é um mecanismo do kernel que efetivamente filtra as chamadas de sistema.

---

# 11. Resumo: onde o Seccomp se encaixa?

Vamos relacionar os mecanismos estudados até aqui:

| Mecanismo | Pergunta que responde |
|---|---|
| Namespaces | O que o processo consegue enxergar? |
| Cgroups | Quanto de recursos o processo pode consumir? |
| Capabilities | Quais privilégios especiais o processo possui? |
| Seccomp | Quais chamadas de sistema o processo pode executar? |
| AppArmor/SELinux | Quais operações o processo pode realizar sobre os recursos? |

Uma configuração de container mais restritiva combina esses mecanismos.

Por exemplo:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

Isso não representa uma política completa de segurança, mas já estabelece uma boa base de hardening.

---

# 12. Exercícios de fixação

Sugiro realizar estes exercícios no Debian 13 antes de avançarmos.

**Exercício 1 — System Calls**

Execute:

```bash
strace -c ls /
```

Identifique as cinco chamadas mais frequentes.

**Exercício 2 — Seccomp**

Execute:

```bash
grep Seccomp /proc/self/status
```

Compare o resultado com:

```bash
docker run --rm alpine cat /proc/1/status
```

Explique a diferença.

**Exercício 3 — Filtro personalizado**

Modifique nosso programa C para bloquear também:

```c
SCMP_SYS(getpid)
```

Observe o comportamento da aplicação.

**Exercício 4 — Docker**

Crie um perfil que bloqueie `uname()` e outro que bloqueie `getpid()`.

Compare os resultados.

**Exercício 5 — Kubernetes**

Crie um Pod com:

```yaml
seccompProfile:
  type: RuntimeDefault
```

Verifique o estado do Seccomp em `/proc/1/status`.

---

## Próxima aula

Depois de Seccomp, nossa próxima etapa será:

**Aula 1.6 — AppArmor e SELinux: Mandatory Access Control (MAC).**

Nela vamos entender como o Linux controla o acesso dos processos a arquivos, diretórios, sockets e outros recursos, complementando o que aprendemos com Capabilities e Seccomp.
