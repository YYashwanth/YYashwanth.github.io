---
layout: post
title: "Designing a highly available and self healing HAProxy cluster"
description: "Designing a highly available HAProxy cluster using CloudWatch, Lambda and SSM documents"
date: 2019-08-23 11:30:18
comments: true
description: "Highly Available, Self Healing HAProxy Cluster"
keywords: "AWS, HAProxy, Load Balancer, IP Address, EC2, Source IP, Systems Manager Documents, Lambda, CloudWatch, LifeCycle Hooks"
categories:
    - HAProxy
    - Lambda
    - AutoScaling Groups
    - CloudWatch Events
tags:
- AWS
- HAProxy
- EC2
- Load Balancing
- AutoScaling Group
- Lifecycle Hooks
- Lambda
- Systems Manager Documents
---

In my previous [article](../haproxyaws) I set up a layer 4 load balancer using HAProxy in a peering configuration. But 
<ul>
  <li>what if a server goes down? I know you are thinking about auto-scaling groups. </li>
  <li>But how would you configure the server to work as a HAProxy peer? </li>
  <li>How would it know the existing stick table information from the other HAProxy server?</li>
</ul>


I deployed the HAProxy servers in auto scaling groups and put a network load balancer in front of them to route the requests to either of the HAProxy servers. As both the HAProxy servers share the same stick table, it will not matter to which HAProxy server the request went to. It would still get delivered to the right instance.

But when a linux server comes up in the ASG, I am manually configuring it to work as a HAProxy server and this is how I automated the entire process:
1) I created lifecycle hooks, for terminate and launch events, in the auto scaling groups 

![Best Jekyll Theme]({{site.baseurl}}/assets/images/ASGHooks.png)


2) Create a cloud watch rule to trigger a lambda function based on those ASG lifecycle events

![CloudWatch Rule]({{site.baseurl}}/assets/images/CWRule.png)

3) Program the lambda function to run SSM documents to configure HAProxy server and update the cluster information in the remaining HAproxy servers so they can share the same stick table.

4) Also program the lambda to remove the instance from the cluster config in remaining servers when an instance is being scaled in.

```python
import boto3
import time

def lambda_handler(event, context):
    """
    This function is the entry point to the lambda function. This function is responsible for invoking the ssm-command on the newly launched server.
    Once the newly launched server is configured, it calls the function that configures rest of the servers in the auto scaling group.
    """
    
    asg_status=event['detail']['LifecycleTransition'].split(":")
    current_instance_id = event['detail']['EC2InstanceId']
    print('Instance id: ' + current_instance_id +' is in '+asg_status[1] + ' state')
    asg_name = event['detail']['AutoScalingGroupName']
    ssm_client = boto3.client('ssm')
    asg_client = boto3.client('autoscaling')
    if asg_status[1]=="EC2_INSTANCE_LAUNCHING":
        status = execute_ssm(ssm_client, asg_name, current_instance_id, 'Linux-InstallHAProxyASG')
        if status == 'Success':
            print('HAProxy Installation successfull.\n\nInstalling Datadog')
            datadog_status = execute_ssm(ssm_client, asg_name, current_instance_id, 'Linux-InstallDatadog')
            if datadog_status == 'Success':
                print('Datadog Installation successfull.\n\nConfiguring ASG peers')
            elif datadog_status == 'Failed':
                print('Datadog Installation failed.\n\n Configuring ASG peers')
            asg_status = update_asg_peers(asg_client, ssm_client, asg_name, current_instance_id)
            if asg_status == 'Success':
                print('asg peers have been configured successfully')
            elif asg_status == 'Failed':
                print('asg peers config has failed')
        elif status == 'Failed':
            print('HAProxy installation failed')
    elif asg_status[1]=="EC2_INSTANCE_TERMINATING":
        print('About to Update the existing peers config on other instances to remove the current instance from peers')
        asg_status = update_asg_peers(asg_client, ssm_client, asg_name, current_instance_id)
        if asg_status == 'Success':
            print('asg peers have been configured successfully')
        elif asg_status == 'Failed':
            print('asg peers config has failed')
    else:
        raise Exception("Unknown state sent in the autoscaling event")
        
        
def update_asg_peers(asg_client, ssm_client, asg_name, current_instance_id):
    """
    This function is responsible for discovering other instances present in the autoscaling group. It excludes the newly launched server and executes the ssm documents against all other servers.
    
    Params:
    ssm_client: It takes the aws ssm client parameter created in the lambda_handler function.
    asg_client: It takes the aws autoscaling group client parameter created in the lambda_handler function.
    asg_name: It takes the name of the autoscaling group that's been passed through the event.
    current_instance_id: This is the instance id of the newly launched server that is to be excluded from the ssm_command_execution
    
    """
    asg_instance_group = asg_client.describe_auto_scaling_groups(AutoScalingGroupNames=[asg_name])
    asg_instance_ids = asg_instance_group['AutoScalingGroups'][0]['Instances']
    for instance in asg_instance_ids:
        if instance['InstanceId'] != current_instance_id and instance['LifecycleState'] != "Terminating:Wait":
            asg_ssm_status = execute_ssm(ssm_client, asg_name, instance['InstanceId'],'Linux-InstallHAProxyASG')
            if asg_ssm_status == 'Success':
                return "Success"
            elif asg_ssm_status == 'Failed':
                return "Failed"

def execute_ssm(ssm_client, asg_name, instance_id, document_name):
    """
    This function is responsible for checking the execution status of an ssm command that is being run against a server.
    
    Params:
    ssm_client: It takes the aws ssm client parameter created in the lambda_handler function.
    execution_command_id: this parameter is the id of the command that has been invoked against a server.
    isntance_id: This parameter is the instance id against which the ssm document is being executed.
    
    """
    ssm_agent_status = 'Inactive'
    for counter in range(10):
        instance_status = ssm_client.describe_instance_information(
            InstanceInformationFilterList=[
                {
                    'key': 'InstanceIds',
                    'valueSet': [
                        instance_id,
                        ]
                }])
        ssm_agent_status = instance_status['InstanceInformationList'][0]['PingStatus']
        if ssm_agent_status == 'Online':
            ssm_command_execution = ssm_client.send_command(
                   InstanceIds=[instance_id],
                   DocumentName=document_name,
                   MaxConcurrency='1',
                   Parameters={
                       'asgName':[
                           asg_name,
                           ]})
            if(ssm_command_execution):
                print('checking execution status')
                ssm_command_execution_status = 'InProgress'
                for counter in range(0,60):
                    if ssm_command_execution_status == 'InProgress':
                        while(True):
                            try:
                                ssm_command_execution_details = ssm_client.get_command_invocation(
                                CommandId=ssm_command_execution['Command']['CommandId'],
                                InstanceId=instance_id)
                                break
                            except Exception as e:
                                time.sleep(1)
                        ssm_command_execution_status = ssm_command_execution_details['Status']
                        time.sleep(5)
                    elif ssm_command_execution_status == 'Success':
                        return 'Success'
                    elif ssm_command_execution_status == 'Failed':
                        return 'Failed'
        else:
            time.sleep(2)


 ```

As a result of this, whenever a server is launched in the auto scaling group, this event triggers the lambda function which then configures this server to work as a HAProxy and updates remaining servers in the cluster to change their cluster information to include the newly launched server.

When a server leaves the ASG, the lambda function, triggered by the lifecycle event, updates all the servers in the group to change their cluster config by removing the scaled-in instance.

This architecture works amazing in a single account architecture. What if you wanted to control all this from a multi-account perspective? I will explain this in a new article. 

If you think of a better way to do it, please do let me know at yyellapragada@gmail.com.