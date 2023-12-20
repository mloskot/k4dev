# Create Windows node

```powershell
$vmName = 'vm-winworker'
New-VM -Name $vmName -Generation 2 -Switch ClusterNatSwitch -NewVHDPath ('F:\_\hyperv\disks\{0}.vhdx' -f $vmName) -NewVHDSizeBytes 128GB -Path 'F:\_\hyperv\config'
Get-VMNetworkAdapter -VMName $vmName | Connect-VMNetworkAdapter -SwitchName 'ClusterNatSwitch'
Get-VMNetworkAdapter -VMName $vmName | Set-VMNetworkAdapter -MacAddressSpoofing On
Set-VMMemory -VMName $vmName -DynamicMemoryEnabled $true -MinimumBytes 2GB -StartupBytes 4GB -MaximumBytes 8GB
Set-VMProcessor -VMName $vmName -Count 4
Set-VMProcessor -VMName $vmName -ExposeVirtualizationExtensions $true
Add-VMScsiController -VMName $vmName
Add-VMDvdDrive -VMName $vmName -ControllerNumber 1 -ControllerLocation 0 -Path 'D:\_\Software\WindowsServer2022\SERVER_EVAL_x64FRE_en-us.iso'
Set-VMFirmware -VMName $vmName -FirstBootDevice (Get-VMDvdDrive -VMName $vmName) -EnableSecureBoot Off
Start-VM -Name $vmName
vmconnect.exe $env:ComputerName 'winworker'
```

After OS installation unmount ISO:

```powershell
Set-VMDvdDrive -VMName 'vm-master' -ControllerNumber 1 -ControllerLocation 0 -Path $null
```

```powershell
Rename-Computer -NewName 'vm-winworker'
Restart-Computer
```

```powershell
New-NetIPAddress -IPAddress 192.168.0.3 -PrefixLength 24 -InterfaceAlias "Ethernet" -DefaultGateway 192.168.0.1
Set-DnsClientServerAddress -ServerAddresses 1.1.1.1,8.8.8.8 -InterfaceAlias "Ethernet"
```

```powershell
Get-NetRoute -InterfaceAlias "Ethernet" -AddressFamily IPv4 -DestinationPrefix "0.0.0.0/0"
Get-DnsClientServerAddress -InterfaceIndex 3 -AddressFamily IPv4
```

```powershell
# lazy way to allow all traffic from Linux nodes, including ICMP
# https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/configure-with-command-line?tabs=powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```

```powershell
# SSH is very convenient
# https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse?tabs=powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Set-Service -Name sshd -StartupType 'Automatic'
Start-Service sshd
```

```powershell
# https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement#deploying-the-public-key
$publicKey = Get-Content -Path $env:USERPROFILE\.ssh\id_rsa.pub
$remotePs1 = "powershell Add-Content -Force -Path $env:ProgramData\ssh\administrators_authorized_keys -Value '$publicKey';icacls.exe ""$env:ProgramData\ssh\administrators_authorized_keys"" /inheritance:r /grant ""Administrators:F"" /grant ""SYSTEM:F"""
ssh Administrator@winworker $remotePs1
```

```powershell
Stop-VM -VMName 'vm-winworker' -Force
Checkpoint-VM -Name 'vm-master' -SnapshotName 'BaseInstallationWithNetworkAndSSH'
```

https://github.com/kubernetes-sigs/sig-windows-tools/blob/master/guides/guide-for-adding-windows-node.md

```powershell
$CONTAINERD_VERSION="1.7.11"
curl.exe -LO https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/Install-Containerd.ps1
.\Install-Containerd.ps1 -ContainerDVersion $CONTAINERD_VERSION
```

```powershell
PS C:\Users\Administrator> .\Install-Containerd.ps1 -ContainerDVersion $CONTAINERD_VERSION
Windows feature 'Containers' is already installed.
WARNING: Windows feature 'Hyper-V' is not installed. Installing it now...
Success Restart Needed Exit Code      Feature Result
------- -------------- ---------      --------------
True    Yes            SuccessRest... {Hyper-V}
WARNING: You must restart this server to finish the installation process.
WARNING: Windows feature 'Hyper-V-PowerShell' is not installed. Installing it now...
True    Yes            SuccessRest... {Hyper-V Module for Windows PowerShell, Re...
WARNING: You must restart this server to finish the installation process.
Please reboot and re-run this script.
PS C:\Users\Administrator> Restart-Computer
```

```powershell
# continue
.\Install-Containerd.ps1 -ContainerDVersion $CONTAINERD_VERSION
```

```powershell
PS C:\Users\Administrator> .\Install-Containerd.ps1 -ContainerDVersion $CONTAINERD_VERSION
```

```powershell
# Test containerd
ctr image pull mcr.microsoft.com/windows/nanoserver:ltsc2022
ctr run --rm  mcr.microsoft.com/windows/nanoserver:ltsc2022 test cmd /c echo hello
```

```powershell
# Optional, ctr.exe friendly alternative
$NERDCTL_VERSION="1.7.2"
Invoke-WebRequest -Uri https://github.com/containerd/nerdctl/releases/download/v$NERDCTL_VERSION/nerdctl-$NERDCTL_VERSION-windows-amd64.tar.gz -OutFile .\nerdctl-$NERDCTL_VERSION-windows-amd64.tar.gz
tar -xzvf nerdctl-$NERDCTL_VERSION-windows-amd64.tar.gz -C "$env:ProgramFiles\containerd"
```

```powershell
$KUBERNETES_VERSION="1.29.0"
curl.exe -LO https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/PrepareNode.ps1
.\PrepareNode.ps1 -KubernetesVersion $KUBERNETES_VERSION
```

```powershell
curl.exe -LO "https://dl.k8s.io/release/v$KUBERNETES_VERSION/bin/windows/amd64/kubectl.exe"
```

```powershell
kubeadm join "192.168.0.2:6443" `
    --token "2wa1qe.x3x1vmhiogpwq73s" `
    --discovery-token-ca-cert-hash "sha256:7359e7582614596facf7681e5183c0105f8c7431a0d6f6572a8840c86b82d354" `
    --cri-socket "npipe:////./pipe/containerd-containerd" --v=5
```


On Linux (e.g. master node):

```bash
ssh Administrator@winworker "mkdir .kube"
scp .kube/config Administrator@winworker:.kube
```

```bash
KUBERNETES_VERSION="1.29.0"
CALICO_VERSION="3.26.1"  # calico-install and calico-node images are not yet available for 3.27.0
curl -L https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/calico/kube-proxy/kube-proxy.yml | sed "s/KUBE_PROXY_VERSION/v${KUBERNETES_VERSION}/g" | kubectl apply -f -
curl -L https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/calico/calico.yml | sed "s/CALICO_VERSION/$CALICO_VERSION/g" | kubectl apply -f -
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/calico/kube-calico-rbac.yml
```