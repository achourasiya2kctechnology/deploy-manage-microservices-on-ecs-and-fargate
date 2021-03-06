[Back to main guide](../README.md)

___

# Build and deploy Contacts-Service

In this section we will build and deploy the **contacts-service** as an ECS service. We will deploy the service on Container instances.

The ECS Task-Definition and Service-Definition are located in the contacts-service folder.

___

## Task 1 - Build contacts-service docker image

1. Navigate to the contacts-service folder

```bash
cd ../contacts-service
```

2. Create ECR Repository for contacts-service

```bash
# Note the repositoryUri - output of below command
aws ecr create-repository --repository-name ecs-workshop/contacts-service
```

3. Build the Docker Image and push it to ECR

```bash
docker build -t <repositoryUri>:latest .
```

4. Login to Elastic Container Registry (ECR)

```bash
aws ecr get-login --no-include-email | bash
```

5. Push the Docker image to ECR

```bash
docker push <repositoryUri>:latest
```

___

## Task 2 - Setup the ALB

You can create a listener with rules to forward requests based on the URL path. This is known as path-based routing. If you are running microservices, you can route traffic to multiple back-end services using path-based routing.

The Application Load Balancer(ALB), Listner and the necessary Target groups have been created by the ecs-workshop CloudFormation Stack deployed in the [Lab Setup](lab-guides/lab-setup.md) section. We will now add rules to the ALB Listener to enable path-based routing to the contacts-service.

```bash 
# Refer to ecs-workshop CloudFormation stack output to get values of:
# <ALBListenerArn>
# <ContactsServiceALBTargetGroupArn>
aws elbv2 create-rule --listener-arn "<ALBListenerArn>" \
--priority 4 \
--conditions Field=path-pattern,Values='/contacts' \
--actions Type=forward,TargetGroupArn="<ContactsServiceALBTargetGroupArn>"

aws elbv2 create-rule --listener-arn "<ALBListenerArn>" \
--priority 5 \
--conditions Field=path-pattern,Values='/impcsv' \
--actions Type=forward,TargetGroupArn="<ContactsServiceALBTargetGroupArn>"
```

Verify the ALB Listener rules have been created.

```bash
aws elbv2 describe-rules --listener-arn "<ALBListenerArn>"
```

___

## Task 3 - Deploy the contacts-service as ECS Service

1. Create a CloudWatch LogGroup to collect contacts-service logs.

```bash
aws logs create-log-group --log-group-name ecs-workshop/contacts-service
```

2. Let us now register the ECS Task Definition for the contacts-service. A **task-definition.json** is located in the contacts-service folder, replace the below parameter placeholders in the file with the values appropriate to your environment.

|Parameters                          | Value                                         |
|------------------------------------|-----------------------------------------------|
|&lt;repositoryUri&gt;               | ECR repository Uri of contacts-service        |
|&lt;AWS-REGION&gt;                  | AWS Region e.g. us-east-1                     |
|&lt;ImageS3BucketName&gt;           | ecs-workshop CloudFormation stack Output      |
|&lt;UserProfileDdbTable&gt;         | ecs-workshop CloudFormation stack Output      |
|&lt;ContactsDdbTable&gt;            | ecs-workshop CloudFormation stack Output      |
|&lt;CognitoUserPoolId&gt;           | ecs-workshop CloudFormation stack Output      |
|&lt;CognitoUserPoolClientId&gt;     | ecs-workshop CloudFormation stack Output      |
|&lt;CognitoIdentityPoolId&gt;       | ecs-workshop CloudFormation stack Output      |

```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

3. We will now deploy the ECS service. Replace the following parameter place holders in the **service-definition.json**.

| Parameter                              | Value                                    |
|----------------------------------------|------------------------------------------|
|&lt;ContactsServiceALBTargetGroupArn&gt;| ecs-workshop CloudFormation stack Output |

```bash
# Deploy the contacts-service
aws ecs create-service --cli-input-json file://service-definition.json

# Verify the contacts-service has been deployed successfully
# The command will wait till the service has been deployed.  No output is returned.
aws ecs wait services-stable \
--cluster ecs-workshop-cluster \
--services contacts-service

# Verify the desired and running tasks for the service
aws ecs describe-services \
--cluster ecs-workshop-cluster \
--services contacts-service \
--query 'services[0].{desiredCount:desiredCount,runningCount:runningCount,pendingCount:pendingCount}'
```

___
[Back to main guide](../README.md)