# Practicum6727-2022

## Kubernetes cluster requirements

  - Register domain name and add it to the cloud provider's DNS service.
<...>

<!-- ## Install Cilium

### with Helm

```bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.11.2 \
   --namespace kube-system
```
### with cilium cli

`cilium install`
-->

## Install Istio

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
kubectl create namespace istio-system
helm install istio-base istio/base -n istio-system
helm install istiod istio/istiod -n istio-system \
  --set sidecarInjectorWebhook.enableNamespacesByDefault=true
```

## Install Calico

### with Helm

```bash
echo '{ installation: {kubernetesProvider: AKS }}' > values.yaml
helm install calico projectcalico/tigera-operator --version v3.22.1 -f values.yaml
```

## Configure namespace with Istio sidecar injection

```bash
kubectl label namespace default istio-injection=enabled
```

## Configure Calico Istio integration

- Enable application layer policy

```bash
calicoctl patch FelixConfiguration default --patch \
   '{"spec": {"policySyncPathPrefix": "/var/run/nodeagent"}}'
```

- Update Istio sidecar injector

```bash
curl https://docs.projectcalico.org/archive/v3.21/manifests/alp/istio-inject-configmap-1.10.yaml -o istio-inject-configmap.yaml
kubectl patch configmap -n istio-system istio-sidecar-injector --patch "$(cat istio-inject-configmap.yaml)"
```

- Add Calico authorization services to the mesh

```bash
kubectl apply -f https://docs.projectcalico.org/archive/v3.21/manifests/alp/istio-app-layer-policy-envoy-v3.yaml
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

### Create Pomerium namespace

```bash
kubectl create ns pomerium
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

#### for Azure

- assign AKS cluster kubelet managed identity `DNS Zone Contributor` role in DNS zone/ These are AKS managed identity and Agentpool identity.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@kubezta.ga
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - dns01:
        azureDNS:
          subscriptionID: 5e2a9ee6-0bd8-4749-8a0a-75f1ab7ca131
          resourceGroupName: practicum
          hostedZoneName: kubezta.ga
          environment: AzurePublicCloud

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
      secretName: authenticate.pomerium.kubezta.ga-tls

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
  rootDomain: pomerium.kubezta.ga
  generateTLS: false
  insecure: true

```

### Deploy Pomerium using Helm

```bash
helm repo add pomerium https://helm.pomerium.io
helm upgrade --install pomerium pomerium/pomerium -f values.yaml -n pomerium --create-namespace
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
    ingress.pomerium.io/policy: '[{"allow":{"and":[{"domain":{"is":"kubezta.ga"}}]}}]'
    ingress.pomerium.io/pass_identity_headers: "true"
spec:
  ingressClassName: pomerium
  rules:
  - host: hello.pomerium.kubezta.ga
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
    - hello.pomerium.kubezta.ga
    secretName: hello.pomerium.kubezta.ga-tls
```

```bash
kubectl apply -f hello-ingress.yaml
```

### Testing

- attempt to log in using valid user from a different domain


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
  - issuer: "authenticate.pomerium.kubezta.ga" # Adjust to match your Authenticate service URL
    audiences:
      - hello.pomerium.kubezta.ga # This should match the value of spec.host in the services Ingress
    fromHeaders:
      - name: "X-Pomerium-Jwt-Assertion"
    jwksUri: https://authenticate.pomerium.kubezta.ga/.well-known/pomerium/jwks.json # Adjust to match your Authenticate service URL
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
      values: ["hello.pomerium.kubezta.ga"]
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
                is: grafana-admin@kubezta.ga
  hosts:
    - "grafana.pomerium.kubezta.ga"
  tls:
  - hosts:
    - grafana.pomerium.kubezta.ga
    secretName: grafana.pomerium.kubezta.ga-tls
persistence:
  type: pvc
  enabled: true
  storageClassName: default
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
    jwk_set_url: https://authenticate.pomerium.kubezta.ga/.well-known/pomerium/jwks.json
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
  - issuer: "authenticate.pomerium.kubezta.ga" # Adjust to match your Authenticate service URL
    forwardOriginalToken: true
    audiences:
      - grafana.pomerium.kubezta.ga # This should match the value of spec.host in the services Ingress
    fromHeaders:
      - name: "X-Pomerium-Jwt-Assertion"
    jwksUri: https://authenticate.pomerium.kubezta.ga/.well-known/pomerium/jwks.json # Adjust to match your Authenticate service URL.
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
      values: ["grafana.pomerium.kubezta.ga"]
```

```bash
kubectl apply -f istio-grafana-policy.yaml
```

## Apply Network Application Policies

### Deploy test application

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
  labels:
    app: database
spec:
  ports:
  - port: 2379
    name: http
  selector:
    app: database
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: database
  labels:
    app: yaobank
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
        version: v1
    spec:
      serviceAccountName: database
      containers:
      - name: database
        image: calico/yaobank-database:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 2379
        command: ["etcd"]
        args:
          - "-advertise-client-urls"
          - "http://database:2379"
          - "-listen-client-urls"
          - "http://0.0.0.0:2379"
---
apiVersion: v1
kind: Service
metadata:
  name: summary
  labels:
    app: summary
spec:
  ports:
  - port: 80
    name: http
    targetPort: 8000
  selector:
    app: summary
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: summary
  labels:
    app: yaobank
    database: reader
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: summary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: summary
  template:
    metadata:
      labels:
        app: summary
        version: v1
    spec:
      serviceAccountName: summary
      containers:
      - name: summary
        image: calico/yaobank-summary:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: customer
  labels:
    app: customer
spec:
  ports:
  - port: 80
    name: http
    targetPort: 8000
  selector:
    app: customer
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: customer
  labels:
    app: yaobank
    summary: reader
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: customer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: customer
  template:
    metadata:
      labels:
        app: customer
        version: v1
    spec:
      serviceAccountName: customer
      containers:
      - name: customer
        image: calico/yaobank-customer:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
---
```

```bash
kubectl apply -f application-stack.yaml
```

### Deploy ingress and Pomerium access policy

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yaobank
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    ingress.pomerium.io/policy: '[{"allow":{"and":[{"domain":{"is":"kubezta.ga"}}]}}]'
    ingress.pomerium.io/pass_identity_headers: "true"
spec:
  ingressClassName: pomerium
  rules:
  - host: customer.yaobank.pomerium.kubezta.ga
    http:
      paths:
      - backend:
          service:
            name: customer
            port:
              name: http
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - customer.yaobank.pomerium.kubezta.ga
    secretName: customer.yaobank.pomerium.kubezta.ga-tls
```

```bash
kubectl apply -f application-ingress.yaml
```

### Deploy Calico application layer policies

```yaml
apiVersion: crd.projectcalico.org/v1
kind: GlobalNetworkPolicy
metadata:
  name: customer
spec:
  selector: app == 'customer'
  ingress:
    - action: Allow
      http:
        methods: ["GET"]
  egress:
    - action: Allow
---
apiVersion: crd.projectcalico.org/v1
kind: GlobalNetworkPolicy
metadata:
  name: summary
spec:
  selector: app == 'summary'
  ingress:
    - action: Allow
      source:
        serviceAccounts:
          names: ["customer"]
  egress:
    - action: Allow
---
apiVersion: crd.projectcalico.org/v1
kind: GlobalNetworkPolicy
metadata:
  name: database
spec:
  selector: app == 'database'
  ingress:
    - action: Allow
      source:
        serviceAccounts:
          names: ["summary"]
  egress:
    - action: Allow
```

```bash
kubectl apply -f application-calico-policies.yaml
```

### Apply Calico policies to user-facing applications

- Allow external only access to nginx pod. Other pods shouldn't be able to access it.

```yaml
apiVersion: crd.projectcalico.org/v1
kind: GlobalNetworkPolicy
metadata:
  name: hello
spec:
  selector: app.kubernetes.io/name == 'nginx'
  # Configuring only ingress disables external access from the pod
  ingress:
    - action: Allow
      source:
        serviceAccounts:
          names: ["pomerium-proxy"] # Allow Pomerium proxy access only
```

- Allow external only access to grafana pod. Other pods shouldn't be able to access it.

```yaml
apiVersion: crd.projectcalico.org/v1
kind: GlobalNetworkPolicy
metadata:
  name: grafana
spec:
  selector: app.kubernetes.io/name == 'grafana'
  ingress:
    - action: Allow
      source:
        serviceAccounts:
          names: ["pomerium-proxy"] # Allow Pomerium proxy access only
  egress:
    - action: Allow

```

**NOTE:** Using unique Service Account per service allows for granular access control policies.

# Future work

## SPIFFE

In current state, both Cilium and Istio have pending PRs for SPIFFE integration. Until those changes are merged, SPIFFE alone will not benefit the framework.
- [Cilium PR](https://github.com/cilium/cilium/pull/17335)
- [Istio PR](https://github.com/istio/istio/pull/37947) # Merged and will be available in Istio 1.14

## High Availability

In order to maintain availability of the critical services such as Pomerium, Istio, Calico, they need to be deployed cross availability zones with multiple replicas. Due to resource constraints, this has not been implemented as the part of the final project demo.
