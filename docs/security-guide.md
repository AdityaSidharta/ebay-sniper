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

#### Token Validation Implementation
```python
import jwt
from datetime import datetime, timedelta
from typing import Optional, Dict, Any

class TokenValidator:
    def __init__(self, user_pool_id: str, region: str):
        self.user_pool_id = user_pool_id
        self.region = region
        self.jwks_url = f"https://cognito-idp.{region}.amazonaws.com/{user_pool_id}/.well-known/jwks.json"
    
    async def validate_token(self, token: str) -> Optional[Dict[str, Any]]:
        """Validate JWT token and return payload if valid"""
        try:
            # Decode header to get key ID
            header = jwt.get_unverified_header(token)
            key_id = header.get('kid')
            
            # Get signing key from JWKS
            signing_key = await self.get_signing_key(key_id)
            
            # Validate token
            payload = jwt.decode(
                token,
                signing_key,
                algorithms=['RS256'],
                audience=self.user_pool_client_id,
                issuer=f"https://cognito-idp.{self.region}.amazonaws.com/{self.user_pool_id}"
            )
            
            # Additional validation
            if payload.get('token_use') != 'access':
                raise jwt.InvalidTokenError('Invalid token use')
            
            if datetime.fromtimestamp(payload.get('exp', 0)) <= datetime.utcnow():
                raise jwt.ExpiredSignatureError('Token has expired')
            
            return payload
            
        except jwt.ExpiredSignatureError:
            logger.warning("Token has expired")
            return None
        except jwt.InvalidTokenError as e:
            logger.warning(f"Invalid token: {str(e)}")
            return None
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
      # Uses DynamoDB automatic encryption at rest via KMS
      # No custom KMS key required - AWS manages encryption automatically
    PointInTimeRecoverySpecification:
      PointInTimeRecoveryEnabled: true
```

#### S3 Encryption
```yaml
S3Bucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketEncryption:
      ServerSideEncryptionConfiguration:
        - ServerSideEncryptionByDefault:
            SSEAlgorithm: aws:kms
            KMSMasterKeyID: !Ref EncryptionKey
          BucketKeyEnabled: true
```

#### Data Storage Security
```python
import boto3
import json
from typing import Dict, Any

class SecureDataHandler:
    def __init__(self):
        self.dynamodb = boto3.resource('dynamodb')
        # DynamoDB handles encryption at rest automatically via KMS
        # No manual encryption/decryption needed for stored data
    
    def store_user_data(self, table_name: str, user_data: Dict[str, Any]) -> None:
        """Store user data - automatically encrypted by DynamoDB"""
        table = self.dynamodb.Table(table_name)
        table.put_item(Item=user_data)
        # Data is automatically encrypted at rest by DynamoDB
    
    def retrieve_user_data(self, table_name: str, user_id: str) -> Dict[str, Any]:
        """Retrieve user data - automatically decrypted by DynamoDB"""
        table = self.dynamodb.Table(table_name)
        response = table.get_item(Key={'userId': user_id})
        # Data is automatically decrypted when retrieved
        return response.get('Item', {})
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

#### Secret Rotation
```python
class SecretRotator:
    def __init__(self, secrets_manager: SecretsManager):
        self.secrets_manager = secrets_manager
    
    async def rotate_ebay_tokens(self, user_id: str) -> None:
        """Rotate eBay OAuth tokens before expiration"""
        try:
            # Get current tokens
            secret_name = f"ebay-tokens/{user_id}"
            tokens = await self.secrets_manager.get_secret(secret_name)
            
            # Refresh access token using refresh token
            new_tokens = await self.ebay_client.refresh_tokens(tokens['refresh_token'])
            
            # Update stored tokens
            await self.secrets_manager.update_secret(secret_name, new_tokens)
            
            logger.info(f"Successfully rotated tokens for user {user_id}")
            
        except Exception as e:
            logger.error(f"Failed to rotate tokens for user {user_id}: {str(e)}")
            raise

# Note: eBay OAuth tokens are stored in DynamoDB user records and are
# automatically encrypted at rest via DynamoDB's encryption features.
# No manual encryption/decryption is required in the application code.
```

## Application Security

### Input Validation

#### Pydantic Models for Validation
```python
from pydantic import BaseModel, validator, Field
from typing import Optional
import re

class CreateBidRequest(BaseModel):
    ebay_item_id: str = Field(..., min_length=1, max_length=20)
    max_bid_amount: int = Field(..., ge=100, le=10000000)  # $1.00 to $100,000
    
    @validator('ebay_item_id')
    def validate_ebay_item_id(cls, v):
        if not re.match(r'^[0-9]+$', v):
            raise ValueError('eBay item ID must contain only digits')
        return v
    
    @validator('max_bid_amount')
    def validate_bid_amount(cls, v):
        if v <= 0:
            raise ValueError('Bid amount must be positive')
        return v

class UpdateUserRequest(BaseModel):
    email: Optional[str] = Field(None, regex=r'^[^@]+@[^@]+\.[^@]+$')
    preferences: Optional[dict] = None
    
    @validator('preferences')
    def validate_preferences(cls, v):
        if v is None:
            return v
        
        allowed_keys = {'emailNotifications', 'bidWinNotifications', 'bidLossNotifications', 'timezone'}
        if not set(v.keys()).issubset(allowed_keys):
            raise ValueError('Invalid preference keys')
        
        return v
```

#### SQL Injection Prevention
```python
# Using DynamoDB expressions to prevent injection
from boto3.dynamodb.conditions import Key, Attr

class UserRepository:
    def __init__(self, table):
        self.table = table
    
    async def get_user_bids(self, user_id: str, status: Optional[str] = None):
        """Safely query user bids with optional status filter"""
        try:
            key_condition = Key('userId').eq(user_id)
            
            if status:
                # Use attribute condition for filtering
                filter_condition = Attr('status').eq(status)
                response = self.table.query(
                    KeyConditionExpression=key_condition,
                    FilterExpression=filter_condition
                )
            else:
                response = self.table.query(KeyConditionExpression=key_condition)
            
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

#### Application-Level Rate Limiting
```python
import time
import redis
from typing import Optional

class RateLimiter:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
    
    async def check_rate_limit(
        self, 
        identifier: str, 
        limit: int, 
        window: int
    ) -> tuple[bool, Optional[int]]:
        """
        Check if request is within rate limit
        Returns (allowed, retry_after_seconds)
        """
        key = f"rate_limit:{identifier}"
        current_time = int(time.time())
        window_start = current_time - window
        
        # Use sliding window counter
        pipe = self.redis.pipeline()
        pipe.zremrangebyscore(key, 0, window_start)
        pipe.zcard(key)
        pipe.zadd(key, {str(current_time): current_time})
        pipe.expire(key, window)
        
        results = pipe.execute()
        current_requests = results[1]
        
        if current_requests >= limit:
            # Calculate retry after
            oldest_request = self.redis.zrange(key, 0, 0, withscores=True)
            if oldest_request:
                retry_after = int(oldest_request[0][1]) + window - current_time
                return False, max(0, retry_after)
            return False, window
        
        return True, None
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

#### Security Alerts
```python
import boto3
import json
from typing import Dict, Any

class SecurityMonitor:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        self.sns = boto3.client('sns')
    
    async def create_security_alarm(
        self, 
        alarm_name: str, 
        log_group: str, 
        filter_pattern: str,
        threshold: int = 5
    ) -> None:
        """Create CloudWatch alarm for security events"""
        
        # Create metric filter
        logs_client = boto3.client('logs')
        logs_client.put_metric_filter(
            logGroupName=log_group,
            filterName=f"{alarm_name}-filter",
            filterPattern=filter_pattern,
            metricTransformations=[
                {
                    'metricName': alarm_name,
                    'metricNamespace': 'Security/Events',
                    'metricValue': '1'
                }
            ]
        )
        
        # Create alarm
        self.cloudwatch.put_metric_alarm(
            AlarmName=alarm_name,
            ComparisonOperator='GreaterThanThreshold',
            EvaluationPeriods=1,
            MetricName=alarm_name,
            Namespace='Security/Events',
            Period=300,
            Statistic='Sum',
            Threshold=threshold,
            ActionsEnabled=True,
            AlarmActions=[
                'arn:aws:sns:us-east-1:123456789012:security-alerts'
            ],
            AlarmDescription=f'Alert when {alarm_name} exceeds threshold'
        )
    
    async def setup_security_monitoring(self) -> None:
        """Setup comprehensive security monitoring"""
        
        security_patterns = {
            'UnauthorizedAccess': '[timestamp, request_id, "UNAUTHORIZED"]',
            'InvalidTokens': '[timestamp, request_id, "Invalid token"]',
            'SuspiciousActivity': '[timestamp, request_id, "SUSPICIOUS"]',
            'FailedLogins': '[timestamp, request_id, "Login failed"]',
            'DataAccessViolations': '[timestamp, request_id, "ACCESS_DENIED"]'
        }
        
        for alarm_name, pattern in security_patterns.items():
            await self.create_security_alarm(
                alarm_name=alarm_name,
                log_group='/aws/lambda/ebay-sniper-api',
                filter_pattern=pattern
            )
```

### Audit Logging

#### Structured Security Logging
```python
import logging
import json
from datetime import datetime
from typing import Dict, Any, Optional

class SecurityLogger:
    def __init__(self, logger_name: str = "security"):
        self.logger = logging.getLogger(logger_name)
        self.logger.setLevel(logging.INFO)
    
    def log_security_event(
        self,
        event_type: str,
        user_id: Optional[str],
        request_id: str,
        details: Dict[str, Any],
        severity: str = "INFO"
    ) -> None:
        """Log security event with structured format"""
        
        security_event = {
            "timestamp": datetime.utcnow().isoformat(),
            "event_type": event_type,
            "user_id": user_id,
            "request_id": request_id,
            "severity": severity,
            "details": details,
            "source": "ebay-sniper-api"
        }
        
        if severity == "CRITICAL":
            self.logger.critical(json.dumps(security_event))
        elif severity == "ERROR":
            self.logger.error(json.dumps(security_event))
        elif severity == "WARNING":
            self.logger.warning(json.dumps(security_event))
        else:
            self.logger.info(json.dumps(security_event))
    
    def log_authentication_event(
        self,
        user_id: str,
        request_id: str,
        success: bool,
        ip_address: str,
        user_agent: str
    ) -> None:
        """Log authentication attempts"""
        
        self.log_security_event(
            event_type="AUTHENTICATION",
            user_id=user_id,
            request_id=request_id,
            details={
                "success": success,
                "ip_address": ip_address,
                "user_agent": user_agent
            },
            severity="WARNING" if not success else "INFO"
        )
    
    def log_data_access(
        self,
        user_id: str,
        request_id: str,
        resource: str,
        action: str,
        success: bool
    ) -> None:
        """Log data access events"""
        
        self.log_security_event(
            event_type="DATA_ACCESS",
            user_id=user_id,
            request_id=request_id,
            details={
                "resource": resource,
                "action": action,
                "success": success
            },
            severity="ERROR" if not success else "INFO"
        )
```

## Compliance & Governance

### GDPR Compliance

#### Data Processing Implementation
```python
from enum import Enum
from typing import List, Dict, Any

class DataPurpose(Enum):
    ACCOUNT_MANAGEMENT = "account_management"
    BID_PROCESSING = "bid_processing"
    COMMUNICATION = "communication"
    ANALYTICS = "analytics"

class GDPRComplianceHandler:
    def __init__(self, dynamodb_client):
        self.dynamodb = dynamodb_client
    
    async def export_user_data(self, user_id: str) -> Dict[str, Any]:
        """Export all user data for GDPR data portability"""
        
        user_data = {
            "user_profile": await self.get_user_profile(user_id),
            "bids": await self.get_user_bids(user_id),
            "bid_history": await self.get_user_bid_history(user_id),
            "preferences": await self.get_user_preferences(user_id)
        }
        
        # Add metadata
        user_data["export_metadata"] = {
            "exported_at": datetime.utcnow().isoformat(),
            "format_version": "1.0",
            "data_controller": "eBay Sniper Application"
        }
        
        return user_data
    
    async def delete_user_data(self, user_id: str) -> None:
        """Delete all user data for GDPR right to erasure"""
        
        # Delete from all tables
        tables = ['Users', 'Bids', 'BidHistory']
        
        for table_name in tables:
            table = self.dynamodb.Table(table_name)
            
            # Query for user's items
            response = table.query(
                KeyConditionExpression=Key('userId').eq(user_id)
            )
            
            # Delete items in batches
            with table.batch_writer() as batch:
                for item in response['Items']:
                    batch.delete_item(Key={
                        'userId': item['userId'],
                        'bidId': item.get('bidId') or item.get('timestamp')
                    })
        
        logger.info(f"Deleted all data for user {user_id}")
    
    async def anonymize_user_data(self, user_id: str) -> None:
        """Anonymize user data while preserving analytics"""
        
        # Generate anonymous identifier
        anonymous_id = hashlib.sha256(f"{user_id}:anonymous".encode()).hexdigest()[:16]
        
        # Update records with anonymized data
        tables = ['BidHistory']  # Keep for analytics
        
        for table_name in tables:
            table = self.dynamodb.Table(table_name)
            
            response = table.query(
                KeyConditionExpression=Key('userId').eq(user_id)
            )
            
            for item in response['Items']:
                table.update_item(
                    Key={'userId': user_id, 'timestamp': item['timestamp']},
                    UpdateExpression='SET userId = :anon_id',
                    ExpressionAttributeValues={':anon_id': anonymous_id}
                )
```

### Security Frameworks

#### SOC 2 Controls Implementation
```python
class SOC2Controls:
    """Implementation of SOC 2 security controls"""
    
    @staticmethod
    def cc6_1_logical_access_controls():
        """CC6.1 - Logical and physical access controls"""
        controls = {
            "multi_factor_authentication": True,
            "role_based_access_control": True,
            "principle_of_least_privilege": True,
            "regular_access_reviews": True,
            "privileged_access_management": True
        }
        return controls
    
    @staticmethod
    def cc6_2_access_authorization():
        """CC6.2 - Access authorization and authentication"""
        controls = {
            "centralized_authentication": "AWS Cognito",
            "strong_password_policy": True,
            "session_management": True,
            "account_lockout_policy": True
        }
        return controls
    
    @staticmethod
    def cc6_7_data_transmission():
        """CC6.7 - Data transmission security"""
        controls = {
            "encryption_in_transit": "TLS 1.2+",
            "certificate_management": "AWS Certificate Manager",
            "secure_protocols": ["HTTPS", "WSS"],
            "certificate_validation": True
        }
        return controls
    
    @staticmethod
    def cc6_8_data_classification():
        """CC6.8 - Data classification and handling"""
        controls = {
            "data_classification_policy": True,
            "pii_handling_procedures": True,
            "data_retention_policy": True,
            "secure_disposal_procedures": True
        }
        return controls
```

## Incident Response

### Security Incident Procedures

#### Incident Detection and Response
```python
class SecurityIncidentHandler:
    def __init__(self, sns_topic_arn: str):
        self.sns = boto3.client('sns')
        self.sns_topic = sns_topic_arn
    
    async def handle_security_incident(
        self,
        incident_type: str,
        severity: str,
        details: Dict[str, Any]
    ) -> None:
        """Handle security incident according to severity"""
        
        incident = {
            "incident_id": str(uuid.uuid4()),
            "type": incident_type,
            "severity": severity,
            "timestamp": datetime.utcnow().isoformat(),
            "details": details,
            "status": "DETECTED"
        }
        
        # Immediate containment for critical incidents
        if severity == "CRITICAL":
            await self.emergency_containment(incident)
        
        # Notify security team
        await self.notify_security_team(incident)
        
        # Log incident
        logger.critical(f"Security incident detected: {json.dumps(incident)}")
    
    async def emergency_containment(self, incident: Dict[str, Any]) -> None:
        """Emergency containment procedures"""
        
        if incident["type"] == "UNAUTHORIZED_ACCESS":
            # Disable affected user accounts
            user_ids = incident["details"].get("affected_users", [])
            for user_id in user_ids:
                await self.disable_user_account(user_id)
        
        elif incident["type"] == "DATA_BREACH":
            # Enable enhanced monitoring
            await self.enable_enhanced_monitoring()
            
        elif incident["type"] == "SYSTEM_COMPROMISE":
            # Scale down affected services
            await self.emergency_scale_down()
    
    async def notify_security_team(self, incident: Dict[str, Any]) -> None:
        """Notify security team of incident"""
        
        message = {
            "incident_id": incident["incident_id"],
            "type": incident["type"],
            "severity": incident["severity"],
            "timestamp": incident["timestamp"],
            "details": incident["details"],
            "runbook": f"https://docs.company.com/security/runbook/{incident['type']}"
        }
        
        await self.sns.publish(
            TopicArn=self.sns_topic,
            Subject=f"SECURITY INCIDENT - {incident['severity']} - {incident['type']}",
            Message=json.dumps(message, indent=2)
        )
```

## Security Testing

### Automated Security Testing

#### Security Test Suite
```python
import pytest
import requests
import jwt
from datetime import datetime, timedelta

class SecurityTestSuite:
    def __init__(self, api_base_url: str):
        self.api_base_url = api_base_url
    
    @pytest.mark.security
    def test_authentication_required(self):
        """Test that endpoints require authentication"""
        
        protected_endpoints = [
            "/users/me",
            "/bids",
            "/ebay/wishlist"
        ]
        
        for endpoint in protected_endpoints:
            response = requests.get(f"{self.api_base_url}{endpoint}")
            assert response.status_code == 401
    
    @pytest.mark.security
    def test_invalid_token_rejection(self):
        """Test that invalid tokens are rejected"""
        
        invalid_tokens = [
            "invalid-token",
            "Bearer invalid-token",
            "Bearer " + "a" * 500,  # Oversized token
            ""
        ]
        
        for token in invalid_tokens:
            headers = {"Authorization": token}
            response = requests.get(
                f"{self.api_base_url}/users/me",
                headers=headers
            )
            assert response.status_code == 401
    
    @pytest.mark.security
    def test_sql_injection_protection(self):
        """Test SQL injection protection"""
        
        injection_payloads = [
            "'; DROP TABLE Users; --",
            "1' OR '1'='1",
            "' UNION SELECT * FROM Users --"
        ]
        
        for payload in injection_payloads:
            response = requests.get(
                f"{self.api_base_url}/bids",
                params={"status": payload},
                headers={"Authorization": self.get_valid_token()}
            )
            # Should not return error or unexpected data
            assert response.status_code in [200, 400, 422]
    
    @pytest.mark.security
    def test_xss_protection(self):
        """Test XSS protection"""
        
        xss_payloads = [
            "<script>alert('xss')</script>",
            "javascript:alert('xss')",
            "<img src=x onerror=alert('xss')>"
        ]
        
        for payload in xss_payloads:
            response = requests.post(
                f"{self.api_base_url}/bids",
                json={"ebayItemId": "123456789", "maxBidAmount": payload},
                headers={"Authorization": self.get_valid_token()}
            )
            
            # Payload should be rejected or sanitized
            if response.status_code == 200:
                assert payload not in response.text
    
    @pytest.mark.security
    def test_rate_limiting(self):
        """Test rate limiting enforcement"""
        
        # Make rapid requests
        responses = []
        for i in range(110):  # Exceed rate limit
            response = requests.get(
                f"{self.api_base_url}/users/me",
                headers={"Authorization": self.get_valid_token()}
            )
            responses.append(response.status_code)
        
        # Should receive 429 responses
        assert 429 in responses
```

## Security Checklist

### Deployment Security Checklist

#### Pre-Deployment
- [ ] All secrets stored in AWS Secrets Manager
- [ ] IAM roles follow least privilege principle
- [ ] KMS encryption enabled for all data stores
- [ ] TLS 1.2+ enforced for all endpoints
- [ ] WAF rules configured and tested
- [ ] Security groups restrict access appropriately
- [ ] CloudTrail logging enabled
- [ ] Backup and recovery procedures tested

#### Post-Deployment
- [ ] Security monitoring alerts configured
- [ ] Penetration testing completed
- [ ] Vulnerability scanning performed
- [ ] Access logs reviewed
- [ ] Incident response procedures tested
- [ ] Compliance requirements validated
- [ ] Security documentation updated

### Ongoing Security Maintenance

#### Weekly Tasks
- [ ] Review security alerts and logs
- [ ] Monitor for failed authentication attempts
- [ ] Check for unauthorized access patterns
- [ ] Review API usage patterns

#### Monthly Tasks
- [ ] Update security patches
- [ ] Review IAM permissions
- [ ] Rotate non-critical secrets
- [ ] Security metrics review

#### Quarterly Tasks
- [ ] Penetration testing
- [ ] Security training updates
- [ ] Incident response drills
- [ ] Compliance audit preparation

#### Annual Tasks
- [ ] Comprehensive security audit
- [ ] Disaster recovery testing
- [ ] Security policy review
- [ ] Third-party security assessments