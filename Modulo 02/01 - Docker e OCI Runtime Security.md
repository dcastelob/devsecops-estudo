# Módulo 2 — Docker e OCI Runtime Security

Agora vamos sair um pouco da teoria dos mecanismos internos do kernel Linux e entender como eles são utilizados na prática para construir e executar containers.

Até aqui, estudamos individualmente:

- Namespaces.
- Cgroups.
- Capabilities.
- Seccomp.
- AppArmor.
- SELinux.

Uma pergunta importante surge:

**Quem configura todos esses mecanismos quando executamos um container?**

Quando você executa:

```bash
docker run -d nginx
```

O Docker precisa realizar uma série de operações no Linux para transformar uma imagem em um processo isolado.

Ele precisa preparar o filesystem, configurar namespaces, aplicar limites de recursos, definir capabilities, configurar Seccomp e iniciar o processo.

É justamente isso que vamos estudar neste módulo.

## 1. Objetivos do módulo

Ao final, você deverá compreender:

1. A arquitetura interna do Docker.
2. O que é OCI (Open Container Initiative).
3. A diferença entre Docker Engine, containerd, runc e crun.
4. Como uma imagem se transforma em um container em execução.
5. Como funciona o processo de criação de um container.
6. O papel do filesystem e dos mounts.
7. Como funciona o isolamento do container.
8. Como inspecionar a configuração de segurança de containers.
9. Como executar containers sem Docker, utilizando diretamente runtimes OCI.
10. Como identificar configurações inseguras em containers.

---

# Aula 2.1 — Arquitetura do Docker

## 1. O que realmente acontece quando executamos docker run?

Vamos começar com um comando conhecido:

```bash
docker run -d --name web nginx
```

Para quem utiliza Docker diariamente, esse comando parece simples.

Entretanto, ele desencadeia uma sequência de operações envolvendo diferentes componentes.

Uma visão simplificada da arquitetura:

```text
+--------------------------------------+
|              Usuário                 |
|                                      |
|          docker run nginx            |
+------------------+-------------------+
                   |
                   v
+--------------------------------------+
|             Docker CLI               |
|                                      |
|       Interpreta o comando           |
+------------------+-------------------+
                   |
                   v
+--------------------------------------+
|            Docker Engine             |
|                                      |
|              dockerd                 |
+------------------+-------------------+
                   |
                   v
+--------------------------------------+
|              containerd              |
|                                      |
|       Gerenciamento de containers    |
+------------------+-------------------+
                   |
                   v
+--------------------------------------+
|                runc                  |
|                                      |
|       OCI Runtime                    |
|                                      |
|  Namespaces / Cgroups / Mounts       |
|  Capabilities / Seccomp / LSM        |
+------------------+-------------------+
                   |
                   v
+--------------------------------------+
|              Linux Kernel            |
|                                      |
|       Processo isolado               |
+--------------------------------------+
```

Essa arquitetura é uma simplificação didática. Existem componentes adicionais, como plugins, snapshotters, serviços de rede e gerenciadores de armazenamento.

Vamos entender cada parte.

## 2. Docker CLI

O Docker CLI é a ferramenta que utilizamos no terminal.

Exemplos:

```bash
docker run
docker ps
docker inspect
docker logs
docker exec
docker stop
```

O CLI não cria diretamente os namespaces ou configura os cgroups.

Ele envia solicitações para a API do Docker Engine.

Podemos verificar a versão:

```bash
docker version
```

Observe que o resultado apresenta informações do cliente e do servidor.

Isso é importante porque o Docker CLI pode estar instalado em uma máquina e controlar um Docker Engine remoto.

## 3. Docker Engine — dockerd

O `dockerd` é o serviço responsável por disponibilizar a API do Docker e coordenar as operações solicitadas pelo cliente.

Entre suas responsabilidades estão:

- Gerenciamento de imagens.
- Gerenciamento de containers.
- Gerenciamento de redes.
- Gerenciamento de volumes.
- Integração com containerd.
- Gerenciamento do ciclo de vida dos containers.

Podemos verificar:

```bash
systemctl status docker
```

E consultar o processo:

```bash
ps aux | grep dockerd
```

Uma observação importante: o `dockerd` não precisa executar dentro do container. Ele é um serviço do host que gerencia os containers.

## 4. containerd

O containerd é um runtime de containers de nível mais alto.

Ele é responsável por gerenciar o ciclo de vida dos containers e coordenar operações como:

- Pull de imagens.
- Gerenciamento de snapshots.
- Criação de containers.
- Execução de tarefas.
- Gerenciamento de processos.
- Integração com runtimes OCI.

É utilizado não apenas pelo Docker, mas também por ambientes Kubernetes.

Por exemplo, em um cluster Kubernetes, o kubelet normalmente conversa com um runtime por meio da CRI (Container Runtime Interface).

O containerd possui integração com CRI.

Isso significa que o Kubernetes pode utilizar containerd sem precisar de Docker Engine.

## 5. runc

Aqui chegamos a um componente muito importante para nosso estudo.

O `runc` é uma implementação de OCI Runtime.

Sua responsabilidade é efetivamente criar e executar containers utilizando os recursos disponibilizados pelo kernel Linux.

É ele que realiza operações relacionadas a:

- Namespaces.
- Cgroups.
- Mounts.
- Capabilities.
- Seccomp.
- Configuração do processo.
- Inicialização do processo principal.

Lembre-se da nossa aula de Namespaces:

```bash
unshare
```

O `runc` executa operações semelhantes, mas seguindo uma especificação padronizada e recebendo uma configuração estruturada.

O `runc` não é um gerenciador completo de imagens e containers como o Docker Engine. Ele é um runtime de baixo nível.

## 6. crun

O `crun` é outra implementação de OCI Runtime.

Possui funcionalidades semelhantes ao `runc`, mas foi desenvolvido em C e tem características próprias de desempenho e consumo de recursos.

É bastante relevante no ecossistema Red Hat e OpenShift.

Podemos resumir:

| Componente | Responsabilidade |
|---|---|
| Docker CLI | Interface de comandos |
| dockerd | Gerenciamento Docker |
| containerd | Gerenciamento do ciclo de vida dos containers |
| runc | Execução de containers OCI |
| crun | Outra implementação de OCI Runtime |

---

# 2.2 — O que é OCI?

OCI significa **Open Container Initiative**.

É uma organização que mantém especificações abertas para padronizar o ecossistema de containers.

Existem três especificações fundamentais:

| Especificação | Responsabilidade |
|---|---|
| Image Specification | Define o formato das imagens |
| Runtime Specification | Define como executar containers |
| Distribution Specification | Define como distribuir imagens |

## 1. OCI Image Specification

Define como uma imagem de container deve ser organizada.

Uma imagem não é simplesmente um arquivo compactado contendo um sistema operacional.

Ela é composta por elementos como:

- Manifest.
- Config.
- Layers.
- Descriptors.

Exemplo conceitual:

```text
Imagem nginx
     |
     v
+------------------------+
| Manifest               |
+------------------------+
     |
     +--------------------------+
     |                          |
     v                          v
+----------------+       +----------------+
| Config         |       | Layers         |
|                |       |                |
| Entrypoint     |       | Layer 1        |
| Cmd            |       | Layer 2        |
| Env            |       | Layer 3        |
| Architecture   |       | Layer 4        |
+----------------+       +----------------+
```

As layers representam partes do filesystem da imagem.

Normalmente, são armazenadas e distribuídas por digest, permitindo identificação e reutilização.

## 2. OCI Runtime Specification

Define como um container deve ser configurado e executado.

Um dos elementos centrais é o arquivo:

```text
config.json
```

Esse arquivo contém informações como:

- Root filesystem.
- Processo inicial.
- Argumentos.
- Variáveis de ambiente.
- Usuário.
- Mounts.
- Namespaces.
- Cgroups.
- Capabilities.
- Seccomp.
- Configurações de segurança.

Isso significa que boa parte do que estudamos no Módulo 1 pode ser representada nesse arquivo.

## 3. OCI Distribution Specification

Define como imagens podem ser distribuídas entre registries e clientes.

Isso permite que diferentes ferramentas trabalhem com registries compatíveis.

Por exemplo:

- Docker Hub.
- Harbor.
- Amazon ECR.
- Quay.
- GitLab Container Registry.

O ponto fundamental é que o padrão OCI permite interoperabilidade entre ferramentas diferentes.

---

# 2.3 — Laboratório: conhecendo o ambiente

Vamos começar nosso laboratório no Debian 13.

Verifique se Docker está instalado:

```bash
docker version
```

Consulte as informações do ambiente:

```bash
docker info
```

Observe especialmente:

```text
Storage Driver
Cgroup Driver
Cgroup Version
Kernel Version
Operating System
Architecture
Runtimes
Default Runtime
Security Options
```

Essas informações serão importantes durante as próximas aulas.

Agora verifique o containerd:

```bash
systemctl status containerd
```

E:

```bash
containerd --version
```

Se o comando estiver disponível:

```bash
runc --version
```

Ou:

```bash
crun --version
```

Não é obrigatório que todas essas ferramentas estejam instaladas.

## 1. Inspecionando o runtime utilizado pelo Docker

Execute:

```bash
docker info | grep -A10 -i runtimes
```

Exemplo:

```text
 Runtimes: io.containerd.runc.v2 runc
 Default Runtime: runc
```

O resultado varia conforme a instalação.

Observe que `io.containerd.runc.v2` é um nome de runtime handler do containerd, não o nome de uma implementação completamente diferente do runc.

---

# 2.4 — Criando nosso primeiro container para investigação

Vamos executar um container que permanecerá ativo:

```bash
docker run -d \
  --name runtime-lab \
  alpine \
  sleep 3600
```

Agora consulte:

```bash
docker ps
```

Inspecione o container:

```bash
docker inspect runtime-lab
```

Para visualizar somente o PID no host:

```bash
docker inspect \
  --format '{{.State.Pid}}' \
  runtime-lab
```

Guarde esse PID:

```bash
PID=$(docker inspect \
  --format '{{.State.Pid}}' \
  runtime-lab)

echo "$PID"
```

Agora vamos observar os namespaces:

```bash
sudo lsns -p "$PID"
```

Compare com os namespaces do host:

```bash
sudo ls -l /proc/1/ns/
sudo ls -l /proc/"$PID"/ns/
```

Observe também o cgroup:

```bash
cat /proc/"$PID"/cgroup
```

E as capabilities:

```bash
grep Cap /proc/"$PID"/status
```

Finalmente, o Seccomp:

```bash
grep Seccomp /proc/"$PID"/status
```

Perceba o que estamos fazendo: estamos investigando os mecanismos do kernel que o runtime configurou para o container.

O Docker não criou um kernel novo. Ele criou um processo no kernel existente, com restrições e isolamento.

---

# 2.5 — Um ponto importante: container não é uma máquina virtual

Vamos reforçar esse conceito.

Quando executamos:

```bash
docker run alpine sleep 3600
```

O processo `sleep` continua sendo um processo Linux convencional.

Podemos encontrá-lo no host:

```bash
ps -ef | grep '[s]leep 3600'
```

Podemos consultar:

```bash
cat /proc/"$PID"/status
```

E observar:

- UID.
- GID.
- Capabilities.
- Threads.
- Seccomp.
- Estado do processo.

O container é uma forma de executar processos com isolamento e restrições.

Isso tem uma consequência muito importante para segurança:

**Uma vulnerabilidade no kernel pode afetar containers e o host, porque ambos compartilham o mesmo kernel.**

Por isso, o runtime e os mecanismos de isolamento precisam ser corretamente configurados.

---

## Exercícios da Aula 2.1

Antes de avançarmos, tente responder:

1. Qual é a diferença entre Docker Engine e containerd?
2. Qual é a responsabilidade do runc?
3. O Kubernetes precisa obrigatoriamente do Docker Engine para executar containers?
4. Qual é a diferença entre OCI Image Specification e OCI Runtime Specification?
5. Quando executamos um container, ele possui um kernel próprio?
6. Qual componente efetivamente configura os namespaces e cgroups?

### Próxima aula — 2.2: Anatomia da criação de um container

Vamos aprofundar o processo que acontece internamente quando executamos `docker run`.

Estudaremos a sequência:

```text
Imagem
  |
  v
Snapshotter
  |
  v
Root Filesystem
  |
  v
Mounts
  |
  v
Namespaces
  |
  v
Cgroups
  |
  v
Capabilities
  |
  v
Seccomp / LSM
  |
  v
Execução do processo
```

Também vamos investigar o `config.json` da especificação OCI e entender como ele representa as configurações de segurança que estudamos no Módulo 1.