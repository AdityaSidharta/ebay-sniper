# eBay API Integration Guide

This document outlines the integration with eBay APIs for the eBay Sniper application.

## Overview

The application integrates with multiple eBay APIs to provide wishlist-based bidding functionality:
- **OAuth API**: User authentication and token management
- **Browse API**: Retrieve wishlist items
- **Trading API**: Place bids and monitor auctions

## eBay Developer Account Setup

### Required Credentials
- **App ID**: Your application identifier
- **Dev ID**: Developer account identifier  
- **Cert ID**: Certificate identifier for secure API calls
- **OAuth Redirect URI**: Must match your application's callback URL

### Environment Configuration
```python
# Sandbox Environment
EBAY_BASE_URL = "https://api.sandbox.ebay.com"
EBAY_AUTH_URL = "https://auth.sandbox.ebay.com"

# Production Environment  
EBAY_BASE_URL = "https://api.ebay.com"
EBAY_AUTH_URL = "https://auth.ebay.com"
```

## OAuth 2.0 Authentication Flow

### Step 1: Authorization URL Generation
```python
def generate_auth_url(state: str) -> str:
    params = {
        'client_id': EBAY_APP_ID,
        'redirect_uri': EBAY_REDIRECT_URI,
        'response_type': 'code',
        'state': state,
        'scope': 'https://api.ebay.com/oauth/api_scope/sell.inventory.readonly https://api.ebay.com/oauth/api_scope/sell.marketing.readonly'
    }
    return f"{EBAY_AUTH_URL}/oauth2/authorize?" + urlencode(params)
```

### Step 2: Token Exchange
```python
async def exchange_code_for_tokens(code: str) -> dict:
    headers = {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Authorization': f'Basic {base64_credentials}'
    }
    
    data = {
        'grant_type': 'authorization_code',
        'code': code,
        'redirect_uri': EBAY_REDIRECT_URI
    }
    
    response = await http_client.post(
        f"{EBAY_AUTH_URL}/oauth2/token",
        headers=headers,
        data=data
    )
    
    return response.json()
```

### Step 3: Token Refresh
```python
async def refresh_access_token(refresh_token: str) -> dict:
    headers = {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Authorization': f'Basic {base64_credentials}'
    }
    
    data = {
        'grant_type': 'refresh_token',
        'refresh_token': refresh_token
    }
    
    response = await http_client.post(
        f"{EBAY_AUTH_URL}/oauth2/token",
        headers=headers,
        data=data
    )
    
    return response.json()
```

## Browse API Integration

### Get User Wishlist
```python
async def get_user_wishlist(access_token: str) -> list:
    headers = {
        'Authorization': f'Bearer {access_token}',
        'Content-Type': 'application/json',
        'X-EBAY-C-MARKETPLACE-ID': 'EBAY_US'
    }
    
    # Note: eBay doesn't have a direct wishlist API
    # This would typically require the user to provide item IDs
    # or use the Browse API to search for specific items
    
    response = await http_client.get(
        f"{EBAY_BASE_URL}/buy/browse/v1/item_summary/search",
        headers=headers,
        params={'q': 'user_search_term', 'limit': 200}
    )
    
    return response.json()
```

### Get Item Details
```python
async def get_item_details(item_id: str, access_token: str) -> dict:
    headers = {
        'Authorization': f'Bearer {access_token}',
        'Content-Type': 'application/json',
        'X-EBAY-C-MARKETPLACE-ID': 'EBAY_US'
    }
    
    response = await http_client.get(
        f"{EBAY_BASE_URL}/buy/browse/v1/item/{item_id}",
        headers=headers
    )
    
    return response.json()
```

## Trading API Integration

### Place Bid
```python
async def place_bid(item_id: str, bid_amount: float, access_token: str) -> dict:
    headers = {
        'X-EBAY-API-COMPATIBILITY-LEVEL': '967',
        'X-EBAY-API-DEV-NAME': EBAY_DEV_ID,
        'X-EBAY-API-APP-NAME': EBAY_APP_ID,
        'X-EBAY-API-CERT-NAME': EBAY_CERT_ID,
        'X-EBAY-API-CALL-NAME': 'PlaceBid',
        'X-EBAY-API-SITEID': '0',
        'Content-Type': 'text/xml'
    }
    
    xml_request = f"""<?xml version="1.0" encoding="utf-8"?>
    <PlaceBidRequest xmlns="urn:ebay:apis:eBLBaseComponents">
        <RequesterCredentials>
            <eBayAuthToken>{access_token}</eBayAuthToken>
        </RequesterCredentials>
        <ItemID>{item_id}</ItemID>
        <Bid>
            <Amount currencyID="USD">{bid_amount}</Amount>
        </Bid>
    </PlaceBidRequest>"""
    
    response = await http_client.post(
        f"{EBAY_BASE_URL}/ws/api.dll",
        headers=headers,
        data=xml_request
    )
    
    return parse_xml_response(response.text)
```

### Get Auction Status
```python
async def get_auction_status(item_id: str, access_token: str) -> dict:
    headers = {
        'X-EBAY-API-COMPATIBILITY-LEVEL': '967',
        'X-EBAY-API-DEV-NAME': EBAY_DEV_ID,
        'X-EBAY-API-APP-NAME': EBAY_APP_ID,
        'X-EBAY-API-CERT-NAME': EBAY_CERT_ID,
        'X-EBAY-API-CALL-NAME': 'GetItem',
        'X-EBAY-API-SITEID': '0',
        'Content-Type': 'text/xml'
    }
    
    xml_request = f"""<?xml version="1.0" encoding="utf-8"?>
    <GetItemRequest xmlns="urn:ebay:apis:eBLBaseComponents">
        <RequesterCredentials>
            <eBayAuthToken>{access_token}</eBayAuthToken>
        </RequesterCredentials>
        <ItemID>{item_id}</ItemID>
        <DetailLevel>ReturnAll</DetailLevel>
    </GetItemRequest>"""
    
    response = await http_client.post(
        f"{EBAY_BASE_URL}/ws/api.dll",
        headers=headers,
        data=xml_request
    )
    
    return parse_xml_response(response.text)
```

## Error Handling

### Common Error Codes
```python
EBAY_ERROR_CODES = {
    '21916611': 'Bid amount is too low',
    '21916585': 'Auction has ended',
    '21919301': 'Item not found',
    '932': 'Authentication token is invalid',
    '21917052': 'Bidding is not allowed for this item'
}

async def handle_ebay_error(error_response: dict) -> None:
    error_code = error_response.get('errorId')
    error_message = error_response.get('message', 'Unknown eBay error')
    
    if error_code in EBAY_ERROR_CODES:
        raise EbayAPIError(EBAY_ERROR_CODES[error_code])
    else:
        raise EbayAPIError(f"eBay API Error: {error_message}")
```

### Retry Logic
```python
import asyncio
from typing import Callable, Any

async def retry_with_backoff(
    func: Callable,
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    backoff_factor: float = 2.0
) -> Any:
    for attempt in range(max_retries + 1):
        try:
            return await func()
        except EbayRateLimitError:
            if attempt == max_retries:
                raise
            
            delay = min(base_delay * (backoff_factor ** attempt), max_delay)
            await asyncio.sleep(delay)
        except EbayAPIError as e:
            if e.is_retryable() and attempt < max_retries:
                delay = min(base_delay * (backoff_factor ** attempt), max_delay)
                await asyncio.sleep(delay)
            else:
                raise
```

## Rate Limiting

### Implementation
```python
import time
from collections import defaultdict

class EbayRateLimiter:
    def __init__(self, calls_per_day: int = 5000):
        self.calls_per_day = calls_per_day
        self.call_history = defaultdict(list)
    
    async def check_rate_limit(self, user_id: str) -> bool:
        now = time.time()
        day_start = now - (now % 86400)  # Start of current day
        
        # Clean old calls
        self.call_history[user_id] = [
            call_time for call_time in self.call_history[user_id]
            if call_time >= day_start
        ]
        
        if len(self.call_history[user_id]) >= self.calls_per_day:
            raise EbayRateLimitError("Daily API limit exceeded")
        
        self.call_history[user_id].append(now)
        return True
```

## Response Parsing

### XML Response Parser
```python
import xml.etree.ElementTree as ET

def parse_xml_response(xml_string: str) -> dict:
    try:
        root = ET.fromstring(xml_string)
        
        # Extract common fields
        result = {
            'ack': root.find('.//{urn:ebay:apis:eBLBaseComponents}Ack').text,
            'timestamp': root.find('.//{urn:ebay:apis:eBLBaseComponents}Timestamp').text,
            'version': root.find('.//{urn:ebay:apis:eBLBaseComponents}Version').text
        }
        
        # Check for errors
        errors = root.findall('.//{urn:ebay:apis:eBLBaseComponents}Errors')
        if errors:
            result['errors'] = []
            for error in errors:
                result['errors'].append({
                    'severity': error.find('.//{urn:ebay:apis:eBLBaseComponents}SeverityCode').text,
                    'code': error.find('.//{urn:ebay:apis:eBLBaseComponents}ErrorCode').text,
                    'message': error.find('.//{urn:ebay:apis:eBLBaseComponents}ShortMessage').text
                })
        
        return result
        
    except ET.ParseError as e:
        raise EbayAPIError(f"Failed to parse eBay XML response: {str(e)}")
```

## Testing

### Mock eBay API Client
```python
class MockEbayClient:
    def __init__(self, sandbox_mode: bool = True):
        self.sandbox_mode = sandbox_mode
        self.mock_responses = {
            'place_bid': {'ack': 'Success', 'bid_id': 'mock_bid_123'},
            'get_item': {
                'ack': 'Success',
                'item': {
                    'item_id': '123456789',
                    'title': 'Test Item',
                    'current_price': 25.00,
                    'end_time': '2024-01-01T23:59:59Z'
                }
            }
        }
    
    async def place_bid(self, item_id: str, amount: float, token: str) -> dict:
        if self.sandbox_mode:
            return self.mock_responses['place_bid']
        else:
            # Call real eBay API
            return await real_place_bid(item_id, amount, token)
```

## Security Considerations

### Token Storage
- eBay OAuth tokens are stored in DynamoDB with encryption at rest
- DynamoDB encryption handles security automatically via KMS
- Implement token rotation before expiration
- Never log access tokens or refresh tokens

### API Request Security
- Always validate item IDs before making API calls
- Implement request signing for sensitive operations
- Use HTTPS for all API communications
- Validate all user inputs before sending to eBay API

## Configuration

### Environment Variables
```bash
# eBay API Configuration
EBAY_APP_ID=your_app_id
EBAY_DEV_ID=your_dev_id
EBAY_CERT_ID=your_cert_id
EBAY_ENVIRONMENT=sandbox  # or production
EBAY_REDIRECT_URI=https://your-domain.com/ebay/callback

# Rate Limiting
EBAY_DAILY_RATE_LIMIT=5000
EBAY_BURST_RATE_LIMIT=100
```

### AWS Secrets Manager
Store sensitive credentials in AWS Secrets Manager:
```python
import boto3

async def get_ebay_credentials() -> dict:
    secrets_client = boto3.client('secretsmanager')
    
    response = secrets_client.get_secret_value(
        SecretId='ebay-sniper/ebay-credentials'
    )
    
    return json.loads(response['SecretString'])
```