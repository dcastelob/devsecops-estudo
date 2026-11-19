# Módulo 1.6 — AppArmor e SELinux: Mandatory Access Control (MAC)

Vamos avançar para outro componente fundamental da segurança de containers: o **Mandatory Access Control (MAC)**.

Até aqui, estudamos mecanismos que controlam diferentes aspectos da execução de processos:

- Namespaces: isolamento de recursos.
- Cgroups: controle de consumo de recursos.
- Capabilities: divisão dos privilégios do root.
- Seccomp: filtragem de chamadas de sistema.

Agora vamos estudar como o Linux controla **quais recursos um processo pode acessar e quais operações pode realizar sobre eles**.

Essa camada é especialmente importante porque, mesmo que uma aplicação esteja executando com usuário não privilegiado e tenha poucas capabilities, ela ainda pode tentar acessar arquivos, diretórios e outros recursos aos quais não deveria ter acesso.

---

## 1. DAC versus MAC: entendendo os modelos de controle de acesso

Antes de estudar AppArmor e SELinux, precisamos entender dois modelos de segurança do Linux.

### 1.1 DAC — Discretionary Access Control

É o modelo tradicional de permissões do Linux.

O DAC considera principalmente:

- Proprietário do arquivo.
- Grupo proprietário.
- Permissões de leitura, escrita e execução.

Por exemplo:

```bash
ls -l /etc/shadow
```

Resultado típico:

```text
-rw-r----- 1 root shadow 1234 /etc/shadow
```

Podemos interpretar:

| Identidade | Permissão |
|---|---|
| Proprietário (root) | Leitura e escrita |
| Grupo (shadow) | Leitura |
| Outros | Nenhuma |

O kernel verifica as permissões considerando a identidade e as credenciais do processo.

Podemos testar:

```bash
sudo -u nobody cat /etc/shadow
```

A operação deverá ser negada.

O DAC é chamado de *Discretionary* porque o proprietário do recurso normalmente pode decidir como compartilhar suas permissões, respeitando as regras do sistema.

### 1.2 MAC — Mandatory Access Control

No MAC, existe uma política de segurança definida pelo administrador ou pelo sistema.

O proprietário de um arquivo não pode simplesmente conceder acesso a qualquer processo se a política MAC proibir essa operação.

Imagine a seguinte situação:

```text
Aplicação: apache

Arquivo: /etc/shadow

Permissões DAC:
    root:shadow 640

Política MAC:
    apache não pode acessar /etc/shadow
```

Mesmo que uma alteração nas permissões DAC permita a leitura, o MAC poderá continuar bloqueando a operação.

Podemos representar a decisão de acesso assim:

```text
             Processo
                 |
                 v
          Solicitação de acesso
                 |
                 v
          +-------------+
          | Verificação |
          | DAC         |
          +------+------+
                 |
                 v
          +-------------+
          | Verificação |
          | MAC         |
          +------+------+
                 |
                 v
          Decisão final
```

Uma operação precisa passar pelos mecanismos de controle aplicáveis. Uma permissão concedida pelo DAC não anula uma restrição imposta pelo MAC.

**Importante:** AppArmor e SELinux são implementações de MAC no Linux, mas utilizam modelos diferentes.

---

# 2. Como o Linux implementa o MAC?

O Linux possui uma infraestrutura chamada **LSM — Linux Security Modules**.

Ela permite que mecanismos de segurança sejam integrados ao kernel e participem das verificações de acesso.

Entre os mecanismos associados ao LSM estão:

- SELinux.
- AppArmor.
- Smack.
- Tomoyo.
- Landlock.

O LSM não é, por si só, um mecanismo de política de segurança. Ele fornece pontos de integração para que os módulos implementem suas próprias regras.

Exemplo simplificado:

```text
                 Aplicação
                     |
                     v
                System Call
                     |
                     v
                  Kernel
                     |
                     v
              LSM Security Hook
                     |
          +----------+----------+
          |                     |
          v                     v
       AppArmor              SELinux
          |                     |
          v                     v
       Perfil                Contexto
       aplicado              de segurança
          |                     |
          +----------+----------+
                     |
                     v
                Decisão MAC
```

Na prática, a arquitetura depende da configuração do kernel. AppArmor e SELinux são normalmente utilizados como mecanismos alternativos de MAC, não como dois sistemas independentes que precisam ser ativados simultaneamente.

## 2.1 Verificando o LSM ativo

No Debian 13, execute:

```bash
cat /sys/kernel/security/lsm
```

Exemplo de resultado:

```text
capability,landlock,yama,apparmor,bpf
```

A lista varia conforme a configuração do kernel.

Também podemos consultar:

```bash
dmesg | grep -i 'LSM:'
```

E:

```bash
cat /proc/cmdline
```

Esse último comando ajuda a identificar parâmetros de inicialização relacionados aos mecanismos de segurança.

---

# 3. AppArmor

O AppArmor é um mecanismo MAC baseado em **perfis associados a aplicações**.

Sua abordagem é relativamente simples de entender:

> Cada aplicação recebe um perfil que determina quais recursos pode acessar e quais operações pode realizar.

Por exemplo, podemos criar um perfil para o Apache permitindo:

- Ler arquivos em `/var/www`.
- Ler configurações em `/etc/apache2`.
- Escrever logs em `/var/log/apache2`.
- Abrir sockets de rede.
- Executar determinados programas auxiliares.

E negar acesso a:

- `/etc/shadow`.
- Diretórios pessoais de outros usuários.
- Arquivos de configuração de outros serviços.
- Outros recursos não previstos na política.

## 3.1 Como o AppArmor identifica os processos?

O AppArmor associa perfis aos executáveis e às transições entre perfis.

Podemos ter:

```text
/usr/sbin/apache2
        |
        v
Perfil AppArmor: usr.sbin.apache2
        |
        +-- /etc/apache2/**      leitura
        |
        +-- /var/www/**          leitura
        |
        +-- /var/log/apache2/**  escrita
        |
        +-- /etc/shadow          negado
```

Diferentemente do SELinux, o AppArmor não depende de atribuir um rótulo de segurança individual a cada arquivo.

Ele utiliza principalmente caminhos de arquivos e regras associadas ao perfil.

## 3.2 Verificando o AppArmor no Debian 13

Verifique o estado:

```bash
sudo aa-status
```

Também podemos verificar:

```bash
systemctl status apparmor
```

Para listar os perfis carregados:

```bash
sudo aa-status --profiled
```

Para identificar os perfis associados aos processos:

```bash
ps -eZ
```

A saída depende do sistema e da ferramenta utilizada; `ps -Z` também pode mostrar informações de contexto de segurança quando disponíveis.

## 3.3 Modos de operação

O AppArmor possui dois modos principais:

| Modo | Comportamento |
|---|---|
| Enforce | Aplica as regras e bloqueia violações |
| Complain | Registra violações, mas não bloqueia as operações |

O modo `Complain` é muito útil durante a criação de perfis.

Podemos consultar o modo de um perfil:

```bash
sudo aa-status
```

Alterar para complain:

```bash
sudo aa-complain /etc/apparmor.d/usr.sbin.apache2
```

Alterar para enforce:

```bash
sudo aa-enforce /etc/apparmor.d/usr.sbin.apache2
```

Esses comandos pressupõem que o perfil exista.

## 3.4 Estrutura de um perfil AppArmor

Vamos observar um exemplo simplificado:

```text
#include <tunables/global>

/usr/bin/minha-aplicacao {
    #include <abstractions/base>

    /usr/bin/minha-aplicacao r,

    /etc/minha-aplicacao/** r,

    /var/log/minha-aplicacao/** rw,

    /tmp/minha-aplicacao/** rw,

    deny /etc/shadow r,
}
```

Algumas permissões:

| Permissão | Significado |
|---|---|
| `r` | Leitura |
| `w` | Escrita |
| `m` | Mapeamento executável |
| `x` | Execução |
| `ix` | Executar mantendo o perfil atual |
| `px` | Executar com transição para outro perfil |
| `ux` | Executar sem confinamento AppArmor |

O AppArmor possui uma linguagem de regras bastante rica. O exemplo acima é apenas uma introdução.

### Um detalhe importante

O perfil precisa considerar bibliotecas, arquivos temporários, sockets, configurações e demais recursos utilizados pela aplicação.

Um perfil excessivamente restritivo pode impedir que uma aplicação funcione.

Por isso, normalmente o desenvolvimento de perfis envolve observação, análise dos logs e ajustes graduais.

---

# 4. Laboratório AppArmor

Vamos criar um pequeno laboratório para entender o funcionamento do mecanismo.

### 4.1 Criando uma aplicação

Crie um script:

```bash
sudo mkdir -p /opt/apparmor-lab

sudo vim /opt/apparmor-lab/teste.sh
```

Conteúdo:

```bash
#!/bin/bash

echo "Aplicacao iniciada"

cat /etc/hostname

cat /etc/shadow

echo "Aplicacao finalizada"
```

Torne-o executável:

```bash
sudo chmod 755 /opt/apparmor-lab/teste.sh
```

Execute:

```bash
sudo /opt/apparmor-lab/teste.sh
```

Como estamos executando com `sudo`, a leitura de `/etc/shadow` provavelmente será permitida pelo DAC.

Agora vamos criar uma política AppArmor que impeça essa aplicação de acessar o arquivo.

### 4.2 Criando um perfil

Crie:

```bash
sudo vim /etc/apparmor.d/opt.apparmor-lab.teste
```

Conteúdo:

```text
#include <tunables/global>

/opt/apparmor-lab/teste.sh {
    #include <abstractions/base>

    /bin/bash ix,

    /opt/apparmor-lab/teste.sh r,

    /etc/hostname r,

    deny /etc/shadow r,
}
```

O perfil é intencionalmente didático. Em um ambiente real, precisaríamos avaliar os caminhos dos executáveis e das bibliotecas utilizadas pelo interpretador.

Carregue:

```bash
sudo apparmor_parser -r /etc/apparmor.d/opt.apparmor-lab.teste
```

Consulte:

```bash
sudo aa-status
```

Execute novamente:

```bash
sudo /opt/apparmor-lab/teste.sh
```

A tentativa de leitura de `/etc/shadow` deverá ser negada pelo AppArmor, mesmo com a aplicação executando como root.

Esse é um dos aspectos mais interessantes do MAC.

**O UID 0 não significa que o processo pode ignorar as restrições do AppArmor.**

Se o laboratório apresentar erros relacionados ao interpretador ou às abstrações, examine os logs antes de ampliar as permissões indiscriminadamente.

---

# 5. SELinux

Agora vamos estudar o segundo mecanismo.

O SELinux utiliza uma abordagem diferente do AppArmor.

Enquanto o AppArmor trabalha principalmente com caminhos e perfis associados às aplicações, o SELinux utiliza **contextos de segurança e políticas baseadas em tipos**.

Essa diferença é fundamental.

O SELinux é muito utilizado em distribuições como:

- Red Hat Enterprise Linux.
- Fedora.
- CentOS Stream.
- Rocky Linux.
- AlmaLinux.

Também é particularmente importante no OpenShift.

## 5.1 O conceito de contexto de segurança

No SELinux, processos, arquivos, diretórios, sockets e outros objetos podem possuir um contexto de segurança.

Um contexto tradicional possui quatro campos:

```text
user:role:type:level
```

Exemplo:

```text
system_u:object_r:httpd_sys_content_t:s0
```

Vamos interpretar:

| Campo | Valor | Significado |
|---|---|---|
| User | `system_u` | Identidade SELinux |
| Role | `object_r` | Papel associado ao objeto |
| Type | `httpd_sys_content_t` | Tipo de segurança |
| Level | `s0` | Nível MLS/MCS |

O campo `type` é especialmente importante no modelo de segurança utilizado pela maioria das instalações SELinux.

## 5.2 Type Enforcement

O SELinux utiliza principalmente um modelo chamado **Type Enforcement (TE)**.

Imagine um processo Apache:

```text
Processo:
httpd_t
```

E arquivos de conteúdo web:

```text
Arquivo:
/var/www/html/index.html

Contexto:
httpd_sys_content_t
```

Uma política pode estabelecer:

```text
httpd_t pode ler httpd_sys_content_t
```

Mas não permitir:

```text
httpd_t pode escrever httpd_sys_content_t
```

Ou seja, o Apache consegue ler os arquivos publicados, mas não consegue modificá-los.

Mesmo que as permissões DAC permitam escrita, o SELinux pode negar a operação.

## 5.3 Verificando os contextos

Em uma distribuição com SELinux habilitado:

```bash
ls -Z /var/www/html
```

Para processos:

```bash
ps -eZ
```

Para verificar o contexto de um arquivo:

```bash
stat -c '%C %n' /var/www/html/index.html
```

Para verificar o estado do SELinux:

```bash
getenforce
```

Os resultados possíveis são:

| Estado | Comportamento |
|---|---|
| Enforcing | Aplica a política e bloqueia violações |
| Permissive | Registra violações, mas não bloqueia |
| Disabled | SELinux desabilitado |

Para obter informações adicionais:

```bash
sestatus
```

---

# 6. Comparando AppArmor e SELinux

Embora ambos implementem MAC, suas abordagens são diferentes.

| Característica | AppArmor | SELinux |
|---|---|---|
| Modelo | Baseado em perfis | Baseado em políticas e contextos |
| Identificação | Principalmente caminhos e executáveis | Contextos de segurança |
| Controle | Perfil associado à aplicação | Regras entre sujeitos e objetos |
| Complexidade inicial | Geralmente menor | Geralmente maior |
| Modos | Enforce e Complain | Enforcing, Permissive e Disabled |
| Distribuições comuns | Debian, Ubuntu, SUSE | RHEL, Fedora, CentOS |
| OpenShift | Não é o mecanismo MAC padrão | Integração nativa |

Não significa que um seja necessariamente mais seguro que o outro.

A segurança depende da qualidade da política, da configuração e da manutenção do sistema.

---

# 7. Relação com Containers e Docker

Agora vamos conectar o assunto ao nosso objetivo principal: segurança de containers.

Quando executamos um container, o runtime pode associar a ele um perfil AppArmor ou um contexto SELinux.

Isso adiciona uma camada de isolamento além de namespaces, cgroups, capabilities e Seccomp.

## 7.1 AppArmor no Docker

Em sistemas com AppArmor, podemos consultar:

```bash
docker info | grep -i apparmor
```

Executar um container com um perfil específico:

```bash
docker run --rm \
  --security-opt apparmor=docker-default \
  alpine sh
```

O perfil `docker-default` é o perfil padrão tradicionalmente utilizado pelo Docker em hosts com suporte ao AppArmor.

Também existe a possibilidade de utilizar:

```bash
--security-opt apparmor=unconfined
```

Isso desabilita o confinamento AppArmor do container.

Não é recomendado em produção.

## 7.2 SELinux no Docker

Em hosts com SELinux habilitado, o Docker pode aplicar rótulos de segurança aos containers.

Por exemplo:

```bash
docker run --rm \
  --security-opt label=type:container_t \
  alpine sh
```

Para volumes, o Docker também possui opções de relabeling:

```bash
docker run --rm \
  -v /dados:/dados:Z \
  alpine sh
```

As opções:

- `:z`: permite compartilhamento do conteúdo entre containers.
- `:Z`: aplica um rótulo privado para o container.

O comportamento exato depende do runtime e da configuração SELinux do host.

---

# 8. SELinux no Kubernetes e OpenShift

Aqui chegamos a uma das partes mais importantes para seu ambiente de trabalho.

O Kubernetes permite configurar contextos SELinux por meio do `securityContext`.

Exemplo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: selinux-lab
spec:
  securityContext:
    seLinuxOptions:
      type: container_t

  containers:
    - name: alpine
      image: alpine:latest
      command:
        - sleep
        - "3600"
```

A aplicação do contexto depende do suporte do sistema operacional, do runtime e das políticas disponíveis no nó.

## 8.1 OpenShift e SELinux

O OpenShift utiliza SELinux como parte importante do isolamento entre workloads.

Isso é especialmente relevante porque diferentes aplicações podem executar no mesmo nó, com usuários Linux distintos ou até com UID aleatório.

O SELinux fornece uma camada adicional de isolamento entre os processos e os recursos do host.

Podemos verificar os contextos dos processos:

```bash
oc debug node/NOME_DO_NODE
```

Dentro do ambiente de debug:

```bash
ps -eZ
```

Também podemos verificar os contextos dos arquivos:

```bash
ls -Z /var/lib/kubelet
```

Os valores exatos dependem da versão do OpenShift e da configuração do nó.

### Uma observação sobre SCC

No OpenShift, a SCC também participa da validação das configurações de segurança relacionadas ao SELinux.

Por exemplo, ela pode controlar:

- Tipos SELinux permitidos.
- Estratégia de atribuição do contexto.
- UID permitido.
- Capabilities.
- Execução privilegiada.

Portanto, a SCC determina quais configurações o Pod pode solicitar, enquanto o SELinux aplica as restrições no kernel.

---

# 9. Comparando as camadas de segurança estudadas

Vamos consolidar o aprendizado.

Imagine uma aplicação web executando em um container.

| Mecanismo | Exemplo de restrição |
|---|---|
| Namespace | Não enxergar os processos do host |
| Cgroup | Não consumir mais de 512 MB |
| Capability | Não possuir `CAP_SYS_ADMIN` |
| Seccomp | Não executar `mount()` |
| AppArmor | Não acessar `/etc/shadow` |
| SELinux | Não permitir escrita em arquivos de conteúdo web |

Observe como os mecanismos são complementares.

Um container bem protegido não depende de uma única camada.

---

# 10. Exercícios de fixação

Sugiro realizar os seguintes exercícios:

**Exercício 1 — DAC e MAC**

Explique por que um processo executando como root pode ter uma operação negada pelo AppArmor ou SELinux.

**Exercício 2 — AppArmor**

No Debian 13:

```bash
sudo aa-status
```

Identifique:

- Quantos perfis estão carregados.
- Quantos estão em Enforce.
- Quantos estão em Complain.

**Exercício 3 — LSM**

Execute:

```bash
cat /sys/kernel/security/lsm
```

Identifique os módulos carregados no seu sistema.

**Exercício 4 — SELinux**

Em uma máquina de laboratório com SELinux:

```bash
getenforce
ls -Z /var/www/html
ps -eZ
```

Identifique os contextos dos arquivos e processos.

**Exercício 5 — Containers**

Compare:

```bash
docker run --rm alpine sh
```

Com:

```bash
docker run --rm \
  --security-opt apparmor=unconfined \
  alpine sh
```

Observe as diferenças na configuração de segurança do container.

---

## Encerramento do Módulo 1

Com esta aula, concluímos a parte principal do estudo dos mecanismos de segurança do kernel Linux:

1. Namespaces — isolamento.
2. Cgroups — controle de recursos.
3. Capabilities — privilégios.
4. Seccomp — filtragem de System Calls.
5. AppArmor e SELinux — controle obrigatório de acesso.

O próximo passo natural é **Módulo 2 — Docker e OCI Runtime Security**, onde vamos entender como essas funcionalidades do kernel são combinadas por ferramentas como Docker, containerd, runc e crun para construir um container.

Isso permitirá conectar os conceitos que estudamos diretamente à execução real de containers.
