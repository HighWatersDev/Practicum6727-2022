# Calico

Calico provides network services plane tying all the components together. This can be replaced with Cilium based on Kubernetes cluster setup.

# SPIRE

SPIRE is implementation of SPIFFE framework and provides attestation service for individual workloads. Each workload is issued a short-lived x509 certificate referencing pod name and service account. These certificates are then used with Istio and Envoy to provide Mutual TLS with unique workload attestation.

# Istio

Istio service mesh provides inter-cluster traffic management via authorization policies and cross-pod traffic encryption through Mutual TLS via SPIRE integration. Istio Authorization policies enable least privileges by limiting workloads' access to only necessary services/ports/operations.

# Cert-Manager

Cert-Manager provides certificate renewal and management across services to ensure uninterrupted certificate issuance.

# Pomerium

Pomerium provides proxy service accepting all inbound traffic from the Internet and validating it through its integration with OIDC Identity Provider. Through the access rules, it authenticates and authorizes all external requests to individual workloads.

# Tests

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
  
- Access `https://grafana.pomerium.kubezta.ga` using other than `grafana-admin@kubezta.ga` email.

  Result: access is denied because the service can only be accessed using `grafana-admin@kubezta.ga` account.

- Access `https://hello.pomerium.kubezta.ga` using `rkirimov3@gatech.edu` email.
  Result: access is denied becuase the service can only be accessed with accounts belonging to `kubezta.ga` domain.

- Check SPIRE certificate
  istioctl pc secret productpage-v1-68c7c59cb5-jkvz8 -n dev -o json | jq -r \
'.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes' | base64 --decode > chain.pem

  openssl x509 -in chain.pem -text

Test webpages
-  /
  - access from kubezta.ga domain - allowed
  - access from gatech.edu domain - denied

- https://productpage.pomerium.kubezta.ga/
  - access from rkirimov3@kubezta.ga - allowed
  - access from nginx@kubezta.ga - denied

- https://grafana.pomerium.kubezta.ga/
  - access from grafana-admin@kubezta.ga - allowed

- https://hello.pomerium.kubezta.ga/
  - access from nginx@kubezta.ga - allowed

- https://hello-secret.pomerium.kubezta.ga/
  - access from nginx@kubezta.ga - allowed
