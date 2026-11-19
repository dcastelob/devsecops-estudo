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
