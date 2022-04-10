# Commands

```bash
kubectl create namespace istio-system
kubectl create namespace cert-manager
kubectl create namespace pomerium
kubectl create namespace falco
kubectl create namespace dev
kubectl label namespace default istio-injection=enabled
kubectl label namespace dev istio-injection=enabled
kubectl label namespace cert-manager istio-injection=enabled
kubectl label namespace pomerium istio-injection=enabled
kubectl label namespace falco istio-injection=enabled

helm install istio-base istio/base -n istio-system
helm install istiod istio/istiod -n istio-system \
  --set sidecarInjectorWebhook.enableNamespacesByDefault=true

echo '{ installation: {kubernetesProvider: AKS }}' > calico-values.yaml
helm install calico projectcalico/tigera-operator --version v3.22.1 -f calico-values.yaml

calicoctl patch FelixConfiguration default --patch \
   '{"spec": {"policySyncPathPrefix": "/var/run/nodeagent"}}'

curl https://docs.projectcalico.org/archive/v3.21/manifests/alp/istio-inject-configmap-1.10.yaml -o istio-inject-configmap.yaml
kubectl patch configmap -n istio-system istio-sidecar-injector --patch "$(cat istio-inject-configmap.yaml)"

kubectl apply -f https://docs.projectcalico.org/archive/v3.21/manifests/alp/istio-app-layer-policy-envoy-v3.yaml

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --set installCRDs=true

kubectl apply -f pomerium-certs.yaml
helm upgrade --install pomerium pomerium/pomerium -f pomerium-values.yaml -n pomerium

helm upgrade --install nginx bitnami/nginx --set service.type=ClusterIP --set podAnnotations='inject.istio.io/templates: "sidecar\,spire"'
kubectl apply -f hello.yaml

helm upgrade --install grafana grafana/grafana --values grafana-values.yaml
kubectl apply -f grafana.yaml

kubectl apply -f yaobank.yaml
```
