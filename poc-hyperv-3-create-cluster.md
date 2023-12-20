# Create cluster

- https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart#operator-based-installation

```bash
sudo kubeadm init
```

```console
...
kubeadm join 192.168.0.2:6443 --token 2wa1qe.x3x1vmhiogpwq73s \
        --discovery-token-ca-cert-hash sha256:7359e7582614596facf7681e5183c0105f8c7431a0d6f6572a8840c86b82d354
```

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
CALICO_VERSION=3.27.0
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v${CALICO_VERSION}/manifests/tigera-operator.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v${CALICO_VERSION}/manifests/custom-resources.yaml
# inspect spec.calicoNetwork.ipPools
kubectl create -f custom-resources.yaml
```

```bash
watch kubectl get pods -n calico-system
```
