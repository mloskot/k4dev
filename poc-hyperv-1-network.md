# Hyper-V Network

Create Virtual Switch and configure Virtual Network with NAT:
- https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network

```powershell
(New-Object Security.Principal.WindowsPrincipal $([Security.Principal.WindowsIdentity]::GetCurrent())).IsInRole([Security.Principal.WindowsBuiltinRole]::Administrator))
```

```powershell
New-VMSwitch -SwitchName "ClusterNatSwitch" -SwitchType Internal -Notes "Virtual Switch with NAT used for networking between nodes of hybrid Kubernets cluster, with Internet access."
```

```powershell
Get-NetAdapter |Format-Table -AutoSize
```

```powershell
New-NetIPAddress -IPAddress 192.168.0.1 -PrefixLength 24 -InterfaceAlias "vEthernet (ClusterNatSwitch)"
```

```powershell
New-NetNAT -Name "ClusterNatNetwork" -InternalIPInterfaceAddressPrefix 192.168.0.0/24
```

```powershell
Get-VMSwitch * | Format-Table Name
```

```powershell
Add-Content -Path "$($Env:WinDir)\system32\Drivers\etc\hosts" -Value "192.168.0.1 gateway.cluster   gateway     # ClusterNatSwitch IP"
Add-Content -Path "$($Env:WinDir)\system32\Drivers\etc\hosts" -Value "192.168.0.2 master.cluster    master      # Kubernetes Linux node (control-plane)"
Add-Content -Path "$($Env:WinDir)\system32\Drivers\etc\hosts" -Value "192.168.0.3 linworker.cluster linworker   # Kubernetes Linux node"
Add-Content -Path "$($Env:WinDir)\system32\Drivers\etc\hosts" -Value "192.168.0.4 winworker.cluster winworker   # Kubernetes Windows node"
```

```powershell
ssh-keygen
```
