### Enumeration list of commands with description (LINUX)

Ping probes - testing connection to the target machine or URL

ping <options> <target>
-c <count> --- number of ping requests to send. Ex. ping -c 5 10.10.10.10.Description:How many requests you want yo send
-i <seconds> --- interval between each packet. Ex ping -i 0.5 10.10.10.10
-s <bytes> --- packet size in bytes
Also you can ping url for example
ping google.com

Nmap scans main descriptions
By nmap you can scan ports , understand versions of the services runned in the target for more information
you can use a command:
nmap --help

main syntax:
nmap <options> <target>

Options:

nmap <target> --- basic scan - checks what the most popular ports are open
-sS --- TCP SYN scan , fast and common
-sT --- also a TCP scan used when SYN scan isn't allowed
-sU --- UDP
-p port,port --- list of ports which you want to scan
-A --- agressive scan you can detect OS, versions also includes script scanning, tracerouse
-sV --- service version
-sC --- run defoult scripts 
-Pn --- skip ping probes
-v/-vv --- verbose mode(more information about scanning)
-T<0-5> --- set fast of scan where 0 is slow 5 fast
-oN output.txt --- save scan results to a file
-p- --- scan all ports 0-65000
-O --- detect OS

Examples that I often use:

nmap -p- -T4 -v <target> --- scans all ports || with fast speed || more information mode
nmap -p 22,80,139 -sCV <target> --- scan only defined ports || run defoult scripts and detect versions

Nikto - Web server vulnerability scanner for more information nikto --help
Syntax:
nikto <option> <url>
Example
nikto -h <url>

Subfinder - Advanced subdomain enumeration for more information subfinder --help
Syntax:
subfinder <option> <host>

Example of host
google.com

Example of command:

subfinder -d <host>

Gobuster - Web directory brute-forcing for more information gobuster --help

Syntax:

gobuster [mode] -u <url> -w <wordlist> [options]

DIR mode useful options:

gobuster dir -u http://example.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

-u <url> --- target url
-w <wordlist> --- path to wordlist
-x <extensions> --- add directory extensions
-t <threads> --- number of threads for default 10
-s <status> --- show only specific HTTP status codes

Exampe of using:

gobuster dir -u http://target.com -w /home/kali/common.txt -x php,html,txt -t 30

Also in gobuster exists different modes which i do not using 

Want to say about wordlists:
SecLists --- collection of usefull wordlists that can be used for brute-forcing , all types like passwords , subdomains,
url directories , SQLi 

ffuf - usually used for brute-forcing subdomains but also can be used for hidden url directories









































