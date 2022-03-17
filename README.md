# Practicum6727-2022

## Install local k8s cluster

### [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

  - kind create cluster --config=kind-cluster.yaml

### [minikube](https://minikube.sigs.k8s.io/docs/start/)

  - minikube start --network-plugin=cni --cni=false

## Install Cilium

### with Helm

```bash
helm install cilium cilium/cilium --version 1.11.2 \
   --namespace kube-system
```
### with cilium cli

`cilium install`

