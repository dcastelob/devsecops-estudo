# Módulo 2 — Docker e OCI Runtime Security
## Aula 2.5 — Volumes, Bind Mounts e Isolamento de Filesystem

Nesta aula vamos aprofundar um dos pontos mais sensíveis da segurança de containers: **a relação entre o filesystem isolado do container e o filesystem real do host**.

Até agora estudamos como o OverlayFS constrói uma visão unificada das camadas de uma imagem. Mas essa visão não precisa conter apenas arquivos da imagem: o runtime também pode montar diretórios, volumes e filesystems externos dentro dela.

É justamente aí que surgem alguns dos riscos mais importantes de segurança.

Vamos seguir uma abordagem prática no Debian 13, utilizando Docker e comandos nativos do Linux, e depois relacionar os conceitos com Kubernetes e OpenShift.

## 1. Os três principais tipos de armazenamento no Docker

O Docker oferece três mecanismos muito utilizados para disponibilizar armazenamento aos containers:

1. Volumes.
2. Bind mounts.
3. Tmpfs mounts.

Embora todos possam disponibilizar diretórios dentro do container, eles possuem características e implicações de segurança diferentes.

### 1.1. Volumes Docker

Um volume é uma unidade de armazenamento gerenciada pelo Docker.

O Docker controla sua criação, localização, montagem e remoção.

Por exemplo:

```bash
docker volume create dados-app
```

Consultar:

```bash
docker volume ls
```

Inspecionar:

```bash
docker volume inspect dados-app
```

Você encontrará informações semelhantes a:

```json
{
    "Driver": "local",
    "Mountpoint": "/var/lib/docker/volumes/dados-app/_data",
    "Name": "dados-app",
    "Scope": "local"
}
```

O diretório `Mountpoint` corresponde à localização física do volume no host, no caso do driver `local`.

Agora vamos utilizá-lo:

```bash
docker run --rm -it \
  --mount type=volume,source=dados-app,target=/dados \
  alpine sh
```

Dentro do container:

```bash
echo "Informacao persistente" > /dados/arquivo.txt
```

Saia:

```bash
exit
```

Crie outro container utilizando o mesmo volume:

```bash
docker run --rm -it \
  --mount type=volume,source=dados-app,target=/dados \
  alpine sh
```

Consulte:

```bash
cat /dados/arquivo.txt
```

O arquivo continua existindo.

Isso acontece porque o volume não pertence ao ciclo de vida de um container específico.

```text
                Docker Host
                    |
                    |
             Volume dados-app
                    |
              /var/lib/docker/
              volumes/dados-app/
                    |
             +------+------+
             |             |
             v             v
         Container A   Container B
           /dados        /dados
```

**Uma característica importante:** volumes Docker são persistentes em relação ao ciclo de vida dos containers, mas não são automaticamente backups. Se o volume for removido ou o disco apresentar falha, os dados poderão ser perdidos.

#### O que acontece com o conteúdo inicial do diretório?

Existe um comportamento interessante.

Imagine uma imagem que já possua:

```text
/var/lib/app/
└── config.yml
```

Se você montar um volume Docker vazio nesse diretório:

```bash
docker run --rm -it \
  -v dados-app:/var/lib/app \
  minha-imagem
```

Por padrão, o Docker pode copiar o conteúdo inicial do diretório da imagem para o volume vazio.

Esse comportamento é conhecido como *volume population*.

É possível desabilitá-lo com:

```bash
docker run --rm -it \
  --mount type=volume,source=dados-app,target=/var/lib/app,volume-nocopy \
  minha-imagem
```

Isso é relevante porque nem sempre queremos que o conteúdo da imagem seja copiado automaticamente para o volume.

### 1.2. Bind mounts

Um bind mount disponibiliza um arquivo ou diretório existente no host dentro do container.

Diferentemente dos volumes, o armazenamento não precisa ser criado ou gerenciado pelo Docker.

Considere:

```bash
mkdir -p ~/lab-docker
echo "Arquivo do host" > ~/lab-docker/arquivo.txt
```

Monte o diretório:

```bash
docker run --rm -it \
  --mount type=bind,source="$HOME/lab-docker",target=/lab-docker \
  alpine sh
```

Dentro do container:

```bash
cat /lab-docker/arquivo.txt
```

Você estará acessando o mesmo arquivo que existe no host.

Faça uma alteração:

```bash
echo "Alteracao feita pelo container" >> /lab-docker/arquivo.txt
```

Saia e consulte no host:

```bash
cat ~/lab-docker/arquivo.txt
```

A alteração estará presente.

A representação é:

```text
              Docker Host
                  |
                  |
          /home/usuario/lab-docker
                  |
                  |
             Bind Mount
                  |
                  v
          /lab-docker
             Container
```

Observe a diferença:

- Volume: Docker gerencia o armazenamento.
- Bind mount: você escolhe um caminho existente no host.

#### Bind mount somente leitura

Um dos principais cuidados de segurança é evitar disponibilizar escrita no host quando ela não for necessária.

Exemplo:

```bash
docker run --rm -it \
  --mount type=bind,source="$HOME/lab-docker",target=/lab-docker,readonly \
  alpine sh
```

Tente:

```bash
touch /lab-docker/novo-arquivo.txt
```

A operação deverá falhar.

O container consegue consultar os arquivos, mas não modificá-los por meio dessa montagem.

**Atenção:** um bind mount somente leitura não impede que o container leia informações sensíveis. Expor `/etc`, diretórios de usuários ou arquivos de configuração continua sendo uma decisão de segurança relevante.

### 1.3. Tmpfs mounts

Um tmpfs é um filesystem temporário mantido na memória virtual do sistema.

Ele não é armazenado como um volume persistente convencional.

Exemplo:

```bash
docker run --rm -it \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=64m \
  alpine sh
```

Dentro do container:

```bash
echo "temporario" > /tmp/arquivo.txt
```

Consulte:

```bash
df -h /tmp
```

O filesystem será apresentado como `tmpfs`.

Quando o container é encerrado e o mount deixa de existir, os dados temporários são descartados.

É útil para:

- Arquivos temporários.
- Cache.
- Dados transitórios.
- Arquivos que não devem ser persistidos no armazenamento convencional do container.

Entretanto, atenção: tmpfs utiliza memória e pode utilizar swap, dependendo da configuração do host. Portanto, não deve ser considerado automaticamente um mecanismo de armazenamento criptografado ou de proteção contra recuperação de dados.

### 1.4. Comparação dos três mecanismos

| Característica | Volume | Bind mount | Tmpfs |
|---|---|---|---|
| Gerenciado pelo Docker | Sim | Não | Sim, como mount |
| Origem | Volume Docker | Caminho do host | Memória virtual |
| Persistência após remover container | Sim | Sim, no host | Não |
| Compartilhamento entre containers | Sim | Sim | Possível, conforme configuração |
| Acesso direto ao filesystem do host | Através do mountpoint | Sim, ao caminho escolhido | Não corresponde a diretório persistente do host |
| Uso típico | Dados persistentes | Configurações e arquivos do host | Temporários |

**Recomendação de segurança:** utilize volumes para dados persistentes, bind mounts apenas quando necessários e tmpfs para dados temporários que não precisam ser persistidos.

---

# 2. Como funcionam os mounts no Linux?

Agora vamos sair um pouco do Docker.

Para compreender o isolamento de filesystem, precisamos entender como o próprio Linux trabalha com montagens.

No Linux, o comando `mount` associa um filesystem ou uma árvore de diretórios a um ponto da árvore de diretórios existente.

Por exemplo:

```bash
sudo mount /dev/sdb1 /mnt
```

A partir desse momento, o conteúdo do filesystem de `/dev/sdb1` estará acessível através de `/mnt`.

Mas existe um detalhe importante: **um mount não é simplesmente uma operação que copia arquivos de um local para outro**.

Ele cria uma associação dentro da estrutura de montagens do kernel.

## 2.1. Consultando os mounts

O Linux disponibiliza informações sobre as montagens de cada processo em:

```text
/proc/<PID>/mountinfo
```

Para consultar as montagens do seu shell:

```bash
cat /proc/self/mountinfo
```

Ou de maneira mais amigável:

```bash
findmnt
```

Para visualizar apenas a árvore:

```bash
findmnt -R /
```

O arquivo `mountinfo` é especialmente importante porque representa a visão de montagens associada ao namespace de montagem do processo.

Não confunda:

```bash
cat /proc/mounts
```

com:

```bash
cat /proc/self/mountinfo
```

O primeiro apresenta uma visão das montagens disponível por meio da interface `/proc`; o segundo fornece informações mais detalhadas, incluindo identificadores de mount, relação entre mounts e campos relacionados à propagação.

## 2.2. Mounts são objetos do kernel

Quando você executa:

```bash
mount --bind /home/usuario/dados /mnt/dados
```

O kernel cria uma nova referência de montagem para disponibilizar aquela árvore em outro ponto.

O conteúdo não é copiado.

Assim:

```text
/home/usuario/dados
          |
          |
       Bind Mount
          |
          v
      /mnt/dados
```

Os dois caminhos apresentam o mesmo conteúdo subjacente.

Se você alterar um arquivo por um dos caminhos, a alteração será visível pelo outro.

## 2.3. Bind mount recursivo

Existe uma diferença importante entre:

```bash
mount --bind origem destino
```

e:

```bash
mount --rbind origem destino
```

O `--bind` monta a árvore indicada, mas não replica recursivamente todos os submounts existentes nela.

O `--rbind` replica também os submounts.

Imagine:

```text
/srv/dados
├── arquivo1
└── backup
    └── (outro filesystem montado)
```

Com `--rbind`, a montagem de `backup` também poderá aparecer no destino.

Isso é relevante para segurança porque um diretório aparentemente simples pode conter outros mounts, inclusive volumes e filesystems que você não pretendia disponibilizar ao container.

---

---

# 3. Mount namespaces: como o container enxerga seu próprio filesystem?

Aqui entramos em um dos conceitos centrais do isolamento de containers.

O Linux possui diferentes tipos de namespaces. Um deles é o **Mount Namespace (`mnt`)**.

Ele permite que diferentes processos tenham visões distintas da árvore de montagens do sistema.

Imagine o host:

```text
HOST
/
├── etc
├── home
├── proc
├── sys
├── var
└── mnt
```

Um processo dentro de outro mount namespace pode ter uma árvore diferente:

```text
CONTAINER
/
├── etc
├── home
├── proc
├── sys
├── var
└── dados
```

Os caminhos podem ter os mesmos nomes, mas as montagens associadas a eles podem ser diferentes.

**Importante:** o mount namespace não cria, sozinho, um filesystem isolado. Ele isola a visão da árvore de montagens. Os arquivos continuam sendo objetos do kernel e podem ser compartilhados entre namespaces quando são disponibilizados por bind mounts, volumes ou outros mecanismos.

## 3.1. Comparando os namespaces do host e do container

Vamos utilizar Docker para observar isso.

Crie um container:

```bash
docker run -d --name mount-lab alpine sleep 3600
```

Descubra o PID do processo principal no host:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' mount-lab)

echo "$PID"
```

Consulte o namespace de montagem do host:

```bash
readlink /proc/self/ns/mnt
```

Agora consulte o namespace do container:

```bash
sudo readlink /proc/"$PID"/ns/mnt
```

Você deverá encontrar identificadores diferentes.

Exemplo ilustrativo:

```text
mnt:[4026531841]
mnt:[4026532765]
```

Isso demonstra que o processo do container está em outro mount namespace.

## 3.2. Comparando as árvores de montagem

No host:

```bash
findmnt
```

No container:

```bash
docker exec mount-lab sh -c 'cat /proc/self/mountinfo'
```

Ou, para uma visualização mais amigável:

```bash
docker exec mount-lab sh -c 'cat /proc/mounts'
```

Observe que o container possui uma árvore de montagens diferente da árvore do host.

Você também pode consultar o `mountinfo` diretamente pelo PID:

```bash
sudo cat /proc/"$PID"/mountinfo
```

Isso é útil quando você precisa investigar um container sem entrar nele.

## 3.3. Entrando no mount namespace

O comando `nsenter` permite executar um processo dentro dos namespaces de outro processo.

Por exemplo:

```bash
sudo nsenter -t "$PID" -m findmnt
```

O parâmetro:

```text
-m
```

significa entrar no mount namespace.

Você também pode entrar em outros namespaces:

| Parâmetro | Namespace |
|---|---|
| `-m` | Mount |
| `-p` | PID |
| `-n` | Network |
| `-i` | IPC |
| `-u` | UTS |
| `-U` | User |
| `-C` | Cgroup |
| `-T` | Time |

Para entrar somente no namespace de montagem, mantendo os demais namespaces do seu processo, utilize `-m`.

Para investigar o container de maneira mais abrangente:

```bash
sudo nsenter -t "$PID" -m -p -n -u -i sh
```

O comando pode exigir cuidados adicionais dependendo da combinação de namespaces e do ambiente. Para esta aula, o importante é perceber que o `nsenter` permite investigar a visão do filesystem de outro processo.

Ao terminar:

```bash
docker rm -f mount-lab
```

---

# 4. pivot_root: como o container recebe seu próprio `/`?

Agora vamos estudar uma operação fundamental na construção do filesystem de um container.

Quando um processo é iniciado no host, ele normalmente utiliza a raiz do filesystem do host:

```text
/
├── etc
├── home
├── usr
├── var
└── ...
```

Mas o processo de um container precisa utilizar outro diretório como sua raiz.

Por exemplo:

```text
/var/lib/docker/overlay2/.../merged
```

Esse diretório contém a visão unificada do filesystem da imagem.

Como o runtime transforma esse diretório em `/` para o processo do container?

Uma das operações utilizadas é `pivot_root()`.

## 4.1. O que é pivot_root?

`pivot_root()` é uma syscall do Linux que troca a raiz do filesystem associada ao mount namespace atual.

Ela recebe dois caminhos:

- `new_root`: novo diretório que será utilizado como `/`.
- `put_old`: diretório dentro do novo root onde a raiz antiga será colocada temporariamente.

Conceitualmente:

Antes:

```text
Mount Namespace
       |
       v
      HOST /
       |
       +-- etc
       +-- usr
       +-- var
       |
       +-- novo-root
             |
             +-- etc
             +-- usr
             +-- bin
```

Depois:

```text
Mount Namespace
       |
       v
    novo-root
       |
       +-- etc
       +-- usr
       +-- bin
       +-- old-root
```

O processo passa a enxergar o conteúdo de `novo-root` como `/`.

A raiz anterior fica acessível temporariamente pelo caminho `put_old`, até ser desmontada.

## 4.2. pivot_root versus chroot

Essa distinção é muito importante.

O comando:

```bash
chroot /novo-root /bin/sh
```

altera a raiz percebida pelo processo, mas não cria, por si só, isolamento de mounts ou de namespaces.

Historicamente, `chroot` não foi concebido como uma fronteira de segurança completa.

Já `pivot_root()` trabalha com a árvore de montagens do mount namespace.

Mesmo assim, `pivot_root()` isoladamente não transforma um processo em um container seguro. É necessário combinar:

- Mount namespaces.
- PID namespaces.
- Network namespaces.
- User namespaces.
- Cgroups.
- Capabilities.
- Seccomp.
- AppArmor ou SELinux.
- Configuração adequada dos mounts.

## 4.3. Como isso se relaciona com o runc?

Quando o runtime OCI inicia um container, ele prepara o ambiente definido no `config.json`.

Entre outras operações, o runtime precisa:

1. Preparar o root filesystem.
2. Criar ou entrar nos namespaces.
3. Configurar os mounts.
4. Tornar o novo root acessível.
5. Realizar a troca de raiz.
6. Aplicar as restrições de segurança.
7. Executar o processo configurado.

A implementação exata e a ordem de algumas operações variam conforme runtime, versão, namespaces e configuração.

O `pivot_root()` é uma das peças que ajudam a transformar o rootfs preparado em uma raiz independente para o processo.

### Uma observação técnica

`pivot_root()` possui restrições: por exemplo, o novo root precisa estar em um mount adequado e o processo precisa estar no mount namespace correto. Em geral, o runtime também precisa tratar a propagação dos mounts e desmontar a raiz antiga.

---

# 5. Mount propagation: private, shared e slave

Este é um dos assuntos mais importantes e, frequentemente, menos compreendidos na administração de containers.

Até agora vimos que um mount namespace possui uma visão própria das montagens.

Mas imagine a seguinte situação:

1. O host possui `/mnt/dados`.
2. O diretório é disponibilizado dentro de um container.
3. Outro filesystem é montado dentro de `/mnt/dados` no host.
4. O container deverá enxergar essa nova montagem?

A resposta depende da configuração de **mount propagation**.

A propagação determina como eventos de montagem e desmontagem são compartilhados entre mounts relacionados.

## 5.1. Shared

Um mount `shared` permite que eventos de montagem e desmontagem sejam propagados entre os membros de um grupo compartilhado.

Exemplo conceitual:

```text
HOST
/mnt/dados
     |
     | shared
     |
     +------------------+
                        |
                        v
                 CONTAINER
                 /dados
```

Se um novo filesystem for montado dentro de `/mnt/dados` no host, a montagem poderá aparecer também no container, desde que os mounts estejam configurados no mesmo grupo de propagação e que as condições de propagação sejam atendidas.

A propagação é bidirecional entre membros do grupo compartilhado.

## 5.2. Slave

Um mount `slave` recebe eventos de montagem de seu mount mestre, mas não propaga eventos de volta para ele.

Representação:

```text
HOST
/mnt/dados
    |
    | shared
    |
    v
CONTAINER
/dados
    |
    | slave
    |
    v
```

Conceitualmente:

- O host monta algo dentro da árvore compartilhada.
- O evento pode chegar ao container.
- Uma montagem originada no lado slave não é propagada de volta ao mestre.

É uma forma de permitir propagação em uma única direção.

## 5.3. Private

Um mount `private` não compartilha eventos de montagem com outros mounts.

```text
HOST
/mnt/dados
    |
    X
    |
CONTAINER
/dados
```

Se um novo filesystem for montado dentro de `/mnt/dados` no host, essa montagem não será propagada para o mount privado do container.

Da mesma maneira, eventos originados no container não serão propagados para o host.

## 5.4. Resumo

| Propagação | Recebe eventos do mestre | Envia eventos ao mestre |
|---|---:|---:|
| `shared` | Sim, entre membros do grupo | Sim |
| `slave` | Sim | Não |
| `private` | Não | Não |

Existe também o modo `unbindable`, que impede que aquele mount seja utilizado como origem de um bind mount.

### E o prefixo `r`?

Você encontrará opções como:

```text
rprivate
rshared
rslave
```

O prefixo `r` significa recursivo.

Isso indica que a configuração será aplicada também aos submounts da árvore.

Por exemplo:

```bash
mount --make-rprivate /mnt/dados
```

Aplica o modo private recursivamente.

Enquanto:

```bash
mount --make-private /mnt/dados
```

Aplica a alteração ao mount indicado, sem necessariamente modificar todos os submounts.

## 5.5. Como consultar a propagação?

Execute:

```bash
findmnt -o TARGET,PROPAGATION
```

Ou:

```bash
findmnt -o TARGET,PROPAGATION / 
```

No arquivo `/proc/self/mountinfo`, procure campos como:

```text
shared:123
master:123
propagate_from:123
```

Esses campos ajudam a identificar os relacionamentos entre grupos de propagação.

Se não houver um campo de compartilhamento correspondente, o mount pode estar em modo privado.

### Por que isso importa para containers?

Imagine que você disponibilize `/mnt` do host em um container que executa um agente de monitoramento.

Se a propagação for configurada de maneira inadequada, novos mounts realizados no host podem aparecer dentro do container, ou mounts criados pelo container podem afetar o host.

Isso pode ser necessário em aplicações especializadas, mas aumenta a complexidade e o risco operacional.

**Regra prática:** prefira `private` ou `rprivate` quando não houver necessidade de propagar eventos de montagem. Utilize `shared` e `slave` conscientemente.

---

---

# 6. Riscos de segurança ao expor diretórios do host

Agora vamos relacionar os conceitos anteriores a situações reais de segurança.

O maior risco de um bind mount é disponibilizar ao container arquivos ou diretórios que pertencem ao host.

O container pode ter namespaces isolados, cgroups, Seccomp e capabilities reduzidas. Porém, se você disponibilizar um arquivo sensível por meio de um mount, o processo poderá acessá-lo conforme as permissões e restrições aplicáveis.

## 6.1. Exposição do diretório raiz do host

Considere:

```bash
docker run --rm -it \
  --mount type=bind,source=/,target=/host \
  alpine sh
```

Dentro do container:

```bash
ls /host
```

Você estará visualizando a árvore de diretórios do host.

Dependendo das permissões e das opções de montagem, poderá também modificar arquivos.

Isso é especialmente perigoso quando o container possui privilégios elevados.

## 6.2. Exposição do socket Docker

Um dos exemplos mais perigosos é disponibilizar o socket do Docker:

```bash
-v /var/run/docker.sock:/var/run/docker.sock
```

Imagine:

```bash
docker run -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  docker:cli sh
```

O processo dentro do container poderá se comunicar com o Docker Engine do host.

Se possuir acesso irrestrito ao socket, poderá solicitar a criação de containers com privilégios elevados, mounts do host e outras configurações perigosas.

Por exemplo, um cliente Docker com acesso irrestrito ao daemon pode solicitar um container que monte `/` do host.

Portanto:

**Ter acesso ao socket Docker equivale, em muitos cenários, a ter controle administrativo sobre o host.**

Não disponibilize esse socket para aplicações comuns.

## 6.3. Exposição de `/proc`, `/sys` e `/dev`

Esses três diretórios merecem atenção especial.

| Diretório | Conteúdo |
|---|---|
| `/proc` | Informações de processos e interfaces do kernel |
| `/sys` | Informações e interfaces de dispositivos e subsistemas do kernel |
| `/dev` | Arquivos de dispositivos |

Os containers normalmente recebem versões controladas dessas estruturas.

Por exemplo, o `/proc` de um container é preparado considerando seu PID namespace e as configurações do runtime.

O `/sys` também pode ter restrições de montagem e acesso.

O `/dev` é disponibilizado de forma controlada, com dispositivos selecionados.

O risco aparece quando você disponibiliza diretamente estruturas do host:

```bash
-v /sys:/host-sys
-v /proc:/host-proc
-v /dev:/host-dev
```

Isso pode expor informações e interfaces que não deveriam estar disponíveis para a aplicação.

## 6.4. `hostPath` e o risco de escape de isolamento

No Kubernetes, o equivalente conceitual a disponibilizar um diretório do host é o volume `hostPath`.

Exemplo:

```yaml
volumes:
  - name: host-root
    hostPath:
      path: /
      type: Directory
```

Depois:

```yaml
volumeMounts:
  - name: host-root
    mountPath: /host
```

O container passa a acessar o filesystem do host por `/host`.

Se combinado com permissões de escrita, privilégios elevados ou acesso a interfaces sensíveis, o risco é significativo.

O `hostPath` não é automaticamente uma vulnerabilidade, mas aumenta muito a superfície de ataque.

Use somente quando existir uma necessidade operacional bem definida, como alguns agentes de monitoramento ou componentes de infraestrutura.

---

# 7. Volumes no Kubernetes: emptyDir, hostPath, PVC e CSI

Agora vamos estudar os mecanismos de armazenamento do Kubernetes.

Existe uma distinção importante:

- Volume: recurso que disponibiliza armazenamento a um Pod.
- PersistentVolume (PV): representação de um recurso de armazenamento persistente no cluster.
- PersistentVolumeClaim (PVC): solicitação de armazenamento feita por uma aplicação.
- CSI: interface que permite integrar soluções de armazenamento ao Kubernetes.

## 7.1. emptyDir

O `emptyDir` cria um diretório temporário associado ao ciclo de vida do Pod.

Exemplo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-lab
spec:
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: dados
          mountPath: /dados
  volumes:
    - name: dados
      emptyDir: {}
```

Aplicar:

```bash
kubectl apply -f emptydir.yaml
```

Entrar:

```bash
kubectl exec -it emptydir-lab -- sh
```

Criar um arquivo:

```bash
echo "dados temporarios" > /dados/arquivo.txt
```

O conteúdo sobrevive à reinicialização do container enquanto o Pod continuar existindo, mas não deve ser considerado persistente após a remoção do Pod.

### emptyDir em memória

Você também pode utilizar memória:

```yaml
volumes:
  - name: temporario
    emptyDir:
      medium: Memory
      sizeLimit: 64Mi
```

O Kubernetes disponibiliza esse volume utilizando um filesystem em memória, normalmente tmpfs.

É útil para dados temporários que não precisam ser gravados no armazenamento convencional do nó.

Lembre-se de que o consumo de memória desse volume pode afetar o limite de memória do Pod.

## 7.2. hostPath

O `hostPath` monta um caminho existente no nó Kubernetes.

Exemplo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-lab
spec:
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: host
          mountPath: /host-data
          readOnly: true
  volumes:
    - name: host
      hostPath:
        path: /var/log
        type: Directory
```

Nesse exemplo, o container terá acesso de leitura ao diretório `/var/log` do nó onde o Pod estiver executando.

Observe a combinação:

```yaml
readOnly: true
```

Ela impede a escrita através daquela montagem.

Ainda assim, informações de logs podem ser sensíveis.

### Tipos de hostPath

O Kubernetes permite especificar o tipo do caminho:

| Tipo | Comportamento |
|---|---|
| `DirectoryOrCreate` | Cria o diretório caso não exista |
| `Directory` | Exige um diretório existente |
| `FileOrCreate` | Cria um arquivo caso não exista |
| `File` | Exige um arquivo existente |
| `Socket` | Exige um socket existente |
| `CharDevice` | Exige um dispositivo de caractere |
| `BlockDevice` | Exige um dispositivo de bloco |

O `hostPath` deve ser tratado com bastante cautela, principalmente em clusters multiusuários.

## 7.3. PersistentVolume e PersistentVolumeClaim

O Kubernetes abstrai o armazenamento persistente através de PVs e PVCs.

A aplicação normalmente não precisa conhecer detalhes de implementação do armazenamento.

Exemplo conceitual:

```text
             Aplicação
                 |
                 v
                Pod
                 |
                 v
                PVC
                 |
                 v
                PV
                 |
                 v
          Storage Backend
```

O backend pode ser:

- NFS.
- Ceph.
- EBS.
- EFS.
- Azure Disk.
- GCE Persistent Disk.
- Outros sistemas de armazenamento.

O PVC representa a solicitação de armazenamento.

Exemplo:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dados-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

A aplicação utiliza:

```yaml
volumeMounts:
  - name: dados
    mountPath: /var/lib/app
volumes:
  - name: dados
    persistentVolumeClaim:
      claimName: dados-app
```

A aplicação acessa `/var/lib/app`, sem precisar conhecer diretamente a implementação física do armazenamento.

**Importante:** `ReadWriteOnce`, `ReadOnlyMany` e `ReadWriteMany` são modos de acesso do volume no Kubernetes. Eles não devem ser interpretados como substitutos de permissões Unix, controles de acesso ou garantias de segurança entre aplicações.

## 7.4. CSI — Container Storage Interface

O CSI é uma interface padronizada para integrar sistemas de armazenamento ao Kubernetes.

Em vez de cada fornecedor precisar implementar sua integração diretamente no código do Kubernetes, ele disponibiliza um driver CSI.

Arquitetura simplificada:

```text
                Kubernetes
                    |
                    v
              PersistentVolume
                    |
                    v
              CSI Controller
                    |
                    v
              Storage Backend
                    |
                    v
              CSI Node Plugin
                    |
                    v
              Mount no Node
                    |
                    v
                 Container
```

O CSI pode participar de operações como:

- Provisionamento de volumes.
- Exclusão de volumes.
- Attach e detach.
- Mount e unmount.
- Expansão de volumes.
- Snapshots, quando suportados.

Um ponto importante: CSI não é um tipo de volume equivalente a `emptyDir` ou `hostPath`. É uma interface de integração entre Kubernetes e os sistemas de armazenamento.

---

# 8. Mount propagation no Kubernetes

O Kubernetes também permite configurar a propagação de montagens por meio de `mountPropagation`.

Existem três valores principais:

| Valor | Comportamento |
|---|---|
| `None` | Não solicita propagação adicional; comportamento padrão do Kubernetes |
| `HostToContainer` | Propaga novas montagens do host para o container |
| `Bidirectional` | Propaga montagens nos dois sentidos |

Exemplo:

```yaml
volumeMounts:
  - name: dados
    mountPath: /dados
    mountPropagation: HostToContainer
```

Com `HostToContainer`, novas montagens realizadas no host dentro da árvore compartilhada podem aparecer no container.

Com `Bidirectional`, montagens realizadas pelo container também podem ser propagadas de volta para o host.

Isso exige cuidado especial.

O modo `Bidirectional` pode permitir que o container influencie a árvore de montagens do host. É uma configuração reservada a workloads especializados e privilegiados.

Em particular, o Kubernetes exige condições adicionais para utilizar `Bidirectional`, incluindo execução em um contexto privilegiado.

Para workloads comuns, prefira não configurar propagação adicional.

---

---

# 9. Técnicas de hardening para filesystem de containers

Vamos consolidar algumas práticas que você pode aplicar tanto no Docker quanto no Kubernetes.

## 9.1. Evite montar o filesystem do host

Evite configurações como:

```bash
-v /:/host
-v /etc:/host-etc
-v /var/run/docker.sock:/var/run/docker.sock
```

Quando precisar de um bind mount, disponibilize somente o diretório estritamente necessário.

## 9.2. Prefira mounts somente leitura

Docker:

```bash
docker run --rm \
  --mount type=bind,source=/dados/config,target=/config,readonly \
  alpine
```

Kubernetes:

```yaml
volumeMounts:
  - name: configuracoes
    mountPath: /config
    readOnly: true
```

## 9.3. Utilize filesystem raiz somente leitura

Docker:

```bash
--read-only
```

Kubernetes:

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

Se a aplicação precisar escrever arquivos temporários, disponibilize diretórios específicos por meio de `tmpfs` ou `emptyDir`.

## 9.4. Execute com usuário não privilegiado

Docker:

```bash
docker run --user 10001:10001 alpine
```

Kubernetes:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001
  allowPrivilegeEscalation: false
```

O UID utilizado precisa ter permissões adequadas nos volumes montados.

No OpenShift, considere que o SCC pode atribuir um UID aleatório dentro de uma faixa autorizada. Por isso, imagens e volumes precisam estar preparados para execução com UID arbitrário.

## 9.5. Combine com outras camadas de segurança

Uma configuração de segurança para Kubernetes pode combinar:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

Essas configurações complementam o isolamento do filesystem.

Nenhuma delas, isoladamente, substitui as demais.

## 9.6. Cuidado com SELinux e AppArmor

Os mounts também interagem com os mecanismos MAC estudados na aula 1.6.

No SELinux, o contexto de segurança dos arquivos pode impedir que um container acesse determinado volume, mesmo quando as permissões Unix parecem permitir.

No Docker, existem opções de rotulagem para bind mounts em ambientes SELinux, como `:z` e `:Z`. Elas têm semânticas diferentes e devem ser utilizadas com cuidado, pois podem alterar os rótulos dos arquivos no host.

No OpenShift, a integração com SELinux é especialmente importante. Não é recomendável desabilitar o isolamento de rótulos para resolver problemas de permissões sem entender a causa.

No AppArmor, o perfil pode restringir operações de montagem e acessos a caminhos, complementando as restrições do mount namespace.

---

# 10. Laboratório prático: investigando mounts e isolamento

Vamos fazer um laboratório que reúne boa parte dos conceitos da aula.

### 10.1. Preparação

No host:

```bash
mkdir -p ~/lab-mount/config
mkdir -p ~/lab-mount/dados

echo "configuracao do host" > ~/lab-mount/config/app.conf
```

### 10.2. Criar um container com diferentes tipos de mount

```bash
docker run -d \
  --name mount-security-lab \
  --read-only \
  --mount type=bind,source="$HOME/lab-mount/config",target=/config,readonly \
  --mount type=bind,source="$HOME/lab-mount/dados",target=/dados \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=32m \
  alpine sleep 3600
```

Temos três situações:

| Caminho no container | Origem | Permissão |
|---|---|---|
| `/` | Root filesystem do container | Somente leitura |
| `/config` | Bind mount do host | Somente leitura |
| `/dados` | Bind mount do host | Leitura e escrita |
| `/tmp` | Tmpfs | Temporário |

### 10.3. Verificar os mounts

```bash
docker exec mount-security-lab sh -c 'cat /proc/self/mountinfo'
```

Ou:

```bash
docker exec mount-security-lab sh -c 'cat /proc/mounts'
```

### 10.4. Testar as permissões

Escrever no root filesystem:

```bash
docker exec mount-security-lab sh -c 'touch /teste'
```

Esperado: falha.

Escrever em `/config`:

```bash
docker exec mount-security-lab sh -c 'touch /config/teste'
```

Esperado: falha.

Escrever em `/dados`:

```bash
docker exec mount-security-lab sh -c 'touch /dados/teste'
```

Esperado: sucesso.

Escrever em `/tmp`:

```bash
docker exec mount-security-lab sh -c 'touch /tmp/teste'
```

Esperado: sucesso.

### 10.5. Comparar os namespaces

No host:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' mount-security-lab)
```

Consultar:

```bash
readlink /proc/self/ns/mnt
sudo readlink /proc/"$PID"/ns/mnt
```

Consultar os mounts do container:

```bash
sudo cat /proc/"$PID"/mountinfo
```

### 10.6. Investigar o bind mount no host

```bash
findmnt -T "$HOME/lab-mount/config"
```

E:

```bash
findmnt -T "$HOME/lab-mount/dados"
```

Observe que os diretórios pertencem ao filesystem do host, mas estão disponibilizados em outro namespace.

### 10.7. Limpeza

```bash
docker rm -f mount-security-lab
rm -rf ~/lab-mount
```

---

# 11. Aplicação no OpenShift: cuidados especiais

Como você trabalha com OpenShift, vale destacar três pontos.

**1. SCC (Security Context Constraints)**

O SCC controla características de segurança dos Pods, incluindo:

- Uso de containers privilegiados.
- Capabilities permitidas.
- Tipos de volumes autorizados.
- Execução como root ou UID restrito.
- Perfis Seccomp, conforme configuração e versão.

Consulte:

```bash
oc get scc
```

E:

```bash
oc describe scc restricted-v2
```

O `hostPath` é um tipo de volume que deve ser cuidadosamente controlado pelas políticas do cluster.

**2. SELinux**

O OpenShift utiliza SELinux para reforçar o isolamento entre workloads.

Um volume pode estar corretamente montado e ainda assim apresentar erros de acesso por causa dos rótulos SELinux.

Não resolva esses problemas desabilitando SELinux ou aplicando permissões excessivas indiscriminadamente.

**3. PVC e permissões**

Um PVC pode estar `Bound` e ainda assim a aplicação não conseguir escrever nele.

É necessário considerar:

- UID/GID do processo.
- Permissões Unix.
- `fsGroup`, quando aplicável.
- Suporte do driver CSI.
- Rótulos SELinux.
- Políticas de segurança do Pod.

Isso é especialmente relevante em imagens que precisam funcionar com UID arbitrário.

---

# 12. Resumo da aula

| Conceito | O que você deve guardar |
|---|---|
| Volume Docker | Armazenamento gerenciado pelo Docker |
| Bind mount | Disponibiliza um caminho do host |
| Tmpfs | Armazenamento temporário em memória virtual |
| Mount namespace | Isola a visão da árvore de montagens |
| `pivot_root` | Troca a raiz do filesystem no mount namespace |
| `chroot` | Altera a raiz percebida, mas não fornece isolamento completo |
| `shared` | Propaga eventos entre mounts do grupo |
| `slave` | Recebe eventos, mas não os propaga de volta |
| `private` | Não propaga eventos |
| `emptyDir` | Volume temporário associado ao Pod |
| `hostPath` | Monta caminho do filesystem do nó |
| PVC | Solicitação de armazenamento persistente |
| CSI | Interface de integração com sistemas de armazenamento |

---

# 13. Exercícios de fixação

Tente responder sem consultar o material:

1. Qual é a diferença fundamental entre um volume Docker e um bind mount?
2. Por que um container com mount namespace isolado ainda pode modificar arquivos do host?
3. Qual é a diferença entre `chroot` e `pivot_root`?
4. O que acontece quando um mount `shared` recebe um novo submount?
5. Qual é a diferença entre `slave` e `private`?
6. Por que montar `/var/run/docker.sock` dentro de um container representa um risco elevado?
7. Qual é a diferença entre `emptyDir` e PVC no Kubernetes?
8. Por que `hostPath` deve ser utilizado com cautela?
9. O que acontece com os dados de um `emptyDir` quando o container reinicia? E quando o Pod é removido?
10. Qual é a relação entre CSI, PV e PVC?

## Próxima aula — 2.6: Segurança do processo dentro do container — PID 1, sinais e ciclo de vida

Na próxima aula vamos estudar outro aspecto fundamental da segurança e operação de containers:

- Como o kernel identifica processos dentro e fora dos PID namespaces.
- O papel do PID 1 dentro do container.
- Por que aplicações podem não receber corretamente sinais `SIGTERM`.
- Diferença entre `ENTRYPOINT`, `CMD` e execução via shell.
- Zombie processes e processos órfãos.
- Uso de `tini` e `dumb-init`.
- Como o runtime encerra containers.
- Relação com `terminationGracePeriodSeconds`, `preStop` e `lifecycle` no Kubernetes.

Esse conhecimento será importante para entender problemas de encerramento de Pods, processos que não terminam corretamente e riscos relacionados à administração de processos dentro de containers.
