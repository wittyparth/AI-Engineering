# Drill: Deploy to AWS ECS

**Time**: 45 min  **Difficulty**: Medium

## Task

Deploy your Dockerized LLM service to AWS ECS Fargate.

## Steps

```bash
# 1. Create ECR repository
aws ecr create-repository --repository-name llm-service

# 2. Build and push
docker build -t llm-service .
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ACCOUNT.dkr.ecr.us-east-1.amazonaws.com
docker tag llm-service:latest $ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/llm-service:latest
docker push $ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/llm-service:latest

# 3. Create ECS cluster
aws ecs create-cluster --cluster-name ai-cluster

# 4. Register task definition (see aws-task-def.json)

# 5. Create service
aws ecs create-service \
    --cluster ai-cluster \
    --service-name llm-service \
    --task-definition llm-service:1 \
    --desired-count 1 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[subnet-...],securityGroups=[sg-...],assignPublicIp=ENABLED}"

# 6. Verify
curl http://<alb-dns-name>/health
```

## CloudFormation/CDK Alternative

```python
# Deploy with CDK (simpler)
from aws_cdk import aws_ecs as ecs, aws_ecr as ecr

class LLMStack(cdk.Stack):
    def __init__(self, scope, id):
        super().__init__(scope, id)
        cluster = ecs.Cluster(self, "AICluster")
        ecs_patterns.ApplicationLoadBalancedFargateService(
            self, "LLMService",
            cluster=cluster,
            task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
                image=ecs.ContainerImage.from_registry(".../llm-service:latest"),
                container_port=8000,
            ),
        )
```
