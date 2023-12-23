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
mloskot@master:~$ k get pods -A -o wide
NAMESPACE          NAME                                       READY   STATUS    RESTARTS   AGE     IP            NODE        NOMINATED NODE   READINESS GATES
calico-apiserver   calico-apiserver-647f755948-8sclm          1/1     Running   0          3h4m    10.85.0.5     master      <none>           <none>
calico-apiserver   calico-apiserver-647f755948-lq9wp          1/1     Running   0          3h4m    10.85.0.6     master      <none>           <none>
calico-system      calico-kube-controllers-5fff76c8f7-b2sfh   1/1     Running   0          3h6m    10.85.0.4     master      <none>           <none>
calico-system      calico-node-rk5sc                          1/1     Running   0          3h6m    192.168.0.2   master      <none>           <none>
calico-system      calico-typha-6fbd5b6d79-jldnw              1/1     Running   0          3h6m    192.168.0.2   master      <none>           <none>
calico-system      csi-node-driver-b4dms                      2/2     Running   0          3h6m    10.85.0.3     master      <none>           <none>
kube-system        calico-node-windows-gjzdl                  2/2     Running   0          4m25s   192.168.0.4   winworker   <none>           <none>
kube-system        coredns-76f75df574-79c8l                   1/1     Running   0          3h40m   10.85.0.2     master      <none>           <none>
kube-system        coredns-76f75df574-wttnp                   0/1     Running   0          3h40m   10.85.0.1     master      <none>           <none>
kube-system        etcd-master                                1/1     Running   0          3h40m   192.168.0.2   master      <none>           <none>
kube-system        kube-apiserver-master                      1/1     Running   0          3h40m   192.168.0.2   master      <none>           <none>
kube-system        kube-controller-manager-master             1/1     Running   0          3h40m   192.168.0.2   master      <none>           <none>
kube-system        kube-proxy-cd6gb                           1/1     Running   0          3h40m   192.168.0.2   master      <none>           <none>
kube-system        kube-proxy-windows-rc8pv                   1/1     Running   0          5m40s   192.168.0.4   winworker   <none>           <none>
kube-system        kube-scheduler-master                      1/1     Running   0          3h40m   192.168.0.2   master      <none>           <none>
tigera-operator    tigera-operator-94d7f7696-96slg            1/1     Running   0          3h9m    192.168.0.2   master      <none>           <none>
```
