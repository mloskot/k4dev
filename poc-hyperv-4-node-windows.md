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
vmconnect.exe $env:ComputerName $vmName
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
Windows feature 'Containers' is already installed.
Windows feature 'Hyper-V' is already installed.
Windows feature 'Hyper-V-PowerShell' is already installed.
Getting ContainerD binaries
Downloading https://github.com/containerd/containerd/releases/download/v1.7.11/containerd-1.7.11-windows-amd64.tar.gz to C:\Program Files\containerd\containerd.tar.gz
x containerd.exe
x containerd-shim-runhcs-v1.exe
x ctr.exe
x containerd-stress.exe
Registering ContainerD as a service
Starting ContainerD service
Downloading https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.27.0/crictl-v1.27.0-windows-amd64.tar.gz to C:\Program Files\containerd\crictl.tar.gz
x crictl.exe

    Directory: C:\Users\Administrator

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----        12/20/2023  10:05 AM                .crictl
Done - please remember to add '--cri-socket "npipe:////./pipe/containerd-containerd"' to your kubeadm join command if your kubernetes version is below 1.25!
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