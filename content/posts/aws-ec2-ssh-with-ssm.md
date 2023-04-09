---
title: "SSH access to AWS EC2 with SSM"
date: 2023-04-08T15:03:03+01:00
draft: false
---

The AWS Well-Architected Framework consists of five pillars that represent the key areas of focus for building and operating reliable, secure, efficient, and cost-effective systems in the cloud - Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization.

By using AWS SSM for secure EC2 access we are matching Security pillar in several ways: 
1. Centralized access control through AWS Identity and Access Management (IAM) policies

2. Encrypted communications - all data is encrypted in transit between the SSM client and the SSM service, and between the SSM service and the EC2 instance

3. No inbound ports - With AWS SSM, you can access your EC2 instances without opening inbound ports in your security groups

4. Auditing and logging - AWS SSM provides audit trails and logs that help you monitor and track user activity, access to resources, and changes to your systems

How to adopt SSH with AWS SSM in your project - try on test EC2 instance first, it's pretty easy

- create EC2 instance in AWS Console, make sure EC2 instances must have access to ssm.{region}.amazonaws.com on port 443 - either SSM Endpoint should be provisioned inside VPC or allow outgoing access to ssm.{region}.amazonaws.com:443 


- create `ssm-agent` IAM policy (or you can skip and use AWS-managed `AmazonSSMManagedInstanceCore`)

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:DescribeAssociation",
                "ssm:GetDeployablePatchSnapshotForInstance",
                "ssm:GetDocument",
                "ssm:DescribeDocument",
                "ssm:GetManifest",
                "ssm:GetParameter",
                "ssm:GetParameters",
                "ssm:ListAssociations",
                "ssm:ListInstanceAssociations",
                "ssm:PutInventory",
                "ssm:PutComplianceItems",
                "ssm:PutConfigurePackageResult",
                "ssm:UpdateAssociationStatus",
                "ssm:UpdateInstanceAssociationStatus",
                "ssm:UpdateInstanceInformation"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssmmessages:CreateControlChannel",
                "ssmmessages:CreateDataChannel",
                "ssmmessages:OpenControlChannel",
                "ssmmessages:OpenDataChannel"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2messages:AcknowledgeMessage",
                "ec2messages:DeleteMessage",
                "ec2messages:FailMessage",
                "ec2messages:GetEndpoint",
                "ec2messages:GetMessages",
                "ec2messages:SendReply"
            ],
            "Resource": "*"
        }
    ]
}
```

- attach newly created `ssm-agent` or AWS-managed `AmazonSSMManagedInstanceCore` IAM policy to EC2 instance


Install requirements

- python3 - `brew install python3`
- [awscli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [session-manager-plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)
- clone `ssh-over-ssm` repo - `git clone git@github.com:elpy1/ssh-over-ssm.git`
- copy `ssh-ssm.sh` to your PATH - `cp ssh-ssm.sh /usr/local/bin/`
- add ssh config block below to your `~/.ssh/config` 

```
Match Host i-*
  ProxyCommand ssh-ssm.sh %h %r
  IdentityFile ~/.ssh/ssm-ssh-tmp
  StrictHostKeyChecking no
  PasswordAuthentication no
  ChallengeResponseAuthentication no
```

Login to AWS using CLI / generate AWS CLI credentials - `yawsso` /  `leapp` / use your own CLI

Check that AWS CLI credentials are valid - `aws sts get-caller-identity` should return your `UserId/Account/Arn`

Check that EC2 instance is registered in AWS SSM 

`aws ssm describe-instance-information` 

should return EC2 instance list with SSM agent installed / ready to connect.

Try to connect to EC2 with SSH via AWS SSM

```
ssh -v  ubuntu@i-0ced59ae123456789

ubuntu@ip-172-35-1-75:~$
```

Done!

Now you can remove 22 port from incoming port list in security groups attached to EC2 instance.

How it works: on AWS side `ssm-agent` daemon on EC2 is establishing connection with AWS ssm endpoint `ssm.{region}.amazonaws.com:443` (private inside VPC or public in the Internet) and registering itself in SSM instance list.

When user is ssh-ing to instance - under the hood command like this is executed (try `set -x` in ProxyCommand script)

```
aws ssm start-session --document-name AWS-StartSSHSession --target i-0ced59ae123456789
```

Then incoming TCP connection to SSM endpoint is proxying to EC2 instance using AWS magic.

GitHub repos / gists used

https://gist.github.com/elpy1/9839ce2a06850fb25b35144bb2f70564
https://github.com/elpy1/ssh-over-ssm