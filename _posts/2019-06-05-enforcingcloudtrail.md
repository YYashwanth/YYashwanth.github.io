---
layout: post
title: "What if someone disables CloudTrail in a Centralised Logging Scenario?"
description: "Using AWS CloudWatch Event Bus and Lambda to enforce CloudTrail logging"
date: 2019-06-05 16:39:18
comments: true
description: "Enforcing Centralised Logging in AWS"
keywords: "AWS, CloudTrail, DevOps, CloudWatch, MultiAccount, Governance, Monitoring, Logging, Centralised"
categories:
    - Amazon Web Services
tags:
- AWS
- Automation
- Governance
- Logging
---

This article details one of the many ways of addressing the scenario where logging is crucial and a centralised logging account is setup in a multi account architecture. Of course, you can enforce service control policies at the root level to avoid any tampering with the logging mechanism, which is a proactive approach, but I wanted to talk about the reactive approach: What if someone disables CloudTrail logging in one of the member accounts?

I did some research online to find a AWS Organizations centric approach for enforcing Cloudtrail and all I could find was an approach tailored for a single account on Linux Academy. I took the learnings from that tutorial and tried an approach to enforce CloudTrail in a multi-account scenario.

To better explain the situation, consider the below scenario:

Acme Corp. is an enterprise comprised of over 5000 employees and 100 departments. They are in the transition period from on-premise to cloud infrastructure and want to isolate certain departments per industry’s standard best practices. The solution architect, responsible for the migration, started with a root account (parent account) and created multiple member accounts (child accounts) and organized them into units. To ensure that the architecture meets certain auditing and compliance standards, the architect recommended a central logging and auditing account which collects logs from all the member accounts and stores them.

But what if someone tries to disable the logging in one of the member accounts?

And if you are new to the concepts and functionalities of AWS Organisations, cross account access roles, creating centralised logging, please refer to the below links:

1.	<a href="https://docs.aws.amazon.com/organizations/latest/userguide/orgs_tutorials_basic.html">Setup AWS organization and member accounts</a>
2.	<a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html">Enable Cross-Account access</a>
3.	<a href="https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-getting-started.html">Enable cloudtrail on the member accounts</a>
4.	<a href="https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-receive-logs-from-multiple-accounts.html">Create central logging repository on AWS S3 for cloudtrail logs.</a>

<b>Actions to be taken in Parent Account<b>
1.	Once all the prerequisites have been met, navigate to your parent account’s (root account) CloudWatch console and select “Event Bus” and add a permission for the child account to send events to the parent account by entering the account number of the child account.

![Best Jekyll Theme]({{site.baseurl}}/assets/images/parent1.png)

 2. Navigate to the Lambda console and create a function and use the code below. The code  is well documented but feel free to drop in a comment for further clarification.

 ```python
import boto3

def lambda_handler(event, context):
     # create an STS client object that represents a live connection
	sts_client = boto3.client('sts')    
	sender_account_id = event['account']
    sender_account_region = event['region']
    
    
    # Call the assume_role method of the STSConnection object and pass the role
    assumedRoleObject = sts_client.assume_role(
        RoleArn="arn:aws:iam::"+sender_account_id+":role/AccesstoDev",
        RoleSessionName="AccesstoDev"
    )
    
   #get the temporary credentials that can be used to make subsequent API calls
    credentials = assumedRoleObject['Credentials']
    
    # Use the temporary credentials that AssumeRole returns to make a 
    # connection to Amazon Cloudtrail  
    cloudtrail_resource = boto3.client(
        'cloudtrail',
        aws_access_key_id = credentials['AccessKeyId'],
        aws_secret_access_key = credentials['SecretAccessKey'],
        aws_session_token = credentials['SessionToken'],
    )

    # Use the Amazon cloudtrail resource object that is now configured to start the trail.
  
    cloudtrail_resource.start_logging(Name='arn:aws:cloudtrail:'+sender_account_region+':'+sender_account_id+':trail/devaccount-trail')


 ```
PS: Exception handlers are being included and the work is in progress. The code will be updated once done. For a PoC, this serves the purpose  

 3. Configure a rule on CloudWatch which triggers a lambda function and sends notification on a SNS topic for every CloudTrail event it receives. To be precise for our requirement, we can specify the name of the event for which the CloudWatch rule has to fire up. Let’s configure the CloudWatch rule to trigger on “StopLogging” event name.

4.	Add another target as a SNS topic, if you want an administrator to be notified of this event. Select the SNS topic you created as the target.

![Best Jekyll Theme]({{site.baseurl}}/assets/images/parent2.png)

<b>Actions to be completed in Child Account<b>
1.	Navigate to the CloudWatch console and create a rule to send any notification to the event bus created in the parent account. Select the “Event bus in another account” as target and enter the account number of the account where we created the event bus.

Now all we need to do is turn of the CloudTrail in the child account and watch it turn itself back on.

If you think of a better way to do it, please do let me know at yyellapragada@gmail.com.