# Commands

```bash
kubectl apply -f spire.yaml
istioctl install -f spire-istio.yaml

kubectl label namespace default istio-injection=enabled

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
kubectl label namespace cert-manager istio-injection=enabled

kubectl create ns pomerium
kubectl label namespace pomerium istio-injection=enabled

kubectl apply -f pomerium-certs.yaml
helm upgrade --install pomerium pomerium/pomerium -f pomerium-values.yaml -n pomerium

helm upgrade --install nginx bitnami/nginx --set service.type=ClusterIP --set podAnnotations='inject.istio.io/templates: "sidecar\,spire"'
kubectl apply -f hello.yaml

helm upgrade --install grafana grafana/grafana --values grafana-values.yaml
kubectl apply -f grafana.yaml

kubectl apply -f yaobank.yaml

kubectl apply -f istio-policies.yaml

kubectl create ns dev
kubectl label namespace dev istio-injection=enabled
kubectl apply -f bookinfo -n dev

kubectl create ns falco
kubectl label namespace falco istio-injection=enabled
helm upgrade --install falco falcosecurity/falco -f falco.yaml -n falco

kubectl exec -it $POD -c $CONTAINER -- bash
curl -I http://database:2379/v2/keys?recursive=true # customer

curl -I http://nginx # hello

curl -I http://productpage.dev:9080 # productpage in dev ns

istioctl pc secret $POD -n dev -o json | jq -r \
'.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes' | base64 --decode > chain.pem
openssl x509 -in chain.pem -text

#####################
helm upgrade --install nginx-secret bitnami/nginx --set service.type=ClusterIP --set podAnnotations='inject.istio.io/templates: "sidecar\,spire"'
kubectl apply -f hello-secret.yaml
```
