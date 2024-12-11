# Phase 2: Session Management - AWS Implementation Plan

## 1. Amazon ElastiCache (Redis) Implementation
```hcl
# Terraform configuration for ElastiCache
resource "aws_elasticache_cluster" "game_cache" {
  cluster_id           = "game-session-cache"
  engine              = "redis"
  node_type           = "cache.t3.micro"
  num_cache_nodes     = 1
  port                = 6379
  parameter_group_name = aws_elasticache_parameter_group.game_cache_params.name
  security_group_ids  = [aws_security_group.redis_sg.id]
}

# Redis data structures remain the same:
sessions:{session_id} = {
    'player_id': str,
    'status': str,
    'rating': int,
    'timestamp': str
}
```

## 2. Gaming Features Implementation

### Session Management with AWS Integration:
```python
from aws_xray_sdk.core import xray_recorder

@xray_recorder.capture('session_management')
class AWSSessionManager:
    def __init__(self):
        self.elasticache = boto3.client('elasticache')
        self.docdb = boto3.client('docdb')
        
    @xray_recorder.capture('create_session')
    async def create_session(self, session_data: dict):
        # Create session in ElastiCache
        # Store persistent data in DocumentDB
```

### Player Authentication using Cognito:
```python
from aws_cognito_client import CognitoClient

class AuthService:
    def __init__(self):
        self.cognito = CognitoClient(
            user_pool_id=settings.COGNITO_USER_POOL_ID,
            client_id=settings.COGNITO_CLIENT_ID
        )
    
    async def register_player(self, username: str, password: str):
        return await self.cognito.sign_up(username, password)

    async def authenticate_player(self, username: str, password: str):
        return await self.cognito.initiate_auth(username, password)
```

### Rating System with AWS Integration:
```python
class AWSRatingSystem:
    def __init__(self):
        self.elasticache = boto3.client('elasticache')
        self.cloudwatch = boto3.client('cloudwatch')
        
    async def update_rating(self, player_id: str, new_rating: int):
        # Update rating in ElastiCache
        # Send metric to CloudWatch
        self.cloudwatch.put_metric_data(
            Namespace='GameMetrics',
            MetricData=[{
                'MetricName': 'PlayerRating',
                'Value': new_rating,
                'Unit': 'None',
                'Dimensions': [{'Name': 'PlayerId', 'Value': player_id}]
            }]
        )
```

## 3. Enhanced AWS Monitoring

### CloudWatch Integration:
```python
# Metrics configuration
cloudwatch_metrics = {
    'Dimensions': [
        {'Name': 'Environment', 'Value': environment},
        {'Name': 'Service', 'Value': service_name}
    ],
    'Metrics': [
        {'Name': 'SessionCreationLatency'},
        {'Name': 'QueueLength'},
        {'Name': 'MatchmakingTime'}
    ]
}
```

### X-Ray Tracing:
```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.fastapi.middleware import XRayMiddleware

app.add_middleware(XRayMiddleware, recorder=xray_recorder)
```

## 4. Data Layer Implementation

### ElastiCache Configuration:
```hcl
resource "aws_elasticache_parameter_group" "game_cache_params" {
  family = "redis6.x"
  
  parameter {
    name  = "maxmemory-policy"
    value = "volatile-lru"
  }
}
```

### DocumentDB Collections:
```javascript
// Players collection in DocumentDB
{
    _id: ObjectId,
    cognito_id: String,
    profile: {
        created_at: DateTime,
        games_played: Number,
        current_rating: Number
    }
    // ... rest remains same
}
```

## 5. Updated Directory Structure:
```
game-session-manager/
├── terraform/
│   ├── elasticache.tf
│   ├── cognito.tf
│   └── monitoring.tf
├── src/
│   ├── api/
│   │   ├── routes/
│   │   └── models/
│   ├── services/
│   │   ├── aws/
│   │   │   ├── cognito.py
│   │   │   ├── elasticache.py
│   │   │   └── cloudwatch.py
│   │   └── game/
│   └── utils/
├── deployment/
│   └── aws/
└── tests/
```

## 6. Success Criteria for Phase 2:
1. ElastiCache cluster operational
2. Cognito user pool managing players
3. Session management working with AWS services
4. CloudWatch showing gaming metrics
5. X-Ray traces providing insights
6. All endpoints integrated with AWS services
7. Data persistence working in ElastiCache and DocumentDB
8. AWS security properly configured

## 7. CloudWatch Alarms:
```hcl
resource "aws_cloudwatch_metric_alarm" "session_latency" {
  alarm_name          = "session-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name        = "SessionCreationLatency"
  namespace          = "GameMetrics"
  period             = "60"
  statistic          = "Average"
  threshold          = "1000"
  alarm_description  = "Session creation taking too long"
  alarm_actions      = [aws_sns_topic.alerts.arn]
}
```

