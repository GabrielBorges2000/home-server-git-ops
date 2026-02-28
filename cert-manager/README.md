# cert-manager – ClusterIssuer Let's Encrypt

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

## GitOps (Flux / Argo CD)

Se o ClusterIssuer for aplicado pelo GitOps, garanta que o **cert-manager** (incluindo CRDs) seja instalado e esteja **Ready** antes do recurso que aplica este manifest. Use dependências (e.g. `dependsOn` no Flux) ou ordem de sync para que o cert-manager seja aplicado primeiro.
