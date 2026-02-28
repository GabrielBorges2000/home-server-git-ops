# DNS do cluster – "server misbehaving"

Se o cert-manager (ou outro pod) falhar com:

```text
Error initializing issuer: Get "https://acme-v02.api.letsencrypt.org/directory": dial tcp: lookup acme-v02.api.letsencrypt.org on 10.43.0.10:53: server misbehaving
```

significa que o **DNS do cluster** (CoreDNS, normalmente em `10.43.0.10:53`) não está conseguindo resolver nomes externos (como `acme-v02.api.letsencrypt.org`). O "server misbehaving" é um SERVFAIL do resolvador upstream usado pelo CoreDNS.

## Solução 1: Ajustar o DNS no nó (recomendado)

O CoreDNS no K3s costuma usar `forward . /etc/resolv.conf`, ou seja, o DNS do **nó**. Se o DNS do nó for ruim (roteador, ISP, etc.), o cluster inteiro sofre.

**No servidor onde roda o K3s**, use um resolvador estável (Google ou Cloudflare):

```bash
# Ver o que está hoje
cat /etc/resolv.conf

# Opção A: trocar temporariamente (até reiniciar)
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee -a /etc/resolv.conf

# Opção B: persistente com systemd-resolved (muitos Linux)
sudo mkdir -p /etc/systemd/resolved.conf.d
echo -e "[Resolve]\nDNS=8.8.8.8 1.1.1.1" | sudo tee /etc/systemd/resolved.conf.d/dns_cloudflare.conf
sudo systemctl restart systemd-resolved
# Se /etc/resolv.conf for link para systemd-resolved, já passa a valer
```

Depois **reinicie os pods do CoreDNS** para que usem o novo DNS do nó:

```bash
kubectl rollout restart deployment coredns -n kube-system
# ou, se for DaemonSet:
kubectl rollout restart daemonset coredns -n kube-system
```

## Solução 2: CoreDNS usar DNS fixo (8.8.8.8 / 1.1.1.1)

Se não quiser mudar o DNS do nó, dá para fazer o CoreDNS encaminhar tudo para servidores fixos.

### K3s – ajuste manual do CoreDNS

1. Abrir o ConfigMap do CoreDNS:

   ```bash
   kubectl edit configmap coredns -n kube-system
   ```

2. No bloco `.:53 { ... }`, achar a linha com `forward`:

   ```text
   forward . /etc/resolv.conf
   ```

   Trocar para:

   ```text
   forward . 8.8.8.8 1.1.1.1
   ```

3. Salvar e sair. O CoreDNS recarrega sozinho; se não, reiniciar:

   ```bash
   kubectl rollout restart deployment coredns -n kube-system
   ```

**Atenção:** em upgrades do K3s esse ConfigMap pode ser recriado e a alteração se perder. Nesse caso repita o passo ou use a Solução 1.

### K3s – usar resolv.conf dedicado para o cluster

No servidor, criar um arquivo só para o K3s:

```bash
echo -e "nameserver 8.8.8.8\nnameserver 1.1.1.1" | sudo tee /etc/k3s-resolv.conf
```

No arquivo de config do K3s (ex.: `/etc/rancher/k3s/config.yaml` ou flags do systemd), garantir:

```yaml
kubelet-arg:
  - "resolv-conf=/etc/k3s-resolv.conf"
```

Reiniciar o K3s e, se precisar, os pods do CoreDNS.

## Conferir se o DNS voltou a funcionar

De dentro do cluster:

```bash
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- nslookup acme-v02.api.letsencrypt.org
```

Se resolver o nome, o problema era DNS; o cert-manager deve conseguir falar com a Let's Encrypt de novo. Reaplicar ou aguardar a próxima reconciliação do Certificate/ClusterIssuer.
