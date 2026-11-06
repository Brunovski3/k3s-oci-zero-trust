# Topologia do cluster

Multi-node **in host único**: o cluster K3s roda na box OCI `lab-brunovski` com o control-plane via systemd, e os workers são **containers Docker** (imagem `rancher/k3s:v1.36.4-k3s1`) rodando o `[agent]` no mesmo host — simulando 1 control-plane + 2 workers numa VPS só.

> Segredo de anatomia: sem topology de VMs separadas, o claim "HA" **não** é cross-host. A disponibilidade que existe hoje é do control-plane único.

## Nodes (coletado em 06/11/2026)

```text
NAME            STATUS     ROLES           AGE   VERSION        INTERNAL-IP   EXTERNAL-IP   OS-IMAGE
k3s-worker-1    NotReady   worker          69d   v1.36.4+k3s1   172.17.0.3    <none>        K3s v1.36.4+k3s1 (aarch64)
k3s-worker-2    NotReady   worker          69d   v1.36.4+k3s1   172.17.0.2    <none>        K3s v1.36.4+k3s1 (aarch64)
lab-brunovski   Ready      control-plane   69d   v1.36.4+k3s1   10.0.0.99     <none>        Oracle Linux Server 10.2
```

**Status do workers**: `NotReady`. Os containers Docker saíram há 7 semanas (`k3s-worker-1/2`: `Exited (0)`), então o cluster hoje sobrevive com o **control-plane only** — todos os workloads (Keycloak, n8n, ProxyGroup) têm 1 réplica que roda via systemd k3s server. Em execução há 69 dias sem being touched.

## Como o worker entra (audit trail).

Cada container worker foi criado com:

```bash
docker run -d --name k3s-worker-<N> \
  -e CRI_CONFIG_FILE=/var/lib/rancher/k3s/agent/etc/crictl.yaml \
  rancher/k3s:v1.36.4-k3s1 agent \
  --server https://172.17.0.1:6443 \
  --token <BOOTSTRAP TOKEN>       # NUNCA commitado — só no gerenciador de segredo
```

- `172.17.0.1` é o gateway bridge Docker do host — é por lá que o agent alcança o `:6443` do server.
- O token de bootstrap veio de `/var/lib/rancher/k3s/server/token` no host (não está no repositório).
- Sem manifest/compose versionado no disco — a criação foi por `docker run` (one-shot), daí a ausência de uma configuração de compose declarado pra replicar. *(dado coletado: `docker inspect` não tem label `com.docker.compose.project`).*

## Coexistência com a outra infra

**A mesma box** roda o stack Docker do GitOps/Komodo (descrição no `devops-homelab`/LAB-DEVOPS): `komodo-core`, `komodo-periphery`, `komodo-mongo`, `floci-oci` (emulador OCI local), `fail2ban` — todos Up de 7 semanas. Também um **container Docker de `n8n`** (`n8n_app`), `Exited` há 7 semanas e sucedido pelo n8n do cluster (deployment Kubernetes).

**k3s server (systemd)** rodando com config default (`ExecStart=/usr/local/bin/k3s server` sem flags extras). Não existe `/etc/rancher/k3s/config.yaml` no host — o token de bootstrap é gerado automaticamente para `/var/lib/rancher/k3s/server/token` (não commitado).
