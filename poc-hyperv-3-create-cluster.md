# Create cluster

- https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart#operator-based-installation

```bash
sudo kubeadm init
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















```
ip addre show eth0 | awk '/inet / {print $2}' | cut -d / -f 1
```