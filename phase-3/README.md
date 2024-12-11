# Phase 3: Real-time Features - AWS Implementation Plan

## 1. AWS API Gateway WebSocket Implementation
```python
# AWS Lambda function for WebSocket handling
def lambda_handler(event, context):
    connection_id = event['requestContext']['connectionId']
    route_key = event['requestContext']['routeKey']
    
    if route_key == '$connect':
        return handle_connect(connection_id)
    elif route_key == '$disconnect':
        return handle_disconnect(connection_id)
    
    return handle_message(event['body'], connection_id)

# Message Types remain same:
{
    "type": "match_found",
    "data": {
        "match_id": str,
        "opponent": str,
        "timestamp": str
    }
}
```

### WebSocket Infrastructure:
```hcl
# API Gateway WebSocket API
resource "aws_apigatewayv2_api" "websocket" {
  name                       = "game-session-websocket"
  protocol_type             = "WEBSOCKET"
  route_selection_expression = "$request.body.action"
}

# Lambda Integration
resource "aws_apigatewayv2_integration" "websocket_lambda" {
  api_id           = aws_apigatewayv2_api.websocket.id
  integration_type = "AWS_PROXY"
  integration_uri  = aws_lambda_function.websocket_handler.invoke_arn
}
```

## 2. Advanced Queue System with AWS Services

### Enhanced Matchmaking Service:
```python
class AWSMatchmaker:
    def __init__(self):
        self.sqs = boto3.client('sqs')
        self.elasticache = boto3.client('elasticache')
        
    async def process_queue(self):
        # Use SQS for queue management
        messages = self.sqs.receive_message(
            QueueUrl=QUEUE_URL,
            MaxNumberOfMessages=10
        )
        
        # Process matches using ElastiCache for state
```

### SQS Queue Setup:
```hcl
resource "aws_sqs_queue" "matchmaking" {
  name                      = "game-matchmaking-queue"
  visibility_timeout_seconds = 30
  message_retention_seconds = 86400
  
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.matchmaking_dlq.arn
    maxReceiveCount     = 3
  })
}
```

## 3. Real-time Data Management with ElastiCache and SNS

### AWS SNS Topics:
```hcl
resource "aws_sns_topic" "game_events" {
  name = "game-events"
}

resource "aws_sns_topic_subscription" "lambda" {
  topic_arn = aws_sns_topic.game_events.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.event_handler.arn
}
```

### Event Handler Lambda:
```python
class AWSEventHandler:
    def __init__(self):
        self.sns = boto3.client('sns')
        self.api_gateway = boto3.client('apigatewaymanagementapi')
        
    async def handle_match_ready(self, player_id: str, match_data: dict):
        # Publish to SNS topic
        self.sns.publish(
            TopicArn=GAME_EVENTS_TOPIC,
            Message=json.dumps({
                'type': 'match_ready',
                'player_id': player_id,
                'data': match_data
            })
        )
```

## 4. Enhanced AWS Monitoring

### CloudWatch Metrics:
```python
class GameMetrics:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        
    def track_connection(self, connection_id: str):
        self.cloudwatch.put_metric_data(
            Namespace='GameMetrics',
            MetricData=[{
                'MetricName': 'ActiveConnections',
                'Value': 1,
                'Unit': 'Count'
            }]
        )
```

### CloudWatch Dashboard:
```hcl
resource "aws_cloudwatch_dashboard" "realtime" {
  dashboard_name = "game-realtime-metrics"
  
  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          metrics = [
            ["GameMetrics", "WebSocketLatency"],
            ["GameMetrics", "MatchmakingTime"],
            ["GameMetrics", "QueueLength"]
          ]
          period = 60
        }
      }
    ]
  })
}
```

## 5. Updated Directory Structure:
```
game-session-manager/
├── terraform/
│   ├── websocket.tf
│   ├── sqs.tf
│   ├── sns.tf
│   └── lambda.tf
├── src/
│   ├── lambda/
│   │   ├── websocket/
│   │   └── events/
│   ├── services/
│   │   ├── matchmaking/
│   │   └── realtime/
│   └── monitoring/
└── tests/
```

## 6. Performance Considerations
- Lambda concurrency limits
- API Gateway WebSocket connection limits
- SQS batch processing
- SNS message filtering
- ElastiCache clustering

## 7. Success Criteria for Phase 3:
1. API Gateway WebSocket endpoints operational
2. Lambda functions handling real-time events
3. SQS queues processing matchmaking efficiently
4. SNS topics delivering notifications
5. CloudWatch metrics showing:
   - WebSocket connection stats
   - Queue performance
   - Event processing latency
6. System scalability under load
7. Error handling and dead letter queues working

## 8. AWS-Specific Monitoring:
```hcl
resource "aws_cloudwatch_metric_alarm" "websocket_errors" {
  alarm_name          = "websocket-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "WebSocketErrors"
  namespace           = "GameMetrics"
  period             = "60"
  statistic          = "Sum"
  threshold          = "5"
  alarm_actions      = [aws_sns_topic.alerts.arn]
}
```

Would you like me to detail any specific AWS component further or show more service integrations?