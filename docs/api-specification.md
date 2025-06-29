# API Specification

This document defines the complete REST API specification for the eBay Sniper application. The API uses individual AWS Lambda functions for each endpoint, deployed via API Gateway with single responsibility design.

## Base Configuration

- **Base URL**: `https://api.ebay-sniper.com/v1`
- **Protocol**: HTTPS only
- **Authentication**: AWS Cognito JWT tokens
- **Content-Type**: `application/json`
- **Rate Limiting**: 100 requests/minute per user

## Authentication

All API endpoints require authentication via AWS Cognito JWT tokens, except for public health checks.

### Headers
```
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

### Authentication Errors
```json
{
  "error": "unauthorized",
  "message": "Invalid or expired token",
  "code": 401
}
```

## Lambda Function Architecture

The eBay Sniper application uses **8 individual Lambda functions** with single responsibility design:

### **API Endpoint Lambda Functions** (5):
- **User Management Lambda**: `/users/*` endpoints
- **eBay OAuth Lambda**: `/ebay/auth/*` endpoints  
- **Wishlist Sync Lambda**: `/wishlist/*` endpoints
- **Bid Management Lambda**: `/bids/*` endpoints
- **Bid History Lambda**: `/bid-history/*` endpoints

### **Background/Scheduled Lambda Functions** (3):
- **Bid Executor Lambda**: Executes scheduled bids and determines outcomes (triggered by EventBridge)
- **Notification Handler Lambda**: Sends email notifications (invoked by Bid Executor)
- **Token Refresh Lambda**: Refreshes eBay OAuth tokens (triggered by EventBridge daily)

## API Endpoints Documentation

The following sections document the HTTP endpoints handled by the 5 API Lambda functions:

---

## User Management Lambda

Handles user profile CRUD operations and preference management.

### GET /users/profile
Get current user profile information.

**Lambda Function**: `user-management`  
**Headers**: Requires authentication

**Response 200**:
```json
{
  "userId": "uuid-string",
  "email": "user@example.com",
  "createdAt": 1640995200,
  "updatedAt": 1640995200,
  "preferences": {
    "emailNotifications": true,
    "bidWinNotifications": true,
    "bidLossNotifications": true,
    "timezone": "America/New_York"
  },
  "isActive": true
}
```

**Error Responses**:
- `404`: User not found
- `401`: Unauthorized

### PUT /users/profile
Update current user profile.

**Lambda Function**: `user-management`  
**Headers**: Requires authentication

**Request Body**:
```json
{
  "email": "newemail@example.com"
}
```

**Response 200**:
```json
{
  "userId": "uuid-string",
  "email": "newemail@example.com",
  "updatedAt": 1640995300,
  "preferences": {
    "emailNotifications": true,
    "bidWinNotifications": true,
    "bidLossNotifications": true,
    "timezone": "America/New_York"
  },
  "isActive": true
}
```

**Error Responses**:
- `400`: Invalid email format
- `409`: Email already exists
- `404`: User not found

### PUT /users/preferences
Update user notification preferences.

**Lambda Function**: `user-management`  
**Headers**: Requires authentication

**Request Body**:
```json
{
  "emailNotifications": false,
  "bidWinNotifications": true,
  "bidLossNotifications": true,
  "timezone": "America/Los_Angeles"
}
```

**Response 200**:
```json
{
  "userId": "uuid-string",
  "email": "user@example.com",
  "updatedAt": 1640995300,
  "preferences": {
    "emailNotifications": false,
    "bidWinNotifications": true,
    "bidLossNotifications": true,
    "timezone": "America/Los_Angeles"
  },
  "isActive": true
}
```

**Error Responses**:
- `400`: Invalid preference values
- `404`: User not found

### DELETE /users/account
Delete current user account and all associated data.

**Lambda Function**: `user-management`  
**Headers**: Requires authentication

**Response 204**: No content

**Error Responses**:
- `404`: User not found

---

## eBay OAuth Lambda

Manages eBay account linking, OAuth authorization flow, and token management.

### GET /ebay/auth/url
Generate eBay OAuth authorization URL.

**Lambda Function**: `ebay-oauth`  
**Headers**: Requires authentication

**Response 200**:
```json
{
  "authUrl": "https://auth.sandbox.ebay.com/oauth2/authorize?client_id=...",
  "state": "random-state-string"
}
```

**Error Responses**:
- `500`: Failed to generate auth URL

### POST /ebay/auth/callback
Handle eBay OAuth callback and exchange code for tokens.

**Lambda Function**: `ebay-oauth`  
**Headers**: Requires authentication

**Request Body**:
```json
{
  "code": "authorization-code-from-ebay",
  "state": "state-string-from-auth-url"
}
```

**Response 200**:
```json
{
  "isLinked": true,
  "accountId": "ebay-user-123",
  "linkedAt": 1640995200,
  "tokenExpiresAt": 1640995200
}
```

**Error Responses**:
- `400`: Invalid OAuth code or state
- `401`: OAuth authorization failed
- `409`: eBay account already linked
- `500`: Token exchange failed

### GET /ebay/auth/status
Check current eBay account linking status.

**Lambda Function**: `ebay-oauth`  
**Headers**: Requires authentication

**Response 200**:
```json
{
  "isLinked": true,
  "accountId": "ebay-user-123",
  "linkedAt": 1640995200,
  "tokenExpiresAt": 1640995200
}
```

### DELETE /ebay/auth/unlink
Unlink eBay account from user profile.

**Lambda Function**: `ebay-oauth`  
**Headers**: Requires authentication

**Response 204**: No content

**Error Responses**:
- `404`: No eBay account linked

---

## Wishlist Sync Lambda

Retrieves and syncs eBay wishlist items, filtering for active auctions.

### GET /wishlist/items
Get synchronized wishlist items.

**Lambda Function**: `wishlist-sync`  
**Headers**: Requires authentication

**Query Parameters**:
- `limit` (optional): Number of items to return (default: 20, max: 100)
- `offset` (optional): Pagination offset (default: 0)

**Response 200**:
```json
{
  "items": [
    {
      "ebayItemId": "123456789",
      "title": "Vintage Camera",
      "currentPrice": 15000,
      "endTime": 1640995200,
      "imageUrl": "https://i.ebayimg.com/...",
      "condition": "Used",
      "sellerInfo": {
        "sellerId": "vintage_seller",
        "feedbackScore": 1250,
        "feedbackPercentage": 99.5
      },
      "isAuctionActive": true
    }
  ],
  "lastSyncAt": 1640995100,
  "totalItems": 15
}
```

**Error Responses**:
- `401`: eBay account not linked
- `403`: eBay API rate limit exceeded
- `404`: User not found
- `500`: eBay API error

### GET /wishlist/items/{itemId}
Get specific item details from wishlist.

**Lambda Function**: `wishlist-sync`  
**Headers**: Requires authentication

**Path Parameters**:
- `itemId`: eBay item ID

**Response 200**:
```json
{
  "ebayItemId": "123456789",
  "title": "Vintage Camera",
  "currentPrice": 15000,
  "endTime": 1640995200,
  "imageUrl": "https://i.ebayimg.com/...",
  "condition": "Used",
  "sellerInfo": {
    "sellerId": "vintage_seller",
    "feedbackScore": 1250,
    "feedbackPercentage": 99.5
  },
  "isAuctionActive": true
}
```

**Error Responses**:
- `404`: Item not found in wishlist
- `401`: eBay account not linked
- `500`: eBay API error

---

## Bid Management Lambda

Handles bid CRUD operations and schedules bid execution.

### POST /bids
Create a new bid for an auction item.

**Lambda Function**: `bid-management`  
**Headers**: Requires authentication

**Request Body**:
```json
{
  "ebayItemId": "123456789",
  "maxBidAmount": 25000
}
```

**Response 200**:
```json
{
  "bidId": "bid-uuid-string",
  "userId": "user-uuid-string",
  "ebayItemId": "123456789",
  "maxBidAmount": 25000,
  "status": "PENDING",
  "auctionEndTime": 1640995200,
  "createdAt": 1640990000,
  "updatedAt": 1640990000,
  "itemTitle": "Vintage Camera",
  "itemImageUrl": "https://i.ebayimg.com/...",
  "currentPrice": 20000
}
```

**Error Responses**:
- `400`: Invalid bid amount or item ID
- `404`: Item not found
- `409`: Bid already exists for this item
- `500`: Database or scheduler error

### GET /bids
List user's active bids.

**Lambda Function**: `bid-management`  
**Headers**: Requires authentication

**Query Parameters**:
- `status` (optional): Filter by bid status (PENDING, PLACED, WON, LOST, CANCELLED)
- `limit` (optional): Number of bids to return (default: 20, max: 100)
- `offset` (optional): Pagination offset (default: 0)

**Response 200**:
```json
{
  "bids": [
    {
      "bidId": "bid-uuid-string",
      "userId": "user-uuid-string",
      "ebayItemId": "123456789",
      "maxBidAmount": 25000,
      "status": "PENDING",
      "auctionEndTime": 1640995200,
      "createdAt": 1640990000,
      "updatedAt": 1640990000,
      "itemTitle": "Vintage Camera",
      "itemImageUrl": "https://i.ebayimg.com/...",
      "currentPrice": 20000
    }
  ],
  "pagination": {
    "total": 5,
    "limit": 20,
    "offset": 0,
    "hasMore": false
  }
}
```

### GET /bids/{bidId}
Get specific bid details.

**Lambda Function**: `bid-management`  
**Headers**: Requires authentication

**Path Parameters**:
- `bidId`: Unique bid identifier

**Response 200**:
```json
{
  "bidId": "bid-uuid-string",
  "userId": "user-uuid-string",
  "ebayItemId": "123456789",
  "maxBidAmount": 25000,
  "status": "PENDING",
  "auctionEndTime": 1640995200,
  "createdAt": 1640990000,
  "updatedAt": 1640990000,
  "itemTitle": "Vintage Camera",
  "itemImageUrl": "https://i.ebayimg.com/...",
  "currentPrice": 20000
}
```

**Error Responses**:
- `404`: Bid not found

### PUT /bids/{bidId}
Update bid amount.

**Lambda Function**: `bid-management`  
**Headers**: Requires authentication

**Path Parameters**:
- `bidId`: Unique bid identifier

**Request Body**:
```json
{
  "maxBidAmount": 30000
}
```

**Response 200**:
```json
{
  "bidId": "bid-uuid-string",
  "userId": "user-uuid-string",
  "ebayItemId": "123456789",
  "maxBidAmount": 30000,
  "status": "PENDING",
  "auctionEndTime": 1640995200,
  "createdAt": 1640990000,
  "updatedAt": 1640991000,
  "itemTitle": "Vintage Camera",
  "itemImageUrl": "https://i.ebayimg.com/...",
  "currentPrice": 20000
}
```

**Error Responses**:
- `400`: Invalid bid amount
- `404`: Bid not found
- `409`: Cannot update bid in current status
- `500`: Database or scheduler error

### DELETE /bids/{bidId}
Cancel a pending bid.

**Lambda Function**: `bid-management`  
**Headers**: Requires authentication

**Path Parameters**:
- `bidId`: Unique bid identifier

**Response 204**: No content

**Error Responses**:
- `404`: Bid not found
- `409`: Cannot cancel bid in current status
- `500`: Database or scheduler error

---

## Bid History Lambda

Queries and retrieves historical bid data with pagination and filtering.

### GET /bid-history
Get user's bid history.

**Lambda Function**: `bid-history`  
**Headers**: Requires authentication

**Query Parameters**:
- `action` (optional): Filter by action type (CREATE, UPDATE, CANCEL, PLACE, WIN, LOSE)
- `startDate` (optional): Start date filter (Unix timestamp)
- `endDate` (optional): End date filter (Unix timestamp)
- `limit` (optional): Number of records to return (default: 20, max: 100)
- `offset` (optional): Pagination offset (default: 0)

**Response 200**:
```json
{
  "items": [
    {
      "historyId": "1640990000#CREATE#bid-uuid",
      "originalBidId": "bid-uuid-string",
      "action": "CREATE",
      "timestamp": 1640990000,
      "details": {
        "bidAmount": 25000,
        "reason": "User created new bid"
      },
      "ebayItemId": "123456789",
      "bidAmount": 25000,
      "finalPrice": null
    }
  ],
  "pagination": {
    "total": 25,
    "limit": 20,
    "offset": 0,
    "hasMore": true
  }
}
```

### GET /bid-history/{bidId}
Get history for a specific bid.

**Lambda Function**: `bid-history`  
**Headers**: Requires authentication

**Path Parameters**:
- `bidId`: Original bid identifier

**Response 200**:
```json
{
  "items": [
    {
      "historyId": "1640990000#CREATE#bid-uuid",
      "originalBidId": "bid-uuid-string",
      "action": "CREATE",
      "timestamp": 1640990000,
      "details": {
        "bidAmount": 25000,
        "reason": "User created new bid"
      },
      "ebayItemId": "123456789",
      "bidAmount": 25000,
      "finalPrice": null
    },
    {
      "historyId": "1640995000#PLACE#bid-uuid",
      "originalBidId": "bid-uuid-string",
      "action": "PLACE",
      "timestamp": 1640995000,
      "details": {
        "bidAmount": 25000,
        "ebayBidId": "ebay-bid-123",
        "executedAt": 1640995000
      },
      "ebayItemId": "123456789",
      "bidAmount": 25000,
      "finalPrice": 25000
    }
  ]
}
```

**Error Responses**:
- `404`: No history found for bid

---

## System Endpoints

### GET /health
Health check endpoint (public, no authentication required).

**Response 200**:
```json
{
  "status": "healthy",
  "timestamp": 1640995200,
  "version": "1.0.0",
  "environment": "production",
  "lambdaFunctions": {
    "userManagement": "healthy",
    "ebayOAuth": "healthy",
    "wishlistSync": "healthy",
    "bidManagement": "healthy",
    "bidHistory": "healthy",
    "bidExecutor": "healthy",
    "notificationHandler": "healthy",
    "tokenRefresh": "healthy"
  },
  "services": {
    "dynamodb": "healthy",
    "ebayApi": "healthy",
    "postmark": "healthy",
    "secretsManager": "healthy",
    "eventBridge": "healthy"
  }
}
```

**Notes**:
- The health check monitors all 8 Lambda functions including background/scheduled functions
- Background Lambda functions are checked via CloudWatch metrics and last execution status
- API endpoint Lambda functions are checked via direct invocation

---

## Error Responses

### Standard Error Format
```json
{
  "error": "error_code",
  "message": "Human readable error message",
  "code": 400,
  "details": {
    "field": "specific error details"
  }
}
```

### Common Error Codes

| HTTP Code | Error Code | Description |
|-----------|------------|-------------|
| 400 | bad_request | Invalid request format or parameters |
| 401 | unauthorized | Missing or invalid authentication |
| 403 | forbidden | Insufficient permissions |
| 404 | not_found | Resource not found |
| 409 | conflict | Resource conflict (e.g., bid already exists) |
| 422 | validation_error | Request validation failed |
| 429 | rate_limit_exceeded | Too many requests |
| 500 | internal_error | Internal server error |
| 502 | service_unavailable | External service unavailable |
| 503 | maintenance | Service under maintenance |

### Validation Errors
```json
{
  "error": "validation_error",
  "message": "Request validation failed",
  "code": 422,
  "details": {
    "maxBidAmount": "Must be greater than current price",
    "ebayItemId": "Invalid item ID format"
  }
}
```

## Rate Limiting

- **Default Limit**: 100 requests per minute per user
- **Burst Limit**: 20 requests per 10 seconds
- **Headers**: Rate limit information included in response headers
  - `X-RateLimit-Limit`: Total requests allowed per window
  - `X-RateLimit-Remaining`: Requests remaining in current window
  - `X-RateLimit-Reset`: Time when rate limit resets (Unix timestamp)

## Data Validation

### Bid Amount
- Must be positive integer (in cents)
- Must be at least $1.00 (100 cents)
- Must be greater than current auction price
- Maximum bid amount: $100,000 (10,000,000 cents)

### eBay Item ID
- Must be valid eBay item identifier
- Must correspond to active auction
- Auction must end more than 10 seconds in the future


### Timestamps
- All timestamps are Unix timestamps (seconds since epoch)
- UTC timezone used for all operations

## Security Considerations

### Authentication
- JWT tokens expire after 1 hour
- Refresh tokens valid for 30 days
- Token validation on every request

### Authorization
- Users can only access their own data
- Admin endpoints require special permissions
- Rate limiting prevents abuse

### Data Protection
- All sensitive data encrypted at rest
- eBay tokens encrypted with AWS KMS
- PII handling complies with privacy regulations

### Input Validation
- All inputs validated and sanitized
- SQL injection prevention (using DynamoDB)
- XSS prevention in returned data