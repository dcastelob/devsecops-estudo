# Super resumo

Os principais mecanismos são:

| Mecanismo | Responsabilidade |
|---|---|
| Namespaces | Isolar a visão que um processo possui dos recursos do sistema |
| Cgroups | Controlar e limitar o consumo de recursos |
| Capabilities | Dividir os privilégios tradicionalmente associados ao root |
| Seccomp | Filtrar chamadas de sistema |
| AppArmor | Restringir o acesso a recursos por meio de perfis |
| SELinux | Aplicar políticas de controle de acesso obrigatório |
| LSM | Infraestrutura do kernel para implementar controles de segurança |

## Namespaces


* O isolamento depende da configuração.

Opções como `--privileged`, `--pid=host`, `--network=host` e compartilhamento de diretórios do host podem enfraquecer significativamente o isolamento.

```bash
sudo apt install util-linux
lsns -p "$PID"
```

O Linux disponibiliza o comando `nsenter`, que permite executar um processo dentro dos namespaces de outro processo.

```bash
sudo nsenter -t "$PID" -p -m -u -i -n sh
```

Aqui:

| Parâmetro | Namespace |
|---|---|
| `-t` | Processo de destino |
| `-p` | PID |
| `-m` | MNT |
| `-u` | UTS |
| `-i` | IPC |
| `-n` | NET |

Você estará executando um shell utilizando os namespaces indicados do container.

Verifique:

```bash
hostname
ip addr
lsns -p $$
```

Para sair:

```bash
exit
```

O nsenter é particularmente interessante para administração e investigação de incidentes em containers.


## Cgroups

### memory.events

Permite verificar eventos relacionados ao consumo de memória.

Exemplo:

```bash
cat /sys/fs/cgroup/memory.events
```

Você poderá encontrar:

```text
low 0
high 0
max 0
oom 0
oom_kill 0
oom_group_kill 0
```

Os principais contadores são:

| Contador | Significado |
|---|---|
| high | Eventos de ultrapassagem de `memory.high` |
| max | Situações em que o uso tentou ultrapassar `memory.max` |
| oom | Situações em que o Cgroup entrou em condição de OOM |
| oom_kill | Processos encerrados pelo OOM Killer |
| oom_group_kill | Eventos de encerramento de grupos |

Esses indicadores são extremamente úteis para diagnosticar problemas em aplicações Kubernetes.

## 6.2. OOM do container versus OOM do host

É importante não confundir dois cenários.

**Cenário A — OOM do Cgroup**

Um container ultrapassa seu limite de memória.

O kernel tenta resolver a situação dentro do contexto do Cgroup.

**Cenário B — OOM global do host**

O próprio servidor fica sem memória disponível.

Nesse caso, o OOM Killer pode precisar encerrar processos de diferentes serviços e containers para preservar o funcionamento do sistema.

São situações diferentes, com impactos diferentes.

Para investigar:

```bash
dmesg -T | grep -i -E 'oom|killed process'
```

Ou:

```bash
journalctl -k | grep -i -E 'oom|killed process'
```

Em ambientes Docker, você também pode verificar:

```bash
docker inspect \
  --format '{{.State.OOMKilled}}' \
  nome-do-container
```

Se retornar:

```text
true
```

O Docker registrou que o container foi encerrado por OOM.

---
