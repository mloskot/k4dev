# Create cluster

- https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart#operator-based-installation

```bash
sudo kubeadm init --pod-network-cidr=10.85.0.0/24 --service-cidr=10.85.10.0/16
```

```console
# copy generated command
kubeadm join 192.168.0.2:6443 --token ... --discovery-token-ca-cert-hash sha256:...
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
sed -i "s|cidr: 192.168.0.0/16|cidr: 10.85.0.0/24|g" custom-resources.yaml
kubectl create -f custom-resources.yaml
```

```bash
#  Optional, but useful for Calico inspections: calicoctl get nodes; calicoctl get ippools
wget -O calicoctl https://github.com/projectcalico/calico/releases/download/v${CALICO_VERSION}/calicoctl-linux-amd64
chmod +x calicoctl
sudo mv calicoctl /usr/local/bin/
```

```bash
# do NOT move further before this command shows Calico nodes running; it may take 2-3 minutes
watch kubectl get pods -n calico-system
```

```console
Every 2.0s: kubectl get pods -n calico-system                                                                                                               master: Sat Dec 23 20:35:48 2023

NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-5fff76c8f7-b2sfh   1/1     Running   0          2m58s
calico-node-rk5sc                          1/1     Running   0          2m58s
calico-typha-6fbd5b6d79-jldnw              1/1     Running   0          2m58s
csi-node-driver-b4dms                      2/2     Running   0          2m58s
```

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## Status

```console
mloskot@master:~$ kubectl get node -A
NAME     STATUS   ROLES           AGE   VERSION
master   Ready    control-plane   38m   v1.29.0
mloskot@master:~$ kubectl get pod -A
NAMESPACE          NAME                                       READY   STATUS    RESTARTS   AGE
calico-apiserver   calico-apiserver-647f755948-8sclm          1/1     Running   0          2m33s
calico-apiserver   calico-apiserver-647f755948-lq9wp          1/1     Running   0          2m33s
calico-system      calico-kube-controllers-5fff76c8f7-b2sfh   1/1     Running   0          4m57s
calico-system      calico-node-rk5sc                          1/1     Running   0          4m57s
calico-system      calico-typha-6fbd5b6d79-jldnw              1/1     Running   0          4m57s
calico-system      csi-node-driver-b4dms                      2/2     Running   0          4m57s
kube-system        coredns-76f75df574-79c8l                   1/1     Running   0          38m
kube-system        coredns-76f75df574-wttnp                   0/1     Running   0          38m
kube-system        etcd-master                                1/1     Running   0          38m
kube-system        kube-apiserver-master                      1/1     Running   0          38m
kube-system        kube-controller-manager-master             1/1     Running   0          38m
kube-system        kube-proxy-cd6gb                           1/1     Running   0          38m
kube-system        kube-scheduler-master                      1/1     Running   0          38m
tigera-operator    tigera-operator-94d7f7696-96slg            1/1     Running   0          7m54s
```

```console
mloskot@master:~$ calicoctl get node -o wide
NAME     ASN       IPV4             IPV6
master   (64512)   192.168.0.2/24

mloskot@master:~$ calicoctl get ippools -o wide
NAME                  CIDR           NAT    IPIPMODE   VXLANMODE     DISABLED   DISABLEBGPEXPORT   SELECTOR
default-ipv4-ippool   10.85.0.0/24   true   Never      CrossSubnet   false      false              all()
```
