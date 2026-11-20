# Módulo 2 — Docker e OCI Runtime Security
## Aula 2.9 — Kubernetes Runtime Security: Admission Policies e Hardening Avançado

Nesta aula vamos avançar da configuração individual de segurança dos Pods para a aplicação de políticas centralizadas no cluster.

Até aqui, aprendemos a configurar:

- Namespaces.
- Cgroups.
- Capabilities.
- Seccomp.
- AppArmor e SELinux.
- SecurityContext.
- Pod Security Standards.
- Pod Security Admission.
- SCC do OpenShift.

Agora vamos responder a uma pergunta importante:

**Como garantir que todos os desenvolvedores sigam essas configurações, sem depender de cada um lembrar de configurar corretamente seus manifestos?**

A resposta está nas Admission Policies.

Vamos estudar duas ferramentas importantes:

- **Kyverno:** políticas de segurança declarativas utilizando YAML.
- **OPA Gatekeeper:** políticas utilizando Open Policy Agent e a linguagem Rego.

Ao final, construiremos políticas de segurança e integraremos sua validação ao GitLab CI e ao ArgoCD.

---

# 1. O que são Admission Controllers?

Primeiro, precisamos compreender onde essas ferramentas atuam.

Quando você executa:

```bash
kubectl apply -f deployment.yaml
```

A requisição passa pelo Kubernetes API Server.

Antes de o objeto ser persistido no etcd, ele passa por diferentes etapas de validação e admissão.

Uma representação simplificada:

```text
              kubectl
                 |
                 v
             API Server
                 |
                 v
          Authentication
                 |
                 v
          Authorization
                 |
                 v
        Admission Controllers
                 |
          +------+------+
          |             |
       Mutating      Validating
       Admission     Admission
          |             |
          +------+------+
                 |
                 v
                etcd
                 |
                 v
              Scheduler
                 |
                 v
               kubelet
                 |
                 v
             Container
```

Existem dois grandes tipos de Admission Controllers:

### 1.1 Mutating Admission

Pode modificar o objeto recebido antes de sua persistência.

Por exemplo, imagine que um desenvolvedor envie:

```yaml
containers:
  - name: app
    image: nginx
```

Uma política mutating poderia acrescentar automaticamente:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

O objeto armazenado passaria a conter essas configurações.

### 1.2 Validating Admission

Valida o objeto e decide se ele pode ser admitido.

Imagine que alguém envie:

```yaml
securityContext:
  privileged: true
```

Uma política validating pode rejeitar a requisição.

A diferença fundamental:

| Tipo | Comportamento |
|---|---|
| Mutating | Modifica o objeto. |
| Validating | Aceita ou rejeita o objeto. |

Uma política de segurança pode utilizar os dois mecanismos, mas devemos ter cuidado para não transformar uma configuração insegura em algo aparentemente seguro apenas adicionando campos.

Por exemplo, adicionar `runAsNonRoot: true` não garante que uma imagem seja compatível com execução não root.

---

# 2. O que é Kyverno?

Kyverno é um mecanismo de políticas desenvolvido especificamente para Kubernetes.

Sua principal característica é permitir a criação de políticas utilizando YAML.

Isso é interessante porque não precisamos aprender uma linguagem de programação específica para começar a construir políticas.

O Kyverno permite:

- Validar recursos.
- Modificar recursos.
- Gerar recursos.
- Verificar imagens.
- Aplicar políticas de segurança.
- Verificar configurações de workloads existentes.

A documentação oficial está disponível em:

[Documentação oficial do Kyverno](https://kyverno.io/docs/?utm_source=chatgpt.com)

## 2.1 Arquitetura simplificada

```text
             Kubernetes API
                    |
                    v
             Kyverno Webhook
                    |
             +------+------+
             |             |
          Mutating      Validating
          Policies      Policies
             |             |
             +------+------+
                    |
                    v
              Decisão final
                    |
             +------+------+
             |             |
           Admit         Deny
```

O Kyverno utiliza webhooks de admissão do Kubernetes.

Quando um recurso é criado ou atualizado, o API Server pode enviar a requisição ao Kyverno para avaliação.

O Kyverno avalia as políticas aplicáveis e retorna o resultado.

## 2.2 Instalação em laboratório

Vamos utilizar Helm para instalar o Kyverno.

Primeiro, adicione o repositório:

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
```

Crie o namespace:

```bash
kubectl create namespace kyverno
```

Instale:

```bash
helm install kyverno kyverno/kyverno \
  --namespace kyverno
```

Verifique:

```bash
kubectl get pods -n kyverno
```

Aguarde os componentes ficarem prontos:

```bash
kubectl wait \
  --for=condition=Ready \
  pods \
  --all \
  -n kyverno \
  --timeout=180s
```

Consulte os deployments:

```bash
kubectl get deployments -n kyverno
```

**Observação:** em produção, fixe uma versão do chart e utilize valores revisados, em vez de instalar automaticamente a versão mais recente. Os nomes dos componentes e valores de configuração podem mudar entre versões.

---

# 3. Criando a primeira política Kyverno

Vamos criar uma política para impedir containers privilegiados.

Crie:

```bash
vim block-privileged.yaml
```

Conteúdo:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: block-privileged
spec:
  validationFailureAction: Enforce

  rules:
    - name: deny-privileged-containers

      match:
        any:
          - resources:
              kinds:
                - Pod

      validate:
        message: "Containers privilegiados não são permitidos."

        pattern:
          spec:
            containers:
              - securityContext:
                  privileged: "false"
```

Vamos entender cada parte.

### `ClusterPolicy`

Define uma política que pode atuar em todo o cluster.

Diferentemente de uma política associada a um namespace específico, uma ClusterPolicy pode abranger vários namespaces, conforme os critérios de seleção.

### `validationFailureAction`

Define o comportamento quando a validação falha.

```yaml
validationFailureAction: Enforce
```

Significa que a requisição será rejeitada quando violar a política.

O modo alternativo:

```yaml
validationFailureAction: Audit
```

Permite a requisição e registra a violação.

### `match`

Define quais recursos serão avaliados.

```yaml
match:
  any:
    - resources:
        kinds:
          - Pod
```

Nesse exemplo, a política será aplicada a Pods.

### `validate`

Define a regra de validação.

Se o container não atender ao padrão, a requisição será rejeitada.

## 3.1 Aplicando a política

```bash
kubectl apply -f block-privileged.yaml
```

Verifique:

```bash
kubectl get clusterpolicy
```

Agora crie um Pod privilegiado:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-test
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
kubectl apply -f privileged-test.yaml
```

A expectativa é receber uma rejeição do Kyverno.

**Uma ressalva importante sobre o exemplo:** o padrão apresentado é didático. Para uma política de produção, devemos validar também initContainers e ephemeralContainers e tratar corretamente a ausência do campo `privileged`. Uma regra baseada em `pattern` deve ser cuidadosamente testada com campos omitidos. Mais adiante construiremos uma versão mais completa usando `foreach` e condições explícitas.

---

# 4. Bloqueando `hostPath`

Agora vamos criar uma política para impedir montagens diretas do filesystem do host.

Lembre-se da aula 2.7.

O uso inadequado de `hostPath` pode permitir acesso a:

- `/etc`
- `/var/run`
- `/proc`
- `/sys`
- Diretórios de dados do host.
- Sockets administrativos.

Crie:

```bash
vim block-hostpath.yaml
```

Conteúdo:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: block-hostpath
spec:
  validationFailureAction: Enforce

  rules:
    - name: deny-hostpath-volumes

      match:
        any:
          - resources:
              kinds:
                - Pod

      validate:
        message: "Volumes hostPath não são permitidos."

        pattern:
          spec:
            =(volumes):
              - X(hostPath): "null"
```

A intenção dessa política é impedir volumes do tipo `hostPath`.

A notação do Kyverno possui operadores especiais:

| Operador | Significado |
|---|---|
| `=` | Campo opcional. |
| `X` | Campo deve estar ausente ou nulo. |
| `!` | Negação em determinados contextos de padrão. |
| `*` | Corresponde a qualquer valor. |

A sintaxe de padrões do Kyverno é própria e merece atenção.

Para uma regra de produção, prefira uma expressão de validação explícita ou uma regra `foreach` que percorra todos os volumes, evitando ambiguidades com listas e campos opcionais.

Uma alternativa de validação com `foreach` é:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-hostpath
spec:
  validationFailureAction: Enforce
  background: true

  rules:
    - name: deny-hostpath-volumes
      match:
        any:
          - resources:
              kinds:
                - Pod

      validate:
        message: "O uso de hostPath não é permitido."

        foreach:
          - list: "request.object.spec.volumes || `[]`"
            deny:
              conditions:
                any:
                  - key: "{{ element.hostPath || '' }}"
                    operator: NotEquals
                    value: ""
```

Essa segunda versão deixa explícito que cada volume será avaliado.

Aplique:

```bash
kubectl apply -f block-hostpath.yaml
```

Agora tente criar:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-test
spec:
  containers:
    - name: app
      image: alpine
      command: ["sleep", "3600"]
      volumeMounts:
        - name: host
          mountPath: /host

  volumes:
    - name: host
      hostPath:
        path: /var/log
```

A requisição deverá ser rejeitada.

---

# 5. Exigindo runAsNonRoot, Seccomp e remoção de capabilities

Agora vamos construir uma política mais interessante.

Nosso objetivo será exigir:

1. Execução como usuário não root.
2. `allowPrivilegeEscalation: false`.
3. Seccomp `RuntimeDefault` ou `Localhost`.
4. Remoção de todas as capabilities.

Crie:

```bash
vim require-security-context.yaml
```

Conteúdo:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-security-context
spec:
  validationFailureAction: Enforce

  rules:
    - name: require-non-root

      match:
        any:
          - resources:
              kinds:
                - Pod

      validate:
        message: "O Pod deve executar como usuário não root."

        pattern:
          spec:
            securityContext:
              runAsNonRoot: true

    - name: require-seccomp

      match:
        any:
          - resources:
              kinds:
                - Pod

      validate:
        message: "O Pod deve utilizar Seccomp RuntimeDefault ou Localhost."

        anyPattern:
          - spec:
              securityContext:
                seccompProfile:
                  type: RuntimeDefault

          - spec:
              securityContext:
                seccompProfile:
                  type: Localhost

    - name: require-container-hardening

      match:
        any:
          - resources:
              kinds:
                - Pod

      validate:
        message: "Os containers devem desabilitar privilege escalation e remover todas as capabilities."

        pattern:
          spec:
            containers:
              - securityContext:
                  allowPrivilegeEscalation: false
                  capabilities:
                    drop:
                      - ALL
```

Essa política apresenta uma limitação importante: ela não trata adequadamente todos os containers do Pod, incluindo initContainers e ephemeralContainers, e não garante que cada container tenha seu próprio `runAsNonRoot`.

Para uma política completa, precisamos iterar por todos os tipos de containers e validar cada um.

Isso é uma boa oportunidade para aprender a escrever políticas mais robustas.

**Conceito importante:** uma política de segurança não deve apenas verificar se um campo aparece em algum lugar do manifesto. Ela precisa garantir que todos os containers estejam protegidos.

---

# 6. OPA Gatekeeper e Rego

Agora vamos estudar a segunda ferramenta.

Open Policy Agent é um mecanismo geral de políticas que pode ser utilizado em diferentes sistemas.

O Gatekeeper é a integração do OPA com Kubernetes.

Documentação:

- [Documentação do Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs/?utm_source=chatgpt.com)
- [Documentação do OPA](https://www.openpolicyagent.org/docs/?utm_source=chatgpt.com)

A principal diferença em relação ao Kyverno é que o Gatekeeper utiliza a linguagem Rego para expressar políticas.

## 6.1 Arquitetura

```text
              Kubernetes API
                     |
                     v
              Admission Webhook
                     |
                     v
                Gatekeeper
                     |
                     v
                   OPA
                     |
                     v
                  Rego
                     |
              +------+------+
              |             |
            Allow          Deny
```

O Gatekeeper utiliza dois recursos principais:

- ConstraintTemplate.
- Constraint.

### ConstraintTemplate

Define a lógica da política.

### Constraint

Define onde e como a política será aplicada.

Essa separação é bastante interessante porque permite reutilizar a mesma lógica em diferentes namespaces ou conjuntos de recursos.

---

# 7. Criando uma política Gatekeeper para bloquear containers privilegiados

Primeiro, instale o Gatekeeper.

Em um laboratório Kubernetes convencional, uma forma de instalação é utilizar o manifesto oficial da versão desejada:

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

Em produção, não utilize diretamente `master`. Fixe uma versão estável e revise os manifests antes da instalação.

Verifique:

```bash
kubectl get pods -n gatekeeper-system
```

## 7.1 Criando o ConstraintTemplate

Arquivo:

```bash
vim template-block-privileged.yaml
```

Conteúdo:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sblockprivileged
spec:
  crd:
    spec:
      names:
        kind: K8sBlockPrivileged

  targets:
    - target: admission.k8s.gatekeeper.sh

      rego: |
        package k8sblockprivileged

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true

          msg := sprintf(
            "Container %v não pode ser privilegiado",
            [container.name]
          )
        }
```

Vamos entender o Rego.

```rego
container := input.review.object.spec.containers[_]
```

Percorre os containers do Pod.

```rego
container.securityContext.privileged == true
```

Verifica se o container está privilegiado.

```rego
violation[{"msg": msg}]
```

Define uma violação da política.

O Gatekeeper utiliza essas violações para decidir se o recurso deve ser rejeitado.

Essa primeira versão verifica apenas containers comuns. Uma política completa também deve percorrer initContainers e ephemeralContainers.

## 7.2 Criando o Constraint

Arquivo:

```bash
vim constraint-block-privileged.yaml
```

Conteúdo:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockPrivileged
metadata:
  name: block-privileged
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

Aplique:

```bash
kubectl apply -f template-block-privileged.yaml
kubectl apply -f constraint-block-privileged.yaml
```

Agora tente criar um Pod privilegiado.

O Gatekeeper deverá rejeitar a solicitação.

---

# 8. Comparação entre Kyverno e Gatekeeper

As duas ferramentas resolvem problemas semelhantes, mas possuem filosofias diferentes.

| Característica | Kyverno | Gatekeeper |
|---|---|---|
| Linguagem de políticas | YAML | Rego |
| Base | Kubernetes | OPA |
| Validação | Sim | Sim |
| Mutação | Sim | Não é seu mecanismo principal |
| Geração de recursos | Sim | Não é seu mecanismo principal |
| Auditoria de recursos existentes | Sim | Sim |
| Curva de aprendizado inicial | Geralmente menor | Requer aprender Rego |
| Reutilização | Policies e ClusterPolicies | ConstraintTemplates e Constraints |

Minha sugestão para seu aprendizado:

Comece com Kyverno para entender a construção de políticas de segurança em Kubernetes.

Depois avance para Rego e Gatekeeper.

O conhecimento de Rego também pode ser aproveitado em outros cenários de Policy as Code.

---

# 9. Criando políticas reutilizáveis por namespace

Agora vamos trabalhar com uma situação realista.

Imagine que você possua três namespaces:

```text
desenvolvimento
homologacao
producao
```

Você deseja aplicar políticas diferentes.

| Namespace | Política |
|---|---|
| desenvolvimento | Audit |
| homologacao | Enforce Baseline |
| producao | Enforce Restricted |

Podemos fazer isso com labels no namespace e políticas Kyverno.

Exemplo:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: desenvolvimento
  labels:
    ambiente: desenvolvimento
    politica-seguranca: audit
```

Outro:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: producao
  labels:
    ambiente: producao
    politica-seguranca: restricted
```

No Kyverno, podemos selecionar namespaces utilizando `namespaceSelector`.

Exemplo:

```yaml
match:
  any:
    - resources:
        kinds:
          - Pod
        namespaceSelector:
          matchLabels:
            ambiente: producao
```

Assim, a política será aplicada aos Pods dos namespaces identificados como produção.

**Atenção:** não utilize labels comuns como único mecanismo de autorização. Usuários que podem modificar labels de namespaces podem alterar a abrangência de políticas. Controle quem pode modificar namespaces e seus labels por RBAC.

---

# 10. Integração com GitLab CI

Agora vamos conectar o conteúdo com seu fluxo DevOps.

Imagine que um desenvolvedor envie um manifesto Kubernetes para o GitLab.

Antes de permitir o merge ou deploy, queremos verificar se o YAML atende às políticas de segurança.

Uma boa abordagem é executar o Kyverno CLI no pipeline.

Documentação:

[Kyverno CLI](https://kyverno.io/docs/kyverno-cli/?utm_source=chatgpt.com)

## 10.1 Instalando o Kyverno CLI

O CLI pode ser obtido por meio dos releases oficiais.

Para este laboratório, considere que o binário `kyverno` esteja disponível no PATH.

Verifique:

```bash
kyverno version
```

## 10.2 Validando manifests

O Kyverno CLI permite testar políticas localmente.

Exemplo:

```bash
kyverno apply block-privileged.yaml \
  --resource deployment.yaml
```

A sintaxe e as opções exatas devem ser conferidas na versão instalada.

Também é possível utilizar:

```bash
kyverno test .
```

para executar testes organizados em diretórios de teste.

## 10.3 Exemplo GitLab CI

Imagine esta estrutura:

```text
projeto/
├── manifests/
│   ├── deployment.yaml
│   └── service.yaml
├── policies/
│   ├── block-privileged.yaml
│   └── require-security-context.yaml
└── .gitlab-ci.yml
```

Exemplo de pipeline:

```yaml
stages:
  - validate
  - build
  - deploy

validate-security:
  stage: validate

  image:
    name: ghcr.io/kyverno/kyverno-cli:latest
    entrypoint: [""]

  script:
    - kyverno apply policies/block-privileged.yaml
        --resource manifests/deployment.yaml

    - kyverno apply policies/require-security-context.yaml
        --resource manifests/deployment.yaml
```

Esse exemplo é ilustrativo: a imagem e o formato dos comandos precisam ser compatíveis com a versão do CLI utilizada. Em um pipeline real, fixe a versão da imagem e teste também todos os recursos envolvidos, não apenas um Deployment.

A ideia é que o job falhe quando um manifesto violar as políticas.

Assim, a aplicação não chega à etapa de deploy.

---

# 11. Integração com ArgoCD

Agora vamos conectar o assunto com GitOps.

Imagine o seguinte fluxo:

```text
             Desenvolvedor
                    |
                    v
                 GitLab
                    |
                    v
             Git Repository
                    |
                    v
                  ArgoCD
                    |
                    v
              Kubernetes API
                    |
                    v
             Admission Webhook
                    |
               Kyverno/OPA
                    |
             +------+------+
             |             |
           Admit          Deny
             |             |
             v             v
          Aplicação      Rejeição
```

O ArgoCD compara o estado desejado no Git com o estado atual do cluster.

Quando tenta aplicar um recurso incompatível com uma política de admissão, o API Server rejeita a operação.

O ArgoCD registra o erro de sincronização.

Por exemplo:

```text
Sync Failed

admission webhook "validate.kyverno.svc" denied the request:
Containers privilegiados não são permitidos.
```

Isso é importante porque a política funciona independentemente de quem está tentando criar o recurso.

Pode ser:

- kubectl.
- ArgoCD.
- Helm.
- Outro controlador.
- Uma aplicação utilizando a API Kubernetes.

A política de admissão é aplicada pelo API Server.

## 11.1 Onde validar: GitLab ou Kubernetes?

A melhor estratégia é utilizar os dois.

**No GitLab CI:**

- Detectar problemas antes do merge.
- Impedir que manifestos inseguros avancem.
- Fornecer feedback rápido ao desenvolvedor.

**No Kubernetes:**

- Garantir que nenhum recurso inseguro seja admitido.
- Proteger o cluster independentemente da origem da requisição.
- Impedir que uma configuração insegura seja aplicada por outro mecanismo.

Essa combinação é conhecida como defesa em profundidade.

---

# 12. Exemplo de política de hardening mais abrangente

Vamos consolidar os requisitos de segurança que estudamos.

Nosso objetivo será exigir:

- Nenhum container privilegiado.
- Nenhum hostPath.
- Execução não root.
- Sem escalada de privilégios.
- Seccomp.
- Remoção de capabilities.

Uma recomendação prática é começar com os Pod Security Standards Restricted e complementar com políticas específicas do Kyverno.

Não precisamos reimplementar tudo o que o Kubernetes já oferece.

Por exemplo, podemos aplicar:

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

E utilizar Kyverno para políticas adicionais, como:

- Bloquear imagens sem digest.
- Exigir registries aprovados.
- Impedir uso de tags `latest`.
- Exigir labels obrigatórias.
- Bloquear volumes específicos.
- Verificar assinaturas de imagens.

Essa separação evita duplicação desnecessária de políticas.

---

# 13. Cuidados importantes em produção

Existem alguns detalhes que merecem atenção.

### 13.1 Políticas podem bloquear aplicações legítimas

Por exemplo:

- Agentes de monitoramento.
- CNI.
- CSI.
- Ferramentas de diagnóstico.
- Workloads que realmente precisam de hostPath.

Não recomendo habilitar `Enforce` em todo o cluster sem primeiro identificar as exceções necessárias.

### 13.2 Não confunda validação com correção

Uma política que exige:

```yaml
runAsNonRoot: true
```

não transforma automaticamente uma imagem incompatível em uma imagem segura.

A aplicação precisa funcionar corretamente com o usuário definido.

### 13.3 Não dependa apenas do CI

Um desenvolvedor pode aplicar recursos diretamente com `kubectl`.

Uma credencial comprometida pode criar recursos fora do pipeline.

Por isso, o controle no cluster é indispensável.

### 13.4 Cuidado com exceções

Evite criar exceções genéricas para namespaces inteiros.

Uma exceção deve ser:

- Justificada.
- Documentada.
- Restrita ao menor escopo possível.
- Revisada periodicamente.

---

# 14. Laboratório final — DevSecOps com Kyverno

Vamos montar um exercício que integra os conceitos da aula.

### Objetivo

Impedir que um Deployment utilize containers privilegiados.

**Etapa 1 — Criar namespace:**

```bash
kubectl create namespace devsecops-lab
```

**Etapa 2 — Criar uma política:**

Utilize a política `block-privileged` apresentada anteriormente.

**Etapa 3 — Criar um Deployment seguro:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-segura
  namespace: devsecops-lab

spec:
  replicas: 1

  selector:
    matchLabels:
      app: app-segura

  template:
    metadata:
      labels:
        app: app-segura

    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: app
          image: nginxinc/nginx-unprivileged:stable

          ports:
            - containerPort: 8080

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true

            capabilities:
              drop:
                - ALL
```

Observação: a imagem e a configuração precisam ser compatíveis com o modo somente leitura; algumas aplicações necessitam de volumes temporários para diretórios graváveis.

**Etapa 4 — Aplicar:**

```bash
kubectl apply -f deployment.yaml
```

**Etapa 5 — Tentar alterar para privilegiado:**

```yaml
securityContext:
  privileged: true
```

A atualização deverá ser rejeitada pela política.

**Etapa 6 — Verificar:**

```bash
kubectl get pods -n devsecops-lab
```

```bash
kubectl describe deployment app-segura -n devsecops-lab
```

```bash
kubectl get events -n devsecops-lab --sort-by=.lastTimestamp
```

---

# 15. Resumo da aula

Hoje estudamos como transformar segurança de containers em políticas centralizadas.

| Tecnologia | Responsabilidade |
|---|---|
| Admission Controllers | Interceptar requisições antes da persistência. |
| Kyverno | Criar políticas Kubernetes utilizando YAML. |
| OPA | Avaliar políticas utilizando Rego. |
| Gatekeeper | Integrar OPA ao Kubernetes. |
| Pod Security Admission | Aplicar Pod Security Standards. |
| GitLab CI | Validar manifestos antes do deploy. |
| ArgoCD | Aplicar o estado desejado do Git no cluster. |

O principal aprendizado:

> Segurança não deve depender exclusivamente da disciplina de quem escreve os manifestos. Ela deve ser imposta por políticas automatizadas no pipeline e no próprio cluster.

## Exercícios de fixação

1. Qual é a diferença entre Mutating Admission e Validating Admission?
2. Por que uma política de admissão é mais abrangente do que uma validação executada somente no GitLab CI?
3. Qual é a diferença entre Kyverno e Gatekeeper?
4. O que são ConstraintTemplates e Constraints?
5. Qual é a finalidade da linguagem Rego?
6. Por que uma política que verifica somente `containers` pode ser insuficiente?
7. Qual é a diferença entre `Audit` e `Enforce` no Kyverno?
8. Por que é importante restringir quem pode modificar labels dos namespaces?
9. Como o ArgoCD reage quando o API Server rejeita um recurso?
10. Por que devemos combinar Pod Security Admission, Kyverno e validação no CI/CD?

---

## Próxima aula — 2.10: Segurança de Imagens de Containers — Vulnerabilidades, SBOM e Supply Chain

Na próxima aula, vamos mudar o foco da configuração do runtime para aquilo que executamos dentro dele: as imagens de containers.

Vamos estudar:

1. Anatomia de imagens OCI.
2. Vulnerabilidades em imagens.
3. Trivy e Grype.
4. CVE, CVSS e EPSS.
5. SBOM com Syft.
6. Assinatura de imagens com Cosign.
7. Verificação de proveniência.
8. Integração de scanners ao GitLab CI.
9. Segurança do Harbor e controle de imagens.
10. Relação entre segurança de imagens, registry e ArgoCD.

Essa será a transição de Runtime Security para Image Security e Software Supply Chain Security.
