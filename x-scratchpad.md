
After VM-s restart:

```
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Thu, 21 Dec 2023 15:40:16 +0100   Thu, 21 Dec 2023 14:59:53 +0100   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Thu, 21 Dec 2023 15:40:16 +0100   Thu, 21 Dec 2023 14:59:53 +0100   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Thu, 21 Dec 2023 15:40:16 +0100   Thu, 21 Dec 2023 14:59:53 +0100   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Thu, 21 Dec 2023 15:40:16 +0100   Thu, 21 Dec 2023 15:00:05 +0100   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized
Addresses:
  InternalIP:  192.168.0.3
  Hostname:    vm-winworker
```

https://platform9.com/kb/kubernetes/cni-plugin-not-initialized-on-master-node
