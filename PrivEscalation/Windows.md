```bash
xfreerdp /v:<target IP address> /u:htb-student /p:<password> 	# RDP to lab target
```
```bash
Get-WmiObject -Class win32_OperatingSystem 	Get information about the operating system
```
```bash
dir c:\ /a 	View all files and directories in the c:\ root directory
```
```bash
tree <directory> 	Graphically displaying the directory structure of a path
```
```bash
tree c:\ /f | more 	Walk through results of the tree command page by page
```
```bash
icacls <directory> 	View the permissions set on a directory
```
```bash
icacls c:\users /grant joe:f 	Grant a user full permissions to a directory
```
```bash
icacls c:\users /remove joe 	Remove a users' permissions on a directory
```
```bash
Get-Service 	PowerShell cmdlet to view running services
```
```bash
help <command> 	Display the help menu for a specific command
```
```bash
get-alias 	List PowerShell aliases
```
```bash
New-Alias -Name "Show-Files" Get-ChildItem 	Create a new PowerShell alias
```
```bash
Get-Module | select Name,ExportedCommands | fl 	View imported PowerShell modules and their associated commands
```
```bash
Get-ExecutionPolicy -List 	View the PowerShell execution policy
```
```bash
Set-ExecutionPolicy Bypass -Scope Process 	Set the PowerShell execution policy to bypass for the current session
```
```bash
wmic os list brief 	Get information about the operating system with wmic
```
```bash
Invoke-WmiMethod 	Call methods of WMI objects
```
```bash
whoami /user 	View the current users' SID
```
```bash
reg query <key> 	View information about a registry key
```
```bash
Get-MpComputerStatus 	Check which Defender protection settings are enabled
```
```bash
sconfig 	Load Server Configuration menu in Windows Server Core
```
```bash
whoami
whoami /priv
whoami /groups
systeminfo
```
### services + paths
Get-WmiObject win32_service | Select Name, StartName, PathName | Format-Table -AutoSize

### scheduled tasks
```bash
schtasks /query /fo LIST /v
```
### shares & SMB
```bash
net view
net use
Get-SmbShare
```
### ACLs (example)
```bash
icacls C:\ProgramData
```
### search for keys/creds
```bash
dir C:\Users\* -Recurse -Include *.rdp,*.config,*.xml,*.ini,*.txt -ErrorAction SilentlyContinue | Select -First 200
findstr /si /m "password" C:\Users\* 2>nul
```
```bash
echo %USERNAME%
set
```
```bash
Get-WmiObject win32_service | Select Name,StartName,PathName | Format-List
```
### or
```bash
wmic service get name,displayname,pathname,startname
```
### check a specific path
```bash
(Get-Acl "C:\\Program Files\\App\\app.exe").Access | Format-Table
```



### use accesschk (Sysinternals)
```bash
.\accesschk.exe -uwcqv Users C:\
```
```bash
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\SYSTEM\CurrentControlSet\Services
```

```bash
dir C:\Users -Recurse -Include *.rdp,*.config,*.xml,*.ini,*.json,*.log,*.bak -ErrorAction SilentlyContinue
findstr /si /n "password" C:\Users\* 2>nul
```
### look for ssh keys
```bash
dir C:\Users\*\*.ssh\* -Recurse 2>nul
```

```bash
whoami /all
```
```bash
Get-SmbShare
net view \\target
crackmapexec smb <ip> -u '' -p ''  # if using CCE
```
