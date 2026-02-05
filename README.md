# AWS AI Agent Learning Project: Customer Support Agent

## Project Overview
Build a conversational AI agent that handles customer support inquiries, demonstrates key AWS services, and mirrors real-world enterprise AI agent patterns.

## Architecture Diagram (Conceptual)

```
User Request (API Gateway)
    ↓
Lambda (Request Handler)
    ↓
Step Functions (Orchestration)
    ├→ Lambda (Conversation Manager)
    │   ├→ DynamoDB (Session State)
    │   └→ Bedrock (Claude API)
    ├→ Lambda (Action Executor)
    │   ├→ SQS (Task Queue)
    │   └→ External APIs
    └→ Lambda (Response Formatter)
        ↓
    S3 (Conversation Logs)
        ↓
    EventBridge (Analytics Events)
```

## Core Components

### 1. API Layer
**Service: API Gateway (REST API)**
- Endpoint: POST /chat
- Handles incoming user messages
- Returns agent responses
- Validates requests

### 2. Request Handler
**Service: Lambda Function (Python/Node.js)**
- Receives request from API Gateway
- Retrieves conversation history from DynamoDB
- Initiates Step Functions workflow
- Returns response to user

### 3. Orchestration Layer
**Service: Step Functions (State Machine)**

```json
{
  "Comment": "Customer Support Agent Workflow",
  "StartAt": "LoadContext",
  "States": {
    "LoadContext": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:LoadContext",
      "Next": "CallBedrock"
    },
    "CallBedrock": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:BedrockInference",
      "Next": "CheckForActions"
    },
    "CheckForActions": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.requiresAction",
          "BooleanEquals": true,
          "Next": "ExecuteAction"
        }
      ],
      "Default": "SaveAndReturn"
    },
    "ExecuteAction": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:ActionExecutor",
      "Next": "SaveAndReturn"
    },
    "SaveAndReturn": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:SaveConversation",
      "End": true
    }
  }
}
```

### 4. Conversation Manager
**Service: Lambda Function**
- Manages conversation state
- Formats prompts for Bedrock
- Handles conversation history
- Implements tool calling logic

**DynamoDB Table: Conversations**
```
Primary Key: sessionId (String)
Sort Key: timestamp (Number)

Attributes:
- userId
- messages (List)
- context (Map)
- lastActivity
- metadata
```

### 5. AI Inference
**Service: Amazon Bedrock**
- Model: Claude 3.5 Sonnet
- System prompt for customer support agent persona
- Tool definitions for actions (create ticket, check status, etc.)

**Lambda Function: BedrockInference**
```python
import boto3
import json

bedrock = boto3.client('bedrock-runtime')

def lambda_handler(event, context):
    messages = event['messages']
    
    response = bedrock.invoke_model(
        modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
        body=json.dumps({
            'anthropic_version': 'bedrock-2023-05-31',
            'max_tokens': 1024,
            'messages': messages,
            'tools': [
                {
                    'name': 'create_support_ticket',
                    'description': 'Create a support ticket',
                    'input_schema': {
                        'type': 'object',
                        'properties': {
                            'title': {'type': 'string'},
                            'description': {'type': 'string'},
                            'priority': {'type': 'string'}
                        }
                    }
                }
            ]
        })
    )
    
    return json.loads(response['body'])
```

### 6. Action Execution
**Service: Lambda Function + SQS**

When the agent decides to take an action:
1. Parse tool use from Bedrock response
2. Send task to SQS queue for processing
3. Lambda consumer processes tasks asynchronously
4. Results stored in DynamoDB

**SQS Queue: AgentActions**
- Standard queue for action requests
- DLQ for failed actions
- Lambda triggered on message arrival

### 7. Conversation Storage
**Service: S3**
- Bucket: conversation-logs-{account-id}
- Structure: `{year}/{month}/{day}/{sessionId}.json`
- Lifecycle policy: Move to Glacier after 90 days
- Used for analytics and training data

### 8. Event Streaming
**Service: EventBridge**
- Custom event bus for agent events
- Events: conversation_started, action_executed, error_occurred
- Routes to:
  - CloudWatch Logs for monitoring
  - Lambda for real-time analytics
  - SNS for alerting

### 9. Monitoring & Observability
**Services: CloudWatch + X-Ray**

**CloudWatch:**
- Custom metrics: conversation_duration, bedrock_latency, error_rate
- Log groups for each Lambda
- Dashboards for system health

**X-Ray:**
- Trace requests end-to-end
- Identify bottlenecks
- Service map visualization

## Sample Conversation Flow

```
User: "I can't log into my account"

1. API Gateway receives request
2. Lambda loads session from DynamoDB
3. Step Functions workflow starts
4. Bedrock generates response asking for details
5. User provides email address
6. Bedrock decides to use create_support_ticket tool
7. SQS receives ticket creation task
8. Action executor Lambda creates ticket in mock system
9. Tool result sent back to Bedrock
10. Bedrock generates final response with ticket number
11. Conversation saved to DynamoDB and S3
12. EventBridge emits action_executed event
```

## Infrastructure as Code

Use AWS SAM or Terraform to define everything:

**SAM Template Structure:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  ChatApi:
    Type: AWS::Serverless::Api
    
  ConversationTable:
    Type: AWS::DynamoDB::Table
    
  BedrockInferenceFunction:
    Type: AWS::Serverless::Function
    
  AgentWorkflow:
    Type: AWS::StepFunctions::StateMachine
    
  # ... etc
```
