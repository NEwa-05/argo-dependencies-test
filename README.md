# argo-dependencies-test
test to validate argoCD and Helm chart dependencies with Traefik Hub APIM

## Create cluster

```bash
sudo k3d cluster create argocd --port 80:80@loadbalancer --port 443:443@loadbalancer --k3s-arg "--disable=traefik@server:0"
k3d kubeconfig get argocd > .kubeconfig
chmod 600 .kubeconfig
```

## Source variables (tokens, dns, ...)

```bash
source .env
```

## Deploy Redis

```bash
helm upgrade --install redis-stack redis-stack/redis-stack-server --namespace redis --values redis/values.yaml --create-namespace
```

## Deploy ArgoCD

### Secrets

Let's use sops and age to make secret hidden and managed by argocd.

Install age and sops

```bash
brew install sops age
```

Generate key:

```bash
age-keygen -o age.agekey
```

create sops config

```bash
cat <<EOF | tee .sops.yaml
creation_rules:
  - path_regex: .*.yml
    encrypted_regex: '^(data|stringData)$'
    age: $(awk -F"key:" '{print $2}' age.agekey |tr -d '\n'' ')
EOF
```

Configure key info

```bash
export SOPS_AGE_KEY_FILE=age.agekey
```

generate secret from variables:

```bash
gsed -i "s|\${TLSCRT}|$(cat .lego/certificates/$CLUSTERNAME.$DOMAINNAME.crt|base64)|g" hubapim/base/wildcard-cert.yaml
gsed -i "s|\${TLSKEY}|$(cat .lego/certificates/$CLUSTERNAME.$DOMAINNAME.key|base64)|g" hubapim/base/wildcard-cert.yaml
sops -e -i hubapim/base/wildcard-cert.yaml
gsed -i 's/${HUB_TOKEN}/'$HUB_TOKEN'/g' hubapim/base/hub-secret.yaml 
sops -e -i hubapim/base/hub-secret.yaml
gsed -i 's/${PORTAL_CLIENTID}/'$PORTAL_CLIENTID'/g' hubapim/overlays/hubapimv1/oidc-secret.yaml
gsed -i 's/${PORTAL_CLIENTSEC}/'$PORTAL_CLIENTSEC'/g' hubapim/overlays/hubapimv1/oidc-secret.yaml
sops -e -i hubapim/overlays/hubapimv1/oidc-secret.yaml
gsed -i 's/${PORTAL_CLIENTID}/'$PORTAL_CLIENTID'/g' hubapim/overlays/hubapimv2/oidc-secret.yaml
gsed -i 's/${PORTAL_CLIENTSEC}/'$PORTAL_CLIENTSEC'/g' hubapim/overlays/hubapimv2/oidc-secret.yaml
sops -e -i hubapim/overlays/hubapimv2/oidc-secret.yaml
```

```bash
helm upgrade --install argocd argo-cd/argo-cd -f argocd/values.yaml --namespace argocd --create-namespace --set configs.secret.argocdServerAdminPassword=${ARGO_PWD_ENCRYPTED}
```

