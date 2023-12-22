# Create Windows node

```bash
# on master or any client with access to master
KUBERNETES_VERSION="1.29.0"
CALICO_VERSION="v3.26.1" # use v-prefix; calico-install and calico-node images are not yet available for 3.27.0
curl -L https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/calico/kube-proxy/kube-proxy.yml | sed "s/KUBE_PROXY_VERSION/v${KUBERNETES_VERSION}/g" | kubectl apply -f -
curl -L https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/calico/calico.yml | sed "s/CALICO_VERSION/$CALICO_VERSION/g" | kubectl apply -f -
```

```bash
# FIXME: https://github.com/kubernetes-sigs/sig-windows-tools/issues/302#issuecomment-1866298999
# kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/calico/kube-calico-rbac.yml
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-node
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: calico-node-win
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-node
subjects:
  - kind: ServiceAccount
    name: calico-node
    namespace: kube-system
EOF
```

```bash
mloskot@master:~$ kubectl get pods -A
NAMESPACE         NAME                              READY   STATUS    RESTARTS      AGE
kube-system       calico-node-windows-czwbf         2/2     Running   0             66s
kube-system       coredns-76f75df574-66grl          1/1     Running   1 (72m ago)   4h10m
kube-system       coredns-76f75df574-tk552          1/1     Running   1 (72m ago)   4h10m
kube-system       etcd-master                       1/1     Running   2 (72m ago)   4h10m
kube-system       kube-apiserver-master             1/1     Running   2 (72m ago)   4h10m
kube-system       kube-controller-manager-master    1/1     Running   2 (72m ago)   4h10m
kube-system       kube-proxy-4g9nh                  1/1     Running   2 (72m ago)   4h10m
kube-system       kube-proxy-windows-zgzhd          1/1     Running   0             103s
kube-system       kube-scheduler-master             1/1     Running   2 (72m ago)   4h10m
tigera-operator   tigera-operator-94d7f7696-529qv   1/1     Running   4 (71m ago)   4h7m
```
