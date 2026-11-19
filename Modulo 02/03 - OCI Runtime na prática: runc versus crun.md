# Módulo 2.3 — OCI Runtime na prática: runc versus crun

Nesta aula vamos aprofundar o funcionamento dos runtimes OCI e entender como eles transformam uma configuração declarativa em um processo Linux efetivamente isolado.

Vamos trabalhar com duas implementações:

- **runc:** runtime OCI amplamente utilizado no ecossistema Docker e containerd.
- **crun:** runtime OCI desenvolvido em C, utilizado, entre outros ambientes, pelo ecossistema Red Hat e CRI-O.

A pergunta central desta aula é:

> Quando um runtime recebe um `config.json`, como ele transforma as configurações descritas nesse arquivo em namespaces, cgroups, mounts, capabilities e filtros Seccomp no kernel Linux?

Ao final, você deverá conseguir executar containers diretamente com os dois runtimes, investigar seus processos e compreender como eles participam da arquitetura do Kubernetes e OpenShift.

---

# 1. O que é um OCI Runtime?

Primeiro, vamos separar três conceitos que frequentemente são confundidos:

| Componente | Responsabilidade |
|---|---|
| Container Engine | Gerencia imagens, containers, redes e volumes |
| Container Manager | Gerencia o ciclo de vida e coordena a execução |
| OCI Runtime | Cria e executa o ambiente isolado do container |

O `runc` e o `crun` pertencem à terceira categoria.

Eles implementam a especificação OCI Runtime, que define como um container deve ser configurado e executado.

A arquitetura pode ser representada assim:

```text
              Usuário
                 |
                 v
           Docker CLI
                 |
                 v
           Docker Engine
                 |
                 v
             containerd
                 |
                 v
             OCI Runtime
              /       \
             /         \
          runc         crun
             \         /
              \       /
                 |
                 v
             Linux Kernel
                 |
                 v
          Processo isolado
```

O ponto importante é que `runc` e `crun` são implementações alternativas da mesma especificação.

Um container não precisa ser criado especificamente pelo Docker para utilizar esses runtimes.

---

# 2. runc versus crun

Vamos comparar as duas implementações.

## 2.1 runc

O `runc` é uma implementação de referência bastante conhecida da OCI Runtime Specification.

É escrito principalmente em Go.

Entre suas responsabilidades estão:

- Criar namespaces.
- Configurar cgroups.
- Preparar o root filesystem.
- Configurar mounts.
- Configurar capabilities.
- Aplicar filtros Seccomp.
- Configurar mecanismos de segurança suportados pelo kernel.
- Iniciar o processo principal.

É utilizado em diversas arquiteturas de containers, incluindo instalações baseadas em containerd.

## 2.2 crun

O `crun` é uma implementação OCI Runtime escrita em C.

Foi desenvolvido com foco em desempenho, baixo consumo de recursos e integração com o ecossistema de containers.

Também implementa a OCI Runtime Specification.

É particularmente relevante em ambientes Red Hat, incluindo instalações que utilizam CRI-O.

## 2.3 Comparação

| Característica | runc | crun |
|---|---|---|
| Linguagem | Go | C |
| Especificação | OCI Runtime | OCI Runtime |
| Integração com containerd | Sim | Possível |
| Integração com CRI-O | Possível | Sim, muito comum |
| Namespaces | Sim | Sim |
| Cgroups | Sim | Sim |
| Capabilities | Sim | Sim |
| Seccomp | Sim | Sim |
| SELinux | Suporte | Suporte |
| AppArmor | Suporte | Suporte |

Não devemos concluir que um runtime é automaticamente mais seguro que o outro apenas por ser escrito em C ou Go.

A segurança depende da implementação, das versões utilizadas, da configuração e dos mecanismos de isolamento habilitados.

---

# 3. Instalação das ferramentas no Debian 13

Vamos preparar o laboratório.

Instale:

```bash
sudo apt update

sudo apt install \
    runc \
    crun \
    skopeo \
    umoci \
    jq \
    curl \
    strace \
    util-linux
```

Verifique as versões:

```bash
runc --version
crun --version
```

Também podemos consultar:

```bash
runc features
```

E:

```bash
crun features
```

Esses comandos ajudam a identificar funcionalidades disponíveis na versão instalada.

**Atenção:** a disponibilidade e a sintaxe de alguns recursos podem variar entre versões. Consulte `runc --help` e `crun --help` se algum comando apresentar diferenças.

---

# 4. Entendendo o ciclo de vida OCI

Um dos conceitos mais importantes desta aula é o ciclo de vida de um container.

A OCI Runtime Specification define operações para controlar esse ciclo.

As principais são:

| Operação | Finalidade |
|---|---|
| `create` | Cria o container e prepara seu ambiente |
| `start` | Inicia o processo configurado |
| `state` | Consulta o estado atual |
| `kill` | Envia um sinal ao processo |
| `delete` | Remove o estado do container |
| `exec` | Não é uma operação básica do ciclo OCI, mas alguns runtimes oferecem esse recurso |

Vamos entender a diferença entre `create` e `start`.

## 4.1 create

Quando executamos:

```bash
runc create meu-container
```

O runtime prepara o ambiente do container:

- Prepara o filesystem.
- Configura os namespaces.
- Configura os mounts.
- Prepara os cgroups.
- Configura as restrições de segurança.
- Prepara o processo inicial.

Entretanto, o processo principal ainda não começou a executar o comando definido em `process.args`.

O container fica no estado `created`.

## 4.2 start

Depois executamos:

```bash
runc start meu-container
```

Agora o runtime inicia o processo principal.

O estado passa para `running`.

Podemos representar o ciclo:

```text
              config.json
                   |
                   v
                create
                   |
                   v
                created
                   |
                   v
                 start
                   |
                   v
                running
                   |
             +-----+-----+
             |           |
             v           v
           kill        Processo
             |          termina
             +-----+-----+
                   |
                   v
                stopped
                   |
                   v
                delete
                   |
                   v
                Removido
```

Essa separação é importante porque permite que um gerenciador de containers prepare o ambiente antes de iniciar efetivamente o processo.

---

# 5. Laboratório 1 — Criando um container diretamente com runc

Vamos executar um container sem Docker e sem containerd.

Utilizaremos um root filesystem Alpine.

**Recomendação:** faça este laboratório em uma VM Debian 13. Os comandos de criação de containers exigem privilégios administrativos e alteram recursos do kernel.

## 5.1 Preparando o bundle OCI

Crie um diretório:

```bash
mkdir -p ~/oci-lab-runc
cd ~/oci-lab-runc
```

Obtenha a imagem Alpine:

```bash
skopeo copy \
  docker://docker.io/library/alpine:latest \
  oci:alpine-image:latest
```

Extraia o filesystem:

```bash
umoci unpack \
  --image alpine-image:latest \
  bundle
```

O diretório deverá conter algo semelhante a:

```text
oci-lab-runc/
├── alpine-image/
│   ├── blobs/
│   └── index.json
│
└── bundle/
    ├── config.json
    └── rootfs/
```

O diretório `bundle` é nosso bundle OCI.

Ele contém:

- `config.json`: configuração do container.
- `rootfs`: filesystem que será disponibilizado ao processo.

## 5.2 Gerando a configuração OCI

Entre no bundle:

```bash
cd ~/oci-lab-runc/bundle
```

Gere uma configuração padrão:

```bash
sudo runc spec
```

O comando cria um `config.json`.

Vamos modificar a configuração do processo.

Abra:

```bash
sudo vim config.json
```

Localize a seção:

```json
"process": {
```

Ajuste `args` para:

```json
"args": [
  "sh",
  "-c",
  "sleep 3600"
]
```

Certifique-se também de que:

```json
"terminal": false
```

O processo ficará executando por uma hora, permitindo nossa investigação.

**Importante:** mantenha os demais elementos gerados pelo `runc spec`, especialmente os mounts e as configurações de namespaces. Não substitua o arquivo inteiro pelo pequeno exemplo acima.

## 5.3 Criando o container

Execute:

```bash
sudo runc create \
  --bundle "$PWD" \
  lab-runc
```

Consulte:

```bash
sudo runc list
```

Você deverá encontrar o container no estado:

```text
ID        PID       STATUS
lab-runc  ...       created
```

Observe que ele ainda não está executando o comando `sleep`.

## 5.4 Iniciando o container

Agora:

```bash
sudo runc start lab-runc
```

Consulte:

```bash
sudo runc list
```

O estado deverá ser:

```text
ID        PID       STATUS
lab-runc  ...       running
```

Consulte os detalhes:

```bash
sudo runc state lab-runc
```

O resultado será um JSON contendo informações como:

- ID.
- PID.
- Status.
- Bundle.
- Annotations.

## 5.5 Investigando o processo

Obtenha o PID:

```bash
PID=$(sudo runc state lab-runc | jq -r '.pid')

echo "$PID"
```

Agora investigue:

```bash
ps -fp "$PID"
```

Consulte os namespaces:

```bash
sudo lsns -p "$PID"
```

Consulte as capabilities:

```bash
sudo grep Cap /proc/"$PID"/status
```

Consulte o Seccomp:

```bash
sudo grep Seccomp /proc/"$PID"/status
```

E o cgroup:

```bash
sudo cat /proc/"$PID"/cgroup
```

Observe como estamos investigando diretamente os mecanismos que o runtime configurou.

## 5.6 Encerrando o container

Envie SIGTERM:

```bash
sudo runc kill lab-runc TERM
```

Consulte:

```bash
sudo runc list
```

Depois remova o estado:

```bash
sudo runc delete lab-runc
```

Pronto. Criamos, executamos, investigamos e removemos um container sem utilizar Docker.

---

# 6. Laboratório 2 — Repetindo o processo com crun

Agora vamos executar o mesmo tipo de container utilizando o `crun`.

A vantagem é que podemos reutilizar o conceito de bundle OCI.

Crie outro laboratório:

```bash
mkdir -p ~/oci-lab-crun
cd ~/oci-lab-crun
```

Copie o rootfs do laboratório anterior:

```bash
cp -a ~/oci-lab-runc/bundle/rootfs .
```

Gere o arquivo de configuração:

```bash
sudo crun spec
```

Edite:

```bash
sudo vim config.json
```

Ajuste o processo:

```json
"args": [
  "sh",
  "-c",
  "sleep 3600"
],
"terminal": false
```

Mantenha as demais configurações geradas.

## 6.1 Criando o container com crun

Execute:

```bash
sudo crun create \
  --bundle "$PWD" \
  lab-crun
```

Consulte:

```bash
sudo crun list
```

Inicie:

```bash
sudo crun start lab-crun
```

Consulte:

```bash
sudo crun state lab-crun
```

Obtenha o PID:

```bash
PID=$(sudo crun state lab-crun | jq -r '.pid')
```

Investigue:

```bash
ps -fp "$PID"

sudo lsns -p "$PID"

sudo grep Cap /proc/"$PID"/status

sudo grep Seccomp /proc/"$PID"/status

sudo cat /proc/"$PID"/cgroup
```

Encerre:

```bash
sudo crun kill lab-crun TERM

sudo crun delete lab-crun
```

Você acabou de executar o mesmo tipo de container utilizando outra implementação da especificação OCI.

### O que acabamos de demonstrar?

O `config.json` descreve o ambiente desejado.

O runtime interpreta essa configuração e utiliza os mecanismos do kernel para construir o container.

A especificação OCI permite que diferentes implementações executem containers seguindo uma estrutura comum.

---

# 7. Investigando o que o runtime faz no kernel

Agora vamos aprofundar um pouco mais.

Uma das maneiras de estudar o funcionamento interno do `runc` é observar as chamadas de sistema realizadas por ele.

Podemos utilizar o `strace`.

Por exemplo:

```bash
sudo strace -f \
  -o /tmp/runc-trace.log \
  runc --root /run/runc-lab \
  run --bundle "$HOME/oci-lab-runc/bundle" trace-lab
```

Esse comando executa o container e registra as chamadas de sistema do runtime e dos processos acompanhados.

Depois podemos pesquisar:

```bash
grep -E 'clone|clone3|unshare|setns' /tmp/runc-trace.log
```

Para observar operações de montagem:

```bash
grep -E 'mount|umount2|pivot_root' /tmp/runc-trace.log
```

Para observar chamadas relacionadas a privilégios:

```bash
grep -E 'capset|prctl|setresuid|setresgid' /tmp/runc-trace.log
```

E para localizar operações relacionadas ao Seccomp:

```bash
grep -i seccomp /tmp/runc-trace.log
```

**Observação:** o `strace` pode alterar o comportamento de processos rastreados, especialmente em relação a operações de segurança e depuração. Além disso, chamadas como `clone3()` podem não aparecer dependendo da versão do runtime e do kernel.

Não espere encontrar uma chamada simples para cada recurso configurado. Algumas operações são feitas por processos auxiliares, bibliotecas ou interfaces diferentes do kernel.

---

# 8. Como o config.json representa a segurança

Agora vamos aprofundar a relação entre o arquivo OCI e o Módulo 1.

O arquivo `config.json` não é apenas uma configuração de inicialização. Ele também descreve grande parte da política de segurança do container.

Vamos analisar os principais campos.

## 8.1 Usuário

```json
"process": {
  "user": {
    "uid": 1000,
    "gid": 1000
  }
}
```

Define a identidade Linux utilizada pelo processo.

Isso se relaciona ao DAC e ao conceito de execução com usuário não privilegiado.

## 8.2 Namespaces

```json
"namespaces": [
  {
    "type": "pid"
  },
  {
    "type": "network"
  },
  {
    "type": "mount"
  },
  {
    "type": "ipc"
  },
  {
    "type": "uts"
  }
]
```

Define os namespaces que serão criados.

O campo `path` pode ser utilizado para associar o container a um namespace existente, em vez de criar um novo.

## 8.3 Cgroups

```json
"resources": {
  "memory": {
    "limit": 536870912
  },
  "cpu": {
    "quota": 100000,
    "period": 100000
  },
  "pids": {
    "limit": 100
  }
}
```

Representa:

- Limite de memória de 512 MiB.
- Limite de CPU equivalente a uma CPU.
- Limite de 100 processos/tarefas.

O runtime aplica essas configurações à hierarquia de cgroups.

## 8.4 Capabilities

```json
"capabilities": {
  "bounding": [],
  "effective": [],
  "permitted": [],
  "inheritable": [],
  "ambient": []
}
```

Os conjuntos de capabilities definem os privilégios especiais disponíveis ao processo.

Para uma aplicação que não precisa de privilégios especiais, uma política restritiva pode remover todas as capabilities.

## 8.5 Seccomp

Exemplo:

```json
"seccomp": {
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": [
        "read",
        "write",
        "close",
        "exit",
        "exit_group"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

Define uma política de filtragem das chamadas de sistema.

O exemplo é extremamente restritivo e não deve ser usado como perfil genérico para aplicações.

## 8.6 SELinux

O OCI também possui campos relacionados à configuração SELinux.

Por exemplo, o processo pode receber um contexto:

```json
"process": {
  "selinuxLabel": "system_u:system_r:container_t:s0"
}
```

Esse contexto só fará sentido em um host com SELinux habilitado e com políticas compatíveis.

## 8.7 AppArmor

Também podemos encontrar configuração relacionada ao AppArmor:

```json
"process": {
  "apparmorProfile": "meu-perfil"
}
```

Nesse caso, o runtime tenta associar o processo ao perfil especificado.

A aplicação depende do suporte do kernel e da disponibilidade do perfil no host.

---

# 9. Como isso se relaciona com containerd e CRI-O?

Agora vamos conectar o aprendizado à arquitetura Kubernetes e OpenShift.

## 9.1 Kubernetes com containerd

Uma arquitetura típica:

```text
             Kubernetes
                  |
                  v
                kubelet
                  |
                  v
                 CRI
                  |
                  v
              containerd
                  |
                  v
            Runtime Handler
                  |
                  v
                 runc
                  |
                  v
             Linux Kernel
```

O kubelet não executa diretamente o `runc`.

Ele utiliza a CRI para conversar com o runtime de containers.

O containerd recebe as solicitações e coordena a criação e o gerenciamento dos containers.

## 9.2 OpenShift com CRI-O

No OpenShift, uma arquitetura comum é:

```text
             Kubernetes
                  |
                  v
                kubelet
                  |
                  v
                 CRI
                  |
                  v
                CRI-O
                  |
                  v
                 crun
                  |
                  v
             Linux Kernel
```

O CRI-O é um runtime de containers desenvolvido especificamente para atender às necessidades da CRI do Kubernetes.

Ele utiliza runtimes OCI para executar os containers.

O `crun` é bastante comum nesse ecossistema.

### Uma distinção importante

Não existe uma obrigação de usar exclusivamente:

- containerd + runc;
- CRI-O + crun.

São combinações comuns, mas existem outras possibilidades, dependendo da configuração e da versão do ambiente.

---

# 10. Comparando os dois runtimes no contexto de segurança

Vamos consolidar.

| Mecanismo | runc | crun |
|---|---|---|
| Namespaces | Configura | Configura |
| Cgroups | Configura | Configura |
| Root filesystem | Prepara | Prepara |
| Mounts | Configura | Configura |
| Capabilities | Aplica | Aplica |
| Seccomp | Aplica | Aplica |
| AppArmor | Suporta | Suporta |
| SELinux | Suporta | Suporta |
| OCI Runtime Specification | Implementa | Implementa |

A principal diferença não está nos conceitos de segurança, mas na implementação.

Ambos precisam respeitar as especificações e as limitações do kernel Linux.

---

# 11. Exercícios de fixação

Sugiro os seguintes exercícios para consolidar a aula.

### Exercício 1 — Ciclo de vida

Crie um container utilizando `runc create`.

Verifique o estado antes e depois de executar `runc start`.

Explique a diferença entre `created` e `running`.

### Exercício 2 — Comparação

Execute o mesmo bundle OCI com `runc` e `crun`.

Compare:

```bash
runc state ID
```

Com:

```bash
crun state ID
```

Investigue os processos no host.

### Exercício 3 — Namespaces

Execute:

```bash
lsns -p PID
```

Compare os namespaces criados pelos dois runtimes.

### Exercício 4 — Seccomp

Altere o `config.json` para bloquear a System Call `uname()`.

Execute o container e tente:

```bash
uname -a
```

Observe o comportamento.

### Exercício 5 — Capabilities

Configure o container para executar sem capabilities.

Verifique:

```bash
grep Cap /proc/PID/status
```

Compare com um container configurado com algumas capabilities.

---

# 12. Conclusão

Nesta aula, vimos que o runtime OCI é o componente responsável por transformar uma configuração declarativa em um ambiente de execução real no kernel Linux.

O fluxo conceitual é:

```text
       config.json
            |
            v
       OCI Runtime
            |
     +------+------+ 
     |      |      |
     v      v      v
 Namespaces Cgroups Mounts
     |      |      |
     +------+------+ 
            |
     +------+------+ 
     |      |      |
     v      v      v
 Capabilities Seccomp LSM
     |      |      |
     +------+------+ 
            |
            v
      Processo Linux
```

A partir daqui, temos uma visão muito mais concreta de como os conceitos do Módulo 1 são aplicados na execução de containers.

## Próxima aula — 2.4: Filesystem de Containers — OverlayFS, Copy-on-Write e Snapshotters

Vamos estudar mais profundamente:

1. Como funciona o OverlayFS.
2. O que são `lowerdir`, `upperdir`, `workdir` e `merged`.
3. Como funciona o Copy-on-Write.
4. Como o Docker utiliza OverlayFS.
5. Como o containerd utiliza snapshotters.
6. Diferenças entre OverlayFS, Btrfs e ZFS.
7. Implicações de segurança relacionadas aos mounts e ao filesystem.

Essa aula será especialmente interessante porque permitirá relacionar o funcionamento interno dos containers ao seu estudo de Btrfs.
