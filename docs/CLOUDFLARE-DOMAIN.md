# Domínio e HTTPS (codeborges.com.br)

Como expor a API e o frontend em **api.codeborges.com.br** e **app.codeborges.com.br** com HTTPS, usando Cloudflare Tunnel, Ingress e cert-manager.

## Visão geral

1. **Cloudflare Tunnel** (cloudflared) já está no cluster e envia tráfego para o Traefik.
2. **Ingress** (Traefik) roteia por host para os Services da API e do Web.
3. **cert-manager** emite certificados Let's Encrypt (HTTP-01) e renova automaticamente.
4. **Secrets** já estão com `NEXT_PUBLIC_API_URL=https://api.codeborges.com.br` para o frontend chamar a API via HTTPS.

## Pré-requisitos

### 1. Traefik

O cluster já usa **Traefik** como Ingress Controller. O Service do Traefik (para configurar no tunnel) costuma ser algo como `traefik` no namespace `traefik`. Confira com:

```bash
kubectl get svc -A | grep -i traefik
```

Use o FQDN desse Service (ex.: `http://traefik.traefik.svc.cluster.local:80`) ao configurar o Public Hostname no Cloudflare Tunnel.

### 2. Instalar cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
```

Aguarde os pods do cert-manager ficarem Ready antes de aplicar o ClusterIssuer e o Ingress.

### 3. DNS no Cloudflare

No Cloudflare (dashboard do domínio codeborges.com.br):

- **api.codeborges.com.br** → CNAME para o seu tunnel (ex.: `<tunnel-id>.cfargotunnel.com`) ou “Proxy through Cloudflare” conforme o tipo do tunnel.
- **app.codeborges.com.br** → mesmo CNAME do tunnel (um tunnel pode servir vários hostnames).

Como você usa **Tunnel Token**, os hostnames são configurados no Zero Trust, não só no DNS. Garanta que o DNS aponte para o tunnel (CNAME para o hostname do tunnel na Cloudflare).

### 4. Configurar o Tunnel no Cloudflare Zero Trust

Em **Zero Trust** (ou **Cloudflare Dashboard** > **Networks** > **Tunnels**):

1. Abra o tunnel que corresponde ao token usado no cluster.
2. Em **Public Hostname** (ou **Routes**), adicione:

| Subdomain / Hostname | Service Type | URL (Origin) |
|----------------------|--------------|--------------|
| **api** (ou api.codeborges.com.br) | HTTP | `http://traefik.traefik.svc.cluster.local:80` |
| **app** (ou app.codeborges.com.br) | HTTP | `http://traefik.traefik.svc.cluster.local:80` |

*(Se o Traefik estiver em outro namespace, use o FQDN correto, ex.: `http://<nome-do-svc>.<namespace>.svc.cluster.local:80`.)*

Assim, todo o tráfego de **api** e **app** vai para o Traefik, que roteia por host para **saas-api** e **saas-web**. O cert-manager consegue responder ao desafio HTTP-01 do Let's Encrypt porque a requisição chega via tunnel até o Ingress.

### 5. Aplicar o GitOps

Ordem sugerida:

```bash
# 1) cert-manager (ClusterIssuer)
kubectl apply -f cert-manager/

# 2) Cloudflare (namespace, secret, deployment)
kubectl apply -f services/cloudflare/

# 3) Ingress (TLS + certificado)
kubectl apply -f saas-freelancer-manager-dev/ingress.yaml
```

O cert-manager vai criar o Secret `freelancer-manager-tls` no namespace `freelancer-manager-dev` e renovar o certificado automaticamente.

## Verificação

- **Certificado**: `kubectl get certificate -n freelancer-manager-dev`
- **Ingress**: `kubectl get ingress -n freelancer-manager-dev`
- Acesse **https://app.codeborges.com.br** (frontend) e **https://api.codeborges.com.br/health** (API).

## Port-forward (alternativa local)

Se quiser acessar sem domínio (apenas para debug):

```bash
# Frontend
kubectl port-forward -n freelancer-manager-dev svc/saas-web 3000:80

# API
kubectl port-forward -n freelancer-manager-dev svc/saas-api 3333:80
```

Acesse http://localhost:3000 e http://localhost:3333. Se o port-forward falhar, confira se os pods estão Ready: `kubectl get pods -n freelancer-manager-dev`.
