
# Create Linux node

Ubuntu Server 22.04 (pick the one for which apt.kubernetes.io is available e.g. Mantic is not)
If you are using virtual machines to deploy the Kubernetes workers, the anti-spoofing protection must be disabled.
- https://www.packtpub.com/product/hands-on-kubernetes-on-windows/9781838821562
- https://medium.com/outsystems-engineering/running-a-tight-ship-deploying-kubernetes-on-a-windows-server-429c9b4e6c3e

```powershell
$vmName = 'vm-master'
New-VM -Name $vmName -Generation 2 -Switch ClusterNatSwitch -NewVHDPath ('F:\_\hyperv\disks\{0}.vhdx' -f $vmName) -NewVHDSizeBytes 128GB -Path 'F:\_\hyperv\config'
Get-VMNetworkAdapter -VMName $vmName | Connect-VMNetworkAdapter -SwitchName 'ClusterNatSwitch'
Get-VMNetworkAdapter -VMName $vmName | Set-VMNetworkAdapter -MacAddressSpoofing On
Set-VMMemory -VMName $vmName -DynamicMemoryEnabled $true -MinimumBytes 2GB -StartupBytes 4GB -MaximumBytes 8GB
Set-VMProcessor -VMName $vmName -Count 4
Add-VMScsiController -VMName $vmName
Add-VMDvdDrive -VMName $vmName -ControllerNumber 1 -ControllerLocation 0 -Path 'D:\_\Software\Ubuntu\ubuntu-22.04.3-live-server-amd64.iso'
Set-VMFirmware -VMName $vmName -FirstBootDevice (Get-VMDvdDrive -VMName $vmName) -EnableSecureBoot Off
Start-VM -Name $vmName
vmconnect.exe $env:ComputerName $vmName
```

After OS installation unmount ISO:

```powershell
Set-VMDvdDrive -VMName 'vm-master' -ControllerNumber 1 -ControllerLocation 0 -Path $null
Restart-VM -Name 'vm-master' -Force
```

For the internal vSwitch, network needs to configure manually:

```text
192.168.0.0/24
192.168.0.2 # master
192.168.0.3 # linworker
```

```bash
$ cat /run/systemd/network/10-netplan-eth0.network
[Match]
Name=eth0

[Network]
LinkLocalAddressing=ipv6
Address=192.168.0.2/24
DNS=1.1.1.1
DNS=8.8.8.8

[Route]
Destination=0.0.0.0/0
Gateway=192.168.0.1
```

```powershell
ssh-keygen -R master
Get-Content  $env:USERPROFILE\.ssh\id_rsa.pub | ssh mloskot@master "cat >> .ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

```bash
ssh mloskot@master
```

```bash
sudo -i bash -c 'echo "mloskot ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers'
```

Install tools that enable some dedicated features for integrating with hypervisors:
- https://www.packtpub.com/product/hands-on-kubernetes-on-windows/9781838821562

```bash
sudo apt -y remove docker docker.io containerd runc
sudo apt -y update && sudo apt -y upgrade && sudo apt -y dist-upgrade 
sudo apt -y install \
    apt-transport-https \
    ca-certificates \
    curl \
    gpg \
    iptables \
    iputils-ping \
    linux-cloud-tools-virtual \
    linux-tools-virtual \
    netcat \
    vim
```

```bash
sudo vim /etc/fstab  # (remove any swap entry)
sudo swapoff -a
```

Enable bridged IPv4 traffic to iptables chains when using Calico
- https://github.com/kubernetes-sigs/sig-windows-tools/blob/master/guides/calico.md#prepare-control-plane-for-calico

```bash
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

```bash
sudo tee /etc/modules-load.d/kubernetes.conf <<EOF
br_netfilter
EOF
```

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
sudo sysctl --system
```

```bash
sudo tee --append /etc/hosts <<< "192.168.0.1 gateway.cluster   gateway     # ClusterNatSwitch IP"
sudo tee --append /etc/hosts <<< "192.168.0.2 master.cluster    master      # Kubernetes Linux node (control-plane)"
sudo tee --append /etc/hosts <<< "192.168.0.3 linworker.cluster linworker   # Kubernetes Linux node"
sudo tee --append /etc/hosts <<< "192.168.0.4 winworker.cluster winworker   # Kubernetes Windows node"
```

```bash
sudo reboot now
```

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

```powershell
Stop-VM -VMName 'vm-master' -Force
Checkpoint-VM -Name 'vm-master' -SnapshotName 'BaseConfiguration'
Start-VM -VMName 'vm-master'
```

## Install containerd

```bash
sudo apt install iptables
```

```bash
CONTAINERD_VERSION=1.7.11
wget https://github.com/containerd/containerd/releases/download/v${CONTAINERD_VERSION}/containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz
sudo tar -xzvf containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz -C /usr/local
```

```bash
# containerd used for Kubernetes needs to start as cgroup, so configure systemd cgroup driver for runc
sudo mkdir /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
# i.e.
# vim /etc/containerd/config.toml
# go to [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
# set SystemdCgroup true
```

```bash
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system
sudo mv containerd.service /usr/local/lib/systemd/system/containerd.service
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

```bash
RUNC_VERSION=1.1.10
wget https://github.com/opencontainers/runc/releases/download/v${RUNC_VERSION}/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

```bash
CNI_PLUGINS_VERSION=1.4.0
sudo wget https://github.com/containernetworking/plugins/releases/download/v${CNI_PLUGINS_VERSION}/cni-plugins-linux-amd64-v${CNI_PLUGINS_VERSION}.tgz
sudo mkdir -p /opt/cni/bin
sudo tar -xzvf cni-plugins-linux-amd64-v${CNI_PLUGINS_VERSION}.tgz -C /opt/cni/bin
```

```bash
NERDCTL_VERSION=1.7.2
wget https://github.com/containerd/nerdctl/releases/download/v${NERDCTL_VERSION}/nerdctl-${NERDCTL_VERSION}-linux-amd64.tar.gz
sudo tar -xzvf nerdctl-${NERDCTL_VERSION}-linux-amd64.tar.gz -C /usr/local/bin 
```

```bash
sudo shutdown now
```

```powershell
Stop-VM -VMName 'vm-master' -Force
Checkpoint-VM -Name 'vm-master' -SnapshotName 'BaseConfigurationWithContainerD'
Start-VM -VMName 'vm-master'
```

```bash
sudo nerdctl run -d --name nginx -p 80:80 nginx:alpine
sudo nerdctl images
sudo nerdctl ps
```

> FIXME: Why if ContainerD 3.26.1 is not tested out with nerdctl as abobe before `kubeadm init` makes kubelet not erady due to `KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized`

## Install Kubernetes tools

```bash
KUBERNETES_VERSION_MM=1.29
curl -fsSL https://pkgs.k8s.io/core:/stable:/v${KUBERNETES_VERSION_MM}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v${KUBERNETES_VERSION_MM}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```bash
sudo apt -y update
sudo apt -y install kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

```bash
sudo shutdown now
```

```powershell
Stop-VM -VMName 'vm-master' -Force
Checkpoint-VM -Name 'vm-master' -SnapshotName 'BaseConfigurationWithKubernetes'
Start-VM -VMName 'vm-master'
```
