---
layout: post
title:  "sso + sts: secure bastion-less access"
date:   2021-04-16 11:00:00 -0500
tags: aws ecs bash google sso
---

In cloud environments, access to resources are typically controled by roles via
a provider's identity management service. This provides a single service that
allows granular control on what a user can do and also the ability view a
user's actions for audit. By controling access via IAM or Azure Active
Directory, security can be improved as rather than opening a port to the
internet and restricting access to a set of IPs behind a firewall, access be
granted by the identity provider instead, often with 2FA. As an operator, this
can often place cumbersome barriers if they need to connect to a resource and
perform a function, like querying a database or debugging an issue when logs
and monitoring metrics do not provide enough detail. Also, common tools like
`scp` are not possible anymore so transferring data requires interacting
with another service.

Additionally, to minimise exposure, cloud resources are typically not
publically accessible but rather a single public facing endpoint
(bastion/jumpbox) is created, which itself has access to the specific
resources. This means a dedicated resource is needed just to provide access.
Also, since this resource is not critical to application that is being hosted
on the cloud provider, it is typically a single instance.

The following steps assume you have installed
[awscli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html).

### enabling SSH through Session Manager

Official steps can be found [here](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-enable-ssh-connections.html)

At a high-level:

1. Ensure the instance you'd like to connect to is a Managed instance

2. Edit local ~/.ssh/config file:

```
# SSH over Session Manager
host i-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
```

If you have multiple aws profiles, you will need to add entries for them as
well. See below for example.

3. Ensure user's role has permission against action `ssm:StartSession` against
   the appropriate resource.

At this point, if you don't have/need SSO, you should be able to connect to a
managed instance via SSM rather than a specific IP/CNAME and port using:

```bash
ssh -qt ec2-user@<instance-id>
```

Note, this will still require key authentication so the private key should be
passed in either using `-i` or via ssh-agent.

### enabling sso

The following assumes you've already configure Google as an SSO provider for
AWS already. If you have not, details can be found
[here](https://aws.amazon.com/blogs/security/how-to-set-up-federated-single-sign-on-to-aws-using-google-apps/)

1. Install [client](https://github.com/cevoaustralia/aws-google-auth) to
   enable authentication:
 
```bash
pip install aws-google-auth
```

2. Test connection:

```bash
aws-google-auth -I <google IDP> -S <google SP> -u <username> -R <region> -p <name of profile> -k -d <timeout> -r <awsRoleARN>
```

Note that the actual timeout limit is dependent on setting in AWS.
If successful, you should see entries in `~/.aws/credentials` and
`~/.aws/config` that correspond to the profile name you provided.

### connecting to a task

1. Ensure the ssm-agent installed on the managed instance is 'recent'
   (a version released after [mid 2019](https://aws.amazon.com/about-aws/whats-new/2019/08/amazon-ecs-now-exposes-runtime-containerids-to-apis-and-ecs-console/))

2. When running `aws ecs describe-tasks`, a `runtimeId` is returned that
   represents the docker id of the task.

3. Using the id, you can connect to the container using docker as normal
   assuming you didn't explicitly remove the shell from your docker image:

```
docker exec -it <id> bash
```

### putting it all together

As a shortcut, I added an alias `a` which allows me to connect to a specified
task in a specified cluster and region using:

```bash
a staging web blue
```

The script below is based on a specific deployment architecture as well as
a cluster and task naming taxonomy. It has the deployments separated into three
environments/accounts: test, staging and production. Additionally, the
application has a worker and web tasks and lastly, there is a blue/green
version of each task. Depending on your deployment, you will need to modify
accordingly.

```bash
#!/bin/bash -e

environment=$1
taskType=$2
version=$3
user="<USERNAME>"
gIDP="<google SSO IDP>"
gSP="<google SSO SP>"
awsRole="<AWS ROLE ARN>"


case "$environment" in
        "production")
                # initial sts call to see if already authenticated
                aws sts get-caller-identity --profile sts-prod-ca &> /dev/null || aws-google-auth -I $gIDP -S $gSP -u $user -R ca-central-1 -p sts-prod-ca -k -d 43200 -r $awsRole
                ;;
        "staging")
                aws sts get-caller-identity --profile sts-staging-ca &> /dev/null || aws-google-auth -I $gIDP -S $gSP -u $user -R ca-central-1 -p sts-staging-ca -k -d 43200 -r $awsRole
                ;;
        "test")
                aws sts get-caller-identity --profile sts-test-ca &> /dev/null || aws-google-auth -I $gIDP -S $gSP -u $user -R ca-central-1 -p sts-test-ca -k -d 43200 -r $awsRole
                ;;
        "default")
                echo "refreshing default profile..."
                aws sts get-caller-identity --profile default &> /dev/null || aws-google-auth -I $gIDP -S $gSP -u $user -R ca-central-1 -p default -k -d 43200 -r $awsRole
                exit
                ;;
esac

case "$taskType" in
        "web")
                clusterType="web"
                ;;
        "worker")
                clusterType="worker"
                ;;
esac

case "$environment" in
        "production")
                clusterName="production-$clusterType"
                profile="--profile sts-prod-ca"
                ;;
        "staging")
                clusterName="staging-$clusterType"
                profile="--profile sts-staging-ca"
                ;;
        "test")
                clusterName="test-$clusterType"
                profile="--profile sts-test-ca"
                ;;
        *)
                echo "Please specify environment name (prod, staging, test, etc...)"
                exit -1
                ;;
esac

taskName=$environment-$version

echo "finding task $taskName..."
task=$(aws ecs describe-tasks $profile --cluster $clusterName --tasks $(aws ecs list-tasks $profile --cluster $clusterName --service-name $taskName --launch-type EC2 | jq -r ".taskArns[0]"))
ecs_id=$(echo $task | jq -r ".tasks[0].containerInstanceArn")
docker_id=$(echo $task | jq -r ".tasks[0].containers[0].runtimeId" | cut -c1-6)

echo "finding instance in $clusterName..."
instance=$(aws ecs describe-container-instances $profile --cluster $clusterName --container-instances $ecs_id)
instance_id=$(echo $instance | jq -r ".containerInstances[0].ec2InstanceId")
case "$environment" in
        "staging")
                instance_id="s$instance_id"
                ;;
        "test")
                instance_id="t$instance_id"
                ;;
esac

echo "connecting [ssh -qt ec2-user@$instance_id docker exec -it $docker_id bash]..."
ssh -qt ec2-user@$instance_id docker exec -it $docker_id bash
```

To handle multiple profiles, I've prepended the AWS instance-ids with `s` for
staging and `t` for test. In this case, the ssh command to connect to a staging
instance might look like:

```bash
ssh -qt ec2-user@si-029c9df929
```

Also, my ~/.ssh/config file looks something like:

```
host i-*
    ProxyCommand sh -c "aws ssm start-session --target %h \
         --document-name AWS-StartSSHSession --parameters 'portNumber=%p' --profile sts-prod-ca"

host si-*
    ProxyCommand sh -c "aws ssm start-session --target $(echo %h | cut -c2-) \
         --document-name AWS-StartSSHSession --parameters 'portNumber=%p' --profile sts-staging-ca"

host ti-*
    ProxyCommand sh -c "aws ssm start-session --target $(echo %h | cut -c2-) \
         --document-name AWS-StartSSHSession --parameters 'portNumber=%p' --profile sts-test-ca"
```
