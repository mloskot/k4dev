# Test

On Linux (e.g. master node):

```bash
ssh Administrator@winworker "mkdir .kube"
scp .kube/config Administrator@winworker:.kube
```

On host

```powershell
scp mloskot@master:.kube/config .\.kubeconfig
$env:KUBECONFIG="$PWD\.kubeconfig"
```

```powershell
kubectl apply -f test-windows-deployment.yaml
```