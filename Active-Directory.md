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
