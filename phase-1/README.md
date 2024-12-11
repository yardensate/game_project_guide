# Phase 1: AWS Infrastructure - Detailed Implementation Plan

## 1. AWS EKS Cluster Setup
### Terraform Configuration
```hcl
# EKS Cluster
resource "aws_eks_cluster" "game_cluster" {
  name     = "game-session-cluster"
  role_arn = aws_iam_role.eks_role.arn

  vpc_config {
    subnet_ids = aws_subnet.private[*].id
  }
}

# Node Group
resource "aws_eks_node_group" "game_nodes" {
  cluster_name    = aws_eks_cluster.game_cluster.name
  node_group_name = "game-session-nodes"
  instance_types  = ["t3.medium"]
  
  scaling_config {
    desired_size = 3
    max_size     = 5
    min_size     = 1
  }
}

# Namespaces
resource "kubernetes_namespace" "game_namespaces" {
  for_each = toset(["gaming", "monitoring", "data"])
  metadata {
    name = "${each.key}-namespace"
  }
}
```

## 2. Basic API with FastAPI
### Core Endpoints (Unchanged):
```python
GET /health - Basic service health
GET /health/detailed - Detailed service status including DocumentDB connections
POST /api/v1/players - Create new player
GET /api/v1/players/{player_id} - Get player info
PUT /api/v1/players/{player_id} - Update player info
POST /api/v1/sessions - Create game session
GET /api/v1/sessions/{session_id} - Get session info
PUT /api/v1/sessions/{session_id} - Update session status
```

### EKS Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-api
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: game-api
        image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/game-api:latest
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
```

## 3. Amazon DocumentDB Integration
```hcl
resource "aws_docdb_cluster" "game_db" {
  cluster_identifier  = "game-session-db"
  engine             = "docdb"
  master_username    = var.db_username
  master_password    = var.db_password
  backup_retention_period = 5
  preferred_backup_window = "07:00-09:00"
}

resource "aws_docdb_cluster_instance" "game_db_instances" {
  count              = 1
  identifier         = "game-session-db-${count.index}"
  cluster_identifier = aws_docdb_cluster.game_db.id
  instance_class     = "db.t3.medium"
}
```

## 4. GitHub Actions Pipeline with AWS
```yaml
name: AWS CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run tests
        run: pytest
      - name: Code quality
        run: black . && flake8

  build:
    needs: test
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v1
      - name: Build and push
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

  deploy:
    needs: build
    steps:
      - name: Deploy to EKS
        run: |
          aws eks update-kubeconfig --name game-session-cluster
          kubectl apply -f deployment/kubernetes/
```

## 5. AWS CloudWatch Monitoring
```hcl
resource "aws_cloudwatch_dashboard" "game_dashboard" {
  dashboard_name = "game-session-metrics"
  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          metrics = [
            ["GameSession", "APILatency"],
            ["GameSession", "RequestCount"],
            ["GameSession", "ErrorRate"]
          ]
          period = 300
          stat = "Average"
        }
      }
    ]
  })
}
```

## 6. CloudWatch Logs Integration
```python
import watchtower
import logging

logger = logging.getLogger(__name__)
logger.addHandler(watchtower.CloudWatchLogHandler(
    log_group='game-session-logs',
    stream_name='api-logs'
))

@app.middleware("http")
async def log_requests(request, call_next):
    logger.info({
        "request_id": request.headers.get("X-Request-ID"),
        "path": request.url.path,
        "method": request.method,
        "timestamp": datetime.now().isoformat()
    })
    response = await call_next(request)
    return response
```

## Initial Directory Structure:
```
game-session-manager/
├── terraform/
│   ├── main.tf
│   ├── eks.tf
│   ├── docdb.tf
│   ├── monitoring.tf
│   └── variables.tf
├── src/
│   └── api/
├── deployment/
│   ├── kubernetes/
│   └── github-actions/
├── tests/
└── README.md
```

## Success Criteria for Phase 1:
1. EKS cluster operational with desired node count
2. APIs deployed and responding correctly
3. DocumentDB cluster operational and accessible
4. GitHub Actions successfully deploying to AWS
5. CloudWatch dashboards showing metrics
6. CloudWatch Logs collecting application logs
7. All IAM roles and security groups properly configured

Would you like me to detail any specific part further?