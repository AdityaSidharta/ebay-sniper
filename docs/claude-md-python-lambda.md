# Backend Overview

This is a Python backend API designed to run on AWS Lambda, serving as the backend for a Next.js frontend application. The project follows serverless best practices and is optimized for Lambda's execution environment.

## Tech Stack

- **Runtime**: Python 3.11+ (Lambda supported version)
- **Framework**: FastAPI with Mangum adapter (or AWS Lambda Powertools)
- **Infrastructure**: AWS Lambda + API Gateway
- **Package Management**: pip with requirements.txt or Poetry
- **Deployment**: AWS SAM, Serverless Framework, or CDK

## Lambda Handler Pattern

### Using FastAPI with Mangum
```python
# src/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from mangum import Mangum
from src.api import users, auth
from src.core.config import settings

app = FastAPI(
    title=settings.PROJECT_NAME,
    version=settings.VERSION,
    docs_url="/api/docs" if settings.DEBUG else None,
)

# CORS configuration for Next.js frontend
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Include routers
app.include_router(auth.router, prefix="/api/auth", tags=["auth"])
app.include_router(users.router, prefix="/api/users", tags=["users"])

# Lambda handler
handler = Mangum(app)
```

### Using AWS Lambda Powertools
```python
# Alternative: src/main.py using Lambda Powertools
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.metrics import MetricUnit

logger = Logger()
tracer = Tracer()
metrics = Metrics()
app = APIGatewayRestResolver()

@app.get("/api/users")
@tracer.capture_method
def get_users():
    logger.info("Fetching users")
    # Implementation
    return {"users": []}

@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
@metrics.log_metrics
def handler(event, context):
    return app.resolve(event, context)
```

## Coding Standards

### Python Style
- Follow PEP 8 guidelines
- Use type hints for all function parameters and return values
- Maximum line length: 88 characters (Black formatter default)
- Use descriptive variable and function names
- Document all functions with docstrings

### Type Hints Example
```python
from typing import Optional, List, Dict, Any
from pydantic import BaseModel

class UserResponse(BaseModel):
    id: str
    email: str
    name: Optional[str] = None

async def get_user_by_id(user_id: str) -> Optional[UserResponse]:
    """
    Retrieve a user by their ID.
    
    Args:
        user_id: The unique identifier of the user
        
    Returns:
        UserResponse if found, None otherwise
    """
    # Implementation
```

### Error Handling
```python
from fastapi import HTTPException, status
from aws_lambda_powertools import Logger

logger = Logger()

class CustomException(Exception):
    """Base exception class for custom errors"""
    pass

def handle_database_error(func):
    """Decorator for handling database errors"""
    async def wrapper(*args, **kwargs):
        try:
            return await func(*args, **kwargs)
        except DatabaseError as e:
            logger.error(f"Database error: {str(e)}")
            raise HTTPException(
                status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
                detail="Database service temporarily unavailable"
            )
    return wrapper
```

## AWS Lambda Best Practices

### Cold Start Optimization
- Keep deployment package small (use Lambda Layers for dependencies)
- Initialize connections outside handler function
- Use connection pooling for databases
- Consider using provisioned concurrency for critical endpoints

```python
# Initialize outside handler for reuse across invocations
import boto3
from src.core.config import settings

# These are initialized once per container
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(settings.DYNAMODB_TABLE_NAME)

def handler(event, context):
    # Use pre-initialized resources
    response = table.get_item(Key={'id': event['id']})
    return response
```

### Memory and Timeout Configuration
- Start with 512MB memory, adjust based on CloudWatch metrics
- Set timeout based on expected response times + buffer
- Monitor Lambda Insights for optimization opportunities

### Environment Variables
```python
# src/core/config.py
from pydantic import BaseSettings
from typing import List

class Settings(BaseSettings):
    # API Configuration
    PROJECT_NAME: str = "Backend API"
    VERSION: str = "1.0.0"
    DEBUG: bool = False
    
    # CORS
    ALLOWED_ORIGINS: List[str] = ["https://your-amplify-app.com"]
    
    # AWS Resources
    DYNAMODB_TABLE_NAME: str
    S3_BUCKET_NAME: str
    
    # Authentication
    JWT_SECRET_KEY: str
    JWT_ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    
    # External Services
    OPENAI_API_KEY: str = None
    
    class Config:
        env_file = ".env"
        case_sensitive = True

settings = Settings()
```

## API Design Patterns

### RESTful Endpoints
```python
from fastapi import APIRouter, Depends, Query
from typing import List, Optional

router = APIRouter()

@router.get("/items", response_model=List[ItemResponse])
async def list_items(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100),
    search: Optional[str] = None,
    current_user: User = Depends(get_current_user)
):
    """List items with pagination and search"""
    return await item_service.list_items(skip, limit, search)

@router.post("/items", response_model=ItemResponse, status_code=201)
async def create_item(
    item: ItemCreate,
    current_user: User = Depends(get_current_user)
):
    """Create a new item"""
    return await item_service.create_item(item, current_user.id)
```

### Authentication Pattern
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import JWTError, jwt

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> User:
    """Validate JWT token and return current user"""
    try:
        payload = jwt.decode(
            credentials.credentials,
            settings.JWT_SECRET_KEY,
            algorithms=[settings.JWT_ALGORITHM]
        )
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid authentication credentials"
            )
    except JWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials"
        )
    
    user = await user_service.get_user(user_id)
    if user is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    return user
```

## Database Patterns

### DynamoDB Integration
```python
from boto3.dynamodb.conditions import Key
from aws_lambda_powertools import Logger
from typing import Optional, List

logger = Logger()

class DynamoDBRepository:
    def __init__(self, table_name: str):
        self.table = boto3.resource('dynamodb').Table(table_name)
    
    async def get_item(self, key: dict) -> Optional[dict]:
        """Get single item from DynamoDB"""
        try:
            response = self.table.get_item(Key=key)
            return response.get('Item')
        except Exception as e:
            logger.error(f"Error getting item: {e}")
            return None
    
    async def query_items(self, index_name: str, key_condition) -> List[dict]:
        """Query items using an index"""
        try:
            response = self.table.query(
                IndexName=index_name,
                KeyConditionExpression=key_condition
            )
            return response.get('Items', [])
        except Exception as e:
            logger.error(f"Error querying items: {e}")
            return []
```

## Testing Approach

### Unit Testing
```python
# tests/unit/test_user_service.py
import pytest
from unittest.mock import Mock, patch
from src.services.user_service import UserService

@pytest.fixture
def user_service():
    return UserService()

@pytest.mark.asyncio
async def test_get_user_success(user_service):
    # Arrange
    mock_user = {"id": "123", "email": "test@example.com"}
    with patch.object(user_service.repository, 'get_item', return_value=mock_user):
        # Act
        result = await user_service.get_user("123")
        # Assert
        assert result.id == "123"
        assert result.email == "test@example.com"
```

### Lambda Handler Testing
```python
# tests/integration/test_handler.py
import json
from src.main import handler

def test_handler_success():
    event = {
        "httpMethod": "GET",
        "path": "/api/users",
        "headers": {"Authorization": "Bearer valid-token"},
        "queryStringParameters": {"limit": "10"}
    }
    context = {}
    
    response = handler(event, context)
    
    assert response["statusCode"] == 200
    body = json.loads(response["body"])
    assert "users" in body
```

## Deployment Configuration

### SAM Template Example
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 30
    MemorySize: 512
    Runtime: python3.11
    Environment:
      Variables:
        STAGE: !Ref Stage
        DYNAMODB_TABLE_NAME: !Ref UserTable

Resources:
  ApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: main.handler
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /{proxy+}
            Method: ANY
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref UserTable
```

## Monitoring and Logging

### Structured Logging
```python
from aws_lambda_powertools import Logger

logger = Logger()

@logger.inject_lambda_context
def handler(event, context):
    logger.info("Request received", extra={
        "path": event.get("path"),
        "method": event.get("httpMethod"),
        "request_id": context.request_id
    })
    
    try:
        # Process request
        result = process_request(event)
        logger.info("Request processed successfully")
        return result
    except Exception as e:
        logger.error("Request failed", extra={"error": str(e)})
        raise
```

### CloudWatch Metrics
```python
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit

metrics = Metrics()

@metrics.log_metrics
def handler(event, context):
    # Add custom metrics
    metrics.add_metric(name="RequestCount", unit=MetricUnit.Count, value=1)
    metrics.add_metric(name="RequestLatency", unit=MetricUnit.Milliseconds, value=100)
    
    # Process request
    return response
```

## Security Best Practices

- Never log sensitive information (passwords, tokens, PII)
- Use AWS Secrets Manager or Parameter Store for secrets
- Implement rate limiting using API Gateway
- Validate all inputs using Pydantic models
- Use least privilege IAM roles
- Enable AWS WAF for API Gateway
- Implement request signing for service-to-service calls

## Performance Optimization

- Use connection pooling for databases
- Implement caching with ElastiCache or DynamoDB
- Minimize Lambda package size
- Use Lambda Layers for shared dependencies
- Consider using Lambda@Edge for global latency
- Batch operations when possible
- Use async/await for I/O operations

## Common Utilities

### Response Formatter
```python
from typing import Any, Optional
import json

def create_response(
    status_code: int,
    body: Any,
    headers: Optional[dict] = None
) -> dict:
    """Create API Gateway Lambda response"""
    default_headers = {
        "Content-Type": "application/json",
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Headers": "Content-Type,Authorization",
    }
    
    if headers:
        default_headers.update(headers)
    
    return {
        "statusCode": status_code,
        "headers": default_headers,
        "body": json.dumps(body) if not isinstance(body, str) else body
    }
```

## Key Commands

```bash
# Local development
pip install -r requirements.txt
python -m uvicorn src.main:app --reload

# Testing
pytest tests/
pytest tests/unit/ -v
pytest tests/integration/ -v

# Linting and formatting
black src/ tests/
flake8 src/ tests/
mypy src/

# Deployment (SAM)
sam build
sam deploy --guided

# Deployment (Serverless)
serverless deploy --stage prod
```

## Important Notes

- Lambda has a 15-minute execution limit
- API Gateway has a 29-second timeout
- Lambda payload limit is 6MB (synchronous)
- Cold starts impact performance - optimize accordingly
- Use X-Ray for distributed tracing
- Monitor costs - Lambda charges per invocation and duration
- Implement idempotency for critical operations
- Handle Lambda throttling gracefully
- Use Dead Letter Queues for failed invocations
- Consider using Step Functions for complex workflows