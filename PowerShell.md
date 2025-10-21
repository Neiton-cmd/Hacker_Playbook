## PowerShell usefull commands

```bash
Get-Service | ? {$_.Status -eq "Running"} | select -First 2 | fl
Get-ACL -Path HKLM:\System\CurrentControlSet\Services\wuauserv | Format-List
```
