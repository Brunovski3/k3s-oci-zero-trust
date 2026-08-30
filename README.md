# k3s-oci-zero-trust

Cluster Kubernetes HA multi-node (1 control-plane + 2 workers) rodando K3s na Oracle Cloud. Exposição sem porta aberta: acesso privado via Tailscale, ingress Traefik e TLS automatizado, workloads administrados via Helm.

Este repo guarda a configuração declarada do cluster e os manifests que o sustentam, para que outro lugar (ou outra pessoa) consiga reconstruir o ambiente sem depender do que está rodando em produção.

## Estrutura

| Caminho | O que vai ter |
|---|---|
| `cluster/` | `config.yaml` do k3s e o setup do cluster (installer, nós, labels de role) |
| `networking/traefik/` | Config do ingress + TLS |
| `networking/tailscale/` | Operator do Tailscale e o ProxyGroup que expõe serviços internos em HTTPS privado |
| `apps/` | Helm releases dos workloads (n8n, Keycloak, ...) |

## Como o manifest entra aqui

Os manifests vivos já existem no servidor. O script `scripts/pull-from-server.sh` faz o rsync dos autodeploy manifests (`/var/lib/rancher/k3s/server/manifests`, onde fica o HelmChart do Traefik, do Tailscale Operator, do n8n etc.) e aponta qualquer segredo que encontrar. Fluxo:

```bash
./scripts/pull-from-server.sh <host-oci> .
# revisão em local-export/ (fora do Git)
# transporte dos manifests limpos para os diretórios acima, com commit
```

O que **nunca** vai pro repo: token do k3s, chaves TLS, credenciais de banco, valores de `stringData`/`kind: Secret`. Segredos ficam fora via `secretKeyRef`.

## Status

Estrutura inicial e ferramenta de coleta prontas. Os manifests de produção estão no servidor e entram em commits de `cluster/`, `networking/` e `apps/` conforme passar pela revisão.

## Licença

MIT.
