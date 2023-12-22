# Prepare Control Plane for Calico

- https://github.com/kubernetes-sigs/sig-windows-tools/blob/master/guides/calico.md

```bash
CALICO_VERSION=3.26.1 # Use version for which latest images are available from https://hub.docker.com/r/sigwindowstools for Windows node
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v${CALICO_VERSION}/manifests/tigera-operator.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v${CALICO_VERSION}/manifests/custom-resources.yaml
# inspect spec.calicoNetwork.ipPools
kubectl create -f custom-resources.yaml
```

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-
```
