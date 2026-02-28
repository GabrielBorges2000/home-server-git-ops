# Deploy com Argo CD

Este repositório pode ser usado como fonte Git para o Argo CD. Para o **cert-manager** e o **ClusterIssuer** funcionarem corretamente, a ordem e as dependências precisam estar certas.

## 1. cert-manager antes do ClusterIssuer

O **ClusterIssuer** (`cert-manager/cluster-issuer.yaml`) depende dos **CRDs do cert-manager**. Se o Argo CD aplicar o ClusterIssuer antes do cert-manager estar instalado, você pode ver:

- `Resource not found in cluster: cert-manager.io/v1/ClusterIssuer:letsencrypt-prod`
- ou o recurso fica em estado de erro até os CRDs existirem.

**Formas de garantir a ordem:**

### Opção A: cert-manager fora do Argo CD (recomendado)

Instale o cert-manager **uma vez** no cluster (antes ou logo após criar a Application que aponta para este repo):

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
```

Depois crie no Argo CD a Application que usa este repositório. O ClusterIssuer será aplicado no sync e funcionará porque os CRDs já existem.

### Opção B: Duas Applications no Argo CD

1. **Application 1 – cert-manager**  
   - Fonte: Helm chart oficial do cert-manager ou o manifest YAML do cert-manager.  
   - Sync primeiro e deixe ficar **Healthy**.

2. **Application 2 – este repo (GitOps)**  
   - Fonte: este repositório Git (caminho onde estão os manifests, ex.: raiz ou `cert-manager/`, `saas-freelancer-manager-dev/`, etc.).  
   - Use **Sync Waves** ou simplesmente crie essa Application **depois** da Application do cert-manager e faça o sync quando o cert-manager já estiver Healthy.

### Opção C: Sync wave no próprio repo

O **ClusterIssuer** já está com a anotação de sync wave:

```yaml
argocd.argoproj.io/sync-wave: "1"
```

Assim, dentro do **mesmo** Application, ele é aplicado depois dos recursos com wave `0` (ou sem wave). Isso não instala o cert-manager; só ordena os recursos deste repo. O cert-manager ainda precisa existir no cluster (Opção A ou B).

## 2. Exemplo de Application (este repositório)

Se você aponta o Argo CD para a **raiz** do repositório (com vários YAMLs/Kustomize), pode usar algo assim:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: home-server-gitops
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/SEU_USUARIO/home-server-git-ops.git
    path: .
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Ajuste `repoURL`, `path` (se usar subpastas) e `project` conforme seu setup. Certifique-se de que o **cert-manager** já está instalado (Opção A ou B).

## 3. Health do ClusterIssuer no Argo CD

Por padrão o Argo CD pode não saber se um **ClusterIssuer** está “saudável”. O recurso pode aparecer como **Progressing** ou **Unknown**.

Se quiser que fique **Healthy** quando o status do ClusterIssuer for “Ready”, você pode configurar um **Resource Custom Health Check** no Argo CD (em `argocd-cm` ConfigMap ou em `argocd-cmd-params-cm`), por exemplo:

```yaml
# ConfigMap argocd-cm (exemplo de health check customizado)
data:
  resource.customizations.health.cert-manager.io_ClusterIssuer: |
    hs = {}
    if obj.status ~= nil and obj.status.conditions ~= nil then
      for i, condition in ipairs(obj.status.conditions) do
        if condition.type == "Ready" and condition.status == "True" then
          hs.status = "Healthy"
          hs.message = "Ready"
          return hs
        end
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for Ready"
    return hs
```

Isso é opcional; o ClusterIssuer pode funcionar normalmente mesmo com health “Unknown”.

## 4. Resumo

| O quê              | Ação |
|--------------------|------|
| cert-manager       | Instalar antes (kubectl ou Application separada) e deixar Healthy. |
| ClusterIssuer      | Já tem `sync-wave: "1"`; é aplicado pelo Argo CD quando você sync este repo. |
| DNS “server misbehaving” | Resolver no cluster (CoreDNS/nó); ver [DNS-CLUSTER.md](DNS-CLUSTER.md). |
| Ingress / TLS      | Após ClusterIssuer e DNS ok, o cert-manager emite os certificados; ver [CLOUDFLARE-DOMAIN.md](CLOUDFLARE-DOMAIN.md). |
