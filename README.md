## Pre-requisites
### required tools
```
sudo apt-get update
sudo apt-get install -y jq unzip
```
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
```
aws configure
...
```
### Create a key
```
aws ec2 create-key-pair \
    --key-name my-key-pair \
    --key-type rsa \
    --key-format pem \
    --query "KeyMaterial" \
    --output text > my-key-pair.pem
```
```
chmod 400 my-key-pair.pem
```

### Creating a security group
```
SECGR=$(aws ec2 create-security-group --group-name myecssgecs --description "security group for eu-west-3"  | jq -r .GroupId)
aws ec2 describe-security-groups --group-id $SECGR 
aws ec2 authorize-security-group-ingress --group-id $SECGR --protocol tcp --port 22 --cidr 0.0.0.0/0 
aws ec2 authorize-security-group-ingress --group-id $SECGR --protocol tcp --port 80 --cidr 0.0.0.0/0 
aws ec2 authorize-security-group-ingress --group-id $SECGR --protocol tcp --port 5432 --source-group $SECGR
aws ec2 authorize-security-group-ingress --group-id $SECGR --protocol tcp --port 6379 --source-group $SECGR
aws ec2 describe-security-groups --group-id $SECGR
```
### Preparing IAM
```
aws iam create-instance-profile --instance-profile-name ecsInstanceRole-profile
```
See https://docs.aws.amazon.com/AmazonECS/latest/developerguide/instance_IAM_role.html
```
aws iam create-role \
    --role-name ecsInstanceRole \
    --assume-role-policy-document '{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}'

```
```
aws iam attach-role-policy \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role \
    --role-name ecsInstanceRole
```

```
aws iam add-role-to-instance-profile \
    --instance-profile-name ecsInstanceRole-profile \
    --role-name ecsInstanceRole
```
## Create an ECS cluster
```
aws ecs create-cluster \
    --cluster-name MyCluster \
    --region eu-west-3
```
```
aws ecs list-clusters --region eu-west-3
```
```
aws ecs describe-clusters --clusters MyCluster
```
## Creating compute
```
cat >ecs_compute.txt <<EOF
#!/bin/sh
yum update -y
yum install -y ecs-init
systemctl enable --now --no-block docker.service
echo "ECS_CLUSTER=MyCluster" >> /etc/ecs/ecs.config
systemctl enable --now --no-block ecs.service
EOF
```
```
aws ec2 run-instances --image-id ami-0fc067f03ad87bb64 \
   --count 3 \
   --instance-type t2.micro \
   --iam-instance-profile Name=ecsInstanceRole-profile \
   --key-name my-key-pair \
   --security-group-ids $SECGR \
   --tag-specifications 'ResourceType=instance, Tags=[{Key=Name,Value=ECSnode}]' \
   --user-data file://ecs_compute.txt
```
## Verifying the compute is available for ECS
Wait for a few minutes ...
```
aws ecs list-container-instances --cluster MyCluster
```
```
{
    "containerInstanceArns": [
        "arn:aws:ecs:eu-west-3:349901144557:container-instance/MyCluster/00cd9de271a44ea98780ba1cb436cb38",
        "arn:aws:ecs:eu-west-3:349901144557:container-instance/MyCluster/2df501fbfa314963bb0d3320e07e07b3",
        "arn:aws:ecs:eu-west-3:349901144557:container-instance/MyCluster/c3c09b7d2ded45bc9c6f73f31922bfa0"
    ]
}
```

## Define a task
```
cat >task.json  <<EOF
{
 "containerDefinitions": [
    {
     "name": "nginx",
     "image": "nginx",
     "portMappings": [
     {
        "containerPort": 80
     }
     ],
     "memory": 50,
     "cpu": 102
   }
 ],
 "family": "web"
}
EOF
```

## creating a task
```
aws ecs register-task-definition  --cli-input-json file://task.json
aws ecs list-task-definition-families
aws ecs list-task-definitions
aws ecs describe-task-definition --task-definition web:1
```

## Exploring versioning
```
aws ecs register-task-definition  --cli-input-json file://task.json
aws ecs list-task-definitions
aws ecs deregister-task-definition --task-definition web:2
aws ecs list-task-definitions
```

## Creating a service
```
aws ecs create-service --cluster MyCluster \
   --service-name web \
   --task-definition web \
   --desired-count 3
```
```
aws ecs list-services --cluster MyCluster
aws ecs describe-services --cluster MyCluster --services web
```
```
aws ecs list-tasks --cluster MyCluster
```
```
{
    "taskArns": [
        "arn:aws:ecs:eu-west-3:349901144557:task/MyCluster/b961ac1995124c3bbd3104fb0326efb9",
        "arn:aws:ecs:eu-west-3:349901144557:task/MyCluster/e0d1fa4c1d5547dfbf54fe77de44f83c",
        "arn:aws:ecs:eu-west-3:349901144557:task/MyCluster/f7f057715ebd4f628f2eef2d99ecb17d"
    ]
}
```
```
aws ecs describe-tasks --cluster MyCluster --tasks "b961ac1995124c3bbd3104fb0326efb9"

```
```
aws ecs update-service --cluster MyCluster --service web --task-definition web --desired-count 2
```

## Cleanup
```
aws ecs update-service --cluster MyCluster --service web --task-definition web --desired-count 0
```
```
aws ecs delete-service --cluster MyCluster --service web 
```
```
aws ec2 terminate-instances \
  --instance-ids $(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=ECSnode" \
    --query "Reservations[].Instances[].InstanceId" \
    --output text)
```
```
aws ecs delete-cluster --cluster MyCluster
```
```
aws ec2 delete-security-group --group-name myecssgecs
```
```
aws ec2 delete-key-pair --key-name my-key-pair
```
```
aws iam detach-role-policy \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role \
    --role-name ecsInstanceRole

aws iam remove-role-from-instance-profile \
    --instance-profile-name ecsInstanceRole-profile \
    --role-name ecsInstanceRole

aws iam delete-role --role-name ecsInstanceRole 

aws iam delete-instance-profile --instance-profile-name ecsInstanceRole-profile
```

