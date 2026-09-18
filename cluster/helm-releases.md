# Versões em produção

Fonte: `helm list -A` no control-plane (`lab-brunovski`), coletado em 06/11/2026. No papel do cluster quem gerencia subida de release é o helm CLI; o estado que conta feio fica no output.

| Release | Namespace | Chart | App | Papel |
| --- | --- | --- | --- | --- |
| `keycloak` | `keycloak` | `keycloakx-7.3.2` | Keycloak **26.7.4** | IAM / IdP OIDC central |
| `tailscale-operator` | `tailscale` | `tailscale-operator-1.102.3` | v1.102.3 | Ingress privado zero-trust |
| `traefik` | `kube-system` | `traefik-41.6.0` | v3.7.13 | Ingress interno (LoadBalancer 10.0.0.99) |
| `traefik-crd` | `kube-system` | `traefik-crd-40.1.4+up40.1.0` | v3.7.1 | CRDs do Traefik, sobrepõe o bootstrap do k3s |

`traefik-bootstrap-helmcharts.yaml`: os HelmChart CRDs de autodeploy originais do k3s (traefik + traefik-crd 40.1.4 da base de disco).

## Fluxo de tráfego

```text
Tailnet (Tailscale)
   └─ HTTPS ──▶ ProxyGroup ingress-proxies (1 réplica)
                  ├─ class: tailscale ──▶ keycloak.ling-becrux.ts.net ──▶ svc/keycloak-keycloakx-http:8080
                  └─ class: tailscale ──▶ n8n.ling-becrux.ts.net     ──▶ svc/n8n-service
Dentro da VPC ──▶ Traefik ingress (10.0.0.99) ──▶ n8n-ingress
```

- Security List da OCI: **nenhuma porta de entrada** pra internet. Nada é público.
- Sem DNS público: os nomes resolvem só na tailnet (MagicDNS).
- TLS: os hostnames da tailnet usam **Let's Encrypt**, com emissão e renovação automatizadas pelo próprio Tailscale Operator (sem expor 80/443 ao mundo).
- Segredo do admin do Keycloak via `secretKeyRef` (secret `keycloak-admin`), nada inline.
