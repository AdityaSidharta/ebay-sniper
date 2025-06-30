# Backend Overview

This is a Python backend API designed to run on AWS Lambda, serving as the backend for a Next.js frontend application. The project follows serverless best practices and is optimized for Lambda's execution environment.

## Tech Stack

- **Runtime**: Python 3.11+ (Lambda supported version)
- **Framework**: AWS Lambda PowerTools
- **Infrastructure**: AWS Lambda + API Gateway
- **Package Management**: uv with requirements.txt
- **Deployment**: AWS SAM

## Lambda Handler Pattern


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
from aws_lambda_powertools import Logger
from aws_lambda_powertools.event_handler.exceptions import BadRequestError, ServiceError

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
            raise ServiceError("Database service temporarily unavailable")
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
users_table = dynamodb.Table(os.getenv('USERS_TABLE'))
bids_table = dynamodb.Table(os.getenv('BIDS_TABLE'))

def handler(event, context):
    # Use pre-initialized resources
    response = users_table.get_item(Key={'userId': event['userId']})
    return response
```

### Memory and Timeout Configuration
- Start with 512MB memory, adjust based on CloudWatch metrics
- Set timeout based on expected response times + buffer
- Monitor Lambda Insights for optimization opportunities

### Configuration
For comprehensive environment variable configuration and reference, see [environment-variables.md](environment-variables.md).

```python
# src/core/config.py
import os

# Simple configuration with defaults
PROJECT_NAME = "eBay Sniper API"
VERSION = "1.0.0"
DEBUG = os.getenv("DEBUG", "false").lower() == "true"

# CORS - defaults to common origins
ALLOWED_ORIGINS = os.getenv("CORS_ORIGINS", "http://localhost:3000").split(",")

# Environment variables are configured via Lambda environment
# See environment-variables.md for complete reference
```

## API Design Patterns

For complete HTTP API endpoint specifications, request/response models, and examples, see [api-specification.md](api-specification.md).

### Implementation Pattern
```python
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.event_handler.exceptions import BadRequestError, NotFoundError
from pydantic import BaseModel, Field

logger = Logger()
tracer = Tracer()
app = APIGatewayRestResolver()

# Pydantic models handle ALL validation - no duplicate layers
class CreateRequest(BaseModel):
    field: str = Field(..., min_length=1, max_length=255)
    amount: int = Field(..., ge=1, le=10000000)

@app.post("/resource")
@tracer.capture_method
def create_resource():
    """Implementation pattern - validation handled by Pydantic automatically"""
    raw_body = app.current_event.json_body
    request = CreateRequest(**raw_body)  # Pydantic handles all validation
    
    # Direct operations - validation already complete
    # Implementation logic here
    
    return {"status": "success"}
```

### Authentication (API Gateway Handles JWT)

```python
from aws_lambda_powertools.event_handler.exceptions import UnauthorizedError, NotFoundError
from typing import Dict, Any
import boto3
import os

def get_current_user(app: APIGatewayRestResolver) -> Dict[str, Any]:
    """Extract user from API Gateway JWT context - no manual validation needed"""
    # API Gateway automatically validates JWT and injects user context
    authorizer_context = app.current_event.request_context.authorizer
    
    # Extract validated user information from API Gateway
    claims = authorizer_context.get('claims', {})
    user_id = claims.get('sub')
    
    if not user_id:
        raise UnauthorizedError("User not authenticated")
    
    # Return user context - no database lookup needed for basic auth
    return {
        'sub': user_id,
        'email': claims.get('email'),
        'cognito:username': claims.get('cognito:username')
    }

def get_current_user_full(app: APIGatewayRestResolver) -> Dict[str, Any]:
    """Get full user data when needed - direct DynamoDB operation"""
    basic_user = get_current_user(app)
    
    # Direct DynamoDB lookup when full user data is needed
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(os.getenv('USERS_TABLE'))
    
    response = table.get_item(Key={'userId': basic_user['sub']})
    user_data = response.get('Item')
    
    if not user_data:
        raise NotFoundError("User profile not found")
    
    return user_data
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
        USERS_TABLE: !Ref UsersTable
        BIDS_TABLE: !Ref BidsTable
        BID_HISTORY_TABLE: !Ref BidHistoryTable

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

For comprehensive security guidelines, implementations, and configurations, see [security-guide.md](security-guide.md).

Key backend security considerations:
- Validate all inputs using Pydantic models
- Use AWS Secrets Manager for credential storage
- Implement least privilege IAM roles
- Never log sensitive information (passwords, tokens, PII)

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

For complete development and deployment commands, see the commands section in [CLAUDE.md](../CLAUDE.md).

```bash
# Essential backend commands
pytest tests/                    # Run backend tests  
ruff format src/ tests/         # Format Python code
ruff check src/ tests/          # Lint Python code  
ty src/                         # Type checking
uvicorn src.main:app --reload   # Start development server
```

## Lambda Functions Architecture

The eBay Sniper backend follows a microservices approach with individual Lambda functions, each responsible for a single action. This architecture provides better scalability, maintainability, and cost optimization.

### Architecture Overview

```
API Gateway → Individual Lambda Functions → DynamoDB/External APIs
EventBridge → Scheduled Lambda Functions → eBay API/Notifications
Bid Executor Lambda → Direct Invocation → Notification Handler Lambda → Email Service
```

### Function List

1. **User Management Lambda** - User CRUD operations
2. **eBay OAuth Lambda** - OAuth flow and token management
3. **Wishlist Sync Lambda** - Retrieve and sync wishlist items
4. **Bid Management Lambda** - Bid CRUD operations
5. **Bid History Lambda** - Historical bid data queries
6. **Bid Executor Lambda** - Scheduled bid execution
7. **Notification Handler Lambda** - Email notifications
8. **Token Refresh Lambda** - OAuth token renewal
9. **Price Update Lambda** - Scheduled price monitoring and alerting

---

### 1. User Management Lambda

**Purpose**: Handle user profile CRUD operations and preference management.

**Trigger**: API Gateway HTTP events
- `GET /users/profile` - Get user profile
- `PUT /users/profile` - Update user profile
- `PUT /users/preferences` - Update notification preferences
- `DELETE /users/account` - Delete user account

**Request/Response Models**:
```python
class UserProfileResponse(BaseModel):
    userId: str
    email: str
    createdAt: int
    updatedAt: int
    preferences: Dict[str, Any]
    isActive: bool

class UpdateUserRequest(BaseModel):
    email: Optional[str] = Field(None, regex=r'^[^@]+@[^@]+\.[^@]+$')
    preferences: Optional[Dict[str, Any]] = None

class PreferencesUpdate(BaseModel):
    emailNotifications: Optional[bool] = None
    bidWinNotifications: Optional[bool] = None
    bidLossNotifications: Optional[bool] = None
    timezone: Optional[str] = None
```

**Environment Variables**: See [environment-variables.md](environment-variables.md) for complete reference

**IAM Permissions**:
- DynamoDB: GetItem, PutItem, UpdateItem, DeleteItem on Users table
- CloudWatch: Logs and metrics

**Error Handling**:
- 404: User not found
- 400: Invalid email format or preferences
- 409: Email already exists
- 500: Database operation failed

**Performance**: Memory: 512MB, Timeout: 30s

---

### 2. eBay OAuth Lambda

**Purpose**: Manage eBay account linking, OAuth authorization flow, and token exchange.

**Trigger**: API Gateway HTTP events
- `GET /ebay/auth/url` - Generate eBay OAuth URL
- `POST /ebay/auth/callback` - Handle OAuth callback
- `DELETE /ebay/auth/unlink` - Unlink eBay account
- `GET /ebay/auth/status` - Check linking status

**Request/Response Models**:
```python
class AuthUrlResponse(BaseModel):
    authUrl: str
    state: str

class OAuthCallbackRequest(BaseModel):
    code: str
    state: str

class EbayAccountStatus(BaseModel):
    isLinked: bool
    accountId: Optional[str] = None
    linkedAt: Optional[int] = None
    tokenExpiresAt: Optional[int] = None
```

**Environment Variables**: See [environment-variables.md](environment-variables.md) for complete reference

**IAM Permissions**:
- DynamoDB: GetItem, UpdateItem on Users table
- Secrets Manager: GetSecretValue
- CloudWatch: Logs and metrics

**Error Handling**:
- 400: Invalid OAuth code or state
- 401: OAuth authorization failed
- 409: eBay account already linked
- 500: Token exchange failed

**Performance**: Memory: 512MB, Timeout: 30s

---

### 3. Wishlist Sync Lambda

**Purpose**: Retrieve and sync eBay wishlist items, filter for active auctions.

**Trigger**: API Gateway HTTP events
- `GET /wishlist/items` - Get wishlist items with live price updates
- `GET /wishlist/items/{itemId}` - Get specific item details

**Request/Response Models**:
```python
class WishlistItem(BaseModel):
    ebayItemId: str
    title: str
    currentPrice: int  # in cents
    endTime: int  # Unix timestamp
    imageUrl: Optional[str] = None
    condition: str
    sellerInfo: Dict[str, Any]
    isAuctionActive: bool

class WishlistResponse(BaseModel):
    items: List[WishlistItem]
    lastSyncAt: int
    totalItems: int
```

**Environment Variables**: See [environment-variables.md](environment-variables.md) for complete reference

**IAM Permissions**:
- DynamoDB: GetItem on Users table
- Secrets Manager: GetSecretValue
- CloudWatch: Logs and metrics

**Error Handling**:
- 401: eBay account not linked
- 403: eBay API rate limit exceeded
- 404: User not found
- 500: eBay API error

**Performance**: Memory: 1024MB, Timeout: 60s

---

### 4. Bid Management Lambda

**Purpose**: Handle bid CRUD operations and schedule bid execution. Resets price alert tracking when user updates bid amount to allow new alerts for updated bid levels.

**Trigger**: API Gateway HTTP events
- `POST /bids` - Create new bid
- `GET /bids` - List user's active bids
- `PUT /bids/{bidId}` - Update bid amount
- `DELETE /bids/{bidId}` - Cancel bid
- `GET /bids/{bidId}` - Get specific bid

**Request/Response Models**:
```python
class CreateBidRequest(BaseModel):
    ebayItemId: str = Field(..., regex=r'^[0-9]+$')
    maxBidAmount: int = Field(..., ge=100, le=10000000)  # $1 to $100,000

class UpdateBidRequest(BaseModel):
    maxBidAmount: int = Field(..., ge=100, le=10000000)

class BidResponse(BaseModel):
    bidId: str
    userId: str
    ebayItemId: str
    maxBidAmount: int
    status: str  # PENDING, PLACED, WON, LOST, CANCELLED
    auctionEndTime: int
    createdAt: int
    updatedAt: int
    itemTitle: Optional[str] = None
    itemImageUrl: Optional[str] = None
    currentPrice: Optional[int] = None
    lastAlertSent: Optional[int] = None  # Unix timestamp of last price alert
    lastAlertBidAmount: Optional[int] = None  # Bid amount when last alert was sent
```

**Environment Variables**: See [environment-variables.md](environment-variables.md) for complete reference

**IAM Permissions**:
- DynamoDB: GetItem, PutItem, UpdateItem, DeleteItem, Query on Bids table
- DynamoDB: PutItem on BidHistory table
- EventBridge Scheduler: CreateSchedule, UpdateSchedule, DeleteSchedule
- CloudWatch: Logs and metrics

**Error Handling**:
- 400: Invalid bid amount or item ID
- 404: Bid or item not found
- 409: Bid already exists for item
- 500: Database or scheduler error

**Performance**: Memory: 512MB, Timeout: 30s

---

### 5. Bid History Lambda

**Purpose**: Query and retrieve historical bid data with pagination and filtering.

**Trigger**: API Gateway HTTP events
- `GET /bid-history` - Get user's bid history (paginated)
- `GET /bid-history/{bidId}` - Get specific bid history

**Request/Response Models**:
```python
class BidHistoryItem(BaseModel):
    historyId: str
    originalBidId: str
    action: str  # CREATE, UPDATE, CANCEL, PLACE, WIN, LOSE
    timestamp: int
    details: Dict[str, Any]
    ebayItemId: str
    bidAmount: int
    finalPrice: Optional[int] = None

class BidHistoryResponse(BaseModel):
    items: List[BidHistoryItem]
    pagination: Dict[str, Any]

```

**Environment Variables**: See [environment-variables.md](environment-variables.md) for complete reference

**IAM Permissions**:
- DynamoDB: Query on BidHistory table
- CloudWatch: Logs and metrics

**Error Handling**:
- 400: Invalid pagination parameters
- 404: No history found
- 500: Database query error

**Performance**: Memory: 512MB, Timeout: 30s

---

### 6. Bid Executor Lambda

**Purpose**: Execute scheduled bids 5 seconds before auction end, wait for auction completion, and determine bid outcome (won/lost).

**Trigger**: EventBridge Scheduler (scheduled execution)

**Request/Response Models**:
```python
class BidExecutionEvent(BaseModel):
    bidId: str
    userId: str
    ebayItemId: str
    maxBidAmount: int
    auctionEndTime: int

class BidExecutionResult(BaseModel):
    bidId: str
    bidPlaced: bool
    bidOutcome: Optional[str] = None  # "WON", "LOST", "FAILED"
    ebayBidId: Optional[str] = None
    finalPrice: Optional[int] = None
    winningPrice: Optional[int] = None
    error: Optional[str] = None
    executedAt: int
    auctionEndedAt: Optional[int] = None
    outcomeCheckedAt: Optional[int] = None
```

**Environment Variables**: See [environment-variables.md](environment-variables.md) for complete reference

**IAM Permissions**:
- DynamoDB: GetItem, UpdateItem on Bids table
- DynamoDB: GetItem on Users table
- DynamoDB: PutItem on BidHistory table
- Secrets Manager: GetSecretValue
- Lambda: InvokeFunction on Notification Handler Lambda
- CloudWatch: Logs and metrics

**Error Handling**:
- Bid not found or already executed
- eBay API errors (item ended, bid too low, etc.)
- Network timeouts and retries during bid placement
- Network timeouts and retries during outcome checking
- Token refresh failures
- Auction outcome checking failures

**Performance**: Memory: 1024MB, Timeout: 300s (5 minutes - to allow waiting for auction completion)

---

### 7. Notification Handler Lambda

**Purpose**: Send email notifications via Postmark when called by Bid Executor Lambda.

**Trigger**: Direct invocation from Bid Executor Lambda

**Request/Response Models**:
```python
class BidNotificationEvent(BaseModel):
    bidId: str
    userId: str
    status: str  # WON, LOST, CANCELLED
    ebayItemId: str
    bidAmount: int
    finalPrice: Optional[int] = None

class EmailNotification(BaseModel):
    to: str
    subject: str
    htmlBody: str
    textBody: str
    templateName: str
    templateData: Dict[str, Any]
```

**Environment Variables**: See [environment-variables.md](environment-variables.md) for complete reference

**IAM Permissions**:
- DynamoDB: GetItem on Users table
- Secrets Manager: GetSecretValue
- CloudWatch: Logs and metrics

**Error Handling**:
- User preferences disable notifications
- Email delivery failures
- Template rendering errors
- Invalid email addresses

**Performance**: Memory: 512MB, Timeout: 30s

---

### 8. Token Refresh Lambda

**Purpose**: Automated renewal of eBay OAuth tokens before expiration.

**Trigger**: EventBridge scheduled rule (daily check for tokens expiring within 7 days)

**Request/Response Models**:
```python
class TokenRefreshResult(BaseModel):
    usersProcessed: int
    tokensRefreshed: int
    failures: List[str]
    nextRunAt: int

class UserTokenStatus(BaseModel):
    userId: str
    refreshed: bool
    newExpiresAt: Optional[int] = None
    error: Optional[str] = None
```

**Environment Variables**: See [environment-variables.md](environment-variables.md) for complete reference

**IAM Permissions**:
- DynamoDB: Scan, UpdateItem on Users table
- Secrets Manager: GetSecretValue
- CloudWatch: Logs and metrics

**Error Handling**:
- Refresh token expired or invalid
- eBay OAuth service errors
- Network timeouts
- Database update failures

**Performance**: Memory: 512MB, Timeout: 120s (2 minutes)

---

### 9. Price Update Lambda

**Purpose**: Scheduled price monitoring and alerting for active bids. Uses smart alert logic to notify users when current price exceeds their bid amount, but only sends alerts on first occurrence or after user updates their bid amount.

**Trigger**: EventBridge scheduled rule (every 1 day)

**Request/Response Models**:
```python
class PriceUpdateResult(BaseModel):
    bidsProcessed: int
    priceUpdates: int
    alertsSent: int
    failures: int
    executionTime: float

class PriceAlertNotification(BaseModel):
    bidId: str
    userId: str
    notificationType: str  # "PRICE_ALERT"
    ebayItemId: str
    bidAmount: int
    currentPrice: int
    itemTitle: str
    priceExcess: int  # amount by which current price exceeds bid
```

**Environment Variables**: See [environment-variables.md](environment-variables.md) for complete reference

**IAM Permissions**:
- DynamoDB: Scan, UpdateItem on Bids table (to update currentPrice)
- DynamoDB: GetItem on Users table (to get eBay tokens)
- Secrets Manager: GetSecretValue
- Lambda: InvokeFunction on Notification Handler Lambda
- CloudWatch: Logs and metrics

**Error Handling**:
- eBay API rate limiting and failures
- Token expiration during price fetch
- Database update failures
- Notification delivery failures
- Network timeouts with retry logic

**Performance**: Memory: 1024MB, Timeout: 300s (5 minutes - to handle multiple API calls)

**Smart Alert Logic**:
- Sends price alert when current price first exceeds user's bid amount
- Prevents notification spam by tracking last alert sent (`lastAlertSent`, `lastAlertBidAmount`)
- Does NOT send repeated alerts for continued price increases on same bid amount
- Resets alert tracking when user updates their bid amount (via Bid Management Lambda)
- Sends new alert only if price exceeds the updated bid amount
- Updates alert tracking fields after sending notifications

**Key Features**:
- Scans all PENDING bids across all users
- Fetches current prices from eBay API using user tokens
- Updates cached currentPrice in Bids table when changed
- Implements smart alert deduplication to prevent email spam
- Tracks alert history with `lastAlertSent` and `lastAlertBidAmount` fields
- Implements retry logic for eBay API failures
- Tracks metrics for monitoring (bids processed, alerts sent, failures)

---

## Function Deployment Configuration

### SAM Template Structure
Each Lambda function will be defined as a separate resource in the SAM template with specific:
- Runtime configuration (Python 3.11)
- Memory and timeout settings
- Environment variables
- IAM policies
- Event triggers
- Dead letter queues for error handling

### Shared Dependencies
Common functionality will be organized in Lambda Layers:
- AWS Lambda PowerTools
- Pydantic models
- eBay API client
- Common utilities and middleware

### Monitoring and Observability
All functions include:
- Structured logging with correlation IDs
- Custom CloudWatch metrics
- X-Ray tracing
- Error rate and latency alarms
- Dead letter queue monitoring

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