# home-server-git-ops

Manifests Kubernetes (GitOps) para o home server.

**Deploy com Argo CD:** veja **[docs/ARGOCD.md](docs/ARGOCD.md)** (ordem cert-manager → ClusterIssuer, sync waves).

## Domínio e HTTPS (codeborges.com.br)

Para expor API e frontend em **api.codeborges.com.br** e **app.codeborges.com.br** com HTTPS, Cloudflare Tunnel, Ingress e renovação automática de certificado, veja **[docs/CLOUDFLARE-DOMAIN.md](docs/CLOUDFLARE-DOMAIN.md)**.
