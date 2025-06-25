# Development Setup Guide

This guide provides step-by-step instructions for setting up a local development environment for the eBay Sniper application.

## Prerequisites

### System Requirements
- **Operating System**: macOS, Linux, or Windows 10/11 with WSL2
- **RAM**: Minimum 8GB, recommended 16GB
- **Storage**: At least 10GB free space
- **Network**: Stable internet connection for AWS services and eBay API

### Required Software

#### Development Tools
```bash
# Package managers
brew install git node python3 awscli # macOS
sudo apt-get install git nodejs npm python3 python3-pip awscli # Ubuntu/Debian
choco install git nodejs python awscli # Windows with Chocolatey
```

#### Version Requirements
- **Git**: Version 2.30+
- **Node.js**: Version 18.x LTS
- **Python**: Version 3.11+
- **AWS CLI**: Version 2.x
- **Docker**: Version 20.x+ (optional, for containerized development)

## Initial Setup

### 1. Repository Setup

#### Clone Repository
```bash
git clone https://github.com/your-org/ebay-sniper.git
cd ebay-sniper
```

#### Configure Git
```bash
git config user.name "Your Name"
git config user.email "your.email@example.com"

# Set up Git hooks
cp .githooks/* .git/hooks/
chmod +x .git/hooks/*
```

### 2. AWS Configuration

#### AWS CLI Setup
```bash
# Configure AWS CLI with your credentials
aws configure

# Verify configuration
aws sts get-caller-identity
```

#### AWS Profile Setup (Recommended)
```bash
# Create development profile
aws configure --profile ebay-sniper-dev
aws configure set region us-east-1 --profile ebay-sniper-dev

# Set default profile for project
export AWS_PROFILE=ebay-sniper-dev
echo 'export AWS_PROFILE=ebay-sniper-dev' >> ~/.bashrc  # or ~/.zshrc
```

### 3. Backend Development Setup

#### Python Environment Setup
```bash
cd backend

# Create virtual environment
python3 -m venv venv

# Activate virtual environment
source venv/bin/activate  # Linux/macOS
# venv\Scripts\activate     # Windows

# Upgrade pip
pip install --upgrade pip

# Install dependencies
pip install -r requirements.txt
pip install -r requirements-dev.txt
```

#### Environment Configuration
```bash
# Copy environment template
cp .env.template .env.local

# Edit environment variables
nano .env.local  # or use your preferred editor
```

#### Environment Variables (.env.local)
```bash
# Development Environment
ENVIRONMENT=dev
DEBUG=true
LOG_LEVEL=DEBUG

# AWS Configuration
AWS_REGION=us-east-1
AWS_PROFILE=ebay-sniper-dev

# DynamoDB Configuration (Local)
DYNAMODB_ENDPOINT=http://localhost:8000
DYNAMODB_TABLE_PREFIX=ebay-sniper-dev-

# eBay API Configuration (Sandbox)
EBAY_ENVIRONMENT=sandbox
EBAY_APP_ID=your_sandbox_app_id
EBAY_DEV_ID=your_sandbox_dev_id
EBAY_CERT_ID=your_sandbox_cert_id
EBAY_REDIRECT_URI=http://localhost:3000/ebay/callback

# Postmark Configuration (Test)
POSTMARK_API_KEY=your_test_postmark_key
POSTMARK_FROM_EMAIL=noreply@localhost

# Security Keys (Generate new ones)
JWT_SECRET_KEY=your_jwt_secret_key_min_32_chars

# CORS Configuration
CORS_ORIGINS=http://localhost:3000,http://127.0.0.1:3000
```

#### Generate Security Keys
```bash
# Generate JWT secret
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

### 4. Frontend Development Setup

#### Node.js Environment Setup
```bash
cd frontend

# Install pnpm (if not already installed)
npm install -g pnpm

# Install dependencies
pnpm install
```

#### Frontend Environment Configuration
```bash
# Copy environment template
cp .env.template .env.local

# Edit environment variables
nano .env.local
```

#### Frontend Environment Variables (.env.local)
```bash
# API Configuration
NEXT_PUBLIC_API_URL=http://localhost:8000

# AWS Cognito Configuration
NEXT_PUBLIC_AWS_REGION=us-east-1
NEXT_PUBLIC_USER_POOL_ID=us-east-1_xxxxxxxxx
NEXT_PUBLIC_USER_POOL_CLIENT_ID=your_client_id

# eBay Configuration
NEXT_PUBLIC_EBAY_ENVIRONMENT=sandbox

# Feature Flags
NEXT_PUBLIC_ENABLE_ANALYTICS=false
NEXT_PUBLIC_ENABLE_DEBUG=true
```

## Local Development Infrastructure

### 1. DynamoDB Local Setup

#### Install DynamoDB Local
```bash
# Create dynamodb directory
mkdir -p infrastructure/dynamodb
cd infrastructure/dynamodb

# Download DynamoDB Local
wget https://s3.us-west-2.amazonaws.com/dynamodb-local/dynamodb_local_latest.tar.gz
tar -xzf dynamodb_local_latest.tar.gz

# Create start script
cat > start-dynamodb.sh << 'EOF'
#!/bin/bash
java -Djava.library.path=./DynamoDBLocal_lib -jar DynamoDBLocal.jar -sharedDb -dbPath ./data
EOF

chmod +x start-dynamodb.sh
```

#### Start DynamoDB Local
```bash
# Start DynamoDB Local
./start-dynamodb.sh

# Verify it's running
aws dynamodb list-tables --endpoint-url http://localhost:8000
```

#### Create Local Tables
```bash
cd ../../backend

# Run table creation script
python scripts/create_local_tables.py
```

#### Table Creation Script
```python
# backend/scripts/create_local_tables.py
import boto3
import json
from botocore.exceptions import ClientError

def create_local_tables():
    dynamodb = boto3.client(
        'dynamodb',
        endpoint_url='http://localhost:8000',
        region_name='us-east-1',
        aws_access_key_id='dummy',
        aws_secret_access_key='dummy'
    )
    
    tables = [
        {
            'TableName': 'ebay-sniper-dev-Users',
            'KeySchema': [
                {'AttributeName': 'userId', 'KeyType': 'HASH'}
            ],
            'AttributeDefinitions': [
                {'AttributeName': 'userId', 'AttributeType': 'S'}
            ],
            'BillingMode': 'PAY_PER_REQUEST'
        },
        {
            'TableName': 'ebay-sniper-dev-Bids',
            'KeySchema': [
                {'AttributeName': 'userId', 'KeyType': 'HASH'},
                {'AttributeName': 'bidId', 'KeyType': 'RANGE'}
            ],
            'AttributeDefinitions': [
                {'AttributeName': 'userId', 'AttributeType': 'S'},
                {'AttributeName': 'bidId', 'AttributeType': 'S'}
            ],
            'BillingMode': 'PAY_PER_REQUEST'
        },
        {
            'TableName': 'ebay-sniper-dev-BidHistory',
            'KeySchema': [
                {'AttributeName': 'userId', 'KeyType': 'HASH'},
                {'AttributeName': 'timestamp', 'KeyType': 'RANGE'}
            ],
            'AttributeDefinitions': [
                {'AttributeName': 'userId', 'AttributeType': 'S'},
                {'AttributeName': 'timestamp', 'AttributeType': 'N'}
            ],
            'BillingMode': 'PAY_PER_REQUEST'
        }
    ]
    
    for table_config in tables:
        try:
            dynamodb.create_table(**table_config)
            print(f"Created table: {table_config['TableName']}")
        except ClientError as e:
            if e.response['Error']['Code'] == 'ResourceInUseException':
                print(f"Table already exists: {table_config['TableName']}")
            else:
                print(f"Error creating table {table_config['TableName']}: {e}")

if __name__ == "__main__":
    create_local_tables()
```

### 2. Local AWS Services (Optional)

#### LocalStack Setup
```bash
# Install LocalStack
pip install localstack

# Create LocalStack configuration
cat > docker-compose.localstack.yml << 'EOF'
version: '3.8'
services:
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      - SERVICES=dynamodb,s3,secretsmanager,kms,cognito-idp
      - DEBUG=1
      - DATA_DIR=/tmp/localstack/data
    volumes:
      - "./tmp/localstack:/tmp/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
EOF

# Start LocalStack
docker-compose -f docker-compose.localstack.yml up -d
```

## Development Workflow

### 1. Backend Development

#### Start Backend Server
```bash
cd backend
source venv/bin/activate

# Start development server
uvicorn src.main:app --reload --host 0.0.0.0 --port 8000

# Or using the development script
python scripts/dev-server.py
```

#### Development Server Script
```python
# backend/scripts/dev-server.py
import uvicorn
import os
from dotenv import load_dotenv

# Load environment variables
load_dotenv('.env.local')

if __name__ == "__main__":
    uvicorn.run(
        "src.main:app",
        host="0.0.0.0",
        port=8000,
        reload=True,
        reload_dirs=["src"],
        log_level="debug" if os.getenv("DEBUG") == "true" else "info"
    )
```

#### Backend Development Commands
```bash
# Run tests
pytest tests/ -v

# Run specific test file
pytest tests/unit/test_user_service.py -v

# Run with coverage
pytest tests/ --cov=src --cov-report=html

# Code formatting
black src/ tests/
isort src/ tests/

# Linting
flake8 src/ tests/
mypy src/

# Type checking
mypy src/ --strict
```

### 2. Frontend Development

#### Start Frontend Server
```bash
cd frontend

# Start development server
pnpm dev

# Start with specific port
pnpm dev --port 3001
```

#### Frontend Development Commands
```bash
# Run tests
pnpm test

# Run tests in watch mode
pnpm test:watch

# Run component tests
pnpm test:components

# Build for production
pnpm build

# Type checking
pnpm type-check

# Linting
pnpm lint

# Format code
pnpm format
```

### 3. Full Stack Development

#### Concurrent Development
```bash
# Install concurrently for running multiple processes
npm install -g concurrently

# Create development script
cat > dev-all.sh << 'EOF'
#!/bin/bash

# Start DynamoDB Local
cd infrastructure/dynamodb && ./start-dynamodb.sh &
DYNAMODB_PID=$!

# Wait for DynamoDB to start
sleep 5

# Start backend
cd ../../backend && source venv/bin/activate && python scripts/dev-server.py &
BACKEND_PID=$!

# Start frontend
cd ../frontend && pnpm dev &
FRONTEND_PID=$!

# Wait for Ctrl+C
echo "Development servers started. Press Ctrl+C to stop all services."
trap "kill $DYNAMODB_PID $BACKEND_PID $FRONTEND_PID" EXIT
wait
EOF

chmod +x dev-all.sh

# Run all development servers
./dev-all.sh
```

## IDE Configuration

### VS Code Setup

#### Recommended Extensions
```json
{
  "recommendations": [
    "ms-python.python",
    "ms-python.flake8",
    "ms-python.black-formatter",
    "ms-python.mypy-type-checker",
    "bradlc.vscode-tailwindcss",
    "esbenp.prettier-vscode",
    "ms-vscode.vscode-typescript-next",
    "amazonwebservices.aws-toolkit-vscode",
    "ms-vscode.vscode-json",
    "redhat.vscode-yaml"
  ]
}
```

#### VS Code Settings
```json
{
  "python.defaultInterpreterPath": "./backend/venv/bin/python",
  "python.linting.enabled": true,
  "python.linting.flake8Enabled": true,
  "python.formatting.provider": "black",
  "python.formatting.blackArgs": ["--line-length", "88"],
  "python.testing.pytestEnabled": true,
  "python.testing.pytestArgs": ["tests"],
  "typescript.preferences.importModuleSpecifier": "relative",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports": true
  },
  "files.exclude": {
    "**/__pycache__": true,
    "**/.pytest_cache": true,
    "**/node_modules": true,
    "**/.next": true
  }
}
```

#### Launch Configuration
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: FastAPI",
      "type": "python",
      "request": "launch",
      "program": "${workspaceFolder}/backend/scripts/dev-server.py",
      "console": "integratedTerminal",
      "envFile": "${workspaceFolder}/backend/.env.local"
    },
    {
      "name": "Python: Tests",
      "type": "python",
      "request": "launch",
      "module": "pytest",
      "args": ["tests/", "-v"],
      "console": "integratedTerminal",
      "cwd": "${workspaceFolder}/backend"
    }
  ]
}
```

### PyCharm Setup

#### Configuration Steps
1. Open PyCharm and select "Open" to open the backend directory
2. Configure Python interpreter: File → Settings → Project → Python Interpreter
3. Select the virtual environment: `backend/venv/bin/python`
4. Configure test runner: Settings → Tools → Python Integrated Tools → Testing → pytest
5. Configure code style: Settings → Editor → Code Style → Python → Use Black formatter

## Debugging

### Backend Debugging

#### Debug Configuration
```python
# backend/src/core/debug.py
import logging
import sys
from typing import Any

def setup_debug_logging():
    """Setup detailed logging for development"""
    logging.basicConfig(
        level=logging.DEBUG,
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=[
            logging.StreamHandler(sys.stdout),
            logging.FileHandler('debug.log')
        ]
    )

def debug_request(request: Any, response: Any = None):
    """Debug HTTP requests and responses"""
    logger = logging.getLogger(__name__)
    
    logger.debug(f"Request: {request.method} {request.url}")
    logger.debug(f"Headers: {dict(request.headers)}")
    
    if hasattr(request, 'body'):
        logger.debug(f"Body: {request.body}")
    
    if response:
        logger.debug(f"Response Status: {response.status_code}")
        logger.debug(f"Response Body: {response.body}")
```

#### Debugging Middleware
```python
# backend/src/middleware/debug.py
from fastapi import Request, Response
from fastapi.middleware.base import BaseHTTPMiddleware
import time
import logging

class DebugMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start_time = time.time()
        
        # Log request
        logging.debug(f"Request: {request.method} {request.url}")
        
        # Process request
        response = await call_next(request)
        
        # Log response
        process_time = time.time() - start_time
        logging.debug(f"Response: {response.status_code} in {process_time:.4f}s")
        
        return response
```

### Frontend Debugging

#### Debug Configuration
```typescript
// frontend/lib/debug.ts
export const isDebugMode = process.env.NODE_ENV === 'development' || 
                          process.env.NEXT_PUBLIC_ENABLE_DEBUG === 'true'

export function debugLog(message: string, data?: any) {
  if (isDebugMode) {
    console.log(`[DEBUG] ${message}`, data)
  }
}

export function debugError(message: string, error?: any) {
  if (isDebugMode) {
    console.error(`[DEBUG ERROR] ${message}`, error)
  }
}

export function debugApiCall(url: string, method: string, data?: any) {
  if (isDebugMode) {
    console.group(`[API] ${method} ${url}`)
    if (data) {
      console.log('Request data:', data)
    }
    console.groupEnd()
  }
}
```

#### React Developer Tools
```bash
# Install React Developer Tools browser extension
# Chrome: https://chrome.google.com/webstore/detail/react-developer-tools/
# Firefox: https://addons.mozilla.org/en-US/firefox/addon/react-devtools/

# Install Redux DevTools (if using Redux)
# Chrome: https://chrome.google.com/webstore/detail/redux-devtools/
```

## Testing in Development

### Running Tests

#### Backend Tests
```bash
cd backend

# Run all tests
pytest

# Run specific test categories
pytest tests/unit/
pytest tests/integration/
pytest tests/e2e/

# Run with coverage
pytest --cov=src --cov-report=html

# Run specific test
pytest tests/unit/test_user_service.py::TestUserService::test_create_user

# Run tests matching pattern
pytest -k "test_user"

# Run tests with verbose output
pytest -v -s
```

#### Frontend Tests
```bash
cd frontend

# Run all tests
pnpm test

# Run tests in watch mode
pnpm test:watch

# Run specific test file
pnpm test UserProfile.test.tsx

# Run tests with coverage
pnpm test:coverage

# Run E2E tests
pnpm test:e2e
```

### Test Data Management

#### Seed Development Data
```python
# backend/scripts/seed_dev_data.py
import asyncio
import boto3
from src.models.user import User
from src.models.bid import Bid
from tests.utils.factories import UserFactory, BidFactory

async def seed_development_data():
    """Seed development database with test data"""
    
    # Create test users
    users = []
    for i in range(10):
        user_data = UserFactory()
        user_data['email'] = f'testuser{i}@example.com'
        users.append(user_data)
    
    # Create test bids
    for user in users:
        for j in range(3):
            bid_data = BidFactory(userId=user['userId'])
            # Insert bid data
    
    print("Development data seeded successfully")

if __name__ == "__main__":
    asyncio.run(seed_development_data())
```

## Troubleshooting

### Common Issues

#### 1. Python Virtual Environment Issues
```bash
# Issue: venv not activating
# Solution: Recreate virtual environment
rm -rf venv
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Issue: Module not found
# Solution: Install in development mode
pip install -e .
```

#### 2. Node.js Dependencies Issues
```bash
# Issue: Package installation fails
# Solution: Clear cache and reinstall
pnpm store prune
rm -rf node_modules pnpm-lock.yaml
pnpm install

# Issue: TypeScript errors
# Solution: Restart TypeScript server
# In VS Code: Ctrl+Shift+P -> TypeScript: Restart TS Server
```

#### 3. AWS/DynamoDB Connection Issues
```bash
# Issue: Cannot connect to DynamoDB Local
# Solution: Check if DynamoDB Local is running
lsof -i :8000

# Restart DynamoDB Local
cd infrastructure/dynamodb
./start-dynamodb.sh

# Issue: AWS credentials not found
# Solution: Verify AWS configuration
aws configure list
aws sts get-caller-identity
```

#### 4. Port Conflicts
```bash
# Issue: Port already in use
# Solution: Find and kill process using port
lsof -ti:8000 | xargs kill -9  # Backend port
lsof -ti:3000 | xargs kill -9  # Frontend port

# Or use different ports
uvicorn src.main:app --port 8001
pnpm dev --port 3001
```

### Performance Issues

#### Backend Performance
```bash
# Monitor backend performance
pip install py-spy
py-spy top --pid $(pgrep -f uvicorn)

# Profile specific requests
pip install line_profiler
# Add @profile decorator to functions
kernprof -l -v src/api/endpoints/users.py
```

#### Frontend Performance
```bash
# Bundle analysis
pnpm build
pnpm analyze

# Lighthouse audit
lighthouse http://localhost:3000 --view
```

## Development Best Practices

### Code Quality

#### Pre-commit Hooks
```bash
# Install pre-commit
pip install pre-commit

# Setup pre-commit hooks
pre-commit install

# Run pre-commit on all files
pre-commit run --all-files
```

#### Pre-commit Configuration (.pre-commit-config.yaml)
```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
      - id: black
        language_version: python3.11

  - repo: https://github.com/pycqa/flake8
    rev: 6.0.0
    hooks:
      - id: flake8

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.0.1
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
```

### Documentation

#### Code Documentation
- Use docstrings for all functions and classes
- Include type hints for all function parameters
- Document API endpoints using FastAPI automatic documentation
- Maintain README files for each major component

#### API Documentation
```python
# Example of well-documented API endpoint
from fastapi import APIRouter, Depends, HTTPException, status
from typing import List, Optional

@router.post("/bids", response_model=BidResponse, status_code=201)
async def create_bid(
    bid_request: CreateBidRequest,
    current_user: User = Depends(get_current_user)
) -> BidResponse:
    """
    Create a new bid for an eBay auction.
    
    Args:
        bid_request: Bid creation request containing item ID and bid amount
        current_user: Currently authenticated user
        
    Returns:
        Created bid information
        
    Raises:
        HTTPException: 400 if bid amount is invalid
        HTTPException: 404 if eBay item not found
        HTTPException: 409 if user already has active bid for this item
    """
    pass
```

### Environment Management

#### Environment Switching
```bash
# Create scripts for easy environment switching
cat > scripts/env-dev.sh << 'EOF'
#!/bin/bash
export AWS_PROFILE=ebay-sniper-dev
export ENVIRONMENT=dev
echo "Switched to development environment"
EOF

cat > scripts/env-staging.sh << 'EOF'
#!/bin/bash
export AWS_PROFILE=ebay-sniper-staging
export ENVIRONMENT=staging
echo "Switched to staging environment"
EOF

chmod +x scripts/env-*.sh

# Usage
source scripts/env-dev.sh
```

This development setup guide provides a comprehensive foundation for developing the eBay Sniper application locally. Follow the steps in order and refer to the troubleshooting section if you encounter any issues.