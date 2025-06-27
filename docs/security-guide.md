# Security Guide

This document outlines the comprehensive security measures, best practices, and compliance requirements for the eBay Sniper application.

## Security Architecture Overview

The eBay Sniper application implements a defense-in-depth security strategy across multiple layers:

### Security Layers
1. **Network Security**: VPC, Security Groups, NACLs
2. **Identity & Access Management**: AWS Cognito, IAM roles
3. **Data Protection**: Encryption at rest and in transit
4. **Application Security**: Input validation, secure coding
5. **Monitoring & Logging**: CloudTrail, CloudWatch, alerting
6. **Compliance**: GDPR, SOC 2, security frameworks

## Authentication & Authorization

### AWS Cognito Configuration

#### User Pool Security Settings
```json
{
  "passwordPolicy": {
    "minimumLength": 12,
    "requireUppercase": true,
    "requireLowercase": true,
    "requireNumbers": true,
    "requireSymbols": true,
    "temporaryPasswordValidityDays": 7
  },
  "mfaConfiguration": "OPTIONAL",
  "enabledMfas": ["SOFTWARE_TOKEN_MFA"],
  "accountRecoverySetting": {
    "recoveryMechanisms": [
      {
        "name": "verified_email",
        "priority": 1
      }
    ]
  }
}
```

#### Token Configuration
```json
{
  "accessTokenValidity": 60,
  "idTokenValidity": 60,
  "refreshTokenValidity": 30,
  "tokenValidityUnits": {
    "accessToken": "minutes",
    "idToken": "minutes",
    "refreshToken": "days"
  }
}
```

### JWT Token Security

#### API Gateway Authorization
AWS API Gateway with Cognito Authorizer handles JWT token validation automatically:

```yaml
HttpApi:
  Type: AWS::Serverless::HttpApi
  Properties:
    Auth:
      Authorizers:
        CognitoAuthorizer:
          JwtConfiguration:
            issuer: !Sub "https://cognito-idp.${AWS::Region}.amazonaws.com/${UserPool}"
            audience:
              - !Ref UserPoolClient
          IdentitySource: "$request.header.Authorization"
```


### Role-Based Access Control

#### IAM Roles and Policies
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UserDataAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/ebay-sniper-*",
      "Condition": {
        "ForAllValues:StringEquals": {
          "dynamodb:LeadingKeys": ["${cognito-identity.amazonaws.com:sub}"]
        }
      }
    }
  ]
}
```

## Data Protection

### Encryption at Rest

#### DynamoDB Encryption
```yaml
UsersTable:
  Type: AWS::DynamoDB::Table
  Properties:
    SSESpecification:
      SSEEnabled: true
    PointInTimeRecoverySpecification:
      PointInTimeRecoveryEnabled: true
```

### Encryption in Transit

#### TLS Configuration
- **Minimum TLS Version**: TLS 1.2
- **API Gateway**: Enforced HTTPS only
- **Certificate Management**: AWS Certificate Manager
- **HSTS Headers**: Strict-Transport-Security enabled

#### Certificate Pinning
```typescript
// Frontend certificate pinning
const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  httpsAgent: new https.Agent({
    checkServerIdentity: (hostname, cert) => {
      // Verify certificate pinning
      const expectedFingerprint = process.env.NEXT_PUBLIC_CERT_FINGERPRINT;
      const actualFingerprint = cert.fingerprint256;
      
      if (expectedFingerprint !== actualFingerprint) {
        throw new Error('Certificate pinning validation failed');
      }
    }
  })
});
```

### Secrets Management

#### AWS Secrets Manager Integration
```python
import boto3
import json
from typing import Dict, Any

class SecretsManager:
    def __init__(self, region: str):
        self.client = boto3.client('secretsmanager', region_name=region)
    
    async def get_secret(self, secret_name: str) -> Dict[str, Any]:
        """Retrieve secret from AWS Secrets Manager"""
        try:
            response = self.client.get_secret_value(SecretId=secret_name)
            return json.loads(response['SecretString'])
        except ClientError as e:
            logger.error(f"Failed to retrieve secret {secret_name}: {str(e)}")
            raise
    
    async def update_secret(self, secret_name: str, secret_value: Dict[str, Any]) -> None:
        """Update secret in AWS Secrets Manager"""
        try:
            self.client.update_secret(
                SecretId=secret_name,
                SecretString=json.dumps(secret_value)
            )
        except ClientError as e:
            logger.error(f"Failed to update secret {secret_name}: {str(e)}")
            raise
```

## Application Security

### Input Validation

#### Consolidated Pydantic Validation
All validation is handled through Pydantic models with built-in FastAPI integration:

```python
from pydantic import BaseModel, Field, validator
from typing import Optional, Dict, Any
import re

class CreateBidRequest(BaseModel):
    """Single source of truth for bid creation validation"""
    ebay_item_id: str = Field(..., min_length=1, max_length=20, regex=r'^[0-9]+$')
    max_bid_amount: int = Field(..., ge=100, le=10000000, description="Amount in cents ($1.00 to $100,000)")
    
    class Config:
        schema_extra = {
            "example": {
                "ebay_item_id": "123456789",
                "max_bid_amount": 25000  # $250.00
            }
        }

class UpdateUserRequest(BaseModel):
    """Consolidated user update validation"""
    email: Optional[str] = Field(None, regex=r'^[^@]+@[^@]+\.[^@]+$')
    preferences: Optional[Dict[str, Any]] = None
    
    @validator('preferences')
    def validate_preferences(cls, v):
        if v is None:
            return v
        
        allowed_keys = {'emailNotifications', 'bidWinNotifications', 'bidLossNotifications', 'timezone'}
        if not set(v.keys()).issubset(allowed_keys):
            raise ValueError('Invalid preference keys')
        
        # Validate specific preference types
        for key, value in v.items():
            if key.endswith('Notifications') and not isinstance(value, bool):
                raise ValueError(f'{key} must be a boolean')
            if key == 'timezone' and not isinstance(value, str):
                raise ValueError('timezone must be a string')
        
        return v

class UserResponse(BaseModel):
    """Standardized user response model"""
    userId: str
    email: str
    createdAt: int
    updatedAt: int
    preferences: Dict[str, Any]
    isActive: bool

class BidResponse(BaseModel):
    """Standardized bid response model"""
    bidId: str
    userId: str
    ebayItemId: str
    maxBidAmount: int
    status: str
    auctionEndTime: int
    createdAt: int
    updatedAt: int
    itemTitle: Optional[str] = None
    itemImageUrl: Optional[str] = None
    currentPrice: Optional[int] = None
```

**Validation Strategy:**
- FastAPI automatically validates requests using Pydantic models
- No duplicate validation layers in service or repository code
- Business logic validation only for cross-entity rules (e.g., duplicate bids)
- DynamoDB operations use safe query patterns by default

#### SQL Injection Prevention
```python
# Using DynamoDB expressions to prevent injection
import boto3
from boto3.dynamodb.conditions import Key, Attr
from typing import Optional, List, Dict, Any

async def get_user_bids(user_id: str, status: Optional[str] = None) -> List[Dict[str, Any]]:
    """Safely query user bids with optional status filter"""
    try:
        dynamodb = boto3.resource('dynamodb')
        table = dynamodb.Table('ebay-sniper-Bids')
        
        key_condition = Key('userId').eq(user_id)
        
        if status:
            # Use attribute condition for filtering
            filter_condition = Attr('status').eq(status)
            response = table.query(
                KeyConditionExpression=key_condition,
                FilterExpression=filter_condition
            )
        else:
            response = table.query(KeyConditionExpression=key_condition)
        
        return response.get('Items', [])
        
    except Exception as e:
        logger.error(f"Error querying user bids: {str(e)}")
        raise
```

### XSS Prevention

#### Output Encoding
```python
import html
import json
from typing import Any, Dict

class ResponseEncoder:
    @staticmethod
    def encode_html(text: str) -> str:
        """Encode HTML entities to prevent XSS"""
        return html.escape(text, quote=True)
    
    @staticmethod
    def safe_json_response(data: Dict[str, Any]) -> str:
        """Create safe JSON response with proper encoding"""
        # Recursively encode string values
        encoded_data = ResponseEncoder._encode_dict(data)
        return json.dumps(encoded_data, ensure_ascii=True)
    
    @staticmethod
    def _encode_dict(obj: Any) -> Any:
        """Recursively encode dictionary values"""
        if isinstance(obj, dict):
            return {key: ResponseEncoder._encode_dict(value) for key, value in obj.items()}
        elif isinstance(obj, list):
            return [ResponseEncoder._encode_dict(item) for item in obj]
        elif isinstance(obj, str):
            return html.escape(obj, quote=True)
        else:
            return obj
```

#### Content Security Policy
```python
# FastAPI middleware for CSP headers
from fastapi import FastAPI, Request, Response
from fastapi.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # Content Security Policy
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; "
            "script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; "
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; "
            "font-src 'self' https://fonts.gstatic.com; "
            "img-src 'self' data: https:; "
            "connect-src 'self' https://api.ebay.com;"
        )
        
        # Other security headers
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        
        return response
```

### Rate Limiting

#### API Gateway Throttling
```yaml
HttpApi:
  Type: AWS::Serverless::HttpApi
  Properties:
    ThrottleConfig:
      BurstLimit: 100    # Max concurrent requests
      RateLimit: 50      # Requests per second
```

## Network Security

### VPC Configuration

#### Security Groups
```yaml
ApiSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Security group for API Lambda functions
    VpcId: !Ref VPC
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        CidrIp: 0.0.0.0/0
        Description: HTTPS traffic
    SecurityGroupEgress:
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        CidrIp: 0.0.0.0/0
        Description: Outbound HTTPS
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: 0.0.0.0/0
        Description: Outbound HTTP (for package downloads only)
```

#### Network ACLs
```yaml
PrivateNetworkAcl:
  Type: AWS::EC2::NetworkAcl
  Properties:
    VpcId: !Ref VPC
    Tags:
      - Key: Name
        Value: Private Network ACL

PrivateNetworkAclEntryInbound:
  Type: AWS::EC2::NetworkAclEntry
  Properties:
    NetworkAclId: !Ref PrivateNetworkAcl
    RuleNumber: 100
    Protocol: 6
    RuleAction: allow
    CidrBlock: 10.0.0.0/8
    PortRange:
      From: 443
      To: 443
```

### WAF Protection

#### Web Application Firewall Rules
```yaml
WebACL:
  Type: AWS::WAFv2::WebACL
  Properties:
    Name: !Sub "${AWS::StackName}-waf"
    Scope: REGIONAL
    DefaultAction:
      Allow: {}
    Rules:
      - Name: RateLimitRule
        Priority: 1
        Statement:
          RateBasedStatement:
            Limit: 2000
            AggregateKeyType: IP
        Action:
          Block: {}
        VisibilityConfig:
          SampledRequestsEnabled: true
          CloudWatchMetricsEnabled: true
          MetricName: RateLimitRule
      
      - Name: SQLInjectionRule
        Priority: 2
        Statement:
          SqliMatchStatement:
            FieldToMatch:
              Body: {}
            TextTransformations:
              - Priority: 0
                Type: URL_DECODE
              - Priority: 1
                Type: HTML_ENTITY_DECODE
        Action:
          Block: {}
        VisibilityConfig:
          SampledRequestsEnabled: true
          CloudWatchMetricsEnabled: true
          MetricName: SQLInjectionRule
```

## Monitoring & Logging

### Security Monitoring

#### CloudTrail Configuration
```yaml
CloudTrail:
  Type: AWS::CloudTrail::Trail
  Properties:
    TrailName: !Sub "${AWS::StackName}-security-trail"
    S3BucketName: !Ref SecurityLogsBucket
    S3KeyPrefix: "cloudtrail-logs/"
    IncludeGlobalServiceEvents: true
    IsMultiRegionTrail: true
    EnableLogFileValidation: true
    EventSelectors:
      - ReadWriteType: All
        IncludeManagementEvents: true
        DataResources:
          - Type: "AWS::DynamoDB::Table"
            Values: 
              - !Sub "${UsersTable}/*"
              - !Sub "${BidsTable}/*"
          - Type: "AWS::S3::Object"
            Values:
              - !Sub "${SecurityLogsBucket}/*"
```