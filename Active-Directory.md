```html
Domain Computers 	Domain Users
Domain Group Information 	Organizational Units (OUs)
Default Domain Policy 	Functional Domain Levels
Password Policy 	Group Policy Objects (GPOs)
Domain Trusts 	Access Control Lists (ACLs)
```
### AD Structure(Simple) 
```html
INLANEFREIGHT.LOCAL/
├── ADMIN.INLANEFREIGHT.LOCAL
│   ├── GPOs
│   └── OU
│       └── EMPLOYEES
│           ├── COMPUTERS
│           │   └── FILE01
│           ├── GROUPS
│           │   └── HQ Staff
│           └── USERS
│               └── barbara.jones
├── CORP.INLANEFREIGHT.LOCAL
└── DEV.INLANEFREIGHT.LOCAL
```

```bash
nslookup INLANEFREIGHT.LOCAL
nslookup 172.16.6.5
nslookup ACADEMY-EA-DC01
```

### Hash Protocol Comparison
```bash
NTLM  Random number  Domain Controller  MD4(UTF-16-LE(password))
NTLMv1  MD4 hash, random number  Domain Controller
NTLMv2 	MD4 hash, random number  Domain Controller
Kerberos  Encrypted ticket using DES, MD5  Domain Controller/Key Distribution Center
```



