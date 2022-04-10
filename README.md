# Practicum6727-2022

## Kubernetes cluster requirements

  - Register domain name and add it to the cloud provider's DNS service.
  - for Azure: assign AKS cluster kubelet managed identity `DNS Zone Contributor` role in DNS zone. These are AKS managed identity and Agentpool identity.

## SPIFFE

SPIFFE and its implementation SPIRE allows for workload attestation through short-lived x509 certificates issued by SPIRE server. These certificates are used by Istio to establish mTLS connections. This provides additional assurance of the identity.
- [Istio PR](https://github.com/istio/istio/pull/37947) # Merged and will be available in Istio 1.14

### Deploy SPIRE

```bash
kubectl apply -f spire.yaml
```

## Install Istio

```bash
istioctl install -f spire-istio.yaml
```

## Install Calico

### with Helm

```bash
echo '{ installation: {kubernetesProvider: AKS }}' > calico-values.yaml
helm install calico projectcalico/tigera-operator --version v3.22.1 -f calico-values.yaml
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

### Create Pomerium namespace

```bash
kubectl create ns pomerium
```

### Enable Istio sidecat injection

```bash
kubectl label namespace pomerium istio-injection=enabled
```

### Deploy certificates and issuers for Pomerium

```bash
kubectl apply -f pomerium-certs.yaml
```

## Configure Auth0 Identity Provider

  Enable Auth0 IdP to work with Pomerium proxy - [reference](https://www.pomerium.com/docs/identity-providers/auth0.html)

## Deploy Pomerium using Helm

```bash
helm repo add pomerium https://helm.pomerium.io
helm upgrade --install pomerium pomerium/pomerium -f pomerium-values.yaml -n pomerium
```

### Create DNS A record for Pomerium ingress

Create DNS A record `*.pomerium.<domain_name>` and assign it to Pomerium ingress IP address of the Load Balancer.

### Define test service - NGINX

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install nginx bitnami/nginx --set service.type=ClusterIP
```

```bash
kubectl apply -f hello.yaml
```

### Testing

- attempt to log in using valid user from a different domain

### Deploy test service - nginx-secret

In light of recent Okta breach, additional option to enable per-route device authentication using secure token device [1](https://www.pomerium.com/guides/enroll-device.html), [2](https://www.pomerium.com/docs/topics/ppl.html#device-matcher)

Currently, there is a bug in the device identity authorization.

```bash
kubectl apply -f hello-secret.yaml
```

### Deploy test service - Grafana

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm upgrade --install grafana grafana/grafana --values grafana-values.yaml
```

```bash
kubectl apply -f grafana.yaml
```

### Deploy test application - yaobank

```bash
kubectl apply -f yaobank.yaml
```

### Apply Istio deny-all policy to default, dev namespaces

```bash
kubectl apply -f istio-deny-all.yaml
```

**NOTE:** Using unique Service Account per service allows for granular access control policies.

**NOTE:** Deny all policy may cause service disruption if not prepared correclty. Hence, it is only applied to default namespace where the applications are deployed. This assists in the transition to the zero trust model.

## Tests

### Test network authorization policies

- From `customer` pod to `database` pod

  ```bash
  curl -I http://database:2379/v2/keys?recursive=true
  ```

  Result: access is denied because `database` pod can only be accessed from `summary` pod as defined in Calico policy.

- From `customer` pod to `nginx` pod

  ```bash
  curl -I http://nginx
  ```

  Result: access is denied because `nginx` pod can only be accessed from Pomerium proxy.

### Test Pomerium policies

- Access `https://grafana.pomerium.kubezta.ga` using other than `grafana-admin@kubezta.ga` email.

  Result: access is denied because the service can only be accessed using `grafana-admin@kubezta.ga` account.

- Access `https://hello.pomerium.kubezta.ga` using `rkirimov3@gatech.edu` email.
  Result: access is denied becuase the service can only be accessed with accounts belonging to `kubezta.ga` domain.

### Test SPIFFE workload identity

```bash
istioctl pc secret productpage-v1-68c7c59cb5-jkvz8 -n dev -o json | jq -r \
'.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes' | base64 --decode > chain.pem
openssl x509 -in chain.pem -text
```

## Container runtime security

### Falco

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
```

```bash
helm upgrade --install falco falcosecurity/falco -f falco.yaml -n falco
```

# Future work

## High Availability

In order to maintain availability of the critical services such as Pomerium, Istio, Calico, they need to be deployed cross availability zones with multiple replicas. Due to resource constraints, this has not been implemented as the part of the final project demo.

## Audit and Visibility

### AquaSec

#### Starboard

```bash
helm repo add aqua https://aquasecurity.github.io/helm-charts
helm repo update
helm install starboard-operator aqua/starboard-operator \
  --namespace starboard-system \
  --create-namespace \
  --set="trivy.ignoreUnfixed=true"
```

View results via Lens Extension

## Logging

- Collect all the logs and aggregate in the centralized location.

## User Interface

- To abstract the configuration complexity, create web user interface to interact with the deployments. This will allow for easier click-through experience while abstracting manual definition of YAML configuration files.
