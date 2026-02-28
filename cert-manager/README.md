# cert-manager – ClusterIssuer Let's Encrypt

## Erro de DNS: "lookup acme-v02.api.letsencrypt.org ... server misbehaving"

Se o issuer falhar com **dial tcp: lookup acme-v02.api.letsencrypt.org on 10.43.0.10:53: server misbehaving**, o DNS do cluster não está resolvendo nomes externos. Veja **[docs/DNS-CLUSTER.md](../docs/DNS-CLUSTER.md)** para corrigir (ajustar DNS no nó ou CoreDNS para usar 8.8.8.8 / 1.1.1.1).

---

O erro **"Resource not found in cluster: cert-manager.io/v1/ClusterIssuer:letsencrypt-prod"** costuma ocorrer quando:

1. O **cert-manager ainda não está instalado** (os CRDs não existem no cluster), ou  
2. O ClusterIssuer foi aplicado **antes** dos CRDs estarem prontos, ou  
3. A versão do cert-manager no cluster só expõe a API **v1alpha2**.

## Ordem correta de aplicação

### 1. Instalar o cert-manager (com CRDs)

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
```

### 2. Aguardar o cert-manager ficar pronto

```bash
kubectl wait --for=condition=Ready pods -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=120s
```

### 3. Aplicar o ClusterIssuer

```bash
kubectl apply -f cert-manager/cluster-issuer.yaml
```

### 4. Verificar

```bash
kubectl get clusterissuer letsencrypt-prod
```

## Se o erro continuar (API v1 não existe)

Em clusters com cert-manager mais antigo, o CRD pode só ter a versão **v1alpha2**. Use o manifest alternativo:

```bash
kubectl apply -f cert-manager/cluster-issuer-v1alpha2.yaml
```

Ou edite o `cluster-issuer.yaml` e troque `apiVersion: cert-manager.io/v1` por `apiVersion: cert-manager.io/v1alpha2`, depois aplique de novo.

## Deploy com Argo CD

Se você usa **Argo CD**, o cert-manager precisa estar instalado no cluster **antes** do sync que aplica o ClusterIssuer (senão aparece "Resource not found"). O ClusterIssuer já tem `argocd.argoproj.io/sync-wave: "1"` para ordenar dentro do mesmo Application.

Veja **[docs/ARGOCD.md](../docs/ARGOCD.md)** para:

- Instalar cert-manager antes (kubectl ou Application separada)
- Exemplo de Application para este repo
- Health check opcional para ClusterIssuer
