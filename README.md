# Practicum6727-2022

## Kubernetes cluster requirements

  - Register domain name and add it to the cloud provider's DNS service.
<...>

## Install local k8s cluster

### [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

  - kind create cluster --config=kind-cluster.yaml

### [minikube](https://minikube.sigs.k8s.io/docs/start/)

  - minikube start --network-plugin=cni --cni=false

## Install Cilium

### with Helm

```bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.11.2 \
   --namespace kube-system
```
### with cilium cli

`cilium install`

## Install Istio

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
kubectl create namespace istio-system
helm install istio-base istio/base -n istio-system
helm install istiod istio/istiod -n istio-system \
  --set sidecarInjectorWebhook.enableNamespacesByDefault=true
```

## Configure namespace with Istio sidecar injection

```bash
kubectl label namespace default istio-injection=enabled
```

## Install Cert-Manager

### Deploy cert-manager with CRDs

```bash
helm repo add jetstack https://charts.jetstack.io
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

### Enable Istio sidecar injection

```bash
kubectl label namespace cert-manager istio-injection=enabled
```

### Define certificate issuer

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: pomerium-ca
  namespace: pomerium
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: pomerium-ca
  namespace: pomerium
spec:
  isCA: true
  secretName: pomerium-ca
  commonName: pomerium ca
  issuerRef:
    name: pomerium-ca
    kind: Issuer
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: pomerium-issuer
  namespace: pomerium
spec:
  ca:
    secretName: pomerium-ca
```

### Deploy issuer

```bash
kubectl apply -f pomerium-issuer.yaml
```

### Define Let's Encrypt Issuer

#### for DigitalOcean

create Kubernetes secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: digitalocean-dns
data:
  access-token: "base64 encoded access-token here"
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@zta-k8s.ga
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: pomerium
    # for digital ocean
    - dns01:
        digitalocean:
          tokenSecretRef:
            name: digitalocean-dns
            key: access-token
```

### Deploy LE Issuer

```bash
kubectl apply -f le-issuer.yaml
```

### Define certificates for Pomerium

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: pomerium-cert
  namespace: pomerium
spec:
  secretName: pomerium-tls
  issuerRef:
    name: pomerium-issuer
    kind: Issuer
  usages:
    - server auth
    - client auth
  dnsNames:
    - pomerium-proxy.pomerium.svc.cluster.local
    - pomerium-authorize.pomerium.svc.cluster.local
    - pomerium-databroker.pomerium.svc.cluster.local
    - pomerium-authenticate.pomerium.svc.cluster.local
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: pomerium-redis-cert
  namespace: pomerium
spec:
  secretName: pomerium-redis-tls
  issuerRef:
    name: pomerium-issuer
    kind: Issuer
  usages:
    - server auth
    - client auth
  dnsNames:
    - pomerium-redis-master.pomerium.svc.cluster.local
    - pomerium-redis-headless.pomerium.svc.cluster.local
    - pomerium-redis-replicas.pomerium.svc.cluster.local

```

### Deploy Pomerium certificates

```bash
kubectl apply -f pomerium-certs.yaml
```

## Configure Auth0 Identity Provider

  Enable Auth0 IdP to work with Pomerium proxy - [reference](https://www.pomerium.com/docs/identity-providers/auth0.html)

## Install Pomerium

### Define custom values.yaml file for Pomerium Helm chart

```yaml
### Ensure image tag => v0.17.0. If not, set it manually
#image:
#  tag: v0.17.0
authenticate:
  idp:
    provider: auth0
    url: https://dev-gp51dybl.us.auth0.com
    clientID: gDEdnEknWN3pzBNSAhynmFAzOF0LDpb6
    clientSecret: FIUhWWFWlzPooV_8sUVPzk7rGp9AeO4C0_y6ngxDCEaVgAYoWQ3Fww6wTh3W82pY
    serviceAccount: ewoiY2xpZW50X2lkIjogIjhxNjk5YWdYYVNBM2RabGpJZzR5WjBTeFJNeGZ3SmdnIiwKInNlY3JldCI6ICJ5STBIbUd5dV9Fclk0TGlzbU9LM1BkSm44cGdNMlVvR1dyX0o5M1Y2TFNKVUd6Wm1MNG04UG45aDl4MFdMdFBSIgp9Cg==
  ingress:
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      ingress.pomerium.io/service_proxy_upstream: "true"
    tls:
      secretName: authenticate.pomerium.zta-k8s.ga-tls

proxy:
  cookie_secret: W5jwcMdMRs7JMuZ1uCy/rnf9NzHsbqnKSJlMKSOEyFQ=
  deployment:
    podAnnotations:
      traffic.sidecar.istio.io/excludeInboundPorts: "80,443"

redis:
  enabled: true
  tls:
    enabled: false

ingress:
  enabled: false

ingressController:
  enabled: true

service:
  authorize:
    headless: false
  databroker:
    headless: false

config:
  sharedSecret: M4pXv+vIBzTHAREK1CBMk8SoYLTWEQ9sicIZ+KY1U0s=
  rootDomain: pomerium.zta-k8s.ga
  generateTLS: false
  insecure: true

```

### Deploy Pomerium using Helm

```bash
helm repo add pomerium https://helm.pomerium.io
helm upgrade --install pomerium pomerium/pomerium -f values.yaml
```

**NOTE** in light of recent Okta breach, additional option to enable per-route device authentication using secure token device [1](https://www.pomerium.com/guides/enroll-device.html), [2](https://www.pomerium.com/docs/topics/ppl.html#device-matcher)

### Create DNS A record for Pomerium ingress

Create DNS A record `*.pomerium.<domain_name>` and assign it to Pomerium ingress
IP address of the Load Balancer.

### Define test service - NGINX

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install nginx bitnami/nginx --set service.type=ClusterIP
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    ingress.pomerium.io/policy: '[{"allow":{"and":[{"domain":{"is":"zta-k8s.ga"}}]}}]'
    ingress.pomerium.io/pass_identity_headers: "true"
spec:
  ingressClassName: pomerium
  rules:
  - host: hello.pomerium.zta-k8s.ga
    http:
      paths:
      - backend:
          service:
            name: nginx
            port:
              name: http
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - hello.pomerium.zta-k8s.ga
    secretName: hello.pomerium.zta-k8s.ga-tls
```

```bash
kubectl apply -f hello-ingress.yaml
```

### Define Istio policy for NGINX service

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: nginx-require-pomerium-jwt
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx # This matches the label applied to our test service
  jwtRules:
  - issuer: "authenticate.pomerium.zta-k8s.ga" # Adjust to match your Authenticate service URL
    audiences:
      - hello.pomerium.zta-k8s.ga # This should match the value of spec.host in the services Ingress
    fromHeaders:
      - name: "X-Pomerium-Jwt-Assertion"
    jwksUri: https://authenticate.pomerium.zta-k8s.ga/.well-known/pomerium/jwks.json # Adjust to match your Authenticate service URL
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: nginx-require-pomerium-jwt
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx
  action: ALLOW
  rules:
  - when:
    - key: request.auth.claims[aud]
      values: ["hello.pomerium.zta-k8s.ga"]
```

### Deploy Istio policy

```bash
kubectl apply -f istio-nginx-policy.yaml
```

### Deploy test service - Grafana

```yaml
ingress:
  enabled: true
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: pomerium
    ingress.pomerium.io/pass_identity_headers: "true"
    ingress.pomerium.io/policy: |
      - allow:
          and:
            - email:
                is: grafana-admin@zta-k8s.ga
  hosts:
    - "grafana.pomerium.zta-k8s.ga"
  tls:
  - hosts:
    - grafana.pomerium.zta-k8s.ga
    secretName: grafana.pomerium.zta-k8s.ga-tls
persistence:
  type: pvc
  enabled: true
  storageClassName: do-block-storage # if running k8s on DigitalOcean
  accessModes:
    - ReadWriteOnce
  size: 10Gi
  finalizers:
    - kubernetes.io/pvc-protection
grafana.ini:
  auth:
    disable_login_form: true
  auth.jwt:
    enabled: true
    header_name: X-Pomerium-Jwt-Assertion
    email_claim: email
    auto_sign_up: true
    jwk_set_url: https://authenticate.pomerium.zta-k8s.ga/.well-known/pomerium/jwks.json
```

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm upgrade --install grafana grafana/grafana --values grafana-values.yaml
```

### Deploy Istio policy for Grafana

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: grafana-require-pomerium-jwt
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: grafana # This matches the label applied to our test service
  jwtRules:
  - issuer: "authenticate.pomerium.zta-k8s.ga" # Adjust to match your Authenticate service URL
    forwardOriginalToken: true
    audiences:
      - grafana.pomerium.zta-k8s.ga # This should match the value of spec.host in the services Ingress
    fromHeaders:
      - name: "X-Pomerium-Jwt-Assertion"
    jwksUri: https://authenticate.pomerium.zta-k8s.ga/.well-known/pomerium/jwks.json # Adjust to match your Authenticate service URL.
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: grafana-require-pomerium-jwt
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: grafana # This matches the label applied to our test service
  action: ALLOW
  rules:
  - when:
    - key: request.auth.claims[aud]
      values: ["grafana.pomerium.zta-k8s.ga"]
```

```bash
kubectl apply -f istio-grafana-policy.yaml
```

## SPIFFE

In current state, both Cilium and Istio have pending PRs for SPIFFE integration. Until those changes are merged, SPIFFE alone will not benefit the framework.
- [Cilium PR](https://github.com/cilium/cilium/pull/17335)
- [Istio PR](https://github.com/istio/istio/pull/37947)
