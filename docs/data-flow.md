# Data Flow Architecture

This document outlines the comprehensive data flow patterns, state management, and information processing workflows throughout the eBay Sniper application.

## Data Flow Overview

The eBay Sniper application follows a modern serverless architecture with clear data flow patterns between frontend, backend, external services, and data storage systems.

### High-Level Data Flow Diagram

```mermaid
graph TB
    %% User Interface Layer
    User[👤 User]
    Frontend[🌐 Next.js Frontend]
    
    %% API Gateway Layer
    APIGateway[🚪 AWS API Gateway]
    
    %% Application Layer
    Lambda[⚡ AWS Lambda Functions]
    
    %% Data Layer
    DynamoDB[(📊 DynamoDB Tables)]
    
    %% External Services
    eBayAPI[🛍️ eBay API]
    Postmark[📧 Postmark Email]
    
    %% Authentication & Scheduling
    Cognito[🔐 AWS Cognito]
    EventBridge[⏰ EventBridge Scheduler]
    SecretsManager[🔑 AWS Secrets Manager]
    
    %% Data Flow Connections
    User --> Frontend
    Frontend --> APIGateway
    APIGateway --> Lambda
    Lambda --> DynamoDB
    Lambda --> eBayAPI
    Lambda --> Postmark
    Lambda --> SecretsManager
    Lambda --> EventBridge
    Cognito --> APIGateway
    EventBridge --> Lambda
```

## Core Data Entities

### Data Models and Relationships

```mermaid
erDiagram
    Users ||--o{ Bids : "creates"
    Users ||--o{ BidHistory : "generates"
    Bids ||--o{ BidHistory : "tracks"
    
    Users {
        string userId PK
        string email
        timestamp createdAt
        timestamp updatedAt
        string ebayAccountId
        json preferences
        boolean isActive
        json ebayTokens "encrypted"
    }
    
    Bids {
        string userId PK
        string bidId SK
        string ebayItemId
        integer maxBidAmount
        string status
        timestamp auctionEndTime
        timestamp createdAt
        timestamp updatedAt
        string schedulerJobId
        string itemTitle
        string itemImageUrl
        integer currentPrice
        integer ttl
    }
    
    BidHistory {
        string userId PK
        integer timestamp SK
        string historyId
        string originalBidId
        string action
        json details
        string ebayItemId
        integer bidAmount
        integer finalPrice
        integer ttl
    }
```

## Frontend Data Flow

### State Management Architecture

#### 1. Client-Side State Management
```typescript
// Frontend state architecture
interface AppState {
  auth: AuthState
  user: UserState
  bids: BidsState
  wishlist: WishlistState
  ui: UIState
}

interface AuthState {
  isAuthenticated: boolean
  user: User | null
  token: string | null
  loading: boolean
  error: string | null
}

interface BidsState {
  activeBids: Bid[]
  bidHistory: BidHistoryItem[]
  loading: boolean
  error: string | null
  pagination: PaginationState
}
```

#### 2. Data Fetching Patterns
```typescript
// Custom hooks for data management
export function useBids() {
  const [state, setState] = useState<BidsState>({
    activeBids: [],
    bidHistory: [],
    loading: false,
    error: null,
    pagination: { hasMore: false, offset: 0, limit: 20 }
  })

  const fetchBids = useCallback(async (filters?: BidFilters) => {
    setState(prev => ({ ...prev, loading: true, error: null }))
    
    try {
      const response = await API.get('ebay-api', '/bids', {
        queryStringParameters: filters
      })
      
      setState(prev => ({
        ...prev,
        activeBids: response.bids,
        pagination: response.pagination,
        loading: false
      }))
    } catch (error) {
      setState(prev => ({
        ...prev,
        error: 'Failed to fetch bids',
        loading: false
      }))
    }
  }, [])

  return { ...state, fetchBids, createBid, updateBid, cancelBid }
}
```

#### 3. Real-time Updates
```typescript
// WebSocket connection for real-time bid updates
export function useRealtimeBids() {
  const [socket, setSocket] = useState<WebSocket | null>(null)
  const { updateBidStatus } = useBids()

  useEffect(() => {
    const ws = new WebSocket(process.env.NEXT_PUBLIC_WS_URL)
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data)
      
      switch (data.type) {
        case 'BID_STATUS_UPDATE':
          updateBidStatus(data.bidId, data.status)
          break
        case 'AUCTION_ENDING_SOON':
          showNotification(`Auction ending in ${data.timeRemaining}`)
          break
      }
    }
    
    setSocket(ws)
    return () => ws.close()
  }, [])
}
```

### Component Data Flow

#### 1. Bid Creation Flow
```typescript
// BidForm component data flow
export function BidForm({ item }: { item: WishlistItem }) {
  const { createBid } = useBids()
  const { user } = useAuth()
  
  const handleSubmit = async (formData: CreateBidRequest) => {
    // Client-side validation
    if (formData.maxBidAmount <= item.currentPrice) {
      setError('Bid must be higher than current price')
      return
    }
    
    // Optimistic update
    const tempBid = {
      ...formData,
      bidId: 'temp-' + Date.now(),
      status: 'PENDING',
      userId: user.userId
    }
    
    addOptimisticBid(tempBid)
    
    try {
      // API call
      const createdBid = await createBid(formData)
      
      // Replace optimistic update with real data
      replaceOptimisticBid(tempBid.bidId, createdBid)
      
      // Navigation
      router.push('/bids')
      
    } catch (error) {
      // Rollback optimistic update
      removeOptimisticBid(tempBid.bidId)
      setError('Failed to create bid')
    }
  }
}
```

## Backend Data Flow

### API Request Processing

#### 1. Request Lifecycle
```mermaid
sequenceDiagram
    participant Client
    participant APIGateway
    participant Lambda
    participant Cognito
    participant DynamoDB
    participant eBayAPI
    
    Client->>APIGateway: HTTP Request
    APIGateway->>Cognito: Validate JWT Token
    Cognito-->>APIGateway: Token Valid
    APIGateway->>Lambda: Invoke Function
    Lambda->>DynamoDB: Query/Update Data
    DynamoDB-->>Lambda: Data Response
    Lambda->>eBayAPI: External API Call
    eBayAPI-->>Lambda: API Response
    Lambda-->>APIGateway: Response
    APIGateway-->>Client: HTTP Response
```

#### 2. Data Processing Pipeline
```python
# Request processing pipeline
from typing import Dict, Any
from fastapi import Request, Depends

async def process_bid_request(
    request: CreateBidRequest,
    current_user: User = Depends(get_current_user)
) -> BidResponse:
    """Complete bid creation data flow"""
    
    # 1. Input validation
    validated_data = await validate_bid_request(request)
    
    # 2. External data fetching
    item_details = await fetch_ebay_item_details(
        validated_data.ebay_item_id,
        current_user.ebay_tokens
    )
    
    # 3. Business logic validation
    await validate_bid_business_rules(validated_data, item_details)
    
    # 4. Data transformation
    bid_entity = transform_to_bid_entity(
        validated_data, 
        item_details, 
        current_user
    )
    
    # 5. Database operations
    async with database_transaction():
        # Create bid record
        created_bid = await bid_repository.create_bid(bid_entity)
        
        # Create history record
        await bid_history_repository.create_history_entry(
            created_bid, 
            action="CREATE"
        )
        
        # Schedule bid execution
        job_id = await schedule_bid_execution(created_bid)
        
        # Update bid with scheduler job ID
        await bid_repository.update_bid(
            created_bid.bid_id, 
            {"scheduler_job_id": job_id}
        )
    
    # 6. Response transformation
    return transform_to_response(created_bid)
```

### Database Access Patterns

#### 1. Query Patterns
```python
# Optimized DynamoDB access patterns
class BidRepository:
    def __init__(self, table):
        self.table = table
    
    async def get_user_bids(
        self, 
        user_id: str, 
        status_filter: Optional[str] = None,
        limit: int = 20,
        last_key: Optional[Dict] = None
    ) -> Dict[str, Any]:
        """Efficient pagination and filtering"""
        
        query_params = {
            'KeyConditionExpression': Key('userId').eq(user_id),
            'Limit': limit,
            'ScanIndexForward': False  # Latest first
        }
        
        if status_filter:
            query_params['FilterExpression'] = Attr('status').eq(status_filter)
        
        if last_key:
            query_params['ExclusiveStartKey'] = last_key
        
        response = self.table.query(**query_params)
        
        return {
            'items': response.get('Items', []),
            'last_key': response.get('LastEvaluatedKey'),
            'has_more': 'LastEvaluatedKey' in response
        }
    
    async def batch_update_bid_statuses(
        self, 
        bid_updates: List[Dict[str, Any]]
    ) -> None:
        """Efficient batch updates for bid status changes"""
        
        with self.table.batch_writer() as batch:
            for update in bid_updates:
                batch.put_item(Item=update)
```

#### 2. Data Consistency Patterns
```python
# Transaction handling for data consistency
from boto3.dynamodb.conditions import Key, Attr
from botocore.exceptions import ClientError

async def execute_bid_transaction(bid_data: Dict[str, Any]) -> Dict[str, Any]:
    """Ensure data consistency across multiple operations"""
    
    try:
        # Start transaction
        with dynamodb.meta.client.transact_write_items as transact:
            
            # 1. Create bid record
            transact.append({
                'Put': {
                    'TableName': 'Bids',
                    'Item': bid_data,
                    'ConditionExpression': 'attribute_not_exists(bidId)'
                }
            })
            
            # 2. Create history record
            history_data = create_history_record(bid_data, 'CREATE')
            transact.append({
                'Put': {
                    'TableName': 'BidHistory',
                    'Item': history_data
                }
            })
            
            # 3. Update user's bid count
            transact.append({
                'Update': {
                    'TableName': 'Users',
                    'Key': {'userId': bid_data['userId']},
                    'UpdateExpression': 'ADD activeBidCount :inc',
                    'ExpressionAttributeValues': {':inc': 1}
                }
            })
            
            # Execute transaction
            transact.execute()
            
    except ClientError as e:
        if e.response['Error']['Code'] == 'TransactionCanceledException':
            raise ConflictError("Bid already exists for this item")
        raise DatabaseError(f"Transaction failed: {str(e)}")
```

## External Service Integration

### eBay API Data Flow

#### 1. Authentication Flow
```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant Backend
    participant eBayAPI
    participant SecretsManager
    
    User->>Frontend: Link eBay Account
    Frontend->>Backend: POST /ebay/auth/link
    Backend->>eBayAPI: Generate Auth URL
    eBayAPI-->>Backend: Auth URL + State
    Backend-->>Frontend: Auth URL
    Frontend->>User: Redirect to eBay
    User->>eBayAPI: Authorize Application
    eBayAPI->>Frontend: Callback with Code
    Frontend->>Backend: POST /ebay/auth/callback
    Backend->>eBayAPI: Exchange Code for Tokens
    eBayAPI-->>Backend: Access + Refresh Tokens
    Backend->>SecretsManager: Store Encrypted Tokens
    Backend-->>Frontend: Success Response
```

#### 2. Data Synchronization
```python
# eBay data synchronization patterns
class eBayDataSyncService:
    def __init__(self, ebay_client, user_repository):
        self.ebay_client = ebay_client
        self.user_repository = user_repository
    
    async def sync_user_wishlist(self, user_id: str) -> List[Dict[str, Any]]:
        """Sync user's eBay wishlist with current auction data"""
        
        # Get user's eBay tokens
        user = await self.user_repository.get_user(user_id)
        if not user.ebay_tokens:
            raise ValueError("User has not linked eBay account")
        
        # Decrypt tokens
        tokens = await decrypt_ebay_tokens(user.ebay_tokens)
        
        # Fetch wishlist from eBay
        wishlist_items = await self.ebay_client.get_wishlist_items(
            tokens['access_token']
        )
        
        # Filter for active auctions
        active_auctions = []
        for item in wishlist_items:
            if await self.is_auction_active(item['ebay_item_id'], tokens):
                item_details = await self.ebay_client.get_item_details(
                    item['ebay_item_id'],
                    tokens['access_token']
                )
                active_auctions.append(item_details)
        
        # Cache results with TTL
        await self.cache_wishlist_data(user_id, active_auctions, ttl=300)
        
        return active_auctions
    
    async def refresh_item_data(
        self, 
        ebay_item_id: str, 
        tokens: Dict[str, str]
    ) -> Dict[str, Any]:
        """Refresh individual item data from eBay"""
        
        try:
            # Get current item data
            item_data = await self.ebay_client.get_item_details(
                ebay_item_id,
                tokens['access_token']
            )
            
            # Update cached data
            await self.update_cached_item_data(ebay_item_id, item_data)
            
            return item_data
            
        except eBayAPIError as e:
            if e.code == 'ITEM_NOT_FOUND':
                # Item was removed, mark bids as invalid
                await self.mark_bids_invalid(ebay_item_id)
            raise
```

### Email Notification Flow

#### 1. Notification Pipeline
```python
# Email notification data flow
class NotificationService:
    def __init__(self, postmark_client, template_engine):
        self.postmark_client = postmark_client
        self.template_engine = template_engine
    
    async def process_bid_notification(
        self, 
        bid_event: Dict[str, Any]
    ) -> None:
        """Process bid-related notifications"""
        
        # Get user preferences
        user = await get_user(bid_event['user_id'])
        
        # Check if notifications are enabled
        if not self.should_send_notification(user, bid_event['type']):
            return
        
        # Prepare notification data
        notification_data = await self.prepare_notification_data(
            bid_event, 
            user
        )
        
        # Render email template
        email_content = await self.template_engine.render(
            template=bid_event['type'],
            data=notification_data
        )
        
        # Send email
        await self.postmark_client.send_email(
            to=user.email,
            subject=email_content['subject'],
            html_body=email_content['html'],
            text_body=email_content['text']
        )
        
        # Log notification
        await self.log_notification_sent(bid_event, user.user_id)
    
    def should_send_notification(
        self, 
        user: User, 
        notification_type: str
    ) -> bool:
        """Check user preferences for notification type"""
        
        preference_map = {
            'BID_WON': user.preferences.bid_win_notifications,
            'BID_LOST': user.preferences.bid_loss_notifications,
            'BID_PLACED': user.preferences.email_notifications,
            'AUCTION_ENDING': user.preferences.email_notifications
        }
        
        return preference_map.get(notification_type, False)
```

## Scheduled Processing

### Bid Execution Data Flow

#### 1. Scheduling Workflow
```mermaid
sequenceDiagram
    participant User
    participant API
    participant EventBridge
    participant BidExecutor
    participant eBayAPI
    participant DynamoDB
    participant Notifications
    
    User->>API: Create Bid
    API->>DynamoDB: Store Bid
    API->>EventBridge: Schedule Execution
    EventBridge-->>API: Job ID
    API->>DynamoDB: Update Bid with Job ID
    
    Note over EventBridge: Wait until 5s before auction end
    
    EventBridge->>BidExecutor: Trigger Execution
    BidExecutor->>DynamoDB: Get Bid Details
    BidExecutor->>eBayAPI: Place Bid
    eBayAPI-->>BidExecutor: Bid Result
    BidExecutor->>DynamoDB: Update Bid Status
    BidExecutor->>DynamoDB: Create History Record
    BidExecutor->>Notifications: Send Result Email
```

#### 2. Bid Execution Logic
```python
# Bid execution data processing
class BidExecutorService:
    def __init__(self, ebay_client, bid_repository, notification_service):
        self.ebay_client = ebay_client
        self.bid_repository = bid_repository
        self.notification_service = notification_service
    
    async def execute_scheduled_bid(self, bid_id: str, user_id: str) -> Dict[str, Any]:
        """Execute a scheduled bid at the optimal time"""
        
        # Get bid details
        bid = await self.bid_repository.get_bid(user_id, bid_id)
        if not bid:
            raise ValueError(f"Bid not found: {bid_id}")
        
        # Verify bid is still valid
        if bid.status != BidStatus.PENDING:
            logger.warning(f"Bid {bid_id} is not in pending status: {bid.status}")
            return {"status": "SKIPPED", "reason": "Bid not pending"}
        
        try:
            # Get current item status
            item_data = await self.get_current_item_data(bid.ebay_item_id, bid.user_id)
            
            # Validate bid is still viable
            validation_result = await self.validate_bid_execution(bid, item_data)
            if not validation_result.valid:
                await self.mark_bid_failed(bid, validation_result.reason)
                return {"status": "FAILED", "reason": validation_result.reason}
            
            # Execute the bid
            bid_result = await self.place_ebay_bid(bid, item_data)
            
            # Update bid status based on result
            await self.process_bid_result(bid, bid_result)
            
            # Send notification
            await self.notification_service.send_bid_result_notification(
                bid, 
                bid_result
            )
            
            return {"status": "SUCCESS", "result": bid_result}
            
        except Exception as e:
            logger.error(f"Bid execution failed for {bid_id}: {str(e)}")
            await self.mark_bid_failed(bid, str(e))
            return {"status": "ERROR", "error": str(e)}
    
    async def place_ebay_bid(
        self, 
        bid: Bid, 
        item_data: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Place the actual bid on eBay"""
        
        # Get user's eBay tokens
        user_tokens = await self.get_user_ebay_tokens(bid.user_id)
        
        # Calculate optimal bid timing
        time_until_end = item_data['end_time'] - time.time()
        if time_until_end > 10:  # More than 10 seconds left
            await asyncio.sleep(time_until_end - 5)  # Wait until 5s before end
        
        # Place the bid
        bid_response = await self.ebay_client.place_bid(
            item_id=bid.ebay_item_id,
            bid_amount=bid.max_bid_amount / 100,  # Convert cents to dollars
            access_token=user_tokens['access_token']
        )
        
        return bid_response
```

## Data Transformation

### Request/Response Transformation

#### 1. API Data Transformation
```python
# Data transformation layer
from typing import Dict, Any, List
from decimal import Decimal

class DataTransformer:
    @staticmethod
    def transform_ebay_item_to_internal(ebay_item: Dict[str, Any]) -> Dict[str, Any]:
        """Transform eBay API response to internal format"""
        
        return {
            'ebay_item_id': ebay_item['itemId'],
            'title': ebay_item['title'],
            'current_price': int(float(ebay_item['price']['value']) * 100),  # Convert to cents
            'end_time': int(datetime.fromisoformat(
                ebay_item['itemEndDate'].replace('Z', '+00:00')
            ).timestamp()),
            'image_urls': [img['imageUrl'] for img in ebay_item.get('image', [])],
            'seller_info': {
                'seller_id': ebay_item['seller']['username'],
                'seller_name': ebay_item['seller'].get('feedbackScore', 0),
                'feedback_score': ebay_item['seller'].get('feedbackScore', 0),
                'feedback_percentage': ebay_item['seller'].get('positiveFeedbackPercent', 0)
            },
            'category_id': ebay_item.get('categoryId'),
            'condition': ebay_item.get('condition', {}).get('conditionDisplayName', 'Unknown'),
            'location': ebay_item.get('itemLocation', {}).get('city', 'Unknown'),
            'shipping_cost': int(float(ebay_item.get('shippingOptions', [{}])[0].get('shippingCost', {}).get('value', '0')) * 100)
        }
    
    @staticmethod
    def transform_bid_to_response(bid: Dict[str, Any]) -> Dict[str, Any]:
        """Transform internal bid data to API response format"""
        
        return {
            'bidId': bid['bidId'],
            'userId': bid['userId'],
            'ebayItemId': bid['ebayItemId'],
            'maxBidAmount': bid['maxBidAmount'],
            'status': bid['status'],
            'auctionEndTime': bid['auctionEndTime'],
            'createdAt': bid['createdAt'],
            'updatedAt': bid['updatedAt'],
            'schedulerJobId': bid.get('schedulerJobId'),
            'itemTitle': bid.get('itemTitle'),
            'itemImageUrl': bid.get('itemImageUrl'),
            'currentPrice': bid.get('currentPrice')
        }
    
    @staticmethod
    def transform_pagination_response(
        items: List[Dict[str, Any]], 
        pagination_info: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Transform paginated data to standard response format"""
        
        return {
            'items': items,
            'pagination': {
                'total': pagination_info.get('total', len(items)),
                'limit': pagination_info.get('limit', 20),
                'offset': pagination_info.get('offset', 0),
                'hasMore': pagination_info.get('has_more', False)
            }
        }
```

### Data Validation Pipeline

#### 1. Input Validation
```python
# Comprehensive data validation
from pydantic import BaseModel, validator, Field
from typing import Optional, List
import re

class CreateBidRequestValidator(BaseModel):
    ebay_item_id: str = Field(..., min_length=1, max_length=20)
    max_bid_amount: int = Field(..., ge=100, le=10000000)  # $1 to $100,000
    
    @validator('ebay_item_id')
    def validate_ebay_item_id(cls, v):
        if not re.match(r'^[0-9]+$', v):
            raise ValueError('eBay item ID must contain only digits')
        return v
    
    @validator('max_bid_amount')
    def validate_bid_amount(cls, v):
        if v % 1 != 0:  # Ensure it's a whole number (cents)
            raise ValueError('Bid amount must be in cents (whole number)')
        return v

class BidBusinessRuleValidator:
    def __init__(self, ebay_client, bid_repository):
        self.ebay_client = ebay_client
        self.bid_repository = bid_repository
    
    async def validate_bid_creation(
        self, 
        request: CreateBidRequestValidator,
        user: User
    ) -> Dict[str, Any]:
        """Comprehensive business rule validation"""
        
        validation_results = {
            'valid': True,
            'errors': [],
            'warnings': []
        }
        
        # Check if item exists and is active
        try:
            item_data = await self.ebay_client.get_item_details(
                request.ebay_item_id,
                user.ebay_tokens['access_token']
            )
        except Exception as e:
            validation_results['valid'] = False
            validation_results['errors'].append(f"Unable to fetch item details: {str(e)}")
            return validation_results
        
        # Validate auction timing
        time_until_end = item_data['end_time'] - time.time()
        if time_until_end < 60:  # Less than 1 minute
            validation_results['valid'] = False
            validation_results['errors'].append("Auction ends too soon to place bid")
        elif time_until_end < 300:  # Less than 5 minutes
            validation_results['warnings'].append("Auction ends very soon")
        
        # Validate bid amount
        if request.max_bid_amount <= item_data['current_price']:
            validation_results['valid'] = False
            validation_results['errors'].append("Bid amount must be higher than current price")
        
        # Check for existing bids
        existing_bids = await self.bid_repository.get_user_item_bids(
            user.user_id,
            request.ebay_item_id
        )
        
        active_bids = [bid for bid in existing_bids if bid['status'] == 'PENDING']
        if active_bids:
            validation_results['valid'] = False
            validation_results['errors'].append("You already have an active bid for this item")
        
        return validation_results
```

## Error Handling and Data Recovery

### Error Propagation Patterns

#### 1. Error Handling Pipeline
```python
# Comprehensive error handling
from enum import Enum
from typing import Dict, Any, Optional

class ErrorSeverity(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

class DataFlowError(Exception):
    def __init__(
        self, 
        message: str, 
        severity: ErrorSeverity,
        context: Optional[Dict[str, Any]] = None,
        recoverable: bool = True
    ):
        self.message = message
        self.severity = severity
        self.context = context or {}
        self.recoverable = recoverable
        super().__init__(message)

class ErrorHandler:
    def __init__(self, logger, notification_service):
        self.logger = logger
        self.notification_service = notification_service
    
    async def handle_data_flow_error(
        self, 
        error: DataFlowError,
        operation_context: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Handle errors in data flow with appropriate recovery"""
        
        # Log error with context
        self.logger.error(
            f"Data flow error: {error.message}",
            extra={
                'severity': error.severity.value,
                'context': error.context,
                'operation': operation_context,
                'recoverable': error.recoverable
            }
        )
        
        # Determine recovery strategy
        if error.recoverable:
            recovery_result = await self.attempt_recovery(error, operation_context)
            if recovery_result['success']:
                return recovery_result
        
        # Alert if critical
        if error.severity in [ErrorSeverity.HIGH, ErrorSeverity.CRITICAL]:
            await self.notification_service.send_error_alert(error, operation_context)
        
        # Return error response
        return {
            'success': False,
            'error': error.message,
            'severity': error.severity.value,
            'recoverable': error.recoverable
        }
    
    async def attempt_recovery(
        self, 
        error: DataFlowError,
        context: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Attempt to recover from data flow errors"""
        
        recovery_strategies = {
            'eBayAPIError': self.recover_ebay_api_error,
            'DatabaseError': self.recover_database_error,
            'ValidationError': self.recover_validation_error
        }
        
        strategy = recovery_strategies.get(type(error).__name__)
        if strategy:
            return await strategy(error, context)
        
        return {'success': False, 'message': 'No recovery strategy available'}
```

### Data Consistency Patterns

#### 1. Eventual Consistency Handling
```python
# Handle eventual consistency in distributed systems
class ConsistencyManager:
    def __init__(self, retry_config):
        self.retry_config = retry_config
    
    async def ensure_data_consistency(
        self, 
        operation: str,
        data: Dict[str, Any],
        consistency_checks: List[callable]
    ) -> bool:
        """Ensure data consistency across distributed components"""
        
        max_retries = self.retry_config.get('max_retries', 3)
        delay = self.retry_config.get('initial_delay', 1)
        
        for attempt in range(max_retries):
            try:
                # Run all consistency checks
                all_consistent = True
                for check in consistency_checks:
                    if not await check(data):
                        all_consistent = False
                        break
                
                if all_consistent:
                    return True
                
                # Wait before retry with exponential backoff
                await asyncio.sleep(delay * (2 ** attempt))
                
            except Exception as e:
                logger.warning(f"Consistency check failed on attempt {attempt + 1}: {str(e)}")
        
        # Consistency not achieved - trigger compensation
        await self.trigger_compensation_logic(operation, data)
        return False
    
    async def trigger_compensation_logic(
        self, 
        operation: str, 
        data: Dict[str, Any]
    ) -> None:
        """Trigger compensation logic for consistency failures"""
        
        compensation_actions = {
            'bid_creation': self.compensate_bid_creation,
            'bid_update': self.compensate_bid_update,
            'bid_execution': self.compensate_bid_execution
        }
        
        action = compensation_actions.get(operation)
        if action:
            await action(data)
```

## Performance Optimization

### Data Caching Strategies

#### 1. Multi-Layer Caching
```python
# Comprehensive caching strategy
from typing import Dict, Any, Optional
import redis
import json

class CacheManager:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.cache_layers = {
            'l1': {'ttl': 60, 'prefix': 'l1:'},      # Short-term cache
            'l2': {'ttl': 300, 'prefix': 'l2:'},     # Medium-term cache
            'l3': {'ttl': 3600, 'prefix': 'l3:'}     # Long-term cache
        }
    
    async def get_cached_data(
        self, 
        key: str, 
        layer: str = 'l2'
    ) -> Optional[Dict[str, Any]]:
        """Get data from specified cache layer"""
        
        cache_key = f"{self.cache_layers[layer]['prefix']}{key}"
        
        try:
            cached_data = self.redis.get(cache_key)
            if cached_data:
                return json.loads(cached_data)
        except Exception as e:
            logger.warning(f"Cache read error for {cache_key}: {str(e)}")
        
        return None
    
    async def set_cached_data(
        self, 
        key: str, 
        data: Dict[str, Any],
        layer: str = 'l2',
        custom_ttl: Optional[int] = None
    ) -> None:
        """Set data in specified cache layer"""
        
        cache_config = self.cache_layers[layer]
        cache_key = f"{cache_config['prefix']}{key}"
        ttl = custom_ttl or cache_config['ttl']
        
        try:
            self.redis.setex(
                cache_key, 
                ttl, 
                json.dumps(data, default=str)
            )
        except Exception as e:
            logger.warning(f"Cache write error for {cache_key}: {str(e)}")
    
    async def invalidate_cache_pattern(self, pattern: str) -> None:
        """Invalidate cache entries matching pattern"""
        
        try:
            keys = self.redis.keys(pattern)
            if keys:
                self.redis.delete(*keys)
        except Exception as e:
            logger.error(f"Cache invalidation error for pattern {pattern}: {str(e)}")

# Usage in data flow
class OptimizedDataService:
    def __init__(self, cache_manager, data_repository):
        self.cache = cache_manager
        self.repository = data_repository
    
    async def get_user_bids_optimized(self, user_id: str) -> List[Dict[str, Any]]:
        """Get user bids with multi-layer caching"""
        
        # Try L1 cache first (most recent data)
        cache_key = f"user_bids:{user_id}"
        cached_bids = await self.cache.get_cached_data(cache_key, 'l1')
        
        if cached_bids:
            return cached_bids
        
        # Try L2 cache
        cached_bids = await self.cache.get_cached_data(cache_key, 'l2')
        
        if cached_bids:
            # Promote to L1 cache
            await self.cache.set_cached_data(cache_key, cached_bids, 'l1')
            return cached_bids
        
        # Fetch from database
        bids = await self.repository.get_user_bids(user_id)
        
        # Cache in both layers
        await self.cache.set_cached_data(cache_key, bids, 'l1')
        await self.cache.set_cached_data(cache_key, bids, 'l2')
        
        return bids
```

## Monitoring and Observability

### Data Flow Monitoring

#### 1. Data Flow Metrics
```python
# Comprehensive monitoring of data flows
import time
from typing import Dict, Any
import boto3

class DataFlowMonitor:
    def __init__(self, cloudwatch_client):
        self.cloudwatch = cloudwatch_client
        self.namespace = 'EbaySniperApp/DataFlow'
    
    async def track_data_flow_operation(
        self, 
        operation: str,
        start_time: float,
        success: bool,
        metadata: Dict[str, Any] = None
    ) -> None:
        """Track data flow operation metrics"""
        
        duration = time.time() - start_time
        
        # Basic metrics
        metrics = [
            {
                'MetricName': f'{operation}_Duration',
                'Value': duration * 1000,  # Convert to milliseconds
                'Unit': 'Milliseconds'
            },
            {
                'MetricName': f'{operation}_Count',
                'Value': 1,
                'Unit': 'Count'
            }
        ]
        
        # Success/failure metrics
        if success:
            metrics.append({
                'MetricName': f'{operation}_Success',
                'Value': 1,
                'Unit': 'Count'
            })
        else:
            metrics.append({
                'MetricName': f'{operation}_Error',
                'Value': 1,
                'Unit': 'Count'
            })
        
        # Add metadata as dimensions
        dimensions = []
        if metadata:
            for key, value in metadata.items():
                if isinstance(value, (str, int, float)):
                    dimensions.append({
                        'Name': key,
                        'Value': str(value)
                    })
        
        # Send to CloudWatch
        try:
            self.cloudwatch.put_metric_data(
                Namespace=self.namespace,
                MetricData=[
                    {**metric, 'Dimensions': dimensions} 
                    for metric in metrics
                ]
            )
        except Exception as e:
            logger.error(f"Failed to send metrics: {str(e)}")
    
    def create_data_flow_dashboard(self) -> str:
        """Create CloudWatch dashboard for data flow monitoring"""
        
        dashboard_body = {
            "widgets": [
                {
                    "type": "metric",
                    "properties": {
                        "metrics": [
                            [self.namespace, "BidCreation_Duration"],
                            [self.namespace, "BidExecution_Duration"],
                            [self.namespace, "EbaySync_Duration"]
                        ],
                        "view": "timeSeries",
                        "stacked": False,
                        "region": "us-east-1",
                        "title": "Data Flow Operation Durations"
                    }
                },
                {
                    "type": "metric", 
                    "properties": {
                        "metrics": [
                            [self.namespace, "BidCreation_Success"],
                            [self.namespace, "BidCreation_Error"],
                            [self.namespace, "BidExecution_Success"],
                            [self.namespace, "BidExecution_Error"]
                        ],
                        "view": "number",
                        "region": "us-east-1",
                        "title": "Success/Error Rates"
                    }
                }
            ]
        }
        
        return json.dumps(dashboard_body)
```

This comprehensive data flow documentation provides a complete picture of how information moves through the eBay Sniper application, from user interactions to data persistence and external service integration. The patterns and examples shown here ensure maintainable, scalable, and observable data flows throughout the system.