# AWS Deployment

## Architecture

```
Route 53 → CloudFront → ALB → ECS Fargate (API)
                                ├── ElastiCache (Redis)
                                ├── RDS (PostgreSQL + pgvector)
                                └── Bedrock (LLM API) / SageMaker (self-hosted)
```

## ECS Fargate (Serverless Containers)

```yaml
# ecs-task-definition.json
{
  "family": "ai-rag-api",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::...",
  "taskRoleArn": "arn:aws:iam::...",
  "containerDefinitions": [{
    "name": "api",
    "image": "123456.dkr.ecr.us-east-1.amazonaws.com/rag-api:latest",
    "memory": 2048,
    "cpu": 1024,
    "portMappings": [{"containerPort": 8000}],
    "environment": [
      {"name": "OPENAI_API_KEY", "valueFrom": "arn:aws:secretsmanager:..."},
      {"name": "QDRANT_HOST", "value": "qdrant.ecs.internal"}
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {"awslogs-group": "/ecs/rag-api"}
    }
  }]
}
```

## CDK Infrastructure as Code

```python
from aws_cdk import (
    aws_ecs as ecs,
    aws_ecr as ecr,
    aws_rds as rds,
    aws_elasticache as elasticache,
    aws_ecs_patterns as patterns,
)

class AIStack(cdk.Stack):
    def __init__(self, scope, id, **kwargs):
        super().__init__(scope, id, **kwargs)

        # ECS Cluster
        cluster = ecs.Cluster(self, "AICluster",
            vpc=my_vpc,
            capacity=ecs.AddCapacityOptions(
                instance_type=ec2.InstanceType("t3.medium"),
            ),
        )

        # Fargate Service
        fargate_service = patterns.NetworkLoadBalancedFargateService(
            self, "AIService",
            cluster=cluster,
            task_image_options=patterns.NetworkLoadBalancedTaskImageOptions(
                image=ecs.ContainerImage.from_registry("rag-api:latest"),
                container_port=8000,
                secrets={
                    "OPENAI_API_KEY": ecs.Secret.from_secrets_manager(openai_secret),
                },
            ),
            desired_count=2,  # Multi-AZ
            auto_scaling_max_capacity=10,
        )
```

## Bedrock Integration

```python
import boto3

bedrock = boto3.client("bedrock-runtime")

response = bedrock.invoke_model(
    modelId="anthropic.claude-4-sonnet",
    contentType="application/json",
    accept="application/json",
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "messages": [{"role": "user", "content": "Hello"}],
        "max_tokens": 1000,
    }),
)
```

## 🔴 Senior: AWS Cost Optimization

| Service | Cost Trap | Solution |
|---------|-----------|----------|
| ECS Fargate | Always-on costs | Use ECS with spot instances + auto-scaling |
| Bedrock | No cost visibility | Budget alerts + token tracking |
| RDS | Large instances | Aurora Serverless v2 |
| ElastiCache | Over-provisioned | Start small, monitor hit rate |
| Secrets Manager | Per-secret cost | Use Parameter Store for non-sensitive |

## Drill: Deploy to AWS

1. Set up ECR repository
2. Build and push Docker image
3. Create ECS Fargate service
4. Set up Application Load Balancer
5. Configure Secrets Manager for API keys
6. Set up CloudWatch logging
7. Verify: `curl https://your-domain.com/health` returns 200
