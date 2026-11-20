# Módulo 2 — Docker e OCI Runtime Security
## Aula 2.6 — Segurança do processo dentro do container: PID 1, sinais e ciclo de vida

Nesta aula vamos estudar o comportamento dos processos dentro de containers Linux, especialmente o papel do processo **PID 1**.

Esse assunto parece simples, mas está por trás de muitos problemas operacionais em Docker, Kubernetes e OpenShift:

- Containers que não encerram mesmo depois de receberem `SIGTERM`.
- Pods que ficam presos durante o encerramento.
- Processos filhos que continuam executando depois que a aplicação principal termina.
- Processos zumbis acumulados.
- Aplicações que ignoram sinais enviados pelo Docker ou Kubernetes.
- Containers que precisam ser encerrados à força com `SIGKILL`.

A ideia central desta aula é compreender que **um container não é uma máquina virtual: ele é um conjunto de processos Linux submetidos a isolamento e controle de recursos**.

Para entender o ciclo de vida do container, precisamos entender primeiro como o kernel administra esses processos.

---

# 1. Como o kernel identifica os processos?

No Linux, cada processo possui um identificador chamado PID (*Process Identifier*).

Quando você executa:

```bash
sleep 3600 &
```

O shell inicia um processo e apresenta seu PID:

```text
[1] 12345
```

Você pode consultar:

```bash
ps -ef
```

Ou:

```bash
ps -o pid,ppid,stat,comm -p 12345
```

Exemplo:

```text
    PID    PPID STAT COMMAND
  12345   12000 S    sleep
```

Os campos representam:

| Campo | Significado |
|---|---|
| PID | Identificador do processo |
| PPID | PID do processo pai |
| STAT | Estado do processo |
| COMMAND | Nome do comando |

Mas existe um detalhe importante quando trabalhamos com containers: **um processo pode possuir PIDs diferentes dependendo do namespace em que é observado**.

## 1.1. PID Namespace

O PID Namespace é responsável por isolar a identificação e a visão dos processos.

Imagine o host:

```text
HOST
PID 1       systemd
 ├── PID 800    dockerd
 ├── PID 950    containerd
 └── PID 1200   processo do container
```

Dentro do container:

```text
CONTAINER
PID 1       aplicação
 ├── PID 2     processo filho
 └── PID 3     outro processo
```

O processo que aparece como PID 1200 no host pode aparecer como PID 1 dentro do container.

Isso ocorre porque o kernel mantém identificadores de processo associados aos diferentes níveis de PID namespaces.

## 1.2. Um processo pode ter vários PIDs

Quando um processo está dentro de um PID namespace aninhado, ele pode ter um PID diferente em cada nível de namespace.

O kernel utiliza estruturas internas para representar esses identificadores.

Uma consequência prática:

- O host consegue identificar o processo pelo PID visível no namespace do host.
- O container enxerga o processo pelo PID correspondente ao seu namespace.
- Processos de namespaces ancestrais não são necessariamente visíveis nos namespaces descendentes.

O isolamento não significa que o processo seja independente do kernel. Todos continuam sendo processos Linux no mesmo kernel.

## 1.3. Laboratório: comparar os PIDs

Crie um container:

```bash
docker run -d --name pid-lab alpine sleep 3600
```

Consulte o PID principal no host:

```bash
docker inspect -f '{{.State.Pid}}' pid-lab
```

Armazene:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' pid-lab)
```

No host:

```bash
ps -o pid,ppid,comm -p "$PID"
```

Agora consulte dentro do container:

```bash
docker exec pid-lab ps -ef
```

O `ps` pode não estar instalado na imagem Alpine. Se necessário:

```bash
docker exec pid-lab sh -c 'apk add --no-cache procps'
```

Você deverá observar que o processo principal aparece como PID 1 dentro do container, mas possui outro PID no host.

Confira diretamente:

```bash
sudo grep -E '^(Name|Pid|PPid|NSpid):' /proc/"$PID"/status
```

Um resultado ilustrativo:

```text
Name:   sleep
Pid:    12345
PPid:   12200
NSpid:  12345 1
```

O campo `NSpid` apresenta os identificadores do processo nos diferentes níveis de PID namespaces, do namespace mais externo ao mais interno.

Essa é uma das maneiras mais interessantes de investigar processos de containers pelo host.

---

# 2. O papel especial do PID 1

Agora chegamos ao ponto central da aula.

Em um PID namespace, o primeiro processo criado recebe o PID 1.

Esse processo possui responsabilidades especiais no Linux.

Ele não é apenas mais um processo: desempenha um papel importante na administração dos processos daquele namespace.

## 2.1. PID 1 como processo inicial

Em uma máquina Linux convencional, normalmente temos:

```text
PID 1: systemd
```

O systemd é responsável por iniciar e administrar serviços, acompanhar processos e realizar outras tarefas de inicialização e gerenciamento do sistema.

Dentro de um container, entretanto, normalmente não executamos um sistema operacional completo.

Podemos ter:

```text
PID 1: nginx
```

Ou:

```text
PID 1: java
```

Ou:

```text
PID 1: python
```

Por exemplo:

```bash
docker run -d --name nginx-lab nginx
```

Consulte:

```bash
docker exec nginx-lab ps -ef
```

O processo principal do Nginx poderá aparecer como PID 1, dependendo do entrypoint da imagem e da forma como o processo foi iniciado.

## 2.2. PID 1 possui comportamento especial com sinais

Existe uma particularidade do kernel Linux que causa muitos problemas em containers.

O processo PID 1 de um PID namespace possui tratamento especial para sinais.

Em particular, sinais para os quais o processo não instalou um handler não necessariamente provocam o comportamento padrão que provocariam em um processo comum.

Por exemplo, em condições normais, `SIGTERM` encerra um processo que não trata esse sinal.

Mas, para o PID 1 de um PID namespace, o kernel aplica proteções especiais: sinais cuja ação padrão seria encerrar o processo não encerram automaticamente esse PID 1 da mesma maneira que encerrariam um processo comum.

Isso significa que uma aplicação que não trata corretamente `SIGTERM` pode não encerrar como você espera.

**Importante:** isso não significa que PID 1 seja imune a sinais. Ele pode receber e tratar sinais, e existem formas de encerrá-lo, inclusive com `SIGKILL`.

O problema aparece quando a aplicação depende do comportamento padrão de sinais e não implementa o tratamento adequado.

## 2.3. Por que isso importa?

Imagine uma aplicação Java:

```bash
java -jar app.jar
```

Se ela for executada diretamente como PID 1 e não tratar corretamente o sinal recebido, o container pode não encerrar de maneira graciosa.

Isso é especialmente problemático em Kubernetes.

Quando um Pod precisa ser removido, o kubelet solicita o encerramento dos containers.

Se a aplicação não responder ao `SIGTERM`, o runtime poderá precisar recorrer ao `SIGKILL` depois do período de tolerância.

O resultado pode ser:

- Requisições interrompidas.
- Transações incompletas.
- Mensagens não processadas.
- Arquivos temporários não removidos.
- Conexões não encerradas corretamente.

Para aplicações que utilizam filas, bancos de dados ou processamento assíncrono, isso pode ter consequências operacionais relevantes.

---

# 3. Sinais Linux: SIGTERM, SIGKILL e outros

Antes de avançarmos, vamos revisar os sinais mais importantes.

Um sinal é uma forma de comunicação assíncrona entre processos, utilizada pelo kernel ou por outros processos.

Você pode consultar a lista:

```bash
kill -l
```

Os principais sinais para o ciclo de vida de containers são:

| Sinal | Número comum | Finalidade |
|---|---:|---|
| `SIGTERM` | 15 | Solicita encerramento |
| `SIGKILL` | 9 | Força encerramento |
| `SIGINT` | 2 | Interrupção |
| `SIGHUP` | 1 | Hangup; frequentemente usado para recarregar configurações |
| `SIGQUIT` | 3 | Solicita encerramento, frequentemente com core dump |
| `SIGCHLD` | 17 | Notifica eventos relacionados a processos filhos |

Os números podem variar em algumas arquiteturas, por isso é melhor utilizar os nomes dos sinais.

## 3.1. SIGTERM

O `SIGTERM` é uma solicitação de encerramento.

Exemplo:

```bash
kill -TERM 12345
```

O processo pode capturar o sinal e executar tarefas antes de sair.

Por exemplo:

1. Interromper o recebimento de novas requisições.
2. Aguardar as requisições em andamento.
3. Finalizar tarefas assíncronas.
4. Fechar conexões.
5. Persistir informações necessárias.
6. Encerrar o processo.

É o comportamento desejável para a maioria das aplicações em containers.

## 3.2. SIGKILL

O `SIGKILL` força o encerramento do processo.

```bash
kill -KILL 12345
```

Diferentemente de `SIGTERM`, o processo não pode capturar, ignorar ou bloquear `SIGKILL`.

Consequentemente, não consegue executar uma rotina de encerramento.

Isso pode deixar operações incompletas.

Por isso, o `SIGKILL` deve ser considerado o último recurso, não o mecanismo normal de encerramento.

## 3.3. Testando os sinais

Execute:

```bash
docker run -d --name signal-lab alpine sleep 3600
```

Consulte o PID:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' signal-lab)
```

Envie `SIGTERM`:

```bash
docker kill --signal=TERM signal-lab
```

Consulte:

```bash
docker inspect signal-lab --format '{{.State.Status}}'
```

Como `sleep` não possui uma rotina de encerramento complexa, ele normalmente termina imediatamente.

Remova o container:

```bash
docker rm signal-lab
```

Agora vamos observar uma aplicação que captura sinais. Para isso, criaremos um pequeno script.

# 4. Criando uma aplicação que trata sinais

Vamos criar um pequeno programa em Shell para observar o comportamento de `SIGTERM`.

No host:

```bash
mkdir -p ~/pid-lab
cd ~/pid-lab
```

Crie `app.sh`:

```bash
#!/bin/sh

echo "Aplicacao iniciada. PID=$$"

shutdown() {
    echo "Recebi SIGTERM. Iniciando encerramento..."

    echo "Finalizando tarefas..."
    sleep 2

    echo "Encerramento concluido."
    exit 0
}

trap shutdown TERM

while true; do
    sleep 5 &
    wait $!
done
```

Torne-o executável:

```bash
chmod +x app.sh
```

Crie o `Dockerfile`:

```dockerfile
FROM alpine:3.22

COPY app.sh /app.sh

RUN chmod +x /app.sh

CMD ["/app.sh"]
```

Construa a imagem:

```bash
docker build -t pid-lab:1.0 .
```

Execute:

```bash
docker run -d --name signal-test pid-lab:1.0
```

Consulte os logs:

```bash
docker logs signal-test
```

Envie `SIGTERM`:

```bash
docker stop --time=10 signal-test
```

Consulte os logs:

```bash
docker logs signal-test
```

Você deverá encontrar as mensagens da rotina de encerramento.

Observe que o Docker aguardou o processo terminar. Como especificamos dez segundos, esse é o tempo máximo de espera antes de recorrer ao `SIGKILL`.

**Uma ressalva:** o exemplo utiliza Shell e serve para demonstrar o tratamento de sinais. Em aplicações reais, é preferível que a aplicação principal trate diretamente os sinais, ou que exista um init apropriado para gerenciar os processos filhos.

---

# 5. ENTRYPOINT, CMD e execução via shell

Agora vamos estudar uma das causas mais comuns de problemas com sinais em containers: a forma como o processo é iniciado.

No Dockerfile, temos duas instruções importantes:

- `ENTRYPOINT`
- `CMD`

Elas podem ser utilizadas em formato *exec* ou *shell*.

## 5.1. Formato exec

Exemplo:

```dockerfile
FROM alpine:3.22

ENTRYPOINT ["sleep"]
CMD ["3600"]
```

Execute:

```bash
docker build -t exec-test .
docker run -d --name exec-test exec-test
```

O processo iniciado será diretamente:

```text
PID 1: sleep 3600
```

O formato exec utiliza uma lista de argumentos e não inicia automaticamente um shell intermediário.

A representação é:

```text
Docker
  |
  v
Runtime
  |
  v
sleep 3600
  |
  v
PID 1
```

Esse formato é geralmente o mais adequado para executar a aplicação principal do container.

## 5.2. Formato shell

Agora considere:

```dockerfile
FROM alpine:3.22

ENTRYPOINT sleep 3600
```

Nesse caso, o Docker utiliza um shell para interpretar o comando.

A representação simplificada é:

```text
PID 1: /bin/sh -c "sleep 3600"
             |
             v
        sleep 3600
```

O processo principal é o shell, não necessariamente o `sleep`.

Isso pode causar problemas porque o sinal enviado ao PID 1 pode não ser encaminhado corretamente ao processo filho.

O comportamento exato depende do shell e da forma como o comando foi construído.

## 5.3. Laboratório: comparar os dois formatos

Crie dois Dockerfiles.

`Dockerfile.exec`:

```dockerfile
FROM alpine:3.22

ENTRYPOINT ["sleep"]
CMD ["3600"]
```

`Dockerfile.shell`:

```dockerfile
FROM alpine:3.22

ENTRYPOINT sleep 3600
```

Construa:

```bash
docker build -f Dockerfile.exec -t pid-exec .
docker build -f Dockerfile.shell -t pid-shell .
```

Execute:

```bash
docker run -d --name test-exec pid-exec
docker run -d --name test-shell pid-shell
```

Investigue:

```bash
docker top test-exec
docker top test-shell
```

Você deverá observar diferenças na árvore de processos.

Agora compare o encerramento:

```bash
docker stop --time=5 test-exec
docker stop --time=5 test-shell
```

O exemplo é simplificado, mas demonstra por que o formato exec é preferível para iniciar aplicações diretamente.

## 5.4. O uso de `exec` em scripts de inicialização

Às vezes precisamos executar tarefas antes de iniciar a aplicação:

- Gerar arquivos de configuração.
- Validar variáveis de ambiente.
- Ajustar permissões.
- Preparar diretórios.

Nesse caso, podemos utilizar um script de inicialização.

Exemplo:

```sh
#!/bin/sh

echo "Preparando aplicacao..."

# Preparacao inicial
mkdir -p /var/lib/app

echo "Iniciando aplicacao..."

exec java -jar /app/app.jar
```

O detalhe importante é:

```bash
exec java -jar /app/app.jar
```

O comando `exec` substitui o processo Shell pelo processo Java.

Assim, o Java passa a ser o PID 1, em vez de ficar como filho de um shell intermediário.

Compare:

```text
Sem exec:

PID 1: /bin/sh
         |
         v
       java
```

```text
Com exec:

PID 1: java
```

Essa é uma prática muito importante em Dockerfiles que utilizam scripts de inicialização.

**Regra prática:** se um script Shell prepara o ambiente e depois inicia a aplicação principal, utilize `exec` para substituir o shell pelo processo da aplicação.

---

# 6. Processos zumbis e processos órfãos

Agora vamos estudar dois conceitos importantes na administração de processos Linux.

## 6.1. Processo zumbi (Zombie)

Um processo zumbi é um processo que já terminou sua execução, mas ainda possui uma entrada na tabela de processos porque seu processo pai não coletou seu status de término.

O kernel mantém essas informações até que o pai execute uma operação como:

- `wait()`
- `waitpid()`
- `waitid()`

Representação:

```text
PID 1: aplicacao
  |
  +-- PID 10: processo filho
              |
              +-- terminou
              |
              +-- continua como ZOMBIE
```

O processo já não está executando, mas sua entrada ainda existe.

No `ps`, normalmente aparece com estado `Z`.

Exemplo:

```bash
ps -eo pid,ppid,stat,comm
```

Uma saída ilustrativa:

```text
    PID    PPID STAT COMMAND
      1       0 S    app
     10       1 Z    worker
```

O estado `Z` indica zombie.

### Por que isso acontece?

Imagine que uma aplicação crie processos filhos:

```text
Aplicacao
  |
  +-- worker-1
  +-- worker-2
  +-- worker-3
```

Os workers terminam, mas a aplicação não chama `wait()` para coletar o status deles.

Esses processos permanecem como zumbis.

Um zumbi não consome CPU como um processo em execução, mas ocupa uma entrada na tabela de processos.

Se muitos zumbis se acumularem, podem contribuir para o esgotamento dos recursos relacionados à criação de processos.

Isso é particularmente preocupante em containers com limites de PIDs.

## 6.2. Processo órfão (Orphan)

Um processo órfão é um processo cujo pai terminou antes dele.

Exemplo:

```text
PID 1: aplicacao
  |
  +-- PID 10: worker
```

Se a aplicação terminar enquanto o worker continuar executando:

```text
PID 10: worker
```

O worker fica sem seu pai original.

O kernel então o reparenta para um processo apropriado, normalmente o init do PID namespace ou um subreaper registrado naquele contexto.

No namespace do container, o processo PID 1 frequentemente assume essa responsabilidade.

O processo que recebe o órfão precisa coletar seu status quando ele terminar.

## 6.3. PID 1 precisa coletar processos filhos

Um init tradicional, como o systemd, possui mecanismos para gerenciar processos e coletar processos filhos.

Mas uma aplicação comum pode não implementar corretamente essa responsabilidade.

Imagine uma aplicação que cria vários processos filhos e não os administra adequadamente.

O resultado pode ser:

- Processos zumbis.
- Processos órfãos.
- Processos que continuam executando após a aplicação principal encerrar.
- Containers que não finalizam como esperado.

É por isso que containers que executam aplicações que criam subprocessos podem se beneficiar de um init mínimo.

---

# 7. Tini e dumb-init

Existem ferramentas pequenas que podem atuar como processo PID 1 do container e ajudar na administração dos processos filhos.

Duas bastante conhecidas são:

- `tini`
- `dumb-init`

Elas não são sistemas operacionais completos. São init systems minimalistas voltados para containers.

## 7.1. Tini

O Tini é um init mínimo que pode atuar como PID 1.

Suas responsabilidades incluem:

- Receber sinais.
- Encaminhar sinais ao processo filho.
- Coletar processos filhos terminados.
- Auxiliar no encerramento correto do container.

Representação:

```text
PID 1: tini
  |
  v
PID 2: aplicacao
  |
  +-- worker
  +-- worker
```

Quando o container recebe um sinal de encerramento, o Tini participa do encaminhamento para o processo principal.

O Tini também atua como subreaper, ajudando a coletar processos órfãos que forem reparentados para ele.

### Utilizando Tini no Docker

O Docker oferece suporte a um init mínimo através da opção:

```bash
docker run --init
```

Exemplo:

```bash
docker run -d \
  --name tini-lab \
  --init \
  alpine \
  sleep 3600
```

Consulte:

```bash
docker top tini-lab
```

Você deverá observar um processo init antes da aplicação.

A implementação utilizada depende da distribuição do Docker e de sua instalação.

O Docker Engine utiliza um binário `docker-init`, baseado no Tini.

## 7.2. Dumb-init

O `dumb-init` é outra implementação de init mínimo.

Ele pode atuar como PID 1 e:

- Encaminhar sinais.
- Gerenciar processos filhos.
- Recolher processos zumbis.
- Administrar grupos de processos.

É bastante utilizado em imagens de containers que precisam de um init leve.

Exemplo conceitual de Dockerfile:

```dockerfile
FROM debian:13-slim

RUN apt-get update \
    && apt-get install -y dumb-init \
    && rm -rf /var/lib/apt/lists/*

ENTRYPOINT ["/usr/bin/dumb-init", "--"]

CMD ["/app/minha-aplicacao"]
```

A estrutura será:

```text
PID 1: dumb-init
  |
  v
PID 2: minha-aplicacao
```

## 7.3. Tini versus dumb-init

| Característica | Tini | dumb-init |
|---|---|---|
| Init mínimo | Sim | Sim |
| Encaminhamento de sinais | Sim | Sim |
| Recolhimento de zumbis | Sim | Sim |
| Uso em containers | Sim | Sim |
| Integração com Docker `--init` | Sim, via docker-init | Não é o binário utilizado pelo `--init` |

### Quando utilizar?

Se sua aplicação:

- Cria subprocessos.
- Executa scripts auxiliares.
- Utiliza workers.
- Inicia processos em background.
- Não administra corretamente os processos filhos.

Um init mínimo pode ser uma boa escolha.

Mas atenção: Tini e dumb-init não corrigem todos os problemas de encerramento.

Se a aplicação principal não trata `SIGTERM`, é necessário corrigir seu comportamento ou implementar uma rotina adequada de encerramento.

O init não substitui a lógica de shutdown da aplicação.

---

# 8. Como o runtime encerra um container?

Agora vamos conectar os conceitos ao funcionamento do Docker e do runtime OCI.

Quando você executa:

```bash
docker stop meu-container
```

O Docker não simplesmente apaga o processo.

Existe um fluxo de encerramento.

De forma simplificada:

```text
          docker stop
               |
               v
          Docker Engine
               |
               v
            containerd
               |
               v
          OCI Runtime
               |
               v
          Processo PID 1
               |
               v
             SIGTERM
               |
               v
       Aplicacao encerra
               |
               v
         Container parado
```

A implementação interna possui mais detalhes, mas esse é o fluxo conceitual.

## 8.1. Primeiro: SIGTERM

O Docker solicita o encerramento do container.

Por padrão, o Docker envia `SIGTERM` ao processo principal.

A aplicação tem a oportunidade de:

- Finalizar requisições.
- Encerrar conexões.
- Persistir dados.
- Finalizar subprocessos.
- Liberar recursos.

## 8.2. Segundo: período de espera

O Docker aguarda o tempo de tolerância configurado.

Por padrão, o `docker stop` utiliza um período de espera de dez segundos, salvo configuração diferente.

Você pode especificar:

```bash
docker stop --time=30 meu-container
```

Nesse caso, o Docker aguarda até trinta segundos.

## 8.3. Terceiro: SIGKILL

Se o processo não terminar dentro do prazo, o Docker envia `SIGKILL`.

```text
docker stop
     |
     v
  SIGTERM
     |
     v
 Aguarda 30s
     |
     +---- Processo terminou?
     |           |
     |          Sim
     |           |
     |           v
     |       Container parado
     |
     Não
     |
     v
  SIGKILL
     |
     v
 Processo encerrado
```

O `SIGKILL` não permite que a aplicação execute uma rotina de encerramento.

Por isso, o tempo de espera precisa ser compatível com o comportamento da aplicação.

## 8.4. STOPSIGNAL

Existe outra instrução interessante no Dockerfile:

```dockerfile
STOPSIGNAL SIGTERM
```

Ela define o sinal que deverá ser utilizado para solicitar o encerramento do container.

Por padrão, o sinal é `SIGTERM`.

Você poderia definir:

```dockerfile
STOPSIGNAL SIGQUIT
```

Isso pode ser útil para aplicações que possuem um sinal específico de encerramento gracioso.

A configuração também pode ser consultada pela inspeção da imagem e do container.

Exemplo:

```bash
docker inspect meu-container \
  --format '{{.Config.StopSignal}}'
```

**Importante:** não altere o `STOPSIGNAL` sem conhecer o comportamento da aplicação. Escolher um sinal inadequado pode provocar encerramento abrupto ou comportamento inesperado.

---

# 9. Ciclo de vida do processo no Kubernetes

Agora vamos estudar o comportamento no Kubernetes.

O Kubernetes adiciona uma camada de gerenciamento sobre o runtime.

Quando um Pod precisa ser encerrado, o kubelet coordena o processo.

O principal parâmetro envolvido é:

```yaml
terminationGracePeriodSeconds: 30
```

Esse valor determina o período de tolerância para o encerramento do Pod.

Por padrão, são trinta segundos.

## 9.1. Fluxo de encerramento de um Pod

Imagine um Pod que será removido:

```bash
kubectl delete pod minha-aplicacao
```

O fluxo simplificado é:

```text
          kubectl delete pod
                  |
                  v
             API Server
                  |
                  v
             Pod marcado
             para remoção
                  |
                  v
                kubelet
                  |
                  v
            Executa preStop
                  |
                  v
             Envia SIGTERM
                  |
                  v
              PID 1
                  |
                  v
          Aplicacao encerra
                  |
                  v
          Container finalizado
```

O processo possui algumas particularidades importantes.

1. O Pod entra em processo de término e recebe um `deletionTimestamp`.
2. O kubelet inicia o encerramento dos containers.
3. O hook `preStop`, se configurado, é executado.
4. Depois do hook, o runtime solicita o encerramento do processo principal usando o sinal configurado.
5. Se o processo não terminar dentro do período de tolerância, ele será encerrado à força.

A remoção do Pod também envolve a atualização de informações de endpoints e o encerramento do tráfego direcionado ao Pod. Essas operações podem ocorrer de maneira concorrente com o processo de término; não se deve presumir que todo o tráfego será interrompido antes da execução do `preStop`.

## 9.2. terminationGracePeriodSeconds

Exemplo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-shutdown
spec:
  terminationGracePeriodSeconds: 45
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "sleep 3600"]
```

Nesse exemplo, o Kubernetes disponibiliza até 45 segundos para o encerramento do Pod.

Se a aplicação não terminar dentro desse prazo, o runtime poderá enviar `SIGKILL`.

**Atenção:** o período de execução do `preStop` faz parte do período de tolerância. Ele não é um tempo adicional independente.

Se o `preStop` consumir 20 segundos em um Pod com tolerância de 30 segundos, restarão aproximadamente 10 segundos para a aplicação encerrar graciosamente.

## 9.3. preStop

O `preStop` é um hook executado antes de o kubelet solicitar o encerramento do processo principal.

Ele pode ser utilizado para:

- Informar a aplicação de que será encerrada.
- Remover o Pod de um mecanismo interno de balanceamento.
- Aguardar a interrupção de novas requisições.
- Executar uma operação de preparação para shutdown.

Exemplo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prestop-lab
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "trap 'echo SIGTERM recebido; exit 0' TERM; while true; do sleep 2; done"]
      lifecycle:
        preStop:
          exec:
            command:
              - sh
              - -c
              - echo "Executando preStop"; sleep 5
```

A sequência será:

```text
1. Kubernetes inicia o encerramento
2. Executa preStop
3. Aguarda o preStop terminar
4. Solicita encerramento com SIGTERM
5. Aplicação executa seu tratamento de sinal
6. Container termina
```

O exemplo utiliza `sleep` para demonstrar a sequência. Em aplicações reais, o hook deve executar uma operação efetivamente necessária, evitando atrasos artificiais.

### Um cuidado importante

Não utilize `preStop` como substituto do tratamento de `SIGTERM`.

O hook é uma preparação para o encerramento. A aplicação ainda deve ser capaz de tratar o sinal e encerrar corretamente.

## 9.4. lifecycle

O Kubernetes permite configurar hooks de ciclo de vida.

Os principais são:

| Hook | Quando é executado |
|---|---|
| `postStart` | Após a criação do container |
| `preStop` | Antes da solicitação de encerramento |

Exemplo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-lab
spec:
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "trap 'echo Encerrando; exit 0' TERM; while true; do sleep 2; done"]
      lifecycle:
        postStart:
          exec:
            command:
              - sh
              - -c
              - echo "Container iniciado"
        preStop:
          exec:
            command:
              - sh
              - -c
              - echo "Preparando encerramento"
```

O `postStart` não é uma garantia de que o processo principal ainda não começou a executar. Ele é executado de forma assíncrona em relação ao entrypoint do container, e a ordem entre os dois não é garantida.

Por isso, não utilize `postStart` como mecanismo de sincronização obrigatório para inicializar a aplicação.

## 9.5. O que acontece quando o grace period termina?

Imagine:

```yaml
terminationGracePeriodSeconds: 10
```

A aplicação recebe a solicitação de encerramento, mas precisa de 25 segundos para concluir suas operações.

O resultado será aproximadamente:

```text
0s   -> Início do encerramento
      -> preStop, se configurado
      -> SIGTERM
10s  -> Prazo de tolerância esgotado
      -> SIGKILL
```

A aplicação não terá tempo suficiente para terminar normalmente.

Isso pode causar perda de trabalho em andamento.

Portanto, o `terminationGracePeriodSeconds` deve ser definido de acordo com o tempo real de encerramento da aplicação.

---

# 10. Laboratório Kubernetes: observar o encerramento de um Pod

Vamos criar um Pod que registra o recebimento de `SIGTERM`.

Salve como `shutdown-lab.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shutdown-lab
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: app
      image: alpine:3.22
      command:
        - /bin/sh
        - -c
        - |
          echo "Aplicacao iniciada. PID=$$"

          shutdown() {
            echo "SIGTERM recebido"
            echo "Executando rotina de encerramento..."
            sleep 5
            echo "Encerramento concluido"
            exit 0
          }

          trap shutdown TERM

          while true; do
            sleep 2 &
            wait $!
          done
      lifecycle:
        preStop:
          exec:
            command:
              - /bin/sh
              - -c
              - |
                echo "Executando preStop"
                sleep 3
```

Aplicar:

```bash
kubectl apply -f shutdown-lab.yaml
```

Acompanhar os logs:

```bash
kubectl logs -f shutdown-lab
```

Em outro terminal, remover:

```bash
kubectl delete pod shutdown-lab
```

Você deverá observar mensagens semelhantes a:

```text
Aplicacao iniciada. PID=1
Executando preStop
SIGTERM recebido
Executando rotina de encerramento...
Encerramento concluido
```

A ordem é o ponto importante.

O `preStop` é executado antes da solicitação de encerramento ao processo principal.

O tratamento de `SIGTERM` acontece depois.

O Pod deve terminar antes de esgotar os 30 segundos.

---

# 11. Problemas comuns em aplicações Java e Spring Boot

Vamos relacionar tudo isso a uma aplicação que executa Java dentro de um container.

Considere:

```dockerfile
FROM eclipse-temurin:21-jre

COPY app.jar /app.jar

ENTRYPOINT ["java", "-jar", "/app.jar"]
```

O processo Java será o processo principal do container.

Isso é positivo porque evita um shell intermediário.

Entretanto, a aplicação também precisa tratar corretamente o encerramento.

## 11.1. O que deve acontecer quando Java recebe SIGTERM?

Uma aplicação Spring Boot deve ter oportunidade de:

1. Parar de aceitar novas requisições.
2. Aguardar requisições em andamento.
3. Finalizar tarefas assíncronas.
4. Encerrar consumidores de filas.
5. Fechar conexões com bancos de dados.
6. Encerrar pools de threads.
7. Liberar recursos.
8. Finalizar o processo.

O Spring Boot oferece suporte a graceful shutdown.

Em versões modernas, você pode configurar:

```yaml
server:
  shutdown: graceful
```

E definir o tempo máximo de espera:

```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

O período de encerramento da aplicação precisa caber dentro do `terminationGracePeriodSeconds`, considerando também o tempo consumido pelo `preStop`.

Exemplo:

```yaml
spec:
  terminationGracePeriodSeconds: 45
  containers:
    - name: app
      image: minha-aplicacao:1.0
```

Uma configuração coerente poderia ser:

```text
terminationGracePeriodSeconds: 45s
preStop:                     5s
Spring graceful shutdown:   30s
Margem operacional:         10s
```

Esses tempos são ilustrativos. O ideal é medir quanto a aplicação realmente demora para encerrar em condições normais e durante períodos de maior carga.

**Importante:** graceful shutdown não garante que todas as operações serão concluídas. A aplicação precisa implementar corretamente o tratamento de interrupções e persistir seu estado de maneira segura.

---

# 12. Diagnóstico: por que meu container não encerra?

Quando um container fica preso durante o encerramento, recomendo investigar nesta ordem.

### 12.1. Identificar o processo principal

Docker:

```bash
docker inspect meu-container \
  --format 'PID={{.State.Pid}}'
```

Consultar o comando:

```bash
docker inspect meu-container \
  --format 'Path={{.Path}} Args={{json .Args}}'
```

### 12.2. Verificar a árvore de processos

```bash
docker top meu-container
```

Ou diretamente no host:

```bash
pstree -ap "$(docker inspect -f '{{.State.Pid}}' meu-container)"
```

### 12.3. Verificar o tratamento de sinais

Investigue se a aplicação possui tratamento de `SIGTERM`.

No caso de Java/Spring Boot, verifique os logs de encerramento e a configuração de graceful shutdown.

### 12.4. Verificar o tempo de encerramento

Docker:

```bash
docker inspect meu-container \
  --format 'StopSignal={{.Config.StopSignal}}'
```

Kubernetes:

```bash
kubectl get pod meu-pod -o yaml
```

Procure:

```yaml
terminationGracePeriodSeconds:
```

E:

```yaml
lifecycle:
```

### 12.5. Verificar os eventos do Pod

```bash
kubectl describe pod meu-pod
```

Consultar logs:

```bash
kubectl logs meu-pod --previous
```

O parâmetro `--previous` é útil quando o container já foi reiniciado e você precisa consultar os logs da execução anterior.

### 12.6. Verificar processos zumbis

No host:

```bash
ps -eo pid,ppid,stat,comm | awk '$3 ~ /^Z/'
```

No container, se houver ferramentas disponíveis:

```bash
ps -eo pid,ppid,stat,comm
```

Procure processos com estado `Z`.

---

# 13. Recomendações de segurança e operação

Vamos consolidar as práticas que considero mais importantes:

| Recomendação | Motivo |
|---|---|
| Utilize `ENTRYPOINT` no formato exec | Evita shell intermediário desnecessário |
| Utilize `exec` em scripts de inicialização | Faz a aplicação assumir o PID 1 |
| Implemente tratamento de SIGTERM | Permite encerramento gracioso |
| Utilize Tini ou dumb-init quando necessário | Ajuda a gerenciar processos filhos |
| Evite executar processos em background sem gerenciamento | Reduz problemas com órfãos e zumbis |
| Configure `terminationGracePeriodSeconds` adequadamente | Evita encerramento prematuro |
| Utilize `preStop` somente quando necessário | Evita atrasos desnecessários |
| Monitore o tempo de encerramento | Permite dimensionar o período de tolerância |

Uma observação: nem toda aplicação precisa obrigatoriamente de Tini ou dumb-init. Uma aplicação que executa diretamente como PID 1, trata os sinais e não cria processos filhos que precisem ser administrados pode funcionar corretamente sem um init adicional.

---

# 14. Resumo da aula

| Conceito | O que você deve guardar |
|---|---|
| PID Namespace | Isola a identificação e a visão dos processos |
| PID 1 | Primeiro processo do namespace, com responsabilidades especiais |
| SIGTERM | Solicitação de encerramento |
| SIGKILL | Encerramento forçado |
| ENTRYPOINT exec | Inicia diretamente o processo |
| ENTRYPOINT shell | Pode introduzir um shell intermediário |
| `exec` | Substitui o processo atual pelo novo programa |
| Zombie | Processo terminado cujo status ainda não foi coletado |
| Orphan | Processo cujo pai terminou antes dele |
| Tini | Init mínimo para containers |
| dumb-init | Outra implementação de init mínimo |
| `docker stop` | Solicita encerramento e aguarda antes de forçar |
| `terminationGracePeriodSeconds` | Tempo de tolerância para encerrar o Pod |
| `preStop` | Hook executado antes da solicitação de encerramento |
| `lifecycle` | Configuração de hooks do container |

---

# 15. Exercícios de fixação

Tente responder sem consultar o material.

**1. PID Namespace**

Por que o mesmo processo pode ter PID 12345 no host e PID 1 dentro do container?

**2. PID 1**

Por que o PID 1 de um PID namespace possui tratamento especial para sinais?

**3. SIGTERM e SIGKILL**

Qual é a diferença entre os dois? Por que devemos preferir `SIGTERM`?

**4. Dockerfile**

Qual é a diferença entre:

```dockerfile
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

e:

```dockerfile
ENTRYPOINT java -jar /app.jar
```

**5. Shell script**

Por que é recomendável utilizar `exec` antes de iniciar a aplicação principal?

**6. Zombie**

Por que um processo zumbi continua aparecendo no `ps`, mesmo depois de terminar sua execução?

**7. Tini**

Qual é a finalidade de executar um init mínimo como PID 1?

**8. Kubernetes**

Qual é a relação entre `preStop`, `SIGTERM` e `terminationGracePeriodSeconds`?

**9. Graceful shutdown**

Se uma aplicação Spring Boot precisa de 25 segundos para encerrar e seu `preStop` consome 5 segundos, um `terminationGracePeriodSeconds: 30` oferece margem suficiente?

**10. Diagnóstico**

Um Pod permanece em encerramento e termina com `SIGKILL`. Quais pontos você investigaria primeiro?

---

## Próxima aula — 2.7: Segurança do Runtime — Privileged Containers, Host Namespaces e Container Escape

Na próxima aula vamos avançar para um assunto central em Container Security:

1. O que significa executar um container privilegiado.
2. Diferenças entre `--privileged`, `--cap-add` e `--cap-drop`.
3. Riscos de utilizar `--pid=host`, `--network=host` e `--ipc=host`.
4. Acesso a dispositivos do host.
5. Como uma configuração inadequada pode permitir acesso ao host.
6. O que é container escape.
7. Relação entre namespaces, capabilities, Seccomp e AppArmor/SELinux na prevenção de escapes.
8. Como investigar configurações perigosas no Docker e Kubernetes.
9. Como o OpenShift utiliza SCC para restringir workloads privilegiados.

Essa aula vai conectar praticamente tudo que estudamos no Módulo 1: namespaces, cgroups, capabilities, Seccomp e MAC. Será uma das aulas mais importantes do nosso módulo de runtime security.
