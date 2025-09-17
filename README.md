# AWS ECS Beginner Crash Course

A comprehensive guide to Amazon Elastic Container Service (ECS) for developers starting their AWS journey.

## 📊 Visual Workflow

### Quick Overview

```
🚀 START
   ↓
1️⃣ Check Prerequisites (AWS CLI, Docker, Permissions)
   ↓
2️⃣ Create Application (Code + Dockerfile)
   ↓
3️⃣ Build & Push to ECR
   ↓
4️⃣ Create ECS Cluster
   ↓
5️⃣ Create Task Definition
   ↓
6️⃣ Setup Networking (Security Groups)
   ↓
7️⃣ Create ECS Service
   ↓
❓ Need Load Balancer?
   ├─ Yes → 8a️⃣ Create ALB → 8b️⃣ Update Service
   └─ No ────────────────────────────────────┐
                                             ↓
9️⃣ Verify Deployment
   ↓
🔟 Monitor & Scale
   ↓
✅ SUCCESS!
```
### Phase Breakdown

| Phase | Steps | Description |
|-------|-------|-------------|
| 🔧 **Setup** | 1-3 | Prepare environment and build application |
| ⚙️ **Configuration** | 4-6 | Setup AWS ECS infrastructure |
| 🚀 **Deployment** | 7-8 | Deploy and configure services |
| 📊 **Operations** | 9-10 | Monitor and maintain application |

## Table of Contents

1. [What is AWS ECS?](#what-is-aws-ecs)
2. [Key Concepts](#key-concepts)
3. [Prerequisites](#prerequisites)
4. [Getting Started](#getting-started)
5. [Step-by-Step Tutorial](#step-by-step-tutorial)
6. [Common Use Cases](#common-use-cases)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)
9. [Cost Optimization](#cost-optimization)
10. [Next Steps](#next-steps)

## What is AWS ECS?

Amazon Elastic Container Service (ECS) is a fully managed container orchestration service that makes it easy to deploy, manage, and scale containerized applications using Docker containers.

### Why Use ECS?

- **Fully Managed**: No need to manage container orchestration infrastructure
- **Scalable**: Automatically scales your applications up or down
- **Secure**: Integrated with AWS security services
- **Cost-Effective**: Pay only for the resources you use
- **Integration**: Works seamlessly with other AWS services

## Key Concepts

### 1. **Cluster**
A logical grouping of compute resources (EC2 instances or Fargate) where your containers run.

### 2. **Task Definition**
A blueprint that describes how a container should run, including:
- Docker image to use
- CPU and memory requirements
- Environment variables
- Port mappings
- Volume mounts

### 3. **Task**
A running instance of a task definition. One task can contain multiple containers.

### 4. **Service**
Ensures a specified number of tasks are running and replaces unhealthy tasks.

### 5. **Container Instance**
An EC2 instance running the ECS agent that's part of an ECS cluster.

### 6. **Launch Types**
- **EC2**: You manage the underlying EC2 instances
- **Fargate**: AWS manages the infrastructure (serverless)

## Prerequisites

### Required Knowledge
- Basic understanding of Docker containers
- Familiarity with AWS console
- Basic networking concepts
- Command line usage

### Required Tools
- AWS CLI installed and configured
- Docker installed locally
- Text editor or IDE
- AWS account with appropriate permissions

### AWS Permissions Needed
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecs:*",
                "ecr:*",
                "iam:PassRole",
                "logs:*",
                "ec2:*"
            ],
            "Resource": "*"
        }
    ]
}
```

## Getting Started

### Step 1: Install AWS CLI

```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Windows
# Download and run the MSI installer from AWS website
```

### Step 2: Configure AWS CLI

```bash
aws configure
# Enter your:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region (e.g., us-east-1)
# - Default output format (json)
```

### Step 3: Verify Setup

```bash
aws sts get-caller-identity
```

## Step-by-Step Tutorial

### Tutorial 1: Deploy a Simple Web App with Fargate

#### Step 1: Create a Simple Web Application

Create a simple Node.js application:

**app.js**
```javascript
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
    res.json({
        message: 'Hello from ECS!',
        timestamp: new Date().toISOString(),
        hostname: require('os').hostname()
    });
});

app.get('/health', (req, res) => {
    res.status(200).json({ status: 'healthy' });
});

app.listen(port, () => {
    console.log(`Server running on port ${port}`);
});
```

**package.json**
```json
{
    "name": "ecs-demo-app",
    "version": "1.0.0",
    "description": "Simple ECS demo application",
    "main": "app.js",
    "scripts": {
        "start": "node app.js"
    },
    "dependencies": {
        "express": "^4.18.0"
    }
}
```

**Dockerfile**
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY . .

EXPOSE 3000

USER node

CMD ["npm", "start"]
```

#### Step 2: Build and Push Docker Image to ECR

```bash
# Create ECR repository
aws ecr create-repository --repository-name ecs-demo-app --region us-east-1

# Get login token
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Build image
docker build -t ecs-demo-app .

# Tag image
docker tag ecs-demo-app:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/ecs-demo-app:latest

# Push image
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/ecs-demo-app:latest
```

#### Step 3: Create ECS Cluster

```bash
# Create cluster
aws ecs create-cluster --cluster-name demo-cluster
```

Or via AWS Console:
1. Go to ECS Console
2. Click "Create Cluster"
3. Choose "Networking only" (Fargate)
4. Enter cluster name: "demo-cluster"
5. Click "Create"

#### Step 4: Create Task Definition

Create **task-definition.json**:
```json
{
    "family": "demo-app-task",
    "networkMode": "awsvpc",
    "requiresCompatibilities": ["FARGATE"],
    "cpu": "256",
    "memory": "512",
    "executionRoleArn": "arn:aws:iam::<account-id>:role/ecsTaskExecutionRole",
    "containerDefinitions": [
        {
            "name": "demo-app",
            "image": "<account-id>.dkr.ecr.us-east-1.amazonaws.com/ecs-demo-app:latest",
            "portMappings": [
                {
                    "containerPort": 3000,
                    "protocol": "tcp"
                }
            ],
            "essential": true,
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/demo-app",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "ecs"
                }
            }
        }
    ]
}
```

Register the task definition:
```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

#### Step 5: Create CloudWatch Log Group

```bash
aws logs create-log-group --log-group-name /ecs/demo-app
```

#### Step 6: Create Security Group

```bash
# Get default VPC ID
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)

# Create security group
SG_ID=$(aws ec2 create-security-group \
    --group-name ecs-demo-sg \
    --description "Security group for ECS demo" \
    --vpc-id $VPC_ID \
    --query "GroupId" --output text)

# Allow inbound traffic on port 3000
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 3000 \
    --cidr 0.0.0.0/0
```

#### Step 7: Create ECS Service

```bash
# Get subnet IDs
SUBNET_IDS=$(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query "Subnets[*].SubnetId" \
    --output text | tr '\t' ',')

# Create service
aws ecs create-service \
    --cluster demo-cluster \
    --service-name demo-service \
    --task-definition demo-app-task \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_IDS],securityGroups=[$SG_ID],assignPublicIp=ENABLED}"
```

#### Step 8: Verify Deployment

```bash
# Check service status
aws ecs describe-services --cluster demo-cluster --services demo-service

# List running tasks
aws ecs list-tasks --cluster demo-cluster --service-name demo-service

# Get task details
TASK_ARN=$(aws ecs list-tasks --cluster demo-cluster --service-name demo-service --query "taskArns[0]" --output text)
aws ecs describe-tasks --cluster demo-cluster --tasks $TASK_ARN
```

### Tutorial 2: Adding Load Balancer

#### Step 1: Create Application Load Balancer

```bash
# Create ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
    --name demo-alb \
    --subnets $(echo $SUBNET_IDS | tr ',' ' ') \
    --security-groups $SG_ID \
    --query "LoadBalancers[0].LoadBalancerArn" --output text)

# Create target group
TG_ARN=$(aws elbv2 create-target-group \
    --name demo-targets \
    --protocol HTTP \
    --port 3000 \
    --vpc-id $VPC_ID \
    --target-type ip \
    --health-check-path /health \
    --query "TargetGroups[0].TargetGroupArn" --output text)

# Create listener
aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

#### Step 2: Update Service with Load Balancer

```bash
aws ecs update-service \
    --cluster demo-cluster \
    --service demo-service \
    --load-balancers targetGroupArn=$TG_ARN,containerName=demo-app,containerPort=3000
```

## Common Use Cases

### 1. **Microservices Architecture**
- Deploy multiple small services
- Independent scaling and deployment
- Service discovery with AWS Cloud Map

### 2. **Batch Processing**
- Run scheduled tasks
- Process data pipelines
- ETL operations

### 3. **Web Applications**
- Scalable web servers
- API backends
- Static content serving

### 4. **CI/CD Pipelines**
- Automated deployments
- Blue/green deployments
- Rolling updates

## Best Practices

### 1. **Security**
```bash
# Use least privilege IAM roles
# Store secrets in AWS Secrets Manager
# Use VPC endpoints for private communication
# Enable container insights for monitoring
```

### 2. **Performance**
- Right-size your containers (CPU/Memory)
- Use health checks
- Implement graceful shutdowns
- Monitor resource utilization

### 3. **Cost Optimization**
- Use Fargate Spot for non-critical workloads
- Implement auto-scaling
- Use reserved capacity for predictable workloads
- Monitor and optimize resource usage

### 4. **Monitoring and Logging**
```json
{
    "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
            "awslogs-group": "/ecs/my-app",
            "awslogs-region": "us-east-1",
            "awslogs-stream-prefix": "ecs"
        }
    }
}
```

## Troubleshooting

### Common Issues

#### 1. **Task Fails to Start**
```bash
# Check task definition
aws ecs describe-task-definition --task-definition my-task

# Check task events
aws ecs describe-tasks --cluster my-cluster --tasks <task-arn>

# Check CloudWatch logs
aws logs get-log-events --log-group-name /ecs/my-app --log-stream-name <stream-name>
```

#### 2. **Service Not Reaching Desired Count**
- Check security groups
- Verify subnet configuration
- Check resource availability
- Review IAM permissions

#### 3. **Load Balancer Health Check Failures**
- Verify health check path
- Check security group rules
- Ensure application responds correctly
- Review target group settings

### Debugging Commands

```bash
# View service events
aws ecs describe-services --cluster my-cluster --services my-service

# Check task logs
aws logs tail /ecs/my-app --follow

# List cluster resources
aws ecs describe-clusters --clusters my-cluster --include STATISTICS
```

## Cost Optimization

### 1. **Choose Right Launch Type**
- **Fargate**: For variable workloads, no infrastructure management
- **EC2**: For consistent workloads, more control over costs

### 2. **Use Spot Instances**
```json
{
    "capacityProviders": ["FARGATE", "FARGATE_SPOT"],
    "defaultCapacityProviderStrategy": [
        {
            "capacityProvider": "FARGATE_SPOT",
            "weight": 4
        },
        {
            "capacityProvider": "FARGATE",
            "weight": 1
        }
    ]
}
```

### 3. **Implement Auto Scaling**
```bash
# Create auto scaling target
aws application-autoscaling register-scalable-target \
    --service-namespace ecs \
    --resource-id service/my-cluster/my-service \
    --scalable-dimension ecs:service:DesiredCount \
    --min-capacity 1 \
    --max-capacity 10

# Create scaling policy
aws application-autoscaling put-scaling-policy \
    --service-namespace ecs \
    --resource-id service/my-cluster/my-service \
    --scalable-dimension ecs:service:DesiredCount \
    --policy-name cpu-scaling \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration file://scaling-policy.json
```

## Next Steps

### Advanced Topics to Explore

1. **Service Mesh with AWS App Mesh**
2. **CI/CD with AWS CodePipeline**
3. **Secrets Management with AWS Secrets Manager**
4. **Container Insights and X-Ray Tracing**
5. **Multi-region Deployments**
6. **Blue/Green Deployments**
7. **ECS with Terraform/CloudFormation**

### Recommended Learning Path

1. ✅ Complete this tutorial
2. 📚 Learn about ECS Service Discovery
3. 🔧 Implement CI/CD pipeline
4. 📊 Set up comprehensive monitoring
5. 🏗️ Design microservices architecture
6. 🔒 Implement advanced security practices
7. 💰 Optimize costs and performance

### Useful Resources

- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [AWS ECS Best Practices Guide](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/)
- [AWS ECS Workshop](https://ecsworkshop.com/)
- [AWS CLI ECS Reference](https://docs.aws.amazon.com/cli/latest/reference/ecs/)

---

## Quick Reference

### Essential Commands

```bash
# Cluster operations
aws ecs create-cluster --cluster-name my-cluster
aws ecs list-clusters
aws ecs delete-cluster --cluster my-cluster

# Task definition operations
aws ecs register-task-definition --cli-input-json file://task-def.json
aws ecs list-task-definitions
aws ecs describe-task-definition --task-definition my-task

# Service operations
aws ecs create-service --cluster my-cluster --service-name my-service --task-definition my-task
aws ecs update-service --cluster my-cluster --service my-service --desired-count 3
aws ecs delete-service --cluster my-cluster --service my-service

# Task operations
aws ecs run-task --cluster my-cluster --task-definition my-task
aws ecs list-tasks --cluster my-cluster
aws ecs stop-task --cluster my-cluster --task task-arn
```

### Key Ports and Protocols

- **HTTP**: 80, 8080
- **HTTPS**: 443, 8443
- **Custom Apps**: 3000, 5000, 8000
- **Health Checks**: Usually same as app port

---

**Happy containerizing! 🐳**

Remember: Start small, iterate often, and always monitor your applications in production.