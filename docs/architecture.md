## Architecture Diagram

```mermaid
graph TB
    %% User Layer
    User[User]
    
    %% Frontend Layer
    Frontend[Next.js Frontend<br/>TypeScript + Tailwind<br/>Hosted on AWS Amplify]
    
    %% Authentication
    Cognito[AWS Cognito<br/>User Authentication]
    
    %% API Layer
    APIGateway[AWS API Gateway<br/>REST/HTTP API]
    
    %% Backend Layer - Individual Lambda Functions
    UserMgmtLambda[User Management<br/>Lambda Function]
    EbayOAuthLambda[eBay OAuth<br/>Lambda Function]
    WishlistSyncLambda[Wishlist Sync<br/>Lambda Function]
    BidMgmtLambda[Bid Management<br/>Lambda Function]
    BidHistoryLambda[Bid History<br/>Lambda Function]
    BidExecutorLambda[Bid Executor<br/>Lambda Function]
    NotificationLambda[Notification Handler<br/>Lambda Function]
    TokenRefreshLambda[Token Refresh<br/>Lambda Function]
    
    %% Data Layer
    UsersTable[Users Table<br/>DynamoDB]
    BidsTable[Bids Table<br/>DynamoDB]
    BidHistoryTable[BidHistory Table<br/>DynamoDB]
    
    %% External Services
    eBayAPI[eBay API<br/>Wishlist, Bidding, Auctions]
    Postmark[Postmark<br/>Email Notifications]
    
    %% Scheduled Functions
    Scheduler[EventBridge/CloudWatch<br/>Scheduled Bidding Jobs]
    
    %% User interactions
    User --> Frontend
    Frontend --> Cognito
    Frontend --> APIGateway
    
    %% API Gateway routes to individual Lambda functions
    APIGateway --> UserMgmtLambda
    APIGateway --> EbayOAuthLambda
    APIGateway --> WishlistSyncLambda
    APIGateway --> BidMgmtLambda
    APIGateway --> BidHistoryLambda
    
    %% Individual Lambda function interactions
    UserMgmtLambda --> UsersTable
    EbayOAuthLambda --> UsersTable
    EbayOAuthLambda --> eBayAPI
    WishlistSyncLambda --> UsersTable
    WishlistSyncLambda --> eBayAPI
    BidMgmtLambda --> BidsTable
    BidMgmtLambda --> BidHistoryTable
    BidMgmtLambda --> Scheduler
    BidHistoryLambda --> BidHistoryTable
    BidExecutorLambda --> BidsTable
    BidExecutorLambda --> BidHistoryTable
    BidExecutorLambda --> UsersTable
    BidExecutorLambda --> eBayAPI
    BidExecutorLambda --> Postmark
    NotificationLambda --> UsersTable
    NotificationLambda --> Postmark
    TokenRefreshLambda --> UsersTable
    TokenRefreshLambda --> eBayAPI
    
    %% Scheduled bidding flow
    Scheduler --> BidExecutorLambda
    Scheduler --> TokenRefreshLambda
    
    %% Authentication flow
    Cognito -.->|JWT Token| APIGateway
    
    %% Styling
    classDef frontend fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    classDef aws fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#000
    classDef lambda fill:#ffecb3,stroke:#ff8f00,stroke-width:2px,color:#000
    classDef external fill:#e8f5e8,stroke:#388e3c,stroke-width:2px,color:#000
    classDef data fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#000
    classDef scheduler fill:#fff8e1,stroke:#f9a825,stroke-width:2px,color:#000
    
    class Frontend frontend
    class Cognito,APIGateway aws
    class UserMgmtLambda,EbayOAuthLambda,WishlistSyncLambda,BidMgmtLambda,BidHistoryLambda lambda
    class BidExecutorLambda,NotificationLambda,TokenRefreshLambda lambda
    class UsersTable,BidsTable,BidHistoryTable data
    class eBayAPI,Postmark external
    class Scheduler scheduler
```

## Key Data Flows

### User Management
1. **User Registration/Login**: User → Frontend → Cognito → Frontend
2. **Get User Profile**: Frontend → API Gateway → User Management Lambda → DynamoDB → Frontend
3. **Update User Profile**: Frontend → API Gateway → User Management Lambda → DynamoDB → Frontend
4. **Update Preferences**: Frontend → API Gateway → User Management Lambda → DynamoDB → Frontend

### eBay Integration
1. **eBay Account Linking**: Frontend → API Gateway → eBay OAuth Lambda → eBay API → DynamoDB → Frontend
2. **OAuth Callback**: eBay → Frontend → API Gateway → eBay OAuth Lambda → DynamoDB
3. **Check Link Status**: Frontend → API Gateway → eBay OAuth Lambda → DynamoDB → Frontend
4. **Unlink Account**: Frontend → API Gateway → eBay OAuth Lambda → DynamoDB → Frontend

### Wishlist Management
1. **Sync Wishlist**: Frontend → API Gateway → Wishlist Sync Lambda → eBay API → Frontend
2. **Get Wishlist Items**: Frontend → API Gateway → Wishlist Sync Lambda → DynamoDB → Frontend
3. **Get Item Details**: Frontend → API Gateway → Wishlist Sync Lambda → eBay API → Frontend

### Bid Management
1. **Create Bid**: Frontend → API Gateway → Bid Management Lambda → DynamoDB → EventBridge Scheduler
2. **View Active Bids**: Frontend → API Gateway → Bid Management Lambda → DynamoDB → Frontend
3. **Update Bid Price**: Frontend → API Gateway → Bid Management Lambda → DynamoDB → EventBridge Scheduler (update)
4. **Cancel Bid**: Frontend → API Gateway → Bid Management Lambda → DynamoDB → EventBridge Scheduler (remove)
5. **Get Specific Bid**: Frontend → API Gateway → Bid Management Lambda → DynamoDB → Frontend

### Bid History & Analytics
1. **View Bid History**: Frontend → API Gateway → Bid History Lambda → DynamoDB → Frontend
2. **Get Bid Statistics**: Frontend → API Gateway → Bid History Lambda → DynamoDB → Frontend
3. **Get Specific History**: Frontend → API Gateway → Bid History Lambda → DynamoDB → Frontend

### Automated Execution
1. **Scheduled Bidding**: EventBridge Scheduler → Bid Executor Lambda → eBay API → DynamoDB → Notification Handler Lambda → Postmark
2. **Token Refresh**: EventBridge Scheduler → Token Refresh Lambda → eBay API → DynamoDB