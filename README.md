# Practicum6727-2022

## Install kind

  - kind create cluster --config=kind-cluster.yaml

## Install Cilium

```bash
helm install cilium cilium/cilium --version 1.11.1 \
   --namespace kube-system \
   --set kubeProxyReplacement=partial \
   --set hostServices.enabled=false \
   --set externalIPs.enabled=true \
   --set nodePort.enabled=true \
   --set hostPort.enabled=true \
   --set bpf.masquerade=false \
   --set image.pullPolicy=IfNotPresent \
   --set ipam.mode=kubernetes
```
