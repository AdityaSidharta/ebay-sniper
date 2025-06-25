# API Specification

This document defines the complete REST API specification for the eBay Sniper application. The API is built using FastAPI and deployed on AWS Lambda with API Gateway.

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

## User Management

### GET /users/me
Get current user profile information.

**Headers**: Requires authentication

**Response 200**:
```json
{
  "userId": "uuid-string",
  "email": "user@example.com",
  "createdAt": 1640995200,
  "updatedAt": 1640995200,
  "ebayAccountId": "ebay-user-123",
  "preferences": {
    "emailNotifications": true,
    "bidWinNotifications": true,
    "bidLossNotifications": true,
    "timezone": "America/New_York"
  },
  "isActive": true
}
```

### PUT /users/me
Update current user profile and preferences.

**Headers**: Requires authentication

**Request Body**:
```json
{
  "preferences": {
    "emailNotifications": false,
    "bidWinNotifications": true,
    "bidLossNotifications": true,
    "timezone": "America/Los_Angeles"
  }
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
  }
}
```

### DELETE /users/me
Delete current user account and all associated data.

**Headers**: Requires authentication

**Response 204**: No content

## eBay Integration

### POST /ebay/auth/link
Initiate eBay account linking process.

**Headers**: Requires authentication

**Response 200**:
```json
{
  "authUrl": "https://auth.ebay.com/oauth2/authorize?client_id=...",
  "state": "random-state-string"
}
```

### POST /ebay/auth/callback
Complete eBay OAuth flow and store tokens.

**Headers**: Requires authentication

**Request Body**:
```json
{
  "code": "ebay-auth-code",
  "state": "random-state-string"
}
```

**Response 200**:
```json
{
  "success": true,
  "ebayAccountId": "ebay-user-123",
  "linkedAt": 1640995200
}
```

### DELETE /ebay/auth/unlink
Unlink eBay account from user profile.

**Headers**: Requires authentication

**Response 204**: No content

### GET /ebay/account/status
Check eBay account connection status.

**Headers**: Requires authentication

**Response 200**:
```json
{
  "isLinked": true,
  "ebayAccountId": "ebay-user-123",
  "tokenStatus": "valid",
  "linkedAt": 1640995200,
  "lastSync": 1640995300
}
```

## eBay Wishlist Integration

### GET /ebay/wishlist
Get current user's eBay wishlist items that are currently active for bidding.

**Headers**: Requires authentication

**Response 200**:
```json
{
  "wishlistItems": [
    {
      "ebayItemId": "123456789",
      "title": "Vintage Watch Collection",
      "description": "Rare vintage watches...",
      "currentPrice": 15000,
      "endTime": 1640995200,
      "imageUrls": [
        "https://i.ebayimg.com/image1.jpg"
      ],
      "sellerInfo": {
        "sellerId": "seller123",
        "sellerName": "VintageDealer",
        "feedbackScore": 1500,
        "feedbackPercentage": 99.5
      },
      "categoryId": "jewelry-watches",
      "condition": "used",
      "location": "New York, NY",
      "shippingCost": 1500,
      "isActive": true,
      "addedToWishlistAt": 1640990000
    }
  ],
  "total": 15
}
```

### GET /ebay/wishlist/{itemId}
Get detailed information about a specific eBay wishlist item.

**Headers**: Requires authentication

**Path Parameters**:
- `itemId` (string): eBay item ID

**Response 200**:
```json
{
  "ebayItemId": "123456789",
  "title": "Vintage Watch Collection",
  "description": "Detailed description...",
  "currentPrice": 15000,
  "endTime": 1640995200,
  "imageUrls": [
    "https://i.ebayimg.com/image1.jpg"
  ],
  "sellerInfo": {
    "sellerId": "seller123",
    "sellerName": "VintageDealer",
    "feedbackScore": 1500,
    "feedbackPercentage": 99.5
  },
  "categoryId": "jewelry-watches",
  "condition": "used",
  "location": "New York, NY",
  "shippingCost": 1500,
  "isActive": true,
  "addedToWishlistAt": 1640990000
}
```

## Bid Management

### GET /bids
Get current user's bids with filtering options.

**Headers**: Requires authentication

**Query Parameters**:
- `status` (string, optional): Filter by status (PENDING, PLACED, WON, LOST, CANCELLED)
- `limit` (number, optional): Results limit (default: 20, max: 100)
- `offset` (number, optional): Results offset (default: 0)

**Response 200**:
```json
{
  "bids": [
    {
      "bidId": "bid-uuid-123",
      "userId": "user-uuid-456",
      "ebayItemId": "123456789",
      "maxBidAmount": 20000,
      "status": "PENDING",
      "auctionEndTime": 1640995200,
      "createdAt": 1640990000,
      "updatedAt": 1640990000,
      "schedulerJobId": "scheduler-job-123",
      "itemTitle": "Vintage Watch Collection",
      "itemImageUrl": "https://i.ebayimg.com/image1.jpg",
      "currentPrice": 15000
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

### POST /bids
Create a new bid for an eBay auction.

**Headers**: Requires authentication

**Request Body**:
```json
{
  "ebayItemId": "123456789",
  "maxBidAmount": 20000
}
```

**Response 201**:
```json
{
  "bidId": "bid-uuid-123",
  "userId": "user-uuid-456",
  "ebayItemId": "123456789",
  "maxBidAmount": 20000,
  "status": "PENDING",
  "auctionEndTime": 1640995200,
  "createdAt": 1640990000,
  "schedulerJobId": "scheduler-job-123",
  "itemTitle": "Vintage Watch Collection",
  "itemImageUrl": "https://i.ebayimg.com/image1.jpg",
  "currentPrice": 15000
}
```

### GET /bids/{bidId}
Get details of a specific bid.

**Headers**: Requires authentication

**Path Parameters**:
- `bidId` (string): Bid UUID

**Response 200**:
```json
{
  "bidId": "bid-uuid-123",
  "userId": "user-uuid-456",
  "ebayItemId": "123456789",
  "maxBidAmount": 20000,
  "status": "PENDING",
  "auctionEndTime": 1640995200,
  "createdAt": 1640990000,
  "updatedAt": 1640990000,
  "schedulerJobId": "scheduler-job-123",
  "itemTitle": "Vintage Watch Collection",
  "itemImageUrl": "https://i.ebayimg.com/image1.jpg",
  "currentPrice": 15000
}
```

### PUT /bids/{bidId}
Update an existing bid (only pending bids can be updated).

**Headers**: Requires authentication

**Path Parameters**:
- `bidId` (string): Bid UUID

**Request Body**:
```json
{
  "maxBidAmount": 25000
}
```

**Response 200**:
```json
{
  "bidId": "bid-uuid-123",
  "userId": "user-uuid-456",
  "ebayItemId": "123456789",
  "maxBidAmount": 25000,
  "status": "PENDING",
  "auctionEndTime": 1640995200,
  "updatedAt": 1640991000,
  "schedulerJobId": "scheduler-job-456",
  "itemTitle": "Vintage Watch Collection",
  "currentPrice": 15000
}
```

### DELETE /bids/{bidId}
Cancel a pending bid.

**Headers**: Requires authentication

**Path Parameters**:
- `bidId` (string): Bid UUID

**Response 204**: No content

## Bid History

### GET /bids/history
Get current user's bid history.

**Headers**: Requires authentication

**Query Parameters**:
- `limit` (number, optional): Results limit (default: 50, max: 100)
- `offset` (number, optional): Results offset (default: 0)

**Response 200**:
```json
{
  "history": [
    {
      "historyId": "history-uuid-123",
      "userId": "user-uuid-456",
      "originalBidId": "bid-uuid-123",
      "action": "CREATE",
      "timestamp": 1640990000,
      "details": {
        "originalAmount": 20000
      },
      "ebayItemId": "123456789",
      "bidAmount": 20000,
      "finalPrice": null
    },
    {
      "historyId": "history-uuid-124",
      "userId": "user-uuid-456",
      "originalBidId": "bid-uuid-123",
      "action": "UPDATE",
      "timestamp": 1640991000,
      "details": {
        "oldAmount": 20000,
        "newAmount": 25000
      },
      "ebayItemId": "123456789",
      "bidAmount": 25000,
      "finalPrice": null
    }
  ],
  "pagination": {
    "total": 15,
    "limit": 50,
    "offset": 0,
    "hasMore": false
  }
}
```

### GET /bids/{bidId}/history
Get history timeline for a specific bid.

**Headers**: Requires authentication

**Path Parameters**:
- `bidId` (string): Bid UUID

**Response 200**:
```json
{
  "bidId": "bid-uuid-123",
  "history": [
    {
      "historyId": "history-uuid-123",
      "action": "CREATE",
      "timestamp": 1640990000,
      "details": {
        "originalAmount": 20000
      },
      "bidAmount": 20000
    },
    {
      "historyId": "history-uuid-124",
      "action": "UPDATE",
      "timestamp": 1640991000,
      "details": {
        "oldAmount": 20000,
        "newAmount": 25000
      },
      "bidAmount": 25000
    }
  ]
}
```

## System Endpoints

### GET /health
Health check endpoint (public, no authentication required).

**Response 200**:
```json
{
  "status": "healthy",
  "timestamp": 1640995200,
  "version": "1.0.0",
  "services": {
    "database": "healthy",
    "ebay_api": "healthy",
    "email_service": "healthy"
  }
}
```

### GET /metrics
System metrics (requires admin authentication).

**Headers**: Requires admin authentication

**Response 200**:
```json
{
  "metrics": {
    "active_users": 1250,
    "pending_bids": 3420,
    "successful_bids_today": 45,
    "failed_bids_today": 3,
    "api_requests_per_minute": 85
  },
  "timestamp": 1640995200
}
```

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