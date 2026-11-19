# Roadmap de estudos: Segurança em ambientes de containers

Minha recomendação é estudar segurança de containers seguindo uma abordagem que começa na infraestrutura Linux, passa pela construção e distribuição de imagens e chega à segurança de Kubernetes e OpenShift.

Como você já trabalha com Linux, Kubernetes, OpenShift, Ansible, GitLab CI e ArgoCD, não vejo necessidade de começar por conceitos básicos de containers. O melhor caminho é aprofundar os mecanismos de segurança e aprender a implementar controles de forma automatizada.

Eu dividiria o roadmap em **10 módulos, organizados em 5 grandes etapas**.

O objetivo final é conseguir projetar, implementar, auditar e monitorar um ambiente Kubernetes/OpenShift seguindo princípios de DevSecOps e defesa em profundidade.

---

# Etapa 1 — Fundamentos de segurança de containers

## Módulo 1. Segurança no Linux e isolamento de processos

Antes de estudar ferramentas de segurança para Kubernetes, é importante compreender como o Linux implementa o isolamento dos containers.

Estude:

| Assunto | Conteúdo |
|---|---|
| Namespaces | PID, NET, MNT, IPC, UTS, USER, CGROUP |
| Cgroups | Limitação e isolamento de recursos |
| Capabilities | Redução dos privilégios de processos |
| Seccomp | Filtragem de chamadas de sistema |
| AppArmor | Controle de acesso obrigatório |
| SELinux | Políticas de segurança e controle de acesso |
| Linux Security Modules | Arquitetura LSM |
| Filesystem | Read-only root filesystem, permissões e mounts |
| Process isolation | PID 1, processos privilegiados e isolamento |

**Prática recomendada:**

Execute containers e investigue seus mecanismos de isolamento.

Exemplos:

```bash
docker run --rm alpine sleep 3600
```

Identifique os namespaces:

```bash
lsns
```

Inspecione as capabilities:

```bash
docker inspect <container>
```

Investigue as capabilities de um processo:

```bash
grep Cap /proc/<PID>/status
```

**Objetivo:** entender o que efetivamente impede um processo dentro de um container de acessar recursos do host.

---

## Módulo 2. Segurança do Docker e runtimes OCI

Estude os mecanismos de segurança do runtime.

Conteúdos:

- Docker Security.
- Containerd.
- CRI-O.
- OCI Runtime Specification.
- runc.
- Rootless containers.
- User namespaces.
- Privileged containers.
- Seccomp profiles.
- AppArmor e SELinux.
- Docker daemon security.
- Socket `/var/run/docker.sock`.
- Runtime socket exposure.

Prática:

Execute containers com diferentes níveis de privilégio:

```bash
docker run --rm alpine id
```

```bash
docker run --rm --privileged alpine id
```

Compare as capabilities:

```bash
docker run --rm \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  alpine sh
```

Um conceito especialmente importante: **um container não é uma máquina virtual**. Seu isolamento depende de mecanismos do kernel compartilhado com o host.

---

# Etapa 2 — Segurança da cadeia de suprimentos

Esta etapa é fundamental para quem trabalha com pipelines GitLab CI, Kaniko, registries e ArgoCD.

## Módulo 3. Segurança de imagens de containers

Estude como identificar e reduzir vulnerabilidades nas imagens.

| Assunto | Ferramentas |
|---|---|
| Análise de vulnerabilidades | Trivy, Grype |
| Análise de pacotes | Syft |
| SBOM | SPDX, CycloneDX |
| Análise de Dockerfile | Hadolint |
| Análise de configurações | Checkov |
| Imagens mínimas | Distroless, Alpine, UBI Minimal |
| Assinatura de imagens | Cosign |
| Verificação de procedência | SLSA, attestations |

Pontos que você precisa dominar:

1. Vulnerabilidades em pacotes do sistema operacional.
2. Vulnerabilidades em dependências da aplicação.
3. Imagens base confiáveis.
4. Imagens imutáveis.
5. Redução da superfície de ataque.
6. Geração e armazenamento de SBOM.
7. Assinatura e verificação de imagens.

**Laboratório sugerido:**

Monte um pipeline:

```text
Git
 |
 v
Build da imagem
 |
 v
Trivy
 |
 v
Syft (SBOM)
 |
 v
Cosign (assinatura)
 |
 v
Harbor / Registry
```

O objetivo é impedir que imagens com vulnerabilidades acima de determinado nível sejam publicadas.

## Módulo 4. Segurança de registries e supply chain

A segurança não termina quando a imagem é construída. É necessário garantir que a imagem armazenada e posteriormente executada seja realmente aquela que foi aprovada.

Estude:

- Segurança do Harbor.
- Autenticação e autorização em registries.
- Controle de acesso a repositórios.
- Assinatura de imagens.
- Verificação de assinatura antes do deploy.
- Imagens identificadas por digest SHA256.
- Proveniência dos artefatos.
- SLSA (Supply-chain Levels for Software Artifacts).
- SBOM e rastreabilidade de dependências.

Ferramentas importantes:

- Harbor.
- Cosign.
- Notation.
- Syft.
- Trivy.
- Kyverno.

**Laboratório:** configurar o Harbor para armazenar imagens assinadas e implementar uma política que impeça o deploy de imagens sem assinatura válida.

---

# Etapa 3 — Segurança do Kubernetes e OpenShift

Esta é provavelmente a etapa mais importante do seu roadmap.

## Módulo 5. Kubernetes Authentication e Authorization

Estude profundamente os mecanismos de controle de acesso.

| Assunto | Conteúdo |
|---|---|
| Authentication | Certificados, tokens, OIDC |
| Authorization | RBAC |
| Service Accounts | Identidade dos workloads |
| API Server | Proteção da API |
| Secrets | Proteção de credenciais |
| etcd | Criptografia em repouso |
| Admission Controllers | Controle de admissões |
| Audit Logs | Auditoria de operações |

Práticas:

1. Criar ServiceAccounts específicas para aplicações.
2. Aplicar o princípio do menor privilégio.
3. Impedir o uso desnecessário de tokens.
4. Auditar permissões excessivas.
5. Investigar caminhos de escalonamento de privilégios.
6. Configurar criptografia dos Secrets em repouso.

Ferramentas:

- kubectl auth can-i.
- kubescape.
- rbac-tool.
- kubeaudit.

Um exercício interessante:

```bash
kubectl auth can-i \
  --list \
  --as=system:serviceaccount:default:default
```

O objetivo é descobrir quais operações uma aplicação consegue executar dentro do cluster.

## Módulo 6. Pod Security e Hardening

Aqui você vai trabalhar diretamente com a segurança dos workloads.

Estude:

- Pod Security Standards.
- Pod Security Admission.
- Security Context.
- runAsNonRoot.
- allowPrivilegeEscalation.
- readOnlyRootFilesystem.
- capabilities.
- seccomp.
- hostNetwork.
- hostPID.
- hostPath.
- privileged containers.
- Resource Requests e Limits.

Exemplo de configuração:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-container
spec:
  automountServiceAccountToken: false

  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: app
      image: nginxinc/nginx-unprivileged:stable

      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL

      resources:
        requests:
          memory: 128Mi
          cpu: 100m
        limits:
          memory: 256Mi
          cpu: 500m
```

Atenção: esse manifesto é um exemplo de hardening, mas algumas aplicações precisam de volumes graváveis e ajustes adicionais.

### Particularidades do OpenShift

Não deixe de estudar:

- Security Context Constraints (SCC).
- `restricted-v2`.
- SELinux no OpenShift.
- Arbitrary UID.
- Random UID.
- RunAsUser.
- Privileged SCC.
- ServiceAccount e SCC.
- Project e Namespace isolation.

Uma diferença importante: no OpenShift, o mecanismo SCC exerce um papel central no controle dos privilégios dos Pods. Não basta transportar configurações de Kubernetes convencional sem entender como as SCCs interferem na admissão.

**Laboratório:** construir uma aplicação que funcione sob `restricted-v2`, sem solicitar privilégios adicionais.

## Módulo 7. Network Security

Um dos pontos mais negligenciados na segurança de ambientes Kubernetes é a comunicação entre os workloads.

Estude:

| Assunto | Conteúdo |
|---|---|
| Network Policies | Isolamento de tráfego |
| Ingress/Egress | Controle de entrada e saída |
| CNI | Funcionamento da rede |
| Service Mesh | Segurança entre serviços |
| mTLS | Autenticação mútua |
| DNS | Segurança de resolução |
| Network Segmentation | Segmentação de workloads |
| Egress Control | Controle de saída para Internet |

Ferramentas:

- Cilium.
- Calico.
- Istio.
- Linkerd.
- Hubble.

Exercícios:

1. Criar políticas de negação de tráfego por padrão.
2. Liberar comunicação somente entre aplicações autorizadas.
3. Restringir saída para destinos externos.
4. Implementar comunicação mTLS entre serviços.
5. Investigar comunicação lateral entre namespaces.

Um cenário interessante:

```text
Namespace A
    |
    | permitido
    v
Namespace B

Namespace C
    |
    | bloqueado
    X
Namespace B
```

O objetivo é impedir que o comprometimento de uma aplicação permita movimentação lateral pelo cluster.

---

# Etapa 4 — Segurança em tempo de execução

## Módulo 8. Container Runtime Security

Até aqui, estudamos principalmente prevenção. Agora precisamos detectar comportamentos maliciosos durante a execução.

Estude:

- Container escape.
- Privilege escalation.
- Runtime threat detection.
- Process monitoring.
- Syscall monitoring.
- File integrity monitoring.
- Kubernetes attack techniques.
- Cryptomining em containers.
- Reverse shells.
- Execução de processos inesperados.
- Acesso indevido ao filesystem do host.

Ferramentas que recomendo conhecer:

| Ferramenta | Finalidade |
|---|---|
| Falco | Detecção de comportamentos suspeitos |
| Tetragon | Observabilidade e enforcement via eBPF |
| Tracee | Segurança e rastreamento via eBPF |
| KubeArmor | Runtime security |
| NeuVector | Segurança de containers e Kubernetes |

### Começaria pelo Falco

É uma excelente ferramenta para aprender detecção de ameaças em ambientes de containers.

Exemplos de comportamentos que você pode monitorar:

- Shell executado dentro de um container.
- Acesso inesperado a `/etc/shadow`.
- Alterações em arquivos sensíveis.
- Execução de binários suspeitos.
- Acesso ao socket do Docker.
- Tentativas de execução de containers privilegiados.

**Laboratório:** instalar Falco no Kubernetes e criar regras personalizadas para detectar comportamentos suspeitos.

## Módulo 9. Vulnerability Management e Compliance

Aqui o objetivo é estabelecer um processo contínuo de avaliação de segurança.

Estude:

- CIS Kubernetes Benchmark.
- CIS Docker Benchmark.
- Kubernetes Security Posture Management (KSPM).
- Vulnerability Management.
- Configuration Assessment.
- CVE e CVSS.
- OWASP Kubernetes Top 10.
- Compliance e auditoria.

Ferramentas:

| Ferramenta | Aplicação |
|---|---|
| Kubescape | Kubernetes security posture |
| kube-bench | CIS Benchmark |
| Trivy | Vulnerabilidades e configurações |
| Checkov | IaC security |
| Kubiscan | Análise de permissões RBAC |
| kubeaudit | Auditoria de configurações |

Minha sugestão é utilizar o Kubescape para avaliar o cluster e o kube-bench para verificar a aderência aos controles CIS.

Não confunda os dois: o CIS Benchmark avalia configurações recomendadas, enquanto ferramentas de KSPM podem avaliar um conjunto mais amplo de riscos e controles.

---

# Etapa 5 — DevSecOps e arquitetura de segurança

## Módulo 10. DevSecOps e Kubernetes Security Architecture

Este módulo reúne tudo o que foi estudado anteriormente.

O objetivo é implementar segurança desde o desenvolvimento até a execução da aplicação.

Estude:

- Secure SDLC.
- Shift Left Security.
- Security Gates.
- Policy as Code.
- GitOps Security.
- Secrets Management.
- Image Admission Policies.
- Supply Chain Security.
- Threat Modeling.
- Incident Response.

Ferramentas:

| Ferramenta | Finalidade |
|---|---|
| Kyverno | Policy as Code |
| OPA Gatekeeper | Policy as Code |
| HashiCorp Vault | Gestão de segredos |
| External Secrets Operator | Integração com cofres de segredos |
| ArgoCD | GitOps |
| Cosign | Assinatura de imagens |
| Trivy | Análise de segurança |

### Arquitetura que recomendo implementar

Considerando GitLab CI, Harbor, ArgoCD e OpenShift, eu montaria o seguinte fluxo:

```text
              DESENVOLVIMENTO
                     |
                     v
                  GitLab
                     |
                     v
             Security Scanning
                     |
          +----------+----------+
          |          |          |
          v          v          v
        SAST       SCA       IaC Scan
          |          |          |
          +----------+----------+
                     |
                     v
                Image Build
                     |
                     v
              Image Scanning
                     |
                     v
                 SBOM
                     |
                     v
              Image Signing
                     |
                     v
                   Harbor
                     |
                     v
                   ArgoCD
                     |
                     v
             Admission Control
                     |
                     v
                 OpenShift
                     |
          +----------+----------+
          |          |          |
          v          v          v
        Falco     Network    Runtime
                  Policies   Security
```

O grande objetivo é garantir que uma imagem não seja simplesmente publicada e executada, mas passe por controles de segurança em cada etapa.

# 6. Cronograma sugerido de estudos

Eu organizaria os estudos em aproximadamente 16 semanas, considerando de 5 a 7 horas semanais.

| Período | Módulo | Resultado esperado |
|---|---|---|
| Semanas 1–2 | Linux Security e isolamento | Dominar namespaces, capabilities e seccomp |
| Semana 3 | Container Runtime | Entender Docker, containerd, CRI-O e runc |
| Semanas 4–5 | Image Security | Implementar scanning, SBOM e assinatura |
| Semana 6 | Registry Security | Proteger Harbor e distribuição de imagens |
| Semanas 7–8 | Kubernetes Access Control | Dominar RBAC, ServiceAccounts e Secrets |
| Semana 9 | Pod Security | Implementar hardening e SCC |
| Semana 10 | Network Security | Implementar isolamento de rede |
| Semanas 11–12 | Runtime Security | Implantar Falco e investigar ameaças |
| Semana 13 | Compliance | CIS Benchmark e KSPM |
| Semanas 14–16 | DevSecOps | Integrar tudo em GitLab, Harbor e ArgoCD |

# 7. Documentação e materiais de referência

Estas são as referências que utilizaria como material principal.

| Material | Importância |
|---|---|
| [Kubernetes Security Documentation](https://kubernetes.io/docs/concepts/security/?utm_source=chatgpt.com) | Essencial |
| [Kubernetes Security Checklist](https://kubernetes.io/docs/concepts/security/security-checklist/?utm_source=chatgpt.com) | Essencial |
| [NIST SP 800-190](https://csrc.nist.gov/pubs/sp/800/190/final?utm_source=chatgpt.com) | Muito importante |
| [NSA/CISA Kubernetes Hardening Guide](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF?utm_source=chatgpt.com) | Muito importante |
| [OWASP Kubernetes Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html?utm_source=chatgpt.com) | Muito importante |
| [Kubernetes Application Security Checklist](https://kubernetes.io/docs/concepts/security/application-security-checklist/?utm_source=chatgpt.com) | Essencial |

Destaco especialmente o documento **NIST SP 800-190**, que apresenta uma visão abrangente dos riscos de segurança associados a containers, incluindo imagens, registries, runtimes, orquestradores e infraestrutura.

Também recomendo a documentação da NSA/CISA. É um excelente material para aprender a pensar em termos de hardening e redução da superfície de ataque.

# 8. Certificação para consolidar o conhecimento

Se você quiser transformar esse estudo em uma certificação, minha principal recomendação seria:

**Certified Kubernetes Security Specialist (CKS)**

A CKS aborda justamente boa parte dos assuntos deste roadmap:

- Cluster Setup.
- Cluster Hardening.
- System Hardening.
- Minimize Microservice Vulnerabilities.
- Supply Chain Security.
- Monitoring, Logging and Runtime Security.

É uma certificação bastante alinhada com o objetivo de trabalhar profissionalmente com segurança em Kubernetes.

Documentação: [CKS — Linux Foundation](https://training.linuxfoundation.org/certification/certified-kubernetes-security-specialist/?utm_source=chatgpt.com).

---

# 9. Minha recomendação: um laboratório prático integrado

Eu não estudaria essas ferramentas isoladamente. Construiria um laboratório progressivo com:

| Componente | Tecnologia |
|---|---|
| Sistema operacional | Debian 13 |
| Containers | Docker / containerd |
| Kubernetes | K3s ou kubeadm |
| Plataforma adicional | OpenShift |
| Registry | Harbor |
| CI/CD | GitLab CI |
| GitOps | ArgoCD |
| Image Scanning | Trivy |
| SBOM | Syft |
| Image Signing | Cosign |
| Runtime Security | Falco |
| KSPM | Kubescape |
| Policy as Code | Kyverno |
| Network Security | Cilium |

A ideia é construir uma plataforma onde você consiga simular ataques, identificar vulnerabilidades, implementar controles e verificar se os controles realmente funcionam.

**Uma observação importante:** eu daria prioridade a três conhecimentos que considero fundamentais e que frequentemente são tratados superficialmente:

1. **Linux Kernel Security:** entender profundamente o isolamento e os mecanismos de proteção do kernel.
2. **Container Escape:** compreender como um invasor pode sair do isolamento do container e alcançar o host.
3. **Kubernetes Privilege Escalation:** compreender como permissões RBAC excessivas, ServiceAccounts e configurações inseguras podem permitir o comprometimento de outros workloads ou do cluster.

Esses três assuntos ajudam a sair de uma abordagem puramente operacional de segurança e entrar efetivamente na engenharia de segurança de containers.

**Minha sugestão para começarmos:** seguir esse roadmap como um curso prático de Container Security 101, começando pelo Módulo 1 (Linux Kernel Security), com explicações conceituais, comandos, laboratórios e exercícios de exploração controlada.
