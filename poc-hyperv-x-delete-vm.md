# Delete VM

```powershell
$vmName = 'vm-master'
$vm = (Get-VM -Name $vmName)
Stop-VM -Name $vm.Name -TurnOff
Remove-Item -Path (Get-VHD -VMId $vm.id).Path
$vm | Remove-VM -Force
```
