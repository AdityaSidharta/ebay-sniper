# Testing Strategy

This document outlines the comprehensive testing strategy for the eBay Sniper application, covering unit testing, integration testing, end-to-end testing, and performance testing.

## Testing Overview

### Testing Pyramid
1. **Unit Tests (70%)**: Fast, isolated tests for individual components
2. **Integration Tests (20%)**: Tests for component interactions
3. **End-to-End Tests (10%)**: Full application workflow tests

### Testing Environments
- **Local Development**: Developer machines with mocked services
- **CI/CD Pipeline**: Automated testing on code commits
- **Staging**: Pre-production environment with real AWS services
- **Production**: Limited smoke tests and monitoring

## Backend Testing (Python/FastAPI)

### Unit Testing Framework

#### Test Structure
```python
# tests/conftest.py
import pytest
import boto3
from moto import mock_dynamodb
from src.main import app
from src.core.config import settings
from fastapi.testclient import TestClient

@pytest.fixture
def client():
    """Test client for FastAPI application"""
    return TestClient(app)

@pytest.fixture
def mock_dynamodb_table():
    """Mock DynamoDB table for testing"""
    with mock_dynamodb():
        dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
        
        # Create mock table
        table = dynamodb.create_table(
            TableName='test-Users',
            KeySchema=[
                {'AttributeName': 'userId', 'KeyType': 'HASH'}
            ],
            AttributeDefinitions=[
                {'AttributeName': 'userId', 'AttributeType': 'S'}
            ],
            BillingMode='PAY_PER_REQUEST'
        )
        
        yield table

@pytest.fixture
def sample_user():
    """Sample user data for testing"""
    return {
        "userId": "test-user-123",
        "email": "test@example.com",
        "createdAt": 1640995200,
        "updatedAt": 1640995200,
        "preferences": {
            "emailNotifications": True,
            "bidWinNotifications": True,
            "bidLossNotifications": True
        },
        "isActive": True
    }

@pytest.fixture
def mock_ebay_client():
    """Mock eBay API client"""
    class MockEbayClient:
        async def get_item_details(self, item_id: str) -> dict:
            return {
                "ebayItemId": item_id,
                "title": "Test Item",
                "currentPrice": 15000,
                "endTime": 1640995200
            }
        
        async def place_bid(self, item_id: str, amount: int, token: str) -> dict:
            return {"success": True, "bidId": "test-bid-123"}
    
    return MockEbayClient()
```

#### Service Layer Tests
```python
# tests/unit/test_user_service.py
import pytest
from unittest.mock import Mock, patch
from src.services.user_service import UserService
from src.models.user import User, UserPreferences

class TestUserService:
    @pytest.fixture
    def user_service(self, mock_dynamodb_table):
        return UserService(table=mock_dynamodb_table)
    
    @pytest.mark.asyncio
    async def test_create_user_success(self, user_service, sample_user):
        """Test successful user creation"""
        # Act
        result = await user_service.create_user(
            user_id=sample_user["userId"],
            email=sample_user["email"],
            preferences=sample_user["preferences"]
        )
        
        # Assert
        assert result.userId == sample_user["userId"]
        assert result.email == sample_user["email"]
        assert result.isActive is True
    
    @pytest.mark.asyncio
    async def test_create_user_duplicate_email(self, user_service, sample_user):
        """Test user creation with duplicate email"""
        # Arrange
        await user_service.create_user(
            user_id=sample_user["userId"],
            email=sample_user["email"],
            preferences=sample_user["preferences"]
        )
        
        # Act & Assert
        with pytest.raises(ValueError, match="User already exists"):
            await user_service.create_user(
                user_id="different-user-id",
                email=sample_user["email"],
                preferences=sample_user["preferences"]
            )
    
    @pytest.mark.asyncio
    async def test_get_user_by_id_success(self, user_service, sample_user):
        """Test successful user retrieval"""
        # Arrange
        await user_service.create_user(
            user_id=sample_user["userId"],
            email=sample_user["email"],
            preferences=sample_user["preferences"]
        )
        
        # Act
        result = await user_service.get_user_by_id(sample_user["userId"])
        
        # Assert
        assert result is not None
        assert result.userId == sample_user["userId"]
        assert result.email == sample_user["email"]
    
    @pytest.mark.asyncio
    async def test_get_user_by_id_not_found(self, user_service):
        """Test user retrieval with non-existent ID"""
        # Act
        result = await user_service.get_user_by_id("non-existent-user")
        
        # Assert
        assert result is None
    
    @pytest.mark.asyncio
    async def test_update_user_preferences(self, user_service, sample_user):
        """Test user preferences update"""
        # Arrange
        await user_service.create_user(
            user_id=sample_user["userId"],
            email=sample_user["email"],
            preferences=sample_user["preferences"]
        )
        
        new_preferences = UserPreferences(
            emailNotifications=False,
            bidWinNotifications=True,
            bidLossNotifications=False
        )
        
        # Act
        result = await user_service.update_user_preferences(
            user_id=sample_user["userId"],
            preferences=new_preferences
        )
        
        # Assert
        assert result.preferences.emailNotifications is False
        assert result.preferences.bidWinNotifications is True
        assert result.preferences.bidLossNotifications is False
```

#### API Endpoint Tests
```python
# tests/unit/test_user_endpoints.py
import pytest
from unittest.mock import patch, AsyncMock
from fastapi.testclient import TestClient

class TestUserEndpoints:
    def test_get_current_user_success(self, client, sample_user):
        """Test successful current user retrieval"""
        with patch('src.services.user_service.UserService.get_user_by_id') as mock_get_user:
            mock_get_user.return_value = AsyncMock(return_value=sample_user)
            
            response = client.get(
                "/users/me",
                headers={"Authorization": "Bearer valid-token"}
            )
            
            assert response.status_code == 200
            data = response.json()
            assert data["userId"] == sample_user["userId"]
            assert data["email"] == sample_user["email"]
    
    def test_get_current_user_unauthorized(self, client):
        """Test current user retrieval without authentication"""
        response = client.get("/users/me")
        
        assert response.status_code == 401
        assert "unauthorized" in response.json()["error"]
    
    def test_update_user_preferences_success(self, client, sample_user):
        """Test successful user preferences update"""
        with patch('src.services.user_service.UserService.update_user_preferences') as mock_update:
            updated_user = sample_user.copy()
            updated_user["preferences"]["emailNotifications"] = False
            mock_update.return_value = AsyncMock(return_value=updated_user)
            
            response = client.put(
                "/users/me",
                json={"preferences": {"emailNotifications": False}},
                headers={"Authorization": "Bearer valid-token"}
            )
            
            assert response.status_code == 200
            data = response.json()
            assert data["preferences"]["emailNotifications"] is False
    
    def test_update_user_preferences_validation_error(self, client):
        """Test user preferences update with invalid data - Pydantic handles validation"""
        # Test multiple validation scenarios in one test since Pydantic is the single validation layer
        test_cases = [
            # Invalid preference key
            ({"preferences": {"invalidField": True}}, "Invalid preference keys"),
            # Invalid notification type
            ({"preferences": {"emailNotifications": "not_a_boolean"}}, "emailNotifications must be a boolean"),
            # Invalid timezone type
            ({"preferences": {"timezone": 123}}, "timezone must be a string"),
            # Invalid email format
            ({"email": "invalid-email"}, "string does not match regex")
        ]
        
        for invalid_data, expected_error in test_cases:
            response = client.put(
                "/users/me",
                json=invalid_data,
                headers={"Authorization": "Bearer valid-token"}
            )
            
            assert response.status_code == 422
            assert expected_error.lower() in response.json()["detail"][0]["msg"].lower()
```

#### Bid Service Tests
```python
# tests/unit/test_bid_service.py
import pytest
from unittest.mock import Mock, patch, AsyncMock
from src.services.bid_service import BidService
from src.models.bid import Bid, BidStatus

class TestBidService:
    @pytest.fixture
    def bid_service(self, mock_dynamodb_table, mock_ebay_client):
        return BidService(
            bids_table=mock_dynamodb_table,
            ebay_client=mock_ebay_client
        )
    
    @pytest.mark.asyncio
    async def test_create_bid_success(self, bid_service):
        """Test successful bid creation"""
        # Act
        result = await bid_service.create_bid(
            user_id="test-user-123",
            ebay_item_id="123456789",
            max_bid_amount=20000
        )
        
        # Assert
        assert result.userId == "test-user-123"
        assert result.ebayItemId == "123456789"
        assert result.maxBidAmount == 20000
        assert result.status == BidStatus.PENDING
    
    @pytest.mark.asyncio
    async def test_create_bid_item_not_found(self, bid_service, mock_ebay_client):
        """Test bid creation with non-existent item"""
        # Arrange
        mock_ebay_client.get_item_details = AsyncMock(side_effect=ValueError("Item not found"))
        
        # Act & Assert
        with pytest.raises(ValueError, match="Item not found"):
            await bid_service.create_bid(
                user_id="test-user-123",
                ebay_item_id="non-existent-item",
                max_bid_amount=20000
            )
    
    @pytest.mark.asyncio
    async def test_create_bid_amount_too_low(self, bid_service):
        """Test bid creation with amount lower than current price"""
        # Act & Assert
        with pytest.raises(ValueError, match="Bid amount must be higher than current price"):
            await bid_service.create_bid(
                user_id="test-user-123",
                ebay_item_id="123456789",
                max_bid_amount=10000  # Lower than current price (15000)
            )
    
    @pytest.mark.asyncio
    async def test_update_bid_success(self, bid_service):
        """Test successful bid update"""
        # Arrange
        bid = await bid_service.create_bid(
            user_id="test-user-123",
            ebay_item_id="123456789",
            max_bid_amount=20000
        )
        
        # Act
        result = await bid_service.update_bid(
            user_id="test-user-123",
            bid_id=bid.bidId,
            max_bid_amount=25000
        )
        
        # Assert
        assert result.maxBidAmount == 25000
        assert result.status == BidStatus.PENDING
    
    @pytest.mark.asyncio
    async def test_cancel_bid_success(self, bid_service):
        """Test successful bid cancellation"""
        # Arrange
        bid = await bid_service.create_bid(
            user_id="test-user-123",
            ebay_item_id="123456789",
            max_bid_amount=20000
        )
        
        # Act
        await bid_service.cancel_bid(
            user_id="test-user-123",
            bid_id=bid.bidId
        )
        
        # Assert
        updated_bid = await bid_service.get_bid_by_id(
            user_id="test-user-123",
            bid_id=bid.bidId
        )
        assert updated_bid.status == BidStatus.CANCELLED
```

### Integration Testing

#### Simplified Validation Testing
With consolidated Pydantic validation, testing is simplified:

```python
# tests/unit/test_validation.py
import pytest
from pydantic import ValidationError
from src.models.requests import CreateBidRequest, UpdateUserRequest

class TestConsolidatedValidation:
    """Test single validation layer - no duplicate testing needed"""
    
    def test_create_bid_validation_success(self):
        """Test valid bid creation data"""
        valid_data = {
            "ebay_item_id": "123456789",
            "max_bid_amount": 25000
        }
        
        # Pydantic handles all validation
        request = CreateBidRequest(**valid_data)
        assert request.ebay_item_id == "123456789"
        assert request.max_bid_amount == 25000
    
    def test_create_bid_validation_errors(self):
        """Test all validation scenarios in consolidated tests"""
        invalid_cases = [
            # Invalid item ID format
            ({"ebay_item_id": "abc123", "max_bid_amount": 25000}, "string does not match regex"),
            # Amount too low
            ({"ebay_item_id": "123456789", "max_bid_amount": 50}, "ensure this value is greater"),
            # Amount too high  
            ({"ebay_item_id": "123456789", "max_bid_amount": 20000000}, "ensure this value is less"),
            # Missing required field
            ({"ebay_item_id": "123456789"}, "field required")
        ]
        
        for invalid_data, expected_error in invalid_cases:
            with pytest.raises(ValidationError) as exc_info:
                CreateBidRequest(**invalid_data)
            assert expected_error.lower() in str(exc_info.value).lower()
```

#### Database Integration Tests
```python
# tests/integration/test_database_integration.py
import pytest
import boto3
import time
from moto import mock_dynamodb
from boto3.dynamodb.conditions import Key

@pytest.mark.integration
class TestDatabaseIntegration:
    @pytest.fixture(scope="class")
    def dynamodb_tables(self):
        """Setup real DynamoDB tables for integration testing"""
        with mock_dynamodb():
            dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
            
            # Create Users table
            users_table = dynamodb.create_table(
                TableName='integration-test-Users',
                KeySchema=[
                    {'AttributeName': 'userId', 'KeyType': 'HASH'}
                ],
                AttributeDefinitions=[
                    {'AttributeName': 'userId', 'AttributeType': 'S'}
                ],
                BillingMode='PAY_PER_REQUEST'
            )
            
            # Create Bids table
            bids_table = dynamodb.create_table(
                TableName='integration-test-Bids',
                KeySchema=[
                    {'AttributeName': 'userId', 'KeyType': 'HASH'},
                    {'AttributeName': 'bidId', 'KeyType': 'RANGE'}
                ],
                AttributeDefinitions=[
                    {'AttributeName': 'userId', 'AttributeType': 'S'},
                    {'AttributeName': 'bidId', 'AttributeType': 'S'}
                ],
                BillingMode='PAY_PER_REQUEST'
            )
            
            yield {
                'users': users_table,
                'bids': bids_table
            }
    
    @pytest.mark.asyncio
    async def test_user_bid_workflow(self, dynamodb_tables):
        """Test complete user and bid workflow"""
        # Direct DynamoDB operations without repository layer
        users_table = dynamodb_tables['users']
        bids_table = dynamodb_tables['bids']
        
        # Create user
        user_data = {
            "userId": "integration-test-user",
            "email": "test@example.com",
            "preferences": {"emailNotifications": True},
            "isActive": True
        }
        users_table.put_item(Item=user_data)
        
        # Create bid
        bid_data = {
            "bidId": "integration-test-bid",
            "userId": "integration-test-user",
            "ebayItemId": "123456789",
            "maxBidAmount": 20000,
            "status": "PENDING"
        }
        bids_table.put_item(Item=bid_data)
        
        # Act & Assert
        user_response = users_table.get_item(Key={'userId': 'integration-test-user'})
        user = user_response['Item']
        assert user["userId"] == "integration-test-user"
        
        bids_response = bids_table.query(
            KeyConditionExpression=Key('userId').eq('integration-test-user')
        )
        bids = bids_response['Items']
        assert len(bids) == 1
        assert bids[0]["bidId"] == "integration-test-bid"
```

#### External API Integration Tests
```python
# tests/integration/test_ebay_api_integration.py
import pytest
from unittest.mock import patch
from src.clients.ebay_client import EbayClient

@pytest.mark.integration
class TestEbayAPIIntegration:
    @pytest.fixture
    def ebay_client(self):
        return EbayClient(environment="sandbox")
    
    @pytest.mark.asyncio
    async def test_ebay_oauth_flow(self, ebay_client):
        """Test eBay OAuth authentication flow"""
        # Test authorization URL generation
        auth_url = ebay_client.generate_auth_url("test-state")
        assert "ebay.com/oauth2/authorize" in auth_url
        assert "test-state" in auth_url
    
    @pytest.mark.asyncio
    @patch('httpx.AsyncClient.post')
    async def test_token_exchange(self, mock_post, ebay_client):
        """Test eBay token exchange"""
        # Mock response
        mock_post.return_value.json.return_value = {
            "access_token": "test-access-token",
            "refresh_token": "test-refresh-token",
            "expires_in": 3600
        }
        
        # Act
        tokens = await ebay_client.exchange_code_for_tokens("test-code")
        
        # Assert
        assert tokens["access_token"] == "test-access-token"
        assert tokens["refresh_token"] == "test-refresh-token"
    
    @pytest.mark.asyncio
    @patch('httpx.AsyncClient.get')
    async def test_get_item_details(self, mock_get, ebay_client):
        """Test eBay item details retrieval"""
        # Mock response
        mock_get.return_value.json.return_value = {
            "itemId": "123456789",
            "title": "Test Item",
            "price": {"value": "150.00", "currency": "USD"},
            "itemEndDate": "2024-01-01T23:59:59.000Z"
        }
        
        # Act
        item = await ebay_client.get_item_details("123456789", "test-token")
        
        # Assert
        assert item["itemId"] == "123456789"
        assert item["title"] == "Test Item"
```

## Frontend Testing (Next.js/TypeScript)

### Component Testing with React Testing Library

#### Test Setup
```typescript
// tests/setup.ts
import '@testing-library/jest-dom'
import { TextEncoder, TextDecoder } from 'util'

// Polyfills for Node.js environment
global.TextEncoder = TextEncoder
global.TextDecoder = TextDecoder

// Mock Next.js router
jest.mock('next/router', () => ({
  useRouter: () => ({
    push: jest.fn(),
    pathname: '/',
    query: {},
    asPath: '/',
  }),
}))

// Mock AWS Amplify
jest.mock('aws-amplify', () => ({
  Auth: {
    currentAuthenticatedUser: jest.fn(),
    signOut: jest.fn(),
  },
  API: {
    get: jest.fn(),
    post: jest.fn(),
    put: jest.fn(),
    del: jest.fn(),
  },
}))
```

#### Component Tests
```typescript
// tests/components/BidForm.test.tsx
import React from 'react'
import { render, screen, fireEvent, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { BidForm } from '@/components/BidForm'

describe('BidForm', () => {
  const mockOnSubmit = jest.fn()
  const mockItem = {
    ebayItemId: '123456789',
    title: 'Test Item',
    currentPrice: 15000,
    endTime: Date.now() + 3600000, // 1 hour from now
  }

  beforeEach(() => {
    mockOnSubmit.mockClear()
  })

  it('renders form with item details', () => {
    render(<BidForm item={mockItem} onSubmit={mockOnSubmit} />)
    
    expect(screen.getByText('Test Item')).toBeInTheDocument()
    expect(screen.getByText('$150.00')).toBeInTheDocument()
    expect(screen.getByLabelText(/maximum bid amount/i)).toBeInTheDocument()
  })

  it('validates minimum bid amount', async () => {
    const user = userEvent.setup()
    render(<BidForm item={mockItem} onSubmit={mockOnSubmit} />)
    
    const bidInput = screen.getByLabelText(/maximum bid amount/i)
    const submitButton = screen.getByRole('button', { name: /place bid/i })
    
    await user.type(bidInput, '100') // Less than current price
    await user.click(submitButton)
    
    expect(screen.getByText(/bid must be higher than current price/i)).toBeInTheDocument()
    expect(mockOnSubmit).not.toHaveBeenCalled()
  })

  it('submits valid bid', async () => {
    const user = userEvent.setup()
    render(<BidForm item={mockItem} onSubmit={mockOnSubmit} />)
    
    const bidInput = screen.getByLabelText(/maximum bid amount/i)
    const submitButton = screen.getByRole('button', { name: /place bid/i })
    
    await user.type(bidInput, '200')
    await user.click(submitButton)
    
    await waitFor(() => {
      expect(mockOnSubmit).toHaveBeenCalledWith({
        ebayItemId: '123456789',
        maxBidAmount: 20000, // $200 in cents
      })
    })
  })

  it('disables form when auction has ended', () => {
    const endedItem = { ...mockItem, endTime: Date.now() - 3600000 } // 1 hour ago
    render(<BidForm item={endedItem} onSubmit={mockOnSubmit} />)
    
    expect(screen.getByText(/auction has ended/i)).toBeInTheDocument()
    expect(screen.getByRole('button', { name: /place bid/i })).toBeDisabled()
  })
})
```

#### Hook Tests
```typescript
// tests/hooks/useBids.test.tsx
import { renderHook, waitFor } from '@testing-library/react'
import { API } from 'aws-amplify'
import { useBids } from '@/hooks/useBids'

jest.mock('aws-amplify')
const mockAPI = API as jest.Mocked<typeof API>

describe('useBids', () => {
  beforeEach(() => {
    jest.clearAllMocks()
  })

  it('fetches bids on mount', async () => {
    const mockBids = [
      {
        bidId: 'bid-1',
        ebayItemId: '123456789',
        maxBidAmount: 20000,
        status: 'PENDING',
      },
    ]

    mockAPI.get.mockResolvedValue({ bids: mockBids })

    const { result } = renderHook(() => useBids())

    await waitFor(() => {
      expect(result.current.bids).toEqual(mockBids)
      expect(result.current.loading).toBe(false)
    })

    expect(mockAPI.get).toHaveBeenCalledWith('ebay-api', '/bids', {})
  })

  it('handles API errors', async () => {
    mockAPI.get.mockRejectedValue(new Error('API Error'))

    const { result } = renderHook(() => useBids())

    await waitFor(() => {
      expect(result.current.error).toBe('Failed to fetch bids')
      expect(result.current.loading).toBe(false)
    })
  })

  it('creates new bid', async () => {
    const newBid = {
      bidId: 'bid-2',
      ebayItemId: '987654321',
      maxBidAmount: 25000,
      status: 'PENDING',
    }

    mockAPI.post.mockResolvedValue(newBid)

    const { result } = renderHook(() => useBids())

    await waitFor(() => {
      result.current.createBid({
        ebayItemId: '987654321',
        maxBidAmount: 25000,
      })
    })

    expect(mockAPI.post).toHaveBeenCalledWith('ebay-api', '/bids', {
      body: {
        ebayItemId: '987654321',
        maxBidAmount: 25000,
      },
    })
  })
})
```

### End-to-End Testing with Playwright

#### E2E Test Setup
```typescript
// tests/e2e/setup.ts
import { test as base, expect, Page } from '@playwright/test'

interface TestFixtures {
  authenticatedPage: Page
}

export const test = base.extend<TestFixtures>({
  authenticatedPage: async ({ page }, use) => {
    // Navigate to login page
    await page.goto('/login')
    
    // Perform login
    await page.fill('[data-testid="email-input"]', 'test@example.com')
    await page.fill('[data-testid="password-input"]', 'TestPassword123!')
    await page.click('[data-testid="login-button"]')
    
    // Wait for authentication to complete
    await page.waitForURL('/dashboard')
    
    await use(page)
  },
})

export { expect }
```

#### E2E Test Cases
```typescript
// tests/e2e/bid-workflow.spec.ts
import { test, expect } from './setup'

test.describe('Bid Management Workflow', () => {
  test('complete bid lifecycle', async ({ authenticatedPage: page }) => {
    // Navigate to wishlist
    await page.click('[data-testid="wishlist-nav"]')
    await page.waitForURL('/wishlist')
    
    // Select an item to bid on
    const firstItem = page.locator('[data-testid="wishlist-item"]').first()
    await firstItem.click('[data-testid="bid-button"]')
    
    // Fill out bid form
    await page.fill('[data-testid="bid-amount-input"]', '200')
    await page.click('[data-testid="submit-bid-button"]')
    
    // Verify bid creation
    await expect(page.locator('[data-testid="success-message"]')).toContainText('Bid placed successfully')
    
    // Navigate to active bids
    await page.click('[data-testid="active-bids-nav"]')
    await page.waitForURL('/bids')
    
    // Verify bid appears in list
    const bidItem = page.locator('[data-testid="bid-item"]').first()
    await expect(bidItem).toContainText('$200.00')
    await expect(bidItem).toContainText('PENDING')
    
    // Update bid amount
    await bidItem.click('[data-testid="edit-bid-button"]')
    await page.fill('[data-testid="bid-amount-input"]', '250')
    await page.click('[data-testid="update-bid-button"]')
    
    // Verify update
    await expect(page.locator('[data-testid="success-message"]')).toContainText('Bid updated successfully')
    await expect(bidItem).toContainText('$250.00')
    
    // Cancel bid
    await bidItem.click('[data-testid="cancel-bid-button"]')
    await page.click('[data-testid="confirm-cancel-button"]')
    
    // Verify cancellation
    await expect(bidItem).toContainText('CANCELLED')
  })

  test('handles invalid bid amounts', async ({ authenticatedPage: page }) => {
    await page.goto('/wishlist')
    
    const firstItem = page.locator('[data-testid="wishlist-item"]').first()
    await firstItem.click('[data-testid="bid-button"]')
    
    // Try to bid below current price
    await page.fill('[data-testid="bid-amount-input"]', '50')
    await page.click('[data-testid="submit-bid-button"]')
    
    // Verify error message
    await expect(page.locator('[data-testid="error-message"]')).toContainText('Bid must be higher than current price')
  })
})
```

## Performance Testing

### Load Testing with Locust

#### Load Test Setup
```python
# tests/performance/locustfile.py
from locust import HttpUser, task, between
import json
import random

class EbaySniperUser(HttpUser):
    wait_time = between(1, 3)
    
    def on_start(self):
        """Called when a user starts"""
        self.login()
    
    def login(self):
        """Authenticate user and get token"""
        response = self.client.post("/auth/login", json={
            "email": "loadtest@example.com",
            "password": "TestPassword123!"
        })
        
        if response.status_code == 200:
            self.token = response.json()["accessToken"]
            self.headers = {"Authorization": f"Bearer {self.token}"}
        else:
            self.token = None
            self.headers = {}
    
    @task(3)
    def get_user_profile(self):
        """Get current user profile"""
        self.client.get("/users/me", headers=self.headers)
    
    @task(5)
    def get_wishlist(self):
        """Get user's eBay wishlist"""
        self.client.get("/ebay/wishlist", headers=self.headers)
    
    @task(4)
    def get_active_bids(self):
        """Get user's active bids"""
        self.client.get("/bids", headers=self.headers)
    
    @task(2)
    def get_bid_history(self):
        """Get user's bid history"""
        self.client.get("/bids/history", headers=self.headers)
    
    @task(1)
    def create_bid(self):
        """Create a new bid"""
        bid_data = {
            "ebayItemId": str(random.randint(100000000, 999999999)),
            "maxBidAmount": random.randint(10000, 50000)  # $100-$500
        }
        
        self.client.post("/bids", json=bid_data, headers=self.headers)
    
    @task(1)
    def update_bid(self):
        """Update an existing bid"""
        # First get bids to find one to update
        response = self.client.get("/bids", headers=self.headers)
        
        if response.status_code == 200:
            bids = response.json().get("bids", [])
            pending_bids = [bid for bid in bids if bid["status"] == "PENDING"]
            
            if pending_bids:
                bid = random.choice(pending_bids)
                update_data = {
                    "maxBidAmount": bid["maxBidAmount"] + random.randint(1000, 5000)
                }
                
                self.client.put(
                    f"/bids/{bid['bidId']}", 
                    json=update_data, 
                    headers=self.headers
                )
```

#### Performance Test Configuration
```python
# tests/performance/config.py
LOAD_TEST_CONFIGS = {
    "smoke": {
        "users": 1,
        "spawn_rate": 1,
        "duration": "1m"
    },
    "load": {
        "users": 100,
        "spawn_rate": 10,
        "duration": "10m"
    },
    "stress": {
        "users": 500,
        "spawn_rate": 50,
        "duration": "20m"
    },
    "spike": {
        "users": 1000,
        "spawn_rate": 100,
        "duration": "5m"
    }
}

PERFORMANCE_THRESHOLDS = {
    "response_time_95th": 2000,  # ms
    "response_time_99th": 5000,  # ms
    "error_rate": 0.01,  # 1%
    "throughput_min": 100,  # requests/second
}
```

### Database Performance Testing

#### DynamoDB Performance Tests
```python
# tests/performance/test_dynamodb_performance.py
import asyncio
import time
import boto3
from concurrent.futures import ThreadPoolExecutor

class DynamoDBPerformanceTest:
    def __init__(self, table_name: str):
        self.dynamodb = boto3.resource('dynamodb')
        self.table = self.dynamodb.Table(table_name)
    
    async def test_read_performance(self, num_reads: int = 1000):
        """Test read performance with concurrent requests"""
        start_time = time.time()
        
        with ThreadPoolExecutor(max_workers=50) as executor:
            futures = []
            
            for i in range(num_reads):
                future = executor.submit(
                    self.table.get_item,
                    Key={'userId': f'user-{i % 100}'}
                )
                futures.append(future)
            
            # Wait for all reads to complete
            for future in futures:
                future.result()
        
        end_time = time.time()
        duration = end_time - start_time
        
        return {
            "total_reads": num_reads,
            "duration": duration,
            "reads_per_second": num_reads / duration
        }
    
    async def test_write_performance(self, num_writes: int = 1000):
        """Test write performance with concurrent requests"""
        start_time = time.time()
        
        with ThreadPoolExecutor(max_workers=50) as executor:
            futures = []
            
            for i in range(num_writes):
                item = {
                    'userId': f'perf-test-user-{i}',
                    'email': f'user{i}@example.com',
                    'createdAt': int(time.time()),
                    'isActive': True
                }
                
                future = executor.submit(self.table.put_item, Item=item)
                futures.append(future)
            
            # Wait for all writes to complete
            for future in futures:
                future.result()
        
        end_time = time.time()
        duration = end_time - start_time
        
        return {
            "total_writes": num_writes,
            "duration": duration,
            "writes_per_second": num_writes / duration
        }
```

## Test Data Management

### Test Data Factory
```python
# tests/utils/factories.py
import factory
import uuid
from datetime import datetime, timedelta
from src.models.user import User
from src.models.bid import Bid, BidStatus

class UserFactory(factory.Factory):
    class Meta:
        model = dict
    
    userId = factory.LazyFunction(lambda: str(uuid.uuid4()))
    email = factory.Sequence(lambda n: f"user{n}@example.com")
    createdAt = factory.LazyFunction(lambda: int(datetime.utcnow().timestamp()))
    updatedAt = factory.LazyFunction(lambda: int(datetime.utcnow().timestamp()))
    preferences = factory.LazyFunction(lambda: {
        "emailNotifications": True,
        "bidWinNotifications": True,
        "bidLossNotifications": True
    })
    isActive = True

class BidFactory(factory.Factory):
    class Meta:
        model = dict
    
    bidId = factory.LazyFunction(lambda: str(uuid.uuid4()))
    userId = factory.LazyFunction(lambda: str(uuid.uuid4()))
    ebayItemId = factory.Sequence(lambda n: f"item{n:09d}")
    maxBidAmount = factory.Faker('random_int', min=10000, max=100000)
    status = BidStatus.PENDING
    auctionEndTime = factory.LazyFunction(
        lambda: int((datetime.utcnow() + timedelta(hours=24)).timestamp())
    )
    createdAt = factory.LazyFunction(lambda: int(datetime.utcnow().timestamp()))
    updatedAt = factory.LazyFunction(lambda: int(datetime.utcnow().timestamp()))

class EbayItemFactory(factory.Factory):
    class Meta:
        model = dict
    
    ebayItemId = factory.Sequence(lambda n: f"item{n:09d}")
    title = factory.Faker('catch_phrase')
    currentPrice = factory.Faker('random_int', min=5000, max=50000)
    endTime = factory.LazyFunction(
        lambda: int((datetime.utcnow() + timedelta(hours=24)).timestamp())
    )
    imageUrls = factory.LazyFunction(lambda: ["https://example.com/image.jpg"])
    sellerInfo = factory.LazyFunction(lambda: {
        "sellerId": "seller123",
        "sellerName": "TestSeller",
        "feedbackScore": 1500,
        "feedbackPercentage": 99.5
    })
```

### Test Database Utilities
```python
# tests/utils/database.py
import boto3
import asyncio
from moto import mock_dynamodb

class TestDatabaseManager:
    def __init__(self):
        self.tables = {}
    
    async def setup_test_tables(self):
        """Setup test DynamoDB tables"""
        with mock_dynamodb():
            dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
            
            # Users table
            self.tables['users'] = dynamodb.create_table(
                TableName='test-Users',
                KeySchema=[
                    {'AttributeName': 'userId', 'KeyType': 'HASH'}
                ],
                AttributeDefinitions=[
                    {'AttributeName': 'userId', 'AttributeType': 'S'}
                ],
                BillingMode='PAY_PER_REQUEST'
            )
            
            # Bids table
            self.tables['bids'] = dynamodb.create_table(
                TableName='test-Bids',
                KeySchema=[
                    {'AttributeName': 'userId', 'KeyType': 'HASH'},
                    {'AttributeName': 'bidId', 'KeyType': 'RANGE'}
                ],
                AttributeDefinitions=[
                    {'AttributeName': 'userId', 'AttributeType': 'S'},
                    {'AttributeName': 'bidId', 'AttributeType': 'S'}
                ],
                BillingMode='PAY_PER_REQUEST'
            )
            
            # BidHistory table
            self.tables['bid_history'] = dynamodb.create_table(
                TableName='test-BidHistory',
                KeySchema=[
                    {'AttributeName': 'userId', 'KeyType': 'HASH'},
                    {'AttributeName': 'timestamp', 'KeyType': 'RANGE'}
                ],
                AttributeDefinitions=[
                    {'AttributeName': 'userId', 'AttributeType': 'S'},
                    {'AttributeName': 'timestamp', 'AttributeType': 'N'}
                ],
                BillingMode='PAY_PER_REQUEST'
            )
    
    async def seed_test_data(self, num_users: int = 10, num_bids_per_user: int = 5):
        """Seed test data into tables"""
        for i in range(num_users):
            user = UserFactory()
            self.tables['users'].put_item(Item=user)
            
            for j in range(num_bids_per_user):
                bid = BidFactory(userId=user['userId'])
                self.tables['bids'].put_item(Item=bid)
    
    async def cleanup_test_data(self):
        """Clean up test data"""
        for table in self.tables.values():
            # Scan and delete all items
            response = table.scan()
            
            with table.batch_writer() as batch:
                for item in response['Items']:
                    batch.delete_item(
                        Key={
                            key: item[key] 
                            for key in table.key_schema.keys()
                        }
                    )
```

## CI/CD Testing Pipeline

### GitHub Actions Workflow
```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  backend-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      - name: Run unit tests
        run: pytest tests/unit/ -v --cov=src --cov-report=xml
      
      - name: Run integration tests
        run: pytest tests/integration/ -v
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml

  frontend-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'pnpm'
      
      - name: Install dependencies
        run: pnpm install
      
      - name: Run unit tests
        run: pnpm test
      
      - name: Run component tests
        run: pnpm test:components
      
      - name: Build application
        run: pnpm build

  e2e-tests:
    runs-on: ubuntu-latest
    needs: [backend-tests, frontend-tests]
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'pnpm'
      
      - name: Install dependencies
        run: pnpm install
      
      - name: Install Playwright
        run: pnpm exec playwright install
      
      - name: Run E2E tests
        run: pnpm test:e2e
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/

  performance-tests:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install Locust
        run: pip install locust
      
      - name: Run performance tests
        run: |
          locust -f tests/performance/locustfile.py \
            --headless \
            --users 50 \
            --spawn-rate 5 \
            --run-time 5m \
            --host https://api-staging.ebay-sniper.com
```

## Test Reporting and Metrics

### Coverage Reporting
```python
# .coveragerc
[run]
source = src
omit = 
    */tests/*
    */venv/*
    */migrations/*
    */__pycache__/*

[report]
precision = 2
show_missing = True
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
```

### Quality Gates
```yaml
# sonar-project.properties
sonar.projectKey=ebay-sniper
sonar.organization=your-org

sonar.sources=src
sonar.tests=tests
sonar.python.coverage.reportPaths=coverage.xml

# Quality gates
sonar.coverage.exclusions=**/migrations/**,**/tests/**
sonar.qualitygate.wait=true

# Thresholds
sonar.coverage.minimum=80
sonar.duplicated_lines_density.maximum=3
sonar.maintainability_rating.minimum=A
sonar.reliability_rating.minimum=A
sonar.security_rating.minimum=A
```

## Testing Best Practices

### General Guidelines
1. **Test Naming**: Use descriptive test names that explain what is being tested
2. **Test Structure**: Follow Arrange-Act-Assert pattern
3. **Test Isolation**: Each test should be independent and not rely on other tests
4. **Mock External Dependencies**: Use mocks for external APIs and services
5. **Data Factories**: Use factories for consistent test data generation

### Performance Testing Guidelines
1. **Baseline Metrics**: Establish performance baselines before making changes
2. **Realistic Load**: Use realistic user behavior patterns in load tests
3. **Gradual Ramp-up**: Gradually increase load to identify breaking points
4. **Monitor Resources**: Watch CPU, memory, and database performance during tests

### Security Testing Guidelines
1. **Input Validation**: Test all input validation rules
2. **Authentication**: Verify authentication and authorization controls
3. **Data Protection**: Test encryption and data handling procedures
4. **Vulnerability Scanning**: Regular automated security scans