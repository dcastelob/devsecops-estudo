# Módulo 2 — Docker e OCI Runtime Security
## Aula 2.8 — Runtime Security no Kubernetes

Vamos avançar para a integração entre Kubernetes, CRI, containerd, CRI-O e os mecanismos de segurança do cluster.

Esta aula é especialmente importante porque, no Kubernetes, a segurança de um container não é definida exclusivamente pelo runtime. Ela resulta de várias decisões tomadas pelo Kubernetes, pelo kubelet, pelo runtime e pelas políticas de admissão.

A ideia central é compreender este fluxo:

```text
              kubectl
                 |
             API Server
                 |
          Admission Control
                 |
              etcd
                 |
              Scheduler
                 |
          Kubernetes Node
                 |
              kubelet
                 |
                CRI
                 |
          +------+------+
          |             |
       containerd      CRI-O
          |             |
        runc           crun
          |             |
          +------+------+
                 |
             Linux Kernel
                 |
          Namespaces/Cgroups
          Capabilities/Seccomp
          AppArmor/SELinux
```

Essa arquitetura é conceitual: o fluxo real envolve também componentes como o controlador de Pods, o gerenciador de volumes e o CNI.

---

# 1. Como o kubelet utiliza o CRI?

Primeiro, precisamos compreender o CRI.

CRI significa Container Runtime Interface.

É uma interface que define como o Kubernetes se comunica com um runtime de containers.

Antes da adoção do CRI, o Kubernetes possuía integrações mais específicas com runtimes. O CRI permitiu padronizar essa comunicação.

O Kubernetes não precisa conhecer todos os detalhes internos do containerd ou do CRI-O.

Ele precisa conhecer a interface oferecida pelo runtime.

## 1.1 O que é responsabilidade do kubelet?

O kubelet é o agente responsável por administrar os Pods de um determinado nó.

Entre suas responsabilidades estão:

- Receber a especificação dos Pods atribuídos ao nó.
- Garantir que os containers estejam em execução.
- Solicitar criação e remoção de containers.
- Monitorar a saúde dos containers.
- Aplicar configurações de recursos.
- Coordenar volumes.
- Executar probes.
- Gerenciar o ciclo de vida dos Pods.

Imagine que você execute:

```bash
kubectl apply -f pod.yaml
```

O kubelet não recebe diretamente esse comando.

O fluxo é aproximadamente:

1. `kubectl` envia a requisição ao API Server.
2. O API Server valida e armazena o objeto.
3. O Scheduler seleciona um nó.
4. O kubelet daquele nó identifica o Pod atribuído a ele.
5. O kubelet utiliza o CRI para solicitar ao runtime a criação dos containers.
6. O runtime prepara os recursos necessários.
7. O processo do container é iniciado.

## 1.2 O CRI é uma API gRPC

A comunicação entre kubelet e runtime acontece por meio de uma API gRPC.

Existem dois serviços principais:

| Serviço | Responsabilidade |
|---|---|
| RuntimeService | Gerenciamento do ciclo de vida de Pods e containers. |
| ImageService | Gerenciamento de imagens. |

Entre as operações do RuntimeService encontramos:

- `RunPodSandbox`
- `StopPodSandbox`
- `RemovePodSandbox`
- `CreateContainer`
- `StartContainer`
- `StopContainer`
- `RemoveContainer`
- `ContainerStatus`

Um detalhe importante: o Kubernetes utiliza o conceito de Pod Sandbox.

O sandbox representa o ambiente compartilhado do Pod, incluindo recursos como os namespaces que os containers daquele Pod podem compartilhar.

No Linux, normalmente existe um container de infraestrutura, frequentemente chamado de `pause`, que mantém os namespaces do Pod.

Os containers de aplicação são iniciados dentro desse ambiente.

Isso ajuda a entender por que containers de um mesmo Pod compartilham determinados recursos, como o network namespace.

---

# 2. Como containerd e CRI-O aplicam políticas de segurança?

Agora vamos conectar o que estudamos sobre OCI Runtime.

O CRI é a interface de comunicação do Kubernetes com o runtime. O OCI Runtime Specification define como o container deve ser configurado e executado.

São coisas diferentes.

## 2.1 Kubernetes com containerd

Uma arquitetura comum é:

```text
kubelet
   |
   | CRI
   |
containerd
   |
   | OCI Runtime
   |
runc
   |
Linux Kernel
```

O containerd recebe as solicitações do kubelet e gerencia o ciclo de vida dos containers.

Ele pode utilizar diferentes runtimes OCI, dependendo da configuração.

O mais conhecido é o `runc`.

O containerd também possui recursos relacionados a:

- Gerenciamento de imagens.
- Snapshots.
- Execução de containers.
- Gerenciamento de tarefas.
- Plugins.
- Runtime handlers.

## 2.2 Kubernetes com CRI-O

No ecossistema OpenShift, encontramos frequentemente:

```text
kubelet
   |
   | CRI
   |
CRI-O
   |
   | OCI Runtime
   |
crun
   |
Linux Kernel
```

O CRI-O foi desenvolvido especificamente para atender às necessidades de execução de containers no Kubernetes.

Ele implementa a interface CRI e utiliza runtimes OCI para executar os containers.

O `crun` é bastante utilizado nesse ecossistema.

Entretanto, não devemos considerar que containerd sempre utiliza runc ou que CRI-O sempre utiliza crun. Essas são combinações comuns, não regras obrigatórias.

## 2.3 Quem aplica as configurações de segurança?

Vamos utilizar um exemplo:

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

O Kubernetes interpreta esses campos e os transmite ao runtime por meio das estruturas e operações da interface CRI.

O runtime prepara a configuração OCI correspondente e solicita ao kernel a criação do ambiente.

O kernel é quem efetivamente aplica os mecanismos de isolamento e controle.

Por exemplo:

| Configuração Kubernetes | Mecanismo envolvido |
|---|---|
| `runAsUser` | UID/GID do processo |
| `allowPrivilegeEscalation` | `no_new_privs` |
| `capabilities` | Linux Capabilities |
| `seccompProfile` | Seccomp |
| `readOnlyRootFilesystem` | Mount somente leitura |
| `privileged` | Configuração privilegiada do runtime |
| `resources.limits` | Cgroups |

Uma ressalva importante: a aplicação concreta de cada configuração depende do suporte do runtime, do kernel e das políticas do nó.

**Conclusão:** o Kubernetes declara o estado desejado; o runtime prepara o container; o kernel aplica os mecanismos de segurança.

---

# 3. Como investigar o runtime efetivamente utilizado por um Pod?

Agora vamos para a parte prática.

No seu ambiente Debian 13 e nos ambientes Kubernetes/OpenShift, é importante saber identificar o runtime utilizado por cada nó.

## 3.1 Descobrindo o runtime pelo Kubernetes

Execute:

```bash
kubectl get nodes
```

Depois:

```bash
kubectl get nodes \
  -o custom-columns='NODE:.metadata.name,RUNTIME:.status.nodeInfo.containerRuntimeVersion'
```

Exemplo de resultado:

```text
NODE              RUNTIME
worker-01         containerd://2.1.0
worker-02         containerd://2.1.0
worker-03         cri-o://1.31.0
```

Os valores são ilustrativos.

Observe que o campo informa o runtime e sua versão.

Também podemos obter mais informações:

```bash
kubectl describe node worker-01
```

Procure:

```text
Container Runtime Version:
```

## 3.2 Investigando pelo OpenShift

No OpenShift:

```bash
oc get nodes
```

Depois:

```bash
oc get nodes \
  -o custom-columns='NODE:.metadata.name,RUNTIME:.status.nodeInfo.containerRuntimeVersion'
```

Você provavelmente encontrará referências ao CRI-O.

Para investigar um nó específico:

```bash
oc describe node nome-do-no
```

## 3.3 Investigando diretamente no Linux

Se você tiver acesso ao nó, poderá consultar:

```bash
crictl info
```

O `crictl` é uma ferramenta de linha de comando para interagir com runtimes CRI.

Verifique:

```bash
crictl version
```

Liste os Pods:

```bash
crictl pods
```

Liste os containers:

```bash
crictl ps -a
```

Inspecione um container:

```bash
crictl inspect CONTAINER_ID
```

Para consultar as imagens:

```bash
crictl images
```

Atenção: o `crictl` precisa estar configurado para utilizar o endpoint CRI correto. Em distribuições diferentes, o socket pode variar.

Exemplos comuns:

```text
containerd:
unix:///run/containerd/containerd.sock

CRI-O:
unix:///var/run/crio/crio.sock
```

No OpenShift, o socket do CRI-O costuma ser:

```text
unix:///var/run/crio/crio.sock
```

## 3.4 Descobrindo o runtime OCI utilizado

No nó, podemos investigar os processos:

```bash
ps -ef | grep -E 'containerd|crio|containerd-shim|conmon|runc|crun'
```

No caso do CRI-O, o `conmon` participa do monitoramento dos containers.

No caso do containerd, encontramos processos `containerd-shim`.

Também podemos investigar a configuração do runtime, mas os caminhos e formatos variam conforme a distribuição e a versão.

**Importante:** não confunda o runtime CRI com o runtime OCI.

- containerd e CRI-O implementam CRI.
- runc e crun implementam OCI Runtime.

---

# 4. Como identificar configurações inseguras em workloads?

Agora vamos analisar os objetos Kubernetes.

A ideia é procurar configurações que possam enfraquecer o isolamento.

## 4.1 Containers privilegiados

Liste os Pods:

```bash
kubectl get pods -A
```

Para procurar `privileged: true`:

```bash
kubectl get pods -A -o json |
jq -r '
.items[] as $pod |
$pod.spec.containers[]? |
select(.securityContext.privileged == true) |
"\($pod.metadata.namespace)/\($pod.metadata.name) container=\(.name) privileged=true"
'
```

Não esqueça os initContainers:

```bash
kubectl get pods -A -o json |
jq -r '
.items[] as $pod |
(
  $pod.spec.initContainers[]?,
  $pod.spec.containers[]?
) |
select(.securityContext.privileged == true) |
"\($pod.metadata.namespace)/\($pod.metadata.name) container=\(.name) privileged=true"
'
```

## 4.2 Host namespaces

Procure Pods que compartilham namespaces do host:

```bash
kubectl get pods -A -o json |
jq -r '
.items[] |
select(
  .spec.hostPID == true or
  .spec.hostNetwork == true or
  .spec.hostIPC == true
) |
"\(.metadata.namespace)/\(.metadata.name) hostPID=\(.spec.hostPID // false) hostNetwork=\(.spec.hostNetwork // false) hostIPC=\(.spec.hostIPC // false)"
'
```

## 4.3 hostPath

Agora vamos identificar montagens do host:

```bash
kubectl get pods -A -o json |
jq -r '
.items[] as $pod |
$pod.spec.volumes[]? |
select(.hostPath != null) |
"\($pod.metadata.namespace)/\($pod.metadata.name) path=\(.hostPath.path)"
'
```

Uma saída poderia ser:

```text
monitoramento/node-exporter path=/proc
monitoramento/node-exporter path=/sys
monitoramento/node-exporter path=/var/log
```

Esses mounts podem ser legítimos em ferramentas de monitoramento, mas precisam ser avaliados.

O fato de um Pod utilizar `hostPath` não significa automaticamente que ele está vulnerável.

Precisamos analisar o caminho, o modo de montagem, as permissões e o propósito da aplicação.

## 4.4 Capabilities perigosas

Podemos procurar capabilities adicionais:

```bash
kubectl get pods -A -o json |
jq -r '
.items[] as $pod |
(
  $pod.spec.initContainers[]?,
  $pod.spec.containers[]?
) |
select(.securityContext.capabilities.add != null) |
"\($pod.metadata.namespace)/\($pod.metadata.name) container=\(.name) capabilities=\(.securityContext.capabilities.add)"
'
```

Investigue principalmente:

```text
SYS_ADMIN
SYS_MODULE
SYS_PTRACE
NET_ADMIN
DAC_OVERRIDE
```

Não significa que todas essas capabilities sejam sempre proibidas. Algumas aplicações especializadas realmente precisam delas.

O importante é que cada concessão seja justificada.

---

# 5. Pod Security Standards (PSS)

Agora chegamos a um dos mecanismos mais importantes do Kubernetes moderno.

O Kubernetes define três níveis de segurança chamados Pod Security Standards.

São eles:

1. Privileged.
2. Baseline.
3. Restricted.

Esses padrões descrevem conjuntos de requisitos de segurança para Pods.

## 5.1 Privileged

É o nível menos restritivo.

Permite configurações amplas, incluindo vários recursos que os outros níveis restringem.

É destinado a workloads que realmente precisam de privilégios elevados.

Exemplos:

- Alguns agentes de infraestrutura.
- Determinados componentes de rede.
- Ferramentas especializadas de administração dos nós.

Não significa que qualquer configuração seja automaticamente segura.

## 5.2 Baseline

É um nível intermediário.

Procura impedir configurações conhecidamente perigosas, sem restringir excessivamente workloads comuns.

Entre as restrições estão:

- Containers privilegiados.
- Host namespaces.
- Determinadas capabilities.
- Alguns tipos de volumes.
- Configurações perigosas de escalada de privilégios.

É um bom ponto de partida para ambientes que precisam de uma política geral de segurança.

## 5.3 Restricted

É o nível mais restritivo.

Além das restrições do Baseline, exige controles adicionais.

Entre eles:

- Execução como usuário não root.
- Remoção de capabilities.
- Restrições de escalada de privilégios.
- Uso de Seccomp.
- Restrições adicionais de segurança.

É o nível mais adequado como referência para aplicações comuns que podem operar com privilégios mínimos.

### Comparação

| Característica | Privileged | Baseline | Restricted |
|---|---|---|---|
| Container privilegiado | Permitido | Bloqueado | Bloqueado |
| Host namespaces | Permitido | Restringido | Restringido |
| Capabilities | Amplo | Restringido | Mais restritivo |
| Execução não root | Não exigida | Não exigida | Exigida |
| Seccomp | Não exigido | Não exigido | Exigido |
| Escalada de privilégios | Não restringida pelo padrão | Restringida em aspectos específicos | Mais restritiva |

Os Pod Security Standards são especificações de requisitos. Eles não aplicam políticas sozinhos.

Para isso existe o Pod Security Admission.

---

# 6. Pod Security Admission (PSA)

O Pod Security Admission é o mecanismo nativo do Kubernetes responsável por aplicar os Pod Security Standards.

Ele permite configurar políticas por namespace.

Existem três modos principais:

| Modo | Comportamento |
|---|---|
| `enforce` | Rejeita Pods que violam a política. |
| `audit` | Permite a criação, mas registra violações na auditoria. |
| `warn` | Permite a criação, mas apresenta avisos ao usuário. |

Essa separação é muito interessante.

Podemos primeiro observar as violações antes de bloquear workloads.

## 6.1 Aplicando Baseline em um namespace

Vamos criar um namespace:

```bash
kubectl create namespace app-baseline
```

Agora aplique os labels:

```bash
kubectl label namespace app-baseline \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=baseline \
  pod-security.kubernetes.io/warn=baseline
```

A partir desse momento, o Admission Controller verificará os Pods criados nesse namespace.

## 6.2 Testando uma configuração proibida

Crie o arquivo `pod-privileged.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-test
  namespace: app-baseline
spec:
  containers:
    - name: app
      image: alpine
      command: ["sleep", "3600"]
      securityContext:
        privileged: true
```

Tente criar:

```bash
kubectl apply -f pod-privileged.yaml
```

Você deverá receber uma mensagem de rejeição semelhante a:

```text
Error from server (Forbidden):
violates PodSecurity "baseline"
```

O Pod não será admitido.

Observe que a validação ocorre na admissão da API, antes de o Pod ser encaminhado para execução em um nó.

## 6.3 Aplicando Restricted

Crie outro namespace:

```bash
kubectl create namespace app-restricted
```

Configure:

```bash
kubectl label namespace app-restricted \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

Agora tente criar um Pod simples:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-test
  namespace: app-restricted
spec:
  containers:
    - name: app
      image: alpine
      command: ["sleep", "3600"]
```

A criação poderá ser rejeitada porque o Pod não declara os controles exigidos pelo nível Restricted.

Vamos ajustar:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-test
  namespace: app-restricted
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: app
      image: alpine
      command: ["sleep", "3600"]

      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
```

Agora tente:

```bash
kubectl apply -f restricted-test.yaml
```

Esse exemplo atende aos principais requisitos de segurança do nível Restricted.

Entretanto, a imagem precisa conseguir executar como usuário não root. Se a imagem exigir UID 0, o Pod poderá ser admitido e depois falhar durante a execução.

Essa distinção é importante:

**Pod Security Admission valida a configuração declarada. Não garante que a aplicação funcione corretamente.**

---

# 7. Aplicando políticas de segurança em namespaces

Vamos entender melhor o papel dos labels.

Exemplo:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: producao
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

Esse namespace passa a utilizar o nível Restricted nos três modos.

Podemos verificar:

```bash
kubectl get namespace producao --show-labels
```

Também podemos consultar:

```bash
kubectl describe namespace producao
```

## 7.1 Utilizando versões específicas

Os labels também permitem fixar a versão da política.

Exemplo:

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.34
```

Isso evita que mudanças de versão do Kubernetes alterem inesperadamente a política aplicada.

A versão deve ser compatível com a versão do cluster.

## 7.2 Estratégia de implantação

Em um ambiente de produção, recomendo uma implantação gradual:

**Etapa 1 — Audit**

```yaml
pod-security.kubernetes.io/audit: restricted
```

Observe as violações existentes.

**Etapa 2 — Warn**

```yaml
pod-security.kubernetes.io/warn: restricted
```

Os usuários passam a receber avisos ao criar workloads incompatíveis.

**Etapa 3 — Enforce**

```yaml
pod-security.kubernetes.io/enforce: restricted
```

As configurações incompatíveis passam a ser rejeitadas.

Essa estratégia reduz o risco de interromper aplicações existentes.

Um detalhe importante: o PSA não altera retroativamente os Pods que já estão executando. A política é aplicada às operações de admissão.

---

# 8. Relação entre Pod Security Admission e SCC no OpenShift

Agora chegamos a um ponto muito importante para seu trabalho com OpenShift.

O Kubernetes possui Pod Security Admission.

O OpenShift possui Security Context Constraints.

Os dois mecanismos controlam aspectos de segurança dos Pods, mas não são equivalentes.

## 8.1 O que a SCC faz?

A SCC controla quais configurações de segurança um Pod pode utilizar.

Entre os controles estão:

- Execução privilegiada.
- UID e grupos.
- Capabilities.
- Tipos de volumes.
- Host namespaces.
- SELinux.
- Seccomp.
- Escalada de privilégios.

Além disso, a SCC pode fornecer valores padrão para determinadas configurações e validar os valores solicitados.

## 8.2 Diferença conceitual

| Característica | PSA | SCC |
|---|---|---|
| Origem | Kubernetes | OpenShift |
| Finalidade | Aplicar Pod Security Standards | Controlar permissões e contextos de segurança dos Pods |
| Aplicação | Labels no namespace | Autorização para utilizar uma SCC |
| Níveis | Privileged, Baseline, Restricted | Políticas SCC específicas |
| UID | Exige não root no Restricted | Pode definir estratégias de UID |
| SELinux | Não é um mecanismo de gestão de contexto SELinux | Possui controles específicos de SELinux |
| Volumes | Restringe tipos no padrão | Controla tipos de volumes permitidos |
| Capabilities | Restringe capabilities | Controla capabilities permitidas e obrigatórias |

Uma diferença especialmente importante:

No PSA, a política é selecionada por labels do namespace.

Na SCC, o Pod precisa ser admitido utilizando uma SCC para a qual sua identidade esteja autorizada.

## 8.3 Investigando SCC no OpenShift

Liste as SCCs:

```bash
oc get scc
```

Consulte:

```bash
oc describe scc restricted-v2
```

Verifique a SCC aplicada a um Pod:

```bash
oc get pod meu-pod -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
```

Descubra quem pode utilizar a SCC privilegiada:

```bash
oc adm policy who-can use scc privileged
```

## 8.4 Uma particularidade importante do OpenShift

No OpenShift, é comum que aplicações sejam executadas com UID atribuído dinamicamente.

Por isso, um manifesto que define:

```yaml
runAsUser: 10001
```

pode não ser compatível com a estratégia de UID permitida pela SCC do projeto.

Em workloads OpenShift, muitas vezes é melhor permitir que a plataforma atribua o UID apropriado.

Também é importante que as imagens sejam preparadas para funcionar com UID arbitrário.

Isso é uma diferença operacional relevante em relação a muitos ambientes Kubernetes tradicionais.

## 8.5 E o PSA no OpenShift?

O Pod Security Admission faz parte do Kubernetes, mas o OpenShift possui seu próprio modelo de segurança com SCC e outras políticas da plataforma.

Não devemos presumir que uma configuração PSA substitua as SCCs.

Em particular, o PSA não substitui os controles de UID, SELinux e volumes específicos fornecidos pelas SCCs.

A presença e o comportamento dos mecanismos de admissão podem variar conforme a versão e a configuração da distribuição.

Para workloads OpenShift, continue tratando SCC como um mecanismo central de segurança.

---

# 9. Laboratório integrado — Auditoria de Runtime Security

Vamos juntar os conhecimentos da aula.

Crie um namespace:

```bash
kubectl create namespace runtime-security-lab
```

Aplique a política Baseline:

```bash
kubectl label namespace runtime-security-lab \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/audit=baseline
```

Agora crie um arquivo chamado `workloads.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: runtime-security-lab
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
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true

        capabilities:
          drop:
            - ALL
```

Aplique:

```bash
kubectl apply -f workloads.yaml
```

Investigue:

```bash
kubectl get pod -n runtime-security-lab
```

```bash
kubectl describe pod secure-app -n runtime-security-lab
```

Consulte a configuração:

```bash
kubectl get pod secure-app \
  -n runtime-security-lab \
  -o yaml
```

Agora investigue o runtime do nó onde o Pod foi executado:

```bash
kubectl get pod secure-app \
  -n runtime-security-lab \
  -o wide
```

Identifique o nó e consulte:

```bash
kubectl get node NOME_DO_NO \
  -o jsonpath='{.status.nodeInfo.containerRuntimeVersion}'
```

Você terá acompanhado o fluxo:

```text
Pod YAML
   |
API Server
   |
Pod Security Admission
   |
Scheduler
   |
kubelet
   |
CRI
   |
containerd / CRI-O
   |
runc / crun
   |
Linux Kernel
```

---

# 10. Checklist de auditoria para ambientes Kubernetes

Este é um checklist que você pode aproveitar em auditorias de clusters.

| Verificação | Comando ou campo |
|---|---|
| Runtime utilizado | `kubectl get nodes -o wide` |
| Containers privilegiados | `securityContext.privileged` |
| Host PID | `spec.hostPID` |
| Host Network | `spec.hostNetwork` |
| Host IPC | `spec.hostIPC` |
| Mounts do host | `spec.volumes.hostPath` |
| Capabilities | `securityContext.capabilities` |
| Usuário não root | `runAsNonRoot` |
| Escalada de privilégios | `allowPrivilegeEscalation` |
| Root filesystem | `readOnlyRootFilesystem` |
| Seccomp | `seccompProfile` |
| Política de namespace | `pod-security.kubernetes.io/*` |
| SCC OpenShift | `openshift.io/scc` |

Uma boa prática é incorporar essas verificações aos pipelines de CI/CD.

Assim, configurações inseguras podem ser identificadas antes de chegarem ao cluster.

---

# 11. Resumo da aula

Hoje conectamos a segurança dos containers com a arquitetura do Kubernetes.

Os principais conceitos são:

1. O kubelet utiliza CRI para conversar com o runtime.
2. containerd e CRI-O implementam CRI.
3. runc e crun implementam OCI Runtime.
4. O runtime prepara os mecanismos de isolamento e segurança.
5. O kernel aplica efetivamente esses mecanismos.
6. Pod Security Standards definem três níveis de segurança.
7. Pod Security Admission aplica esses padrões por namespace.
8. SCC é um mecanismo próprio do OpenShift que controla permissões e contextos de segurança.
9. A auditoria de workloads deve considerar tanto configurações do Pod quanto políticas do cluster.

## Exercícios de fixação

1. Qual é a diferença entre CRI e OCI Runtime Specification?
2. O kubelet executa diretamente o `runc`?
3. Qual é a finalidade do Pod Sandbox?
4. Como descobrir se um nó utiliza containerd ou CRI-O?
5. Qual a diferença entre `privileged`, `baseline` e `restricted`?
6. Qual é a diferença entre os modos `enforce`, `audit` e `warn`?
7. O PSA consegue substituir integralmente a SCC do OpenShift?
8. Por que um Pod pode ser admitido pelo PSA Restricted e ainda falhar durante sua inicialização?
9. Qual é o benefício de aplicar políticas por namespace?
10. Como você investigaria um Pod que solicita `CAP_SYS_ADMIN` e `hostPath`?

---

## Próxima aula — 2.9: Kubernetes Runtime Security — Admission Policies e Hardening Avançado

Na próxima aula, podemos avançar para:

1. Validação de segurança com Kyverno.
2. OPA Gatekeeper e Rego.
3. Como bloquear containers privilegiados.
4. Como impedir o uso de `hostPath`.
5. Como exigir `runAsNonRoot`, Seccomp e remoção de capabilities.
6. Como criar políticas reutilizáveis para namespaces.
7. Como integrar validações de segurança ao GitLab CI e ArgoCD.

Essa aula será especialmente interessante para conectar Container Security com seu fluxo de DevSecOps e GitOps.
