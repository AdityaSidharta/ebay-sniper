
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
    
    %% Backend Layer
    Lambda[AWS Lambda<br/>FastAPI + Python]
    
    %% Data Layer
    DynamoDB[Amazon DynamoDB<br/>User Data, Bids, BidHistory]
    
    %% External Services
    eBayAPI[eBay API<br/>Wishlist, Bidding, Auctions]
    Postmark[Postmark<br/>Email Notifications]
    
    %% Scheduled Functions
    Scheduler[EventBridge/CloudWatch<br/>Scheduled Bidding Jobs]
    BidExecutor[Lambda Function<br/>Execute Bids 5s Before Close]
    
    %% User interactions
    User --> Frontend
    Frontend --> Cognito
    Frontend --> APIGateway
    
    %% API Gateway routes to Lambda
    APIGateway --> Lambda
    
    %% Lambda interactions
    Lambda --> DynamoDB
    Lambda --> eBayAPI
    Lambda --> Postmark
    Lambda --> Scheduler
    
    %% Scheduled bidding flow
    Scheduler --> BidExecutor
    BidExecutor --> eBayAPI
    BidExecutor --> DynamoDB
    BidExecutor --> Postmark
    
    %% Authentication flow
    Cognito -.->|JWT Token| APIGateway
    
    %% Styling
    classDef frontend fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    classDef aws fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#000
    classDef external fill:#e8f5e8,stroke:#388e3c,stroke-width:2px,color:#000
    classDef data fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#000
    classDef scheduler fill:#fff8e1,stroke:#f9a825,stroke-width:2px,color:#000
    
    class Frontend frontend
    class Cognito,APIGateway,Lambda aws
    class DynamoDB data
    class eBayAPI,Postmark external
    class Scheduler,BidExecutor scheduler
```

## Key Data Flows

### User Management
1. **User Registration/Login**: User → Frontend → Cognito → Frontend
2. **eBay Account Linking**: Frontend → API Gateway → Lambda → eBay API → DynamoDB
3. **User Preferences**: Frontend → API Gateway → Lambda → DynamoDB → Frontend

### Wishlist & Bidding
1. **Wishlist Item Retrieval**: Frontend → API Gateway → Lambda → eBay API → Frontend
2. **Create Bid**: Frontend → API Gateway → Lambda → DynamoDB → Scheduler
3. **View Active Bids**: Frontend → API Gateway → Lambda → DynamoDB → Frontend
4. **Update Bid Price**: Frontend → API Gateway → Lambda → DynamoDB → Scheduler (update job)
5. **Cancel Bid**: Frontend → API Gateway → Lambda → DynamoDB → Scheduler (remove job)

### History & Monitoring
1. **View Bid History**: Frontend → API Gateway → Lambda → DynamoDB → Frontend

### Automated Execution
1. **Scheduled Bidding**: Scheduler → BidExecutor → eBay API → DynamoDB → Postmark
