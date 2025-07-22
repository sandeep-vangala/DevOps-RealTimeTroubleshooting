**Use Case:** In production, EKS clusters host applications exposed via Application Load Balancers (ALBs) or Network Load Balancers (NLBs). TLS certificates (managed by AWS Certificate Manager or stored in Secrets Manager) need regular rotation to maintain security. A Lambda function can automate certificate rotation, update Secrets Manager, and notify the EKS ingress controller to reload the new certificate.

**Trigger: **CloudWatch Events (EventBridge) rule scheduled monthly or triggered by ACM certificate renewal events.

**Real-Time Scenario:** Your EKS-hosted web application uses a TLS certificate for HTTPS. The Lambda function detects expiring certificates, requests a new one from ACM, updates Secrets Manager, and triggers a Kubernetes job to refresh the ingress configuration.


```

import boto3
import logging
import json
from datetime import datetime, timedelta

# Set up logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Initialize AWS clients
acm_client = boto3.client('acm')
secrets_client = boto3.client('secretsmanager')
ssm_client = boto3.client('ssm')

# Configuration
SECRET_ID = 'my-app/tls-cert'
CERT_ARN = 'arn:aws:acm:us-east-1:ACCOUNT_ID:certificate/CERT_ID'
DOMAIN_NAME = 'app.example.com'
SSM_DOCUMENT = 'AWS-RunShellScript'
EKS_INSTANCE_ID = 'i-xxxxxxxxxxxxxxxxx'  # EC2 instance with kubectl access
KUBECTL_COMMAND = 'kubectl apply -f /path/to/ingress-reload-job.yaml'





def lambda_handler(event, context):
    try:
        # Step 1: Check certificate expiry
        logger.info(f"Checking certificate {CERT_ARN} expiry")
        cert = acm_client.describe_certificate(CertificateArn=CERT_ARN)
        expiry = cert['Certificate']['NotAfter']
        if expiry < datetime.utcnow() + timedelta(days=7):
            logger.info("Certificate nearing expiry, requesting new certificate")
            
            # Step 2: Request new certificate
            new_cert = acm_client.request_certificate(
                DomainName=DOMAIN_NAME,
                ValidationMethod='DNS',
                IdempotencyToken=f'rotation-{int(datetime.utcnow().timestamp())}'
            )
            new_cert_arn = new_cert['CertificateArn']
            logger.info(f"Requested new certificate: {new_cert_arn}")
            
            # Step 3: Update Secrets Manager
            secrets_client.update_secret(
                SecretId=SECRET_ID,
                SecretString=json.dumps({'certificate_arn': new_cert_arn})
            )
            logger.info(f"Updated secret {SECRET_ID} with new ARN")
            
            # Step 4: Trigger Kubernetes job to reload ingress
            logger.info("Triggering Kubernetes job to reload ingress")
            ssm_client.send_command(
                InstanceIds=[EKS_INSTANCE_ID],
                DocumentName=SSM_DOCUMENT,
                Parameters={'commands': [KUBECTL_COMMAND]}
            )
            logger.info("Sent SSM command to reload ingress")
            
            return {
                'statusCode': 200,
                'body': json.dumps(f"Rotated certificate, new ARN: {new_cert_arn}")
            }
        else:
            logger.info("Certificate not due for rotation")
            return {
                'statusCode': 200,
                'body': json.dumps("No rotation needed")
            }

    except Exception as e:
        logger.error(f"Error during certificate rotation: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps(f"Error: {str(e)}")
        }

**IAM Permissions:**


{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "acm:DescribeCertificate",
                "acm:RequestCertificate"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:UpdateSecret",
                "secretsmanager:GetSecretValue"
            ],
            "Resource": "arn:aws:secretsmanager:*:*:secret:my-app/tls-cert*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}


```

Setup:

1) Replace ACCOUNT_ID and CERT_ID in CERT_ARN with your values.
2) Store the certificate ARN in Secrets Manager under my-app/tls-cert.
3) Create the Lambda function with Python 3.9+ and attach the IAM role.
4) Set an EventBridge rule (e.g., cron(0 0 1 * ? *) for monthly checks).
5) Configure your EKS ingress (e.g., NGINX) to fetch the certificate from Secrets Manager or integrate with Jenkins to trigger a reload job.
**Production Notes:**

1) Ensure domain validation is pre-configured in ACM.
2) Integrate with Jenkins to run a kubectl command to reload the ingress controller.
3) Use SNS for failure notifications if rotation fails.
