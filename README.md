# k3s-oci-zero-trust

Cluster **K3s na Oracle Cloud** com **1 control-plane + 2 workers em containers** (topologia single-host: server via systemd, agentes Docker no mesmo host — detalhes em [`cluster/topology.md`](cluster/topology.md)).

Zero porta aberta: os serviços são publicados via **Tailscale Operator (ProxyGroup)**, só resolvem dentro da tailnet, com **TLS Let's Encrypt** emitido e renovado pelo próprio operator. Workloads principais: **Keycloak (IAM, via chart `keycloakx`)** e **n8n (manifests declarativos)**.

> Estado da infra: control-plane online a 69+ dias; os containers workers estão `Exited` (há 7 semanas), então hoje o cluster roda **single-node**. Reerguer os workers (`docker start k3s-worker-1 k3s-worker-2`) é o próximo passo.

## Estado real do cluster

| Camada | Rodando |
|---|---|
| Control-plane | `lab-brunovski` (K3s via systemd, Oracle Linux 10.2, aarch64) |
| Workers | 2x containers `rancher/k3s:v1.36.4-k3s1` (agent `:6443`), hoje `Exited`/NotReady |
| IAM | Keycloak **26.7.4** (chart `keycloakx-7.3.2`), ns `keycloak`, admin via `secretKeyRef` |
| Exposição privada | Tailscale Operator **v1.102.3**, ns `tailscale`, ProxyGroup `ingress-proxies` |
| Ingress interno | Traefik **v3.7.13** (chart `41.6.0`, 6 revisões), LoadBalancer interno `10.0.0.99` |
| Automação alternativa | Docker stack do host (`komodo-core/periphery/mongo`, `floci-oci`, `fail2ban`) — viva e em paralelo |
| Automação | n8n (`n8nio/n8n:latest`), PVC dedicado, ingress interno (Traefik) + privado (tailnet) |

Detalhe completo (com fluxo de tráfego): [`cluster/helm-releases.md`](cluster/helm-releases.md).

## Estrutura

```text
.
├── cluster/               # releases helm de produção e bootstrap do k3s
│   ├── helm-releases.md   # versões de chart/app e fluxo de tráfego
│   └── traefik-bootstrap-helmcharts.yaml
├── networking/
│   ├── tailscale/         # proxy-group.yaml + keycloak-ingress.yaml (class tailscale)
│   └── traefik/           # (ingress interno — sobrevive no bootstrap do k3s)
├── apps/
│   ├── keycloak/values.yaml    # spec do chart keycloakx como está em produção
│   └── n8n/                    # deployment, pvc, service e os dois ingress
├── scripts/pull-from-server.sh # coleta manifests vivos do servidor
└── docs/                  # (todo: diagrama da arquitetura + prints)
```

## Coletar o estado do servidor

O script faz rsync dos manifests que **rodam** (não os que existem na teoria):

```bash
./scripts/pull-from-server.sh <host-oci> .
```

O export cai em `local-export/` (fora do Git) e marca segredos. Nunca sobem para este repo: token do k3s (`server/token`), chaves TLS (`tls/`, `cred/`), `stringData`/`kind: Secret`.

## Prova de exposição (zero-trust)

- Security List da OCI: sem regra de entrada pública.
- Sem DNS público: os nomes (`*.ling-becrux.ts.net`) só resolvem na tailnet.
- TLS: certificado Let's Encrypt emitido pelo Tailscale Operator pra cada service (sem 80/443 aberto).
- Keycloak: ninguém guarda senha de app — tudo OIDC via `proxy-group`; plataforma sem backend delega autenticação pra ele.
