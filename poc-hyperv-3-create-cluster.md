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

```bash
mloskot@master:~$ kubectl get node
NAME     STATUS   ROLES           AGE   VERSION
master   Ready    control-plane   26m   v1.29.0
mloskot@master:~$ kubectl get pods -A
NAMESPACE         NAME                              READY   STATUS    RESTARTS        AGE
kube-system       coredns-76f75df574-66grl          1/1     Running   0               27m
kube-system       coredns-76f75df574-tk552          1/1     Running   0               27m
kube-system       etcd-master                       1/1     Running   1 (7m21s ago)   27m
kube-system       kube-apiserver-master             1/1     Running   1 (7m21s ago)   27m
kube-system       kube-controller-manager-master    1/1     Running   1 (7m21s ago)   27m
kube-system       kube-proxy-4g9nh                  1/1     Running   1 (7m21s ago)   27m
kube-system       kube-scheduler-master             1/1     Running   1 (7m21s ago)   27m
tigera-operator   tigera-operator-94d7f7696-529qv   1/1     Running   2 (6m46s ago)   24m
```
