---
title: "Publicly hosting the backend APIs"
tags:
  - AWS
  - Manga Alert Italia
---

Hey there! In today's blog post I'm going to talk about a topic I had to wrap my head around for quite a while. My goal was to host the backend APIs, enforcing communication over HTTPS, without resorting to pricy solutions (Load Balancers I'm talking to you).

# Automatic Load Balancer
I know, I know, it's included in the AWS Free Tier, so for the first 12 months it's entirely free. Problem is I want to create a "long-term" solution with my application, so I want to keep it hosted even when the Free Tier will expire, so there's no chance in hell I'm ever gonna pay $16-20 a month to keep it, that's madness.

# API Gateway + NLB
Again, this is too pricy: the API Gateway is not, but the Network Load Balancer costs a little less than an ALB, so it's *not feasible*. 

# Cloudfront CDN: automatic SSL certificates management
This could have been a really good solution to be honest. I can enforce HTTPS through the CDN (also the SSL certificates would be free and entirely AWS-managed, renewal included, so that would be really great), problem is that the communication between the CDN and the EC2 on which the backend is hosted would be over HTTP, and the EC2 should still be accessible by the whole Internet. You could argue that I could restrict, through the security group rules, the number of IPs that can access the machine, but it's simply not doable: there's a list of IPs associated to AWS' CDNs, but they change regularly and they are also A LOT, not to mention that this is just a "workaround" to the initial problem, so no. 

# Final Solution: SSL certificates by Let's Encrypt, automatic renewal and Nginx

### SSL certificates by Let's Encrypt
Let's Encrypt provides free SSL certificates, and they can be obtained through the use of their ```certbot``` through command line, exactly what I need. The only downside is that certificates will expire in just 60 days and the renewal action must be automated in some way.

### Automatic renewal: AWS Lambda and Secrets Manager to store certificates
The most simple solution that came to my mind has been to automate the SSL certificates renewals through an AWS Lambda triggered every 60 days. This Lambda will save certificates as secrets on Secrets Manager to centralize their management. ECS instances will retrieve these certificates every time a new instance will spawn. Plus, at the end of the renewal pipeline, the main backend instance will be deleted and respawned by the Lambda function to retrieve the newly renewed certificates. 

```python
import os
import boto3
import logging
import json 
from certbot._internal import main as certbot_main
from botocore.exceptions import ClientError

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    MAIN_DOMAIN_NAME = "api.mangaalertitalia.it"
    SECRET_NAME = "ssl/api.mangaalertitalia.it"
    DOMAIN_NAMES = [
        "api.mangaalertitalia.it",
        "www.api.mangaalertitalia.it"
    ]
    REGION_NAME = "eu-west-1"
    EMAIL = "aloisi.lorenzo99@gmail.com"

    # Create directories for Certbot
    os.makedirs('/tmp/etc/letsencrypt', exist_ok=True)
    os.makedirs('/tmp/var/lib/letsencrypt', exist_ok=True)
    os.makedirs('/tmp/var/log/letsencrypt', exist_ok=True)

    # Run Certbot to obtain the certificate
    certbot_args = [
        'certonly',
        '--non-interactive',
        '--agree-tos',
        '--email', EMAIL,
        '--dns-route53',
        '-d', DOMAIN_NAMES[0],
        '-d', DOMAIN_NAMES[1],
        '--config-dir', '/tmp/etc/letsencrypt',
        '--work-dir', '/tmp/var/lib/letsencrypt',
        '--logs-dir', '/tmp/var/log/letsencrypt'
    ]

    try:
        certbot_main.main(certbot_args)

    except Exception as e:
        logger.error(f"Certbot failed: {e}")
        raise e

    # Read the certificate files
    cert_path = f'/tmp/etc/letsencrypt/live/{MAIN_DOMAIN_NAME}/fullchain.pem'
    key_path = f'/tmp/etc/letsencrypt/live/{MAIN_DOMAIN_NAME}/privkey.pem'

    with open(cert_path, 'r') as cert_file:
        certificate = cert_file.read()

    with open(key_path, 'r') as key_file:
        private_key = key_file.read()

    # Update AWS Secrets Manager
    client = boto3.client('secretsmanager', region_name=REGION_NAME)

    secret_value = {
        'certificate': certificate,
        'private_key': private_key
    }
    secret_string = json.dumps(secret_value)

    try:
        client.put_secret_value(
            SecretId=SECRET_NAME,
            SecretString=secret_string
        )
        logger.info(f"Successfully updated secret {SECRET_NAME}")
        
    except ClientError as e:
        logger.error(f"Error updating secret: {e}")
        raise e

    return {
        'statusCode': 200,
        'body': f"SSL certificate for {MAIN_DOMAIN_NAME} renewed and updated in Secrets Manager."
    }
```

### Nginx reverse proxy
There's nothing much to say about this choice: I used Nginx as a reverse proxy, to redirect HTTPS traffic from ```https://api.mangaalertitalia.it``` to the local Flask server. 
