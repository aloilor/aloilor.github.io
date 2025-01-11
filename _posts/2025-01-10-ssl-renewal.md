---
title: "Automatic SSL certificates renewal with AWS Lambda, Secrets Manager and Let's Encrypt"
tags:
  - AWS
  - Manga Alert Italia
---

Hey there and happy new year! This is a follow-up post to the previous [Publicly hosting the backend APIs](https://aloilor.github.io/hosting-backend-apis/), I'm going to share some more info about the automatic SSL certificates renewal system I had to come up with to do everything free.

## SSL certificates by Let's Encrypt
Let's Encrypt provides free SSL certificates, and they can be obtained through the use of their ```certbot``` through command line, exactly what I need. The only downside is that certificates will expire in just 60 days and the renewal action must be automated in some way.

## Automatic renewal: AWS Lambda and Secrets Manager to store certificates
The most simple (and free) solution that came to my mind has been to automate the SSL certificates renewals through an AWS Lambda triggered every 45 days. This Lambda will save certificates as secrets on Secrets Manager to centralize their management. ECS instances will retrieve these certificates every time a new instance will spawn. Plus, at the end of the renewal pipeline, the main backend instance will be deleted and respawned by the Lambda function to retrieve the newly renewed certificates. 

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

    # ECS cluster and service
    ECS_CLUSTER_NAME = "manga-alert-ecs-cluster" 
    ECS_SERVICE_NAME = "manga-alert-main-backend-service"
    ECS_TASK_NAME = "manga-alert-ecs-main-backend"


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

    # Force a new ECS deployment so the main backend task can fetch the new certificate
    try:
        ecs_client = boto3.client('ecs', region_name=REGION_NAME)
        ecs_client.update_service(
            cluster=ECS_CLUSTER_NAME,
            service=ECS_SERVICE_NAME,
            taskDefinition= ECS_TASK_NAME,
            forceNewDeployment=True
        )
        logger.info(f"Forcing new deployment for ECS service {ECS_SERVICE_NAME} in cluster {ECS_CLUSTER_NAME}")
    except ClientError as e:
        logger.error(f"Error updating ECS service: {e}")
        raise e


    return {
        'statusCode': 200,
        'body': f"SSL certificate for {MAIN_DOMAIN_NAME} renewed and updated in Secrets Manager."
    }
```

### AWS EventBridge trigger every 45 days

```terraform
# EventBridge rule to trigger every 45 days
resource "aws_cloudwatch_event_rule" "ssl_renew_certs_schedule" {
  name                = "ssl-renew-certificates-every-45days"
  description         = "Triggers the ssl-renew-certs Lambda every 45 days"
  schedule_expression = "rate(45 days)"
}

# EventBridge target
resource "aws_cloudwatch_event_target" "ssl_renew_certs_target" {
  rule      = aws_cloudwatch_event_rule.ssl_renew_certs_schedule.name
  target_id = "ssl-renew-certs-lambda-target"
  arn       = aws_lambda_function.lambda_ssl_renew_certs.arn
}

# Permission for EventBridge to invoke the Lambda
resource "aws_lambda_permission" "ssl_renew_certs_eventbridge_permission" {
  statement_id  = "AllowEventBridgeInvokeLambda"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.lambda_ssl_renew_certs.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.ssl_renew_certs_schedule.arn
}

```


