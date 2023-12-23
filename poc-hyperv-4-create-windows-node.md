# Create Windows node

## Create VM

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
# on host, see https://github.com/microsoft/MSLab/tree/bfeab657b8ff2d037045ede0c471f2bef8e2f42f/Scripts
$username = "Administrator"
$password = "Pantera1979"
$passwordSecureString = New-Object -TypeName System.Security.SecureString
$password.ToCharArray() | ForEach-Object { $passwordSecureString.AppendChar($_) }
$credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $username, $passwordSecureString
```

```powershell
Invoke-Command -VMName 'vm-winworker' -Credential $credential -ScriptBlock { `
  Rename-Computer -NewName 'winworker'; `
}
```

```powershell
Invoke-Command -VMName 'vm-winworker' -Credential $credential -ScriptBlock { `
    New-NetIPAddress -IPAddress 192.168.0.4 -PrefixLength 24 -InterfaceAlias "Ethernet" -DefaultGateway 192.168.0.1; `
    Set-DnsClientServerAddress -ServerAddresses 1.1.1.1,8.8.8.8 -InterfaceAlias "Ethernet"; `
}
```

```powershell
Invoke-Command -VMName 'vm-winworker' -Credential $credential -ScriptBlock { `
    Get-NetRoute -InterfaceAlias "Ethernet" -AddressFamily IPv4 -DestinationPrefix "0.0.0.0/0";
    Get-DnsClientServerAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4;
}
```

```powershell
# lazy way to allow all traffic from Linux nodes, including ICMP
# https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/configure-with-command-line?tabs=powershell
Invoke-Command -VMName 'vm-winworker' -Credential $credential -ScriptBlock { `
    Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False; `
}
```

```powershell
# SSH is very convenient
# https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse?tabs=powershell
Invoke-Command -VMName 'vm-winworker' -Credential $credential -ScriptBlock { `
    Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0; `
    Set-Service -Name sshd -StartupType 'Automatic'; `
    Start-Service sshd; `
}
```

```powershell
# Fix broken SSH on Windows
# - https://stackoverflow.com/a/77705199/151641
# - https://github.com/PowerShell/Win32-OpenSSH/issues/1942#issuecomment-1868015179
Invoke-Command -VMName 'vm-winworker' -Credential $credential -ScriptBlock { `
    $content = Get-Content -Path $env:ProgramData\ssh\sshd_config;
    $content = $content -replace '.*Match Group administrators.*', '';
    $content = $content -replace '.*AuthorizedKeysFile.*__PROGRAMDATA__.*', '';
    Set-Content -Path $env:ProgramData\ssh\sshd_config -Value $content;
}
```

```powershell
Invoke-Command -VMName 'vm-winworker' -Credential $credential -ScriptBlock { `
    Restart-Computer  -Force; `
}
```

```powershell
# https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement#deploying-the-public-key
ssh-keygen -R winworker
$publicKey = Get-Content -Path $env:USERPROFILE\.ssh\id_rsa.pub
$remoteCmd = "powershell New-Item -Force -ItemType Directory -Path C:\Users\Administrator\.ssh; Add-Content -Force -Path C:\Users\Administrator\.ssh\authorized_keys -Value '$publicKey';icacls.exe ""C:\Users\Administrator\.ssh\authorized_keys "" /inheritance:r /grant ""Administrators:F"" /grant ""SYSTEM:F"""
ssh Administrator@winworker $remoteCmd
```

```powershell
Stop-VM -VMName 'vm-winworker' -Force;
Checkpoint-VM -Name 'vm-winworker' -SnapshotName 'BaseInstallationWithNetworkAndSSH';
Start-VM -VMName 'vm-winworker';
```

## Join node

- https://github.com/kubernetes-sigs/sig-windows-tools/blob/master/guides/guide-for-adding-windows-node.md

```powershell
ssh Administrator@winworker powershell
```

```powershell
# TODO: For some reason this remote invocation does not work, it gets interrupted at untaring, outputs
#   x containerd.exe
# and terminates
# Invoke-Command -VMName 'vm-winworker' -Credential $credential -ScriptBlock { `
#     $CONTAINERD_VERSION="1.7.11";
#     curl.exe -LO https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/Install-Containerd.ps1; `
#     .\Install-Containerd.ps1 -ContainerDVersion $CONTAINERD_VERSION; `
# }
$CONTAINERD_VERSION="1.7.11";
curl.exe -LO https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/Install-Containerd.ps1;
.\Install-Containerd.ps1 -ContainerDVersion $CONTAINERD_VERSION;
# if reboot is required, then re-run the installer script after reboot
$CONTAINERD_VERSION="1.7.11";
.\Install-Containerd.ps1 -ContainerDVersion $CONTAINERD_VERSION;
```

```powershell
# Test containerd
ctr image pull mcr.microsoft.com/windows/nanoserver:ltsc2022
ctr run --rm  mcr.microsoft.com/windows/nanoserver:ltsc2022 test cmd /c echo hello
```

```powershell
# Optional, ctr.exe friendly alternative
$NERDCTL_VERSION="1.7.2";
Invoke-WebRequest -Uri https://github.com/containerd/nerdctl/releases/download/v$NERDCTL_VERSION/nerdctl-$NERDCTL_VERSION-windows-amd64.tar.gz -OutFile .\nerdctl-$NERDCTL_VERSION-windows-amd64.tar.gz;
tar -xzvf nerdctl-$NERDCTL_VERSION-windows-amd64.tar.gz -C "$env:ProgramFiles\containerd";
```

```powershell
$KUBERNETES_VERSION="1.29.0";
curl.exe -LO https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/PrepareNode.ps1;
.\PrepareNode.ps1 -KubernetesVersion $KUBERNETES_VERSION;
```

```powershell
curl.exe -LO "https://dl.k8s.io/release/v$KUBERNETES_VERSION/bin/windows/amd64/kubectl.exe"
```

```powershell
kubeadm join 192.168.0.2:6443 `
    --token hncx9q.2d0btvn9xh6ot6os `
    --discovery-token-ca-cert-hash sha256:588ebcc05cb86c30fae5b6bf1108b35ffaf9c641a708826dc6ccf0695ddc5849
```

On Linux (e.g. master node):

```bash
ssh Administrator@winworker "mkdir .kube"
scp .kube/config Administrator@winworker:.kube
```
