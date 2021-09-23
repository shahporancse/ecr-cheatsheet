## Step-1: What are we going to learn? 
- We are going to using ENV ECR with Secret Manager,S3, SSM Parameter.

## Step-2:  Create Application Load Balancer
- We are going to using ENV ECR with Secret Manager,S3, SSM Parameter.

# Passing environment variables

## General
To be able to get properties from either SSM or Secrets mgr, the taskexecution IAM role needs additional permissions. See https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html for all details.  

* Go to IAM in AWS mgm console
* click on Roles
* click ecsTaskExecutionRole
* click Add inline policy
* open JSON tab, and paste the following:  

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "ssm:GetParameters",
        "kms:Decrypt"
      ],
      "Resource": [
        "arn:aws:secretsmanager:eu-central-1:##youraccount-id##:secret:*",
        "arn:aws:ssm:eu-central-1:##youraccount-id##:parameter/*",
        "arn:aws:kms:eu-central-1:##youraccount-id##:*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::ecs-course-##youraccount-id##/environment-demo.env"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::ecs-course-##youraccount-id##"
      ]
    }
  ]
}
```

* provide a name for this policy : "ecs-course-parameter-source"


Let's also create a dedicated ECS service and task definition for this demo scenario. For that we use a plain Ubuntu image.
* within section "ECS" in AWS mgm console
* click Task definition
* click Create new task definition
* select EC2, click Next
 enter as name *td-plain-ubuntu**
* select task execution role properly to our recently created one
* click Add container
* container name: plain-ubuntu
* image: ubuntu:latest
* Memory limit: 128
* entrypoint: sleep,infinity

* go to ECS clusters overview
* click Create in tab Services to create a new service
* name: plain-ubuntu
* select previously created task definition
* Create , no further modification required

## SSM Parameter Store

Open AWS mgm console and navigate to AWS Systems Manager
* click on Parameter Store
* click Create Parameter
* provide name: plain-variable-from-ssm
* Type: String
* Value: plain variable from SSM Parameter store
* click Create parameter

* click Create parameter
* provide name: enrypted-variable-from-ssm
 Type *SecureString**
* use default KMS store My current account
* Value: encrypted variable from SSM Parameter store

check parameter:

bash
```
aws ssm get-parameter --name plain-variable-from-ssm
```

Now update the container definition within our task definition, to provide both parameters to the container.
Open section Environment variables and provide name as "plain-from-ssm" & drop-down-box value ValueFrom and in the value textbox, enter the ARN of the corresponding parameter (the ARN you grab with aws ssm get-parameter as shown above)

After updating the task definition, redeploy the plain-Ubuntu service , followed by ssh to the EC2 box and check the env inside the Ubuntu docker container.

##Testing:
```
- chmod 400 key.pem
- ssh -i key.pem ec2-user@ec2-34-255-137-124.eu-west-1.compute.amazonaws.com
- docker container ls | grep ubuntu
- docker exec -it cont ainer_id "env"
```
 


## Secrets manager

## Connecting to a private docker repository

https://docs.aws.amazon.com/AmazonECS/latest/developerguide/private-auth-container-instances.html  
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/private-auth.html

Use Secrets manager to specify your username & password for your DockerHub account.

* Open the AWS Secrets Manager console at https://console.aws.amazon.com/secretsmanager/.
* Choose Store a new secret.
* For Select secret type, choose Other type of secrets.
* Select Plaintext and enter your private registry credentials using the following format:

```
{
  "username" : "<<your-username>>",
  "password" : "<<passwd-or-access-key>>"
}
```
* Choose Next
* For Secret name, type an optional path and name, such as production/MyAwesomeAppSecret or development/TestSecret, and choose Next
* Review and Save
 click on the secret and copy the *ARN**

* create new task definition
* launchtype EC2
* name: td-private-repo-demo
* add container
  * Image : gkoenig/ecs-course-private-demo:1.0
  * mark the checkbox Private repository authentication
   Secrets Manager ARN: *provide the ARN of the secret which you created in the previous step**
* click Create

* start simply a manual task, based on the above task definition td-private-repo-demo
* open public URL to show the added text output

## File from S3
This functionality is limited to EC2 based tasks/services only !!

### create env file and upload to S3

```
ACCOUNTID=$(aws sts get-caller-identity | awk '{print $1}')
aws s3 mb s3://ecs-course-$ACCOUNTID
echo "variable1-from-s3=value1" > environment-demo.env
echo "variable2-from-s3=value2" >> environment-demo.env
aws s3 cp ./environment-demo.env s3://ecs-course-$ACCOUNTID/
```

### add env file to task definition

* open (EC2 based) task definition
* open Container definition
* in section Environment Files click on "+" sign
* provide full S3 ARN to the recently uploaded .env file
  arn:aws:s3:::ecs-course-$ACCOUNTID/environment-demo.env