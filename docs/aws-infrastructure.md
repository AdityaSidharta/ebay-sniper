# AWS Infrastructure Guide

This document outlines the AWS infrastructure architecture, configurations, and deployment specifications for the eBay Sniper application.

## Infrastructure Overview

The application uses a serverless architecture leveraging multiple AWS services for scalability, reliability, and cost-effectiveness.

### Core Services
- **AWS Lambda**: Backend API and scheduled functions
- **API Gateway**: HTTP API endpoint management
- **DynamoDB**: NoSQL database for user data and bids
- **Cognito**: User authentication and authorization
- **EventBridge**: Scheduled bid execution
- **Amplify**: Frontend hosting and deployment
- **Secrets Manager**: Secure credential storage
- **CloudWatch**: Logging and monitoring

Note: Email notifications are handled by **Postmark** (external service), not AWS SES.

## AWS SAM Template

### Main Template Structure
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Description: eBay Sniper Application Infrastructure

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
    Description: Deployment environment

  EbayAppId:
    Type: String
    Description: eBay API Application ID
    NoEcho: true

  PostmarkApiKey:
    Type: String
    Description: Postmark API Key for email notifications
    NoEcho: true

Globals:
  Function:
    Runtime: python3.11
    Timeout: 30
    MemorySize: 512
    Environment:
      Variables:
        ENVIRONMENT: !Ref Environment
        CORS_ORIGINS: !Sub "${AmplifyApp}.amplifyapp.com"
        POSTMARK_API_KEY: !Ref PostmarkApiKey
        EBAY_APP_ID: !Ref EbayAppId
    Layers:
      - !Ref DependenciesLayer
```

### DynamoDB Tables

#### Users Table
```yaml
UsersTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: !Sub "${AWS::StackName}-Users"
    BillingMode: PAY_PER_REQUEST
    AttributeDefinitions:
      - AttributeName: userId
        AttributeType: S
    KeySchema:
      - AttributeName: userId
        KeyType: HASH
    PointInTimeRecoverySpecification:
      PointInTimeRecoveryEnabled: true
    SSESpecification:
      SSEEnabled: true
    Tags:
      - Key: Environment
        Value: !Ref Environment
      - Key: Service
        Value: ebay-sniper
```

#### Bids Table
```yaml
BidsTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: !Sub "${AWS::StackName}-Bids"
    BillingMode: PAY_PER_REQUEST
    AttributeDefinitions:
      - AttributeName: userId
        AttributeType: S
      - AttributeName: bidId
        AttributeType: S
      - AttributeName: auctionEndTime
        AttributeType: N
    KeySchema:
      - AttributeName: userId
        KeyType: HASH
      - AttributeName: bidId
        KeyType: RANGE
    PointInTimeRecoverySpecification:
      PointInTimeRecoveryEnabled: true
    SSESpecification:
      SSEEnabled: true
    TimeToLiveSpecification:
      AttributeName: ttl
      Enabled: true
    Tags:
      - Key: Environment
        Value: !Ref Environment
      - Key: Service
        Value: ebay-sniper
```

#### BidHistory Table
```yaml
BidHistoryTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: !Sub "${AWS::StackName}-BidHistory"
    BillingMode: PAY_PER_REQUEST
    AttributeDefinitions:
      - AttributeName: userId
        AttributeType: S
      - AttributeName: historyId
        AttributeType: S
    KeySchema:
      - AttributeName: userId
        KeyType: HASH
      - AttributeName: historyId
        KeyType: RANGE
    PointInTimeRecoverySpecification:
      PointInTimeRecoveryEnabled: true
    SSESpecification:
      SSEEnabled: true
    TimeToLiveSpecification:
      AttributeName: ttl
      Enabled: true
    Tags:
      - Key: Environment
        Value: !Ref Environment
      - Key: Service
        Value: ebay-sniper
```

### Lambda Functions

The eBay Sniper application uses 8 individual Lambda functions, each responsible for a specific functionality. This microservices approach provides better scalability, maintainability, and cost optimization.

#### 1. User Management Function
```yaml
UserManagementFunction:
  Type: AWS::Serverless::Function
  Properties:
    FunctionName: !Sub "${AWS::StackName}-user-management"
    CodeUri: src/user_management/
    Handler: main.handler
    Description: Handle user profile CRUD operations and preferences
    Timeout: 30
    MemorySize: 512
    Environment:
      Variables:
        USERS_TABLE: !Ref UsersTable
        CORS_ORIGINS: !Sub "${AmplifyApp}.amplifyapp.com"
    Events:
      UserProfileGet:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /users/profile
          Method: GET
          Auth:
            Authorizer: CognitoAuthorizer
      UserProfileUpdate:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /users/profile
          Method: PUT
          Auth:
            Authorizer: CognitoAuthorizer
      UserPreferencesUpdate:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /users/preferences
          Method: PUT
          Auth:
            Authorizer: CognitoAuthorizer
      UserAccountDelete:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /users/account
          Method: DELETE
          Auth:
            Authorizer: CognitoAuthorizer
    Policies:
      - DynamoDBCrudPolicy:
          TableName: !Ref UsersTable
```

#### 2. eBay OAuth Function
```yaml
EbayOAuthFunction:
  Type: AWS::Serverless::Function
  Properties:
    FunctionName: !Sub "${AWS::StackName}-ebay-oauth"
    CodeUri: src/ebay_oauth/
    Handler: main.handler
    Description: Manage eBay account linking and OAuth flow
    Timeout: 30
    MemorySize: 512
    Environment:
      Variables:
        USERS_TABLE: !Ref UsersTable
        SECRET_ARN: !Ref EbayCredentials
        EBAY_APP_ID: !Ref EbayAppId
        EBAY_REDIRECT_URI: !Sub "https://${AmplifyApp}.amplifyapp.com/ebay/callback"
        EBAY_ENVIRONMENT: !Ref Environment
    Events:
      AuthUrl:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /ebay/auth/url
          Method: GET
          Auth:
            Authorizer: CognitoAuthorizer
      AuthCallback:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /ebay/auth/callback
          Method: POST
          Auth:
            Authorizer: CognitoAuthorizer
      AuthStatus:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /ebay/auth/status
          Method: GET
          Auth:
            Authorizer: CognitoAuthorizer
      AuthUnlink:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /ebay/auth/unlink
          Method: DELETE
          Auth:
            Authorizer: CognitoAuthorizer
    Policies:
      - DynamoDBCrudPolicy:
          TableName: !Ref UsersTable
      - Statement:
          Effect: Allow
          Action:
            - secretsmanager:GetSecretValue
          Resource: !Ref EbayCredentials
```

#### 3. Wishlist Sync Function
```yaml
WishlistSyncFunction:
  Type: AWS::Serverless::Function
  Properties:
    FunctionName: !Sub "${AWS::StackName}-wishlist-sync"
    CodeUri: src/wishlist_sync/
    Handler: main.handler
    Description: Retrieve and sync eBay wishlist items
    Timeout: 60
    MemorySize: 1024
    Environment:
      Variables:
        USERS_TABLE: !Ref UsersTable
        SECRET_ARN: !Ref EbayCredentials
        EBAY_ENVIRONMENT: !Ref Environment
    Events:
      WishlistItems:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /wishlist/items
          Method: GET
          Auth:
            Authorizer: CognitoAuthorizer
      WishlistItemDetails:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /wishlist/items/{itemId}
          Method: GET
          Auth:
            Authorizer: CognitoAuthorizer
    Policies:
      - DynamoDBReadPolicy:
          TableName: !Ref UsersTable
      - Statement:
          Effect: Allow
          Action:
            - secretsmanager:GetSecretValue
          Resource: !Ref EbayCredentials
```

#### 4. Bid Management Function
```yaml
BidManagementFunction:
  Type: AWS::Serverless::Function
  Properties:
    FunctionName: !Sub "${AWS::StackName}-bid-management"
    CodeUri: src/bid_management/
    Handler: main.handler
    Description: Handle bid CRUD operations and scheduling
    Timeout: 30
    MemorySize: 512
    Environment:
      Variables:
        BIDS_TABLE: !Ref BidsTable
        BID_HISTORY_TABLE: !Ref BidHistoryTable
        SCHEDULER_GROUP_NAME: !Ref ScheduleGroup
        BID_EXECUTOR_FUNCTION_ARN: !GetAtt BidExecutorFunction.Arn
    Events:
      CreateBid:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /bids
          Method: POST
          Auth:
            Authorizer: CognitoAuthorizer
      ListBids:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /bids
          Method: GET
          Auth:
            Authorizer: CognitoAuthorizer
      GetBid:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /bids/{bidId}
          Method: GET
          Auth:
            Authorizer: CognitoAuthorizer
      UpdateBid:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /bids/{bidId}
          Method: PUT
          Auth:
            Authorizer: CognitoAuthorizer
      CancelBid:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /bids/{bidId}
          Method: DELETE
          Auth:
            Authorizer: CognitoAuthorizer
    Policies:
      - DynamoDBCrudPolicy:
          TableName: !Ref BidsTable
      - DynamoDBWritePolicy:
          TableName: !Ref BidHistoryTable
      - Statement:
          Effect: Allow
          Action:
            - scheduler:CreateSchedule
            - scheduler:UpdateSchedule
            - scheduler:DeleteSchedule
          Resource: !Sub "arn:aws:scheduler:${AWS::Region}:${AWS::AccountId}:schedule/${ScheduleGroup}/*"
```

#### 5. Bid History Function
```yaml
BidHistoryFunction:
  Type: AWS::Serverless::Function
  Properties:
    FunctionName: !Sub "${AWS::StackName}-bid-history"
    CodeUri: src/bid_history/
    Handler: main.handler
    Description: Query and retrieve historical bid data
    Timeout: 30
    MemorySize: 512
    Environment:
      Variables:
        BID_HISTORY_TABLE: !Ref BidHistoryTable
    Events:
      BidHistory:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /bid-history
          Method: GET
          Auth:
            Authorizer: CognitoAuthorizer
      SpecificHistory:
        Type: HttpApi
        Properties:
          ApiId: !Ref HttpApi
          Path: /bid-history/{bidId}
          Method: GET
          Auth:
            Authorizer: CognitoAuthorizer
    Policies:
      - DynamoDBReadPolicy:
          TableName: !Ref BidHistoryTable
```

#### 6. Bid Executor Function
```yaml
BidExecutorFunction:
  Type: AWS::Serverless::Function
  Properties:
    FunctionName: !Sub "${AWS::StackName}-bid-executor"
    CodeUri: src/bid_executor/
    Handler: main.handler
    Description: Execute scheduled bids, wait for auction completion, and determine bid outcome
    Timeout: 300
    MemorySize: 1024
    Environment:
      Variables:
        BIDS_TABLE: !Ref BidsTable
        BID_HISTORY_TABLE: !Ref BidHistoryTable
        USERS_TABLE: !Ref UsersTable
        SECRET_ARN: !Ref EbayCredentials
        NOTIFICATION_FUNCTION_NAME: !Ref NotificationHandlerFunction
    Policies:
      - DynamoDBCrudPolicy:
          TableName: !Ref BidsTable
      - DynamoDBReadPolicy:
          TableName: !Ref UsersTable
      - DynamoDBWritePolicy:
          TableName: !Ref BidHistoryTable
      - Statement:
          Effect: Allow
          Action:
            - secretsmanager:GetSecretValue
          Resource: !Ref EbayCredentials
      - Statement:
          Effect: Allow
          Action:
            - lambda:InvokeFunction
          Resource: !GetAtt NotificationHandlerFunction.Arn
```

#### 7. Notification Handler Function
```yaml
NotificationHandlerFunction:
  Type: AWS::Serverless::Function
  Properties:
    FunctionName: !Sub "${AWS::StackName}-notification-handler"
    CodeUri: src/notification_handler/
    Handler: main.handler
    Description: Send email notifications when invoked by Bid Executor
    Timeout: 30
    MemorySize: 512
    Environment:
      Variables:
        USERS_TABLE: !Ref UsersTable
        SECRET_ARN: !Ref EbayCredentials
    Policies:
      - DynamoDBReadPolicy:
          TableName: !Ref UsersTable
      - Statement:
          Effect: Allow
          Action:
            - secretsmanager:GetSecretValue
          Resource: !Ref EbayCredentials
```

#### 8. Token Refresh Function
```yaml
TokenRefreshFunction:
  Type: AWS::Serverless::Function
  Properties:
    FunctionName: !Sub "${AWS::StackName}-token-refresh"
    CodeUri: src/token_refresh/
    Handler: main.handler
    Description: Automated renewal of eBay OAuth tokens
    Timeout: 120
    MemorySize: 512
    Environment:
      Variables:
        USERS_TABLE: !Ref UsersTable
        SECRET_ARN: !Ref EbayCredentials
        TOKEN_EXPIRY_THRESHOLD: "7"
    Events:
      ScheduledRefresh:
        Type: ScheduleV2
        Properties:
          ScheduleExpression: "rate(1 day)"
          Name: !Sub "${AWS::StackName}-token-refresh-schedule"
          Description: "Check and refresh tokens daily"
    Policies:
      - DynamoDBCrudPolicy:
          TableName: !Ref UsersTable
      - Statement:
          Effect: Allow
          Action:
            - secretsmanager:GetSecretValue
          Resource: !Ref EbayCredentials
```

### API Gateway Configuration

#### HTTP API
```yaml
HttpApi:
  Type: AWS::Serverless::HttpApi
  Properties:
    Name: !Sub "${AWS::StackName}-api"
    Description: eBay Sniper HTTP API
    CorsConfiguration:
      AllowOrigins:
        - !Sub "https://${AmplifyApp}.amplifyapp.com"
        - "http://localhost:3000"
      AllowMethods:
        - GET
        - POST
        - PUT
        - DELETE
        - OPTIONS
      AllowHeaders:
        - Content-Type
        - Authorization
        - X-Amz-Date
        - X-Api-Key
      MaxAge: 86400
    Auth:
      Authorizers:
        CognitoAuthorizer:
          JwtConfiguration:
            issuer: !Sub "https://cognito-idp.${AWS::Region}.amazonaws.com/${UserPool}"
            audience:
              - !Ref UserPoolClient
          IdentitySource: "$request.header.Authorization"
    ThrottleConfig:
      BurstLimit: 100
      RateLimit: 50
```

### Cognito Configuration

#### User Pool
```yaml
UserPool:
  Type: AWS::Cognito::UserPool
  Properties:
    UserPoolName: !Sub "${AWS::StackName}-users"
    AliasAttributes:
      - email
    AutoVerifiedAttributes:
      - email
    Policies:
      PasswordPolicy:
        MinimumLength: 12
        RequireUppercase: true
        RequireLowercase: true
        RequireNumbers: true
        RequireSymbols: true
        TemporaryPasswordValidityDays: 7
    MfaConfiguration: OPTIONAL
    EnabledMfas:
      - SOFTWARE_TOKEN_MFA
    Schema:
      - Name: email
        AttributeDataType: String
        Mutable: true
        Required: true
      - Name: name
        AttributeDataType: String
        Mutable: true
        Required: false
    UserPoolTags:
      Environment: !Ref Environment
      Service: ebay-sniper
    AccountRecoverySetting:
      RecoveryMechanisms:
        - Name: verified_email
          Priority: 1
```

#### User Pool Client
```yaml
UserPoolClient:
  Type: AWS::Cognito::UserPoolClient
  Properties:
    UserPoolId: !Ref UserPool
    ClientName: !Sub "${AWS::StackName}-web-client"
    GenerateSecret: false
    ExplicitAuthFlows:
      - ALLOW_USER_SRP_AUTH
      - ALLOW_REFRESH_TOKEN_AUTH
    SupportedIdentityProviders:
      - COGNITO
    RefreshTokenValidity: 30
    RefreshTokenValidityUnits:
      RefreshToken: days
    AccessTokenValidity: 60
    AccessTokenValidityUnits:
      AccessToken: minutes
    IdTokenValidity: 60
    IdTokenValidityUnits:
      IdToken: minutes
    TokenValidityUnits:
      AccessToken: minutes
      IdToken: minutes
      RefreshToken: days
    ReadAttributes:
      - email
      - name
    WriteAttributes:
      - email
      - name
```

### EventBridge Scheduling

#### Schedule Group
```yaml
ScheduleGroup:
  Type: AWS::Scheduler::ScheduleGroup
  Properties:
    Name: !Sub "${AWS::StackName}-bid-schedules"
    Tags:
      - Key: Environment
        Value: !Ref Environment
      - Key: Service
        Value: ebay-sniper
```

#### Execution Role for Scheduler
```yaml
SchedulerExecutionRole:
  Type: AWS::IAM::Role
  Properties:
    RoleName: !Sub "${AWS::StackName}-scheduler-execution-role"
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            Service: scheduler.amazonaws.com
          Action: sts:AssumeRole
    Policies:
      - PolicyName: LambdaInvoke
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - lambda:InvokeFunction
              Resource: !GetAtt BidExecutorFunction.Arn
```

### Security Configuration

#### Secrets Manager
```yaml
EbayCredentials:
  Type: AWS::SecretsManager::Secret
  Properties:
    Name: !Sub "${AWS::StackName}/ebay-credentials"
    Description: eBay API credentials
    SecretString: !Sub |
      {
        "app_id": "${EbayAppId}",
        "dev_id": "REPLACE_WITH_DEV_ID",
        "cert_id": "REPLACE_WITH_CERT_ID",
        "environment": "${Environment}"
      }
    Tags:
      - Key: Environment
        Value: !Ref Environment
      - Key: Service
        Value: ebay-sniper
```

### Lambda Layers

#### Dependencies Layer
```yaml
DependenciesLayer:
  Type: AWS::Serverless::LayerVersion
  Properties:
    LayerName: !Sub "${AWS::StackName}-dependencies"
    Description: Python dependencies for eBay Sniper application
    ContentUri: layers/dependencies/
    CompatibleRuntimes:
      - python3.11
    RetentionPolicy: Delete
  Metadata:
    BuildMethod: python3.11
```

### CloudWatch Configuration

#### Log Groups
```yaml
ApiLogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    LogGroupName: !Sub "/aws/lambda/${ApiFunction}"
    RetentionInDays: 30

BidExecutorLogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    LogGroupName: !Sub "/aws/lambda/${BidExecutorFunction}"
    RetentionInDays: 30
```

#### CloudWatch Alarms
```yaml
ApiErrorAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub "${AWS::StackName}-api-errors"
    AlarmDescription: API function error rate alarm
    MetricName: Errors
    Namespace: AWS/Lambda
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 2
    Threshold: 5
    ComparisonOperator: GreaterThanThreshold
    Dimensions:
      - Name: FunctionName
        Value: !Ref ApiFunction
    TreatMissingData: notBreaching

BidExecutorFailureAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub "${AWS::StackName}-bid-executor-failures"
    AlarmDescription: Bid executor failure alarm
    MetricName: Errors
    Namespace: AWS/Lambda
    Statistic: Sum
    Period: 60
    EvaluationPeriods: 1
    Threshold: 1
    ComparisonOperator: GreaterThanOrEqualToThreshold
    Dimensions:
      - Name: FunctionName
        Value: !Ref BidExecutorFunction
    TreatMissingData: notBreaching
```

### Amplify Configuration

#### Amplify App
```yaml
AmplifyApp:
  Type: AWS::Amplify::App
  Properties:
    Name: !Sub "${AWS::StackName}-frontend"
    Description: eBay Sniper Frontend Application
    Repository: https://github.com/your-org/ebay-sniper-frontend
    BuildSpec: |
      version: 1
      frontend:
        phases:
          preBuild:
            commands:
              - pnpm install
          build:
            commands:
              - pnpm run build
        artifacts:
          baseDirectory: .next
          files:
            - '**/*'
        cache:
          paths:
            - node_modules/**/*
            - .next/cache/**/*
    EnvironmentVariables:
      - Name: NEXT_PUBLIC_API_URL
        Value: !Sub "https://${HttpApi}.execute-api.${AWS::Region}.amazonaws.com"
      - Name: NEXT_PUBLIC_USER_POOL_ID
        Value: !Ref UserPool
      - Name: NEXT_PUBLIC_USER_POOL_CLIENT_ID
        Value: !Ref UserPoolClient
      - Name: NEXT_PUBLIC_AWS_REGION
        Value: !Ref AWS::Region
    Tags:
      - Key: Environment
        Value: !Ref Environment
      - Key: Service
        Value: ebay-sniper

AmplifyBranch:
  Type: AWS::Amplify::Branch
  Properties:
    AppId: !GetAtt AmplifyApp.AppId
    BranchName: !If
      - IsProduction
      - main
      - !Ref Environment
    EnableAutoBuild: true
    EnvironmentVariables:
      - Name: AMPLIFY_DIFF_DEPLOY
        Value: false
```

## Resource Naming Convention

### Naming Pattern
- **Stack Prefix**: `ebay-sniper-{environment}`
- **Resources**: `{stack-name}-{resource-type}`
- **Functions**: `{stack-name}-{function-purpose}`
- **Tables**: `{stack-name}-{table-name}`

### Examples
- Stack: `ebay-sniper-prod`
- API Function: `ebay-sniper-prod-api`
- Users Table: `ebay-sniper-prod-Users`

## Environment-Specific Configuration

### Development Environment
```yaml
Conditions:
  IsDevelopment: !Equals [!Ref Environment, "dev"]
  IsStaging: !Equals [!Ref Environment, "staging"]
  IsProduction: !Equals [!Ref Environment, "prod"]

# Development-specific overrides
ApiFunction:
  Properties:
    Environment:
      Variables:
        DEBUG: !If [IsDevelopment, "true", "false"]
        LOG_LEVEL: !If [IsDevelopment, "DEBUG", "INFO"]
```

### Production Environment
```yaml
# Production-specific configurations
ApiFunction:
  Properties:
    ReservedConcurrencyLimit: !If [IsProduction, 100, !Ref "AWS::NoValue"]
    ProvisionedConcurrencyConfig: !If
      - IsProduction
      - ProvisionedConcurrencyConfigs:
          - ProvisionedConcurrency: 10
      - !Ref "AWS::NoValue"

DynamoDBTable:
  Properties:
    BillingMode: !If [IsProduction, "PROVISIONED", "PAY_PER_REQUEST"]
    ProvisionedThroughput: !If
      - IsProduction
      - ReadCapacityUnits: 20
        WriteCapacityUnits: 20
      - !Ref "AWS::NoValue"
```

## Deployment Commands

### Build and Deploy
```bash
# Build SAM application
sam build

# Deploy to development
sam deploy --parameter-overrides Environment=dev --config-env dev

# Deploy to staging
sam deploy --parameter-overrides Environment=staging --config-env staging

# Deploy to production
sam deploy --parameter-overrides Environment=prod --config-env prod
```

### Configuration Files

#### samconfig.toml
```toml
version = 0.1
[default]
[default.deploy]
[default.deploy.parameters]
stack_name = "ebay-sniper"
s3_bucket = "your-deployment-bucket"
s3_prefix = "ebay-sniper"
region = "us-east-1"
confirm_changeset = true
capabilities = "CAPABILITY_IAM"

[dev]
[dev.deploy]
[dev.deploy.parameters]
stack_name = "ebay-sniper-dev"
parameter_overrides = "Environment=dev"

[prod]
[prod.deploy]
[prod.deploy.parameters]
stack_name = "ebay-sniper-prod"
parameter_overrides = "Environment=prod"
```

## Monitoring and Observability

### CloudWatch Dashboards
- API performance metrics
- DynamoDB read/write capacity
- Lambda function performance
- Error rates and patterns
- Cost optimization metrics

### X-Ray Tracing
- Enable X-Ray tracing for all Lambda functions
- Trace eBay API calls and database operations
- Monitor request latency and bottlenecks

### Log Aggregation
- Structured logging with JSON format
- Correlation IDs for request tracking
- Log retention policies (30 days default)
- Search and alerting capabilities

## Security Best Practices

### IAM Policies
- Principle of least privilege
- Service-specific roles
- Resource-level permissions
- Regular policy reviews

### Data Encryption
- Encryption at rest (DynamoDB automatic encryption)
- Encryption in transit (TLS 1.2+)
- AWS Secrets Manager handles credential encryption
- Secrets management

### Network Security
- VPC endpoints for AWS services
- Security groups and NACLs
- API Gateway throttling
- WAF protection

## Cost Optimization

### Resource Sizing
- Right-size Lambda memory allocation
- Monitor DynamoDB capacity utilization
- Use reserved capacity for predictable workloads
- Implement auto-scaling policies

### Cost Monitoring
- AWS Cost Explorer integration
- Budget alerts and notifications
- Resource tagging for cost allocation
- Regular cost optimization reviews

## Backup and Recovery

### Data Backup
- DynamoDB point-in-time recovery
- Cross-region backup for critical data
- Automated backup scheduling
- Backup retention policies

### Disaster Recovery
- Multi-AZ deployment
- Cross-region failover capabilities
- Recovery time objectives (RTO): 15 minutes
- Recovery point objectives (RPO): 1 hour

## Compliance and Governance

### Compliance Requirements
- SOC 2 Type II compliance
- GDPR data protection requirements
- PCI DSS for payment data (if applicable)
- Regular security assessments

### Governance Policies
- Resource tagging standards
- Change management procedures
- Access control policies
- Audit logging requirements