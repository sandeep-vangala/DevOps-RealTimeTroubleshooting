I Automated IAM Access Key Rotation on AWS — Here’s How
How I used Lambda, CloudWatch, and Secrets Manager to rotate access keys securely without breaking anything


One of the most overlooked but critical security practices in AWS is rotating IAM access keys regularly.

Let me be honest: I used to ignore it too. We all do — until an audit hits, or worse, credentials leak. AWS recommends rotating access keys every 90 days, but if you’re managing users or CI/CD systems with hardcoded access keys, doing it manually just isn’t scalable.

That’s why I automated the entire key rotation process using:

AWS Lambda for logic
CloudWatch Events for scheduling
Secrets Manager for securely storing new keys
IAM API for full key management
The result? No manual key updates, no expired credentials breaking workflows, and full compliance with key rotation policies.

Here’s how I did it — step by step.

1. Identify IAM Users That Require Rotation
Before jumping into automation, I first identified which IAM users in my org actually needed automated key rotation.

These users:

Used static credentials in long-running systems
Had access to S3, DynamoDB, or internal APIs
Were not using IAM roles or federated access
I tagged each of these IAM users with a custom tag:

aws iam tag-user \
  --user-name cicd-user \
  --tags Key=RotateKey,Value=True
This tagging helped me keep control over who gets rotated and avoided accidentally modifying service accounts that didn’t use access keys.

2. Create a Lambda Function to Rotate Keys
The core of this system is a Lambda function that:

Scans all tagged IAM users
Deletes the oldest key (if 2 keys exist)
Creates a new one
Updates AWS Secrets Manager with the new key
Here’s the logic:
```
import boto3
import datetime
iam = boto3.client('iam')
secrets = boto3.client('secretsmanager')
def lambda_handler(event, context):
    users = iam.list_users()['Users']
    for user in users:
        username = user['UserName']
        tags = iam.list_user_tags(UserName=username)['Tags']
        tag_dict = {tag['Key']: tag['Value'] for tag in tags}
        if tag_dict.get('RotateKey') == 'True':
            rotate_keys_for_user(username)
def rotate_keys_for_user(username):
    keys = iam.list_access_keys(UserName=username)['AccessKeyMetadata']
    
    if len(keys) == 2:
        # Delete oldest key
        oldest = sorted(keys, key=lambda x: x['CreateDate'])[0]
        iam.delete_access_key(UserName=username, AccessKeyId=oldest['AccessKeyId'])
    # Create new key
    new_key = iam.create_access_key(UserName=username)['AccessKey']
    # Store in Secrets Manager
    secret_name = f"/iam/{username}/access_key"
    secrets.put_secret_value(
        SecretId=secret_name,
        SecretString=json.dumps({
            "AccessKeyId": new_key['AccessKeyId'],
            "SecretAccessKey": new_key['SecretAccessKey']
        })
    )
    print(f"Rotated keys for {username}")
    ```
This guarantees:

IAM users always have a fresh key
No downtime (because one key is always valid)
Credentials are securely stored and accessible
3. Schedule Lambda Using CloudWatch Events
I set up a CloudWatch rule to run this Lambda weekly:

aws events put-rule \
  --schedule-expression "cron(0 4 ? * MON *)" \
  --name RotateIAMKeysWeekly
And connected it to the Lambda function:

aws events put-targets \
  --rule RotateIAMKeysWeekly \
  --targets "Id"="1","Arn"="arn:aws:lambda:region:acct-id:function:RotateKeys"
Now key rotation happens every Monday at 4 AM UTC — before business hours and without human involvement.

4. IAM Role for Lambda Execution
The Lambda function needs these IAM permissions to perform key management and Secrets Manager writes:

{
  "Effect": "Allow",
  "Action": [
    "iam:ListUsers",
    "iam:ListUserTags",
    "iam:ListAccessKeys",
    "iam:CreateAccessKey",
    "iam:DeleteAccessKey",
    "secretsmanager:PutSecretValue"
  ],
  "Resource": "*"
}
In production, I scope Resource down to specific users or secrets for tighter security.

5. Version Secrets and Send Alerts
To prevent breaking changes when keys rotate, I enabled secret versioning. Each time a new key is written to Secrets Manager, the old version is retained for fallback.

Also, I integrated Slack alerting into Lambda to notify the DevOps team after every successful rotation:

import requests
def notify_slack(username):
    webhook_url = "https://hooks.slack.com/services/your/webhook"
    message = f"🔁 Rotated IAM access keys for `{username}`"
    requests.post(webhook_url, json={"text": message})
Now we have a full audit trail and visibility — without checking logs manually.

6. Build a Fallback for Broken Key Rotation
There’s always a chance something fails — like the secret update step. To protect against that, I added this fallback logic:

If key rotation fails, don’t delete the existing key
Add a CloudWatch alarm if new secret isn’t created
Use SNS to alert the admin team immediately
Here’s a simple CloudWatch alarm using CLI:

aws cloudwatch put-metric-alarm \
  --alarm-name IAMKeyRotationFailure \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --dimensions Name=FunctionName,Value=RotateKeys \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:region:acct-id:NotifyOps
Final Thought
Before this system, I used to rotate IAM keys manually — exporting them, pasting them into CI/CD pipelines, and storing them in plaintext configs. It was error-prone and easy to forget.

Now?

Key rotation is automatic and secure
Keys are safely stored in Secrets Manager
My team is notified instantly when changes happen
I pass audits without scrambling
Security doesn’t have to be complicated — but it does have to be consistent.
