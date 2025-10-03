# Templates de Testing y Calidad de Código

## Objetivo
Proporcionar templates estandarizados para implementar testing comprehensivo y asegurar la calidad del código en aplicaciones SAAS, incluyendo unit tests, integration tests, E2E tests y herramientas de análisis de código.

## 1. Unit Testing Templates

### pytest Configuration (pytest.ini)
```ini
[tool:pytest]
minversion = 6.0
addopts = 
    -ra
    --strict-markers
    --strict-config
    --cov=src
    --cov-report=term-missing:skip-covered
    --cov-report=html:htmlcov
    --cov-report=xml
    --cov-fail-under=80
    --junitxml=reports/junit.xml
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    integration: marks tests as integration tests
    unit: marks tests as unit tests
    smoke: marks tests as smoke tests
    regression: marks tests as regression tests
    api: marks tests as API tests
    database: marks tests as database tests
    external: marks tests that require external services
filterwarnings =
    error
    ignore::UserWarning
    ignore::DeprecationWarning
```

### conftest.py
```python
import pytest
import asyncio
import asyncpg
import aioredis
from unittest.mock import AsyncMock, MagicMock
from httpx import AsyncClient
from fastapi.testclient import TestClient
import os
from typing import AsyncGenerator, Generator
import tempfile
import shutil

# Test configuration
TEST_DATABASE_URL = "postgresql://test_user:test_pass@localhost:5432/test_db"
TEST_REDIS_URL = "redis://localhost:6379/1"

@pytest.fixture(scope="session")
def event_loop():
    """Create an instance of the default event loop for the test session."""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture(scope="session")
async def test_db_pool():
    """Create test database connection pool."""
    pool = await asyncpg.create_pool(TEST_DATABASE_URL, min_size=1, max_size=5)
    
    # Setup test database schema
    async with pool.acquire() as connection:
        await connection.execute("""
            CREATE SCHEMA IF NOT EXISTS test_users;
            CREATE TABLE IF NOT EXISTS test_users.users (
                id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
                email VARCHAR(255) UNIQUE NOT NULL,
                password_hash VARCHAR(255) NOT NULL,
                first_name VARCHAR(100) NOT NULL,
                last_name VARCHAR(100) NOT NULL,
                is_active BOOLEAN DEFAULT true,
                created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
            );
        """)
    
    yield pool
    
    # Cleanup
    async with pool.acquire() as connection:
        await connection.execute("DROP SCHEMA IF EXISTS test_users CASCADE;")
    
    await pool.close()

@pytest.fixture
async def db_connection(test_db_pool):
    """Get database connection for test."""
    async with test_db_pool.acquire() as connection:
        # Start transaction
        transaction = connection.transaction()
        await transaction.start()
        
        yield connection
        
        # Rollback transaction
        await transaction.rollback()

@pytest.fixture(scope="session")
async def redis_client():
    """Create Redis client for testing."""
    client = aioredis.from_url(TEST_REDIS_URL)
    
    yield client
    
    # Cleanup
    await client.flushdb()
    await client.close()

@pytest.fixture
def temp_dir():
    """Create temporary directory for tests."""
    temp_dir = tempfile.mkdtemp()
    yield temp_dir
    shutil.rmtree(temp_dir)

@pytest.fixture
def mock_email_service():
    """Mock email service."""
    mock = AsyncMock()
    mock.send_email.return_value = {"message_id": "test-123", "status": "sent"}
    return mock

@pytest.fixture
def mock_payment_service():
    """Mock payment service."""
    mock = AsyncMock()
    mock.create_payment_intent.return_value = {
        "id": "pi_test_123",
        "client_secret": "pi_test_123_secret",
        "status": "requires_payment_method"
    }
    mock.confirm_payment.return_value = {
        "id": "pi_test_123",
        "status": "succeeded"
    }
    return mock

@pytest.fixture
def sample_user_data():
    """Sample user data for testing."""
    return {
        "email": "test@example.com",
        "password": "SecurePassword123!",
        "first_name": "John",
        "last_name": "Doe"
    }

@pytest.fixture
async def authenticated_client(app, sample_user_data):
    """Create authenticated test client."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        # Register user
        response = await client.post("/api/auth/register", json=sample_user_data)
        assert response.status_code == 201
        
        # Login
        login_data = {
            "email": sample_user_data["email"],
            "password": sample_user_data["password"]
        }
        response = await client.post("/api/auth/login", json=login_data)
        assert response.status_code == 200
        
        token = response.json()["access_token"]
        client.headers.update({"Authorization": f"Bearer {token}"})
        
        yield client

@pytest.fixture(autouse=True)
def setup_test_environment(monkeypatch):
    """Setup test environment variables."""
    monkeypatch.setenv("ENVIRONMENT", "test")
    monkeypatch.setenv("DATABASE_URL", TEST_DATABASE_URL)
    monkeypatch.setenv("REDIS_URL", TEST_REDIS_URL)
    monkeypatch.setenv("JWT_SECRET_KEY", "test-secret-key")
    monkeypatch.setenv("EMAIL_BACKEND", "mock")
    monkeypatch.setenv("PAYMENT_BACKEND", "mock")

# Custom pytest markers
def pytest_configure(config):
    """Configure pytest with custom markers."""
    config.addinivalue_line(
        "markers", "slow: mark test as slow running"
    )
    config.addinivalue_line(
        "markers", "integration: mark test as integration test"
    )
    config.addinivalue_line(
        "markers", "unit: mark test as unit test"
    )

# Test data factories
class UserFactory:
    """Factory for creating test users."""
    
    @staticmethod
    def build(**kwargs):
        """Build user data."""
        default_data = {
            "email": "test@example.com",
            "password": "SecurePassword123!",
            "first_name": "John",
            "last_name": "Doe"
        }
        default_data.update(kwargs)
        return default_data
    
    @staticmethod
    async def create(db_connection, **kwargs):
        """Create user in database."""
        user_data = UserFactory.build(**kwargs)
        
        query = """
            INSERT INTO test_users.users (email, password_hash, first_name, last_name)
            VALUES ($1, $2, $3, $4)
            RETURNING id, email, first_name, last_name, created_at
        """
        
        row = await db_connection.fetchrow(
            query,
            user_data["email"],
            "hashed_password",  # In real tests, hash the password
            user_data["first_name"],
            user_data["last_name"]
        )
        
        return dict(row)

@pytest.fixture
def user_factory():
    """User factory fixture."""
    return UserFactory
```

### Unit Test Examples

#### test_user_service.py
```python
import pytest
from unittest.mock import AsyncMock, patch
from src.services.user_service import UserService
from src.models.user import User
from src.exceptions import UserNotFoundError, EmailAlreadyExistsError

@pytest.mark.unit
class TestUserService:
    """Test cases for UserService."""
    
    @pytest.fixture
    def user_service(self, db_connection, redis_client, mock_email_service):
        """Create UserService instance."""
        return UserService(
            db_connection=db_connection,
            redis_client=redis_client,
            email_service=mock_email_service
        )
    
    @pytest.mark.asyncio
    async def test_create_user_success(self, user_service, sample_user_data):
        """Test successful user creation."""
        # Act
        user = await user_service.create_user(sample_user_data)
        
        # Assert
        assert user.email == sample_user_data["email"]
        assert user.first_name == sample_user_data["first_name"]
        assert user.last_name == sample_user_data["last_name"]
        assert user.is_active is True
        assert user.id is not None
    
    @pytest.mark.asyncio
    async def test_create_user_duplicate_email(self, user_service, sample_user_data, user_factory, db_connection):
        """Test user creation with duplicate email."""
        # Arrange
        await user_factory.create(db_connection, email=sample_user_data["email"])
        
        # Act & Assert
        with pytest.raises(EmailAlreadyExistsError):
            await user_service.create_user(sample_user_data)
    
    @pytest.mark.asyncio
    async def test_get_user_by_id_success(self, user_service, user_factory, db_connection):
        """Test getting user by ID."""
        # Arrange
        created_user = await user_factory.create(db_connection)
        
        # Act
        user = await user_service.get_user_by_id(created_user["id"])
        
        # Assert
        assert user.id == created_user["id"]
        assert user.email == created_user["email"]
    
    @pytest.mark.asyncio
    async def test_get_user_by_id_not_found(self, user_service):
        """Test getting non-existent user."""
        # Arrange
        non_existent_id = "00000000-0000-0000-0000-000000000000"
        
        # Act & Assert
        with pytest.raises(UserNotFoundError):
            await user_service.get_user_by_id(non_existent_id)
    
    @pytest.mark.asyncio
    async def test_update_user_success(self, user_service, user_factory, db_connection):
        """Test successful user update."""
        # Arrange
        created_user = await user_factory.create(db_connection)
        update_data = {"first_name": "Jane", "last_name": "Smith"}
        
        # Act
        updated_user = await user_service.update_user(created_user["id"], update_data)
        
        # Assert
        assert updated_user.first_name == "Jane"
        assert updated_user.last_name == "Smith"
        assert updated_user.email == created_user["email"]  # Unchanged
    
    @pytest.mark.asyncio
    async def test_delete_user_success(self, user_service, user_factory, db_connection):
        """Test successful user deletion."""
        # Arrange
        created_user = await user_factory.create(db_connection)
        
        # Act
        result = await user_service.delete_user(created_user["id"])
        
        # Assert
        assert result is True
        
        # Verify user is deleted
        with pytest.raises(UserNotFoundError):
            await user_service.get_user_by_id(created_user["id"])
    
    @pytest.mark.asyncio
    async def test_send_verification_email(self, user_service, user_factory, db_connection, mock_email_service):
        """Test sending verification email."""
        # Arrange
        created_user = await user_factory.create(db_connection)
        
        # Act
        result = await user_service.send_verification_email(created_user["id"])
        
        # Assert
        assert result is True
        mock_email_service.send_email.assert_called_once()
        
        # Verify email content
        call_args = mock_email_service.send_email.call_args
        assert call_args[1]["to"] == created_user["email"]
        assert "verification" in call_args[1]["subject"].lower()
    
    @pytest.mark.asyncio
    @patch('src.services.user_service.generate_verification_token')
    async def test_verify_user_email(self, mock_generate_token, user_service, user_factory, db_connection):
        """Test email verification."""
        # Arrange
        created_user = await user_factory.create(db_connection)
        verification_token = "test-token-123"
        mock_generate_token.return_value = verification_token
        
        # Send verification email first
        await user_service.send_verification_email(created_user["id"])
        
        # Act
        result = await user_service.verify_email(verification_token)
        
        # Assert
        assert result is True
        
        # Verify user is marked as verified
        user = await user_service.get_user_by_id(created_user["id"])
        assert user.is_verified is True

@pytest.mark.unit
class TestUserModel:
    """Test cases for User model."""
    
    def test_user_creation(self, sample_user_data):
        """Test User model creation."""
        # Act
        user = User(**sample_user_data)
        
        # Assert
        assert user.email == sample_user_data["email"]
        assert user.first_name == sample_user_data["first_name"]
        assert user.last_name == sample_user_data["last_name"]
    
    def test_user_full_name_property(self, sample_user_data):
        """Test User full_name property."""
        # Arrange
        user = User(**sample_user_data)
        
        # Act
        full_name = user.full_name
        
        # Assert
        expected = f"{sample_user_data['first_name']} {sample_user_data['last_name']}"
        assert full_name == expected
    
    def test_user_to_dict(self, sample_user_data):
        """Test User to_dict method."""
        # Arrange
        user = User(**sample_user_data)
        
        # Act
        user_dict = user.to_dict()
        
        # Assert
        assert isinstance(user_dict, dict)
        assert user_dict["email"] == sample_user_data["email"]
        assert "password" not in user_dict  # Password should not be included
```

## 2. Integration Testing

### test_api_integration.py
```python
import pytest
from httpx import AsyncClient
from fastapi import status
import json

@pytest.mark.integration
class TestUserAPIIntegration:
    """Integration tests for User API endpoints."""
    
    @pytest.mark.asyncio
    async def test_user_registration_flow(self, app, sample_user_data):
        """Test complete user registration flow."""
        async with AsyncClient(app=app, base_url="http://test") as client:
            # Register user
            response = await client.post("/api/auth/register", json=sample_user_data)
            
            # Assert registration success
            assert response.status_code == status.HTTP_201_CREATED
            user_data = response.json()
            assert user_data["email"] == sample_user_data["email"]
            assert user_data["is_verified"] is False
            
            # Login with new user
            login_data = {
                "email": sample_user_data["email"],
                "password": sample_user_data["password"]
            }
            response = await client.post("/api/auth/login", json=login_data)
            
            # Assert login success
            assert response.status_code == status.HTTP_200_OK
            token_data = response.json()
            assert "access_token" in token_data
            assert "refresh_token" in token_data
    
    @pytest.mark.asyncio
    async def test_protected_endpoint_access(self, authenticated_client):
        """Test accessing protected endpoints."""
        # Get user profile
        response = await authenticated_client.get("/api/users/me")
        
        # Assert success
        assert response.status_code == status.HTTP_200_OK
        user_data = response.json()
        assert "email" in user_data
        assert "id" in user_data
    
    @pytest.mark.asyncio
    async def test_user_profile_update(self, authenticated_client):
        """Test user profile update."""
        # Update profile
        update_data = {
            "first_name": "Jane",
            "last_name": "Smith"
        }
        response = await authenticated_client.put("/api/users/me", json=update_data)
        
        # Assert update success
        assert response.status_code == status.HTTP_200_OK
        updated_user = response.json()
        assert updated_user["first_name"] == "Jane"
        assert updated_user["last_name"] == "Smith"
        
        # Verify changes persist
        response = await authenticated_client.get("/api/users/me")
        user_data = response.json()
        assert user_data["first_name"] == "Jane"
        assert user_data["last_name"] == "Smith"
    
    @pytest.mark.asyncio
    async def test_password_change_flow(self, authenticated_client, sample_user_data):
        """Test password change flow."""
        # Change password
        password_data = {
            "current_password": sample_user_data["password"],
            "new_password": "NewSecurePassword123!"
        }
        response = await authenticated_client.post("/api/auth/change-password", json=password_data)
        
        # Assert password change success
        assert response.status_code == status.HTTP_200_OK
        
        # Verify old password no longer works
        async with AsyncClient(app=authenticated_client.app, base_url="http://test") as client:
            login_data = {
                "email": sample_user_data["email"],
                "password": sample_user_data["password"]
            }
            response = await client.post("/api/auth/login", json=login_data)
            assert response.status_code == status.HTTP_401_UNAUTHORIZED
            
            # Verify new password works
            login_data["password"] = "NewSecurePassword123!"
            response = await client.post("/api/auth/login", json=login_data)
            assert response.status_code == status.HTTP_200_OK

@pytest.mark.integration
@pytest.mark.database
class TestDatabaseIntegration:
    """Integration tests for database operations."""
    
    @pytest.mark.asyncio
    async def test_user_crud_operations(self, db_connection, user_factory):
        """Test complete CRUD operations for users."""
        # Create
        user_data = user_factory.build(email="crud@example.com")
        created_user = await user_factory.create(db_connection, **user_data)
        assert created_user["email"] == "crud@example.com"
        
        # Read
        query = "SELECT * FROM test_users.users WHERE id = $1"
        row = await db_connection.fetchrow(query, created_user["id"])
        assert row["email"] == "crud@example.com"
        
        # Update
        update_query = """
            UPDATE test_users.users 
            SET first_name = $1 
            WHERE id = $2 
            RETURNING first_name
        """
        updated_row = await db_connection.fetchrow(update_query, "Updated", created_user["id"])
        assert updated_row["first_name"] == "Updated"
        
        # Delete
        delete_query = "DELETE FROM test_users.users WHERE id = $1"
        await db_connection.execute(delete_query, created_user["id"])
        
        # Verify deletion
        row = await db_connection.fetchrow(query, created_user["id"])
        assert row is None
    
    @pytest.mark.asyncio
    async def test_database_constraints(self, db_connection, user_factory):
        """Test database constraints."""
        # Create user
        user_data = user_factory.build(email="constraint@example.com")
        await user_factory.create(db_connection, **user_data)
        
        # Try to create duplicate email
        with pytest.raises(Exception):  # Should raise unique constraint violation
            await user_factory.create(db_connection, **user_data)
    
    @pytest.mark.asyncio
    async def test_transaction_rollback(self, db_connection, user_factory):
        """Test transaction rollback behavior."""
        # Start transaction
        async with db_connection.transaction():
            # Create user within transaction
            user_data = user_factory.build(email="transaction@example.com")
            created_user = await user_factory.create(db_connection, **user_data)
            
            # Verify user exists within transaction
            query = "SELECT * FROM test_users.users WHERE id = $1"
            row = await db_connection.fetchrow(query, created_user["id"])
            assert row is not None
            
            # Raise exception to trigger rollback
            raise Exception("Test rollback")
        
        # Verify user doesn't exist after rollback (this won't work in our fixture setup)
        # In real tests, you'd need a different setup to test this properly
```

## 3. End-to-End Testing

### playwright.config.js
```javascript
// Playwright configuration for E2E testing
const { defineConfig, devices } = require('@playwright/test');

module.exports = defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['junit', { outputFile: 'reports/e2e-results.xml' }],
    ['json', { outputFile: 'reports/e2e-results.json' }]
  ],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    actionTimeout: 10000,
    navigationTimeout: 30000,
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] },
    },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120000,
  },
});
```

### e2e/auth.spec.js
```javascript
const { test, expect } = require('@playwright/test');

test.describe('Authentication Flow', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/');
  });

  test('user can register and login', async ({ page }) => {
    // Navigate to registration
    await page.click('text=Sign Up');
    await expect(page).toHaveURL('/register');

    // Fill registration form
    const timestamp = Date.now();
    const email = `test${timestamp}@example.com`;
    
    await page.fill('[data-testid="email-input"]', email);
    await page.fill('[data-testid="password-input"]', 'SecurePassword123!');
    await page.fill('[data-testid="first-name-input"]', 'John');
    await page.fill('[data-testid="last-name-input"]', 'Doe');
    
    // Submit registration
    await page.click('[data-testid="register-button"]');
    
    // Verify registration success
    await expect(page.locator('[data-testid="success-message"]')).toBeVisible();
    await expect(page).toHaveURL('/login');
    
    // Login with new account
    await page.fill('[data-testid="email-input"]', email);
    await page.fill('[data-testid="password-input"]', 'SecurePassword123!');
    await page.click('[data-testid="login-button"]');
    
    // Verify login success
    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('[data-testid="user-menu"]')).toBeVisible();
  });

  test('shows validation errors for invalid input', async ({ page }) => {
    await page.click('text=Sign Up');
    
    // Try to submit empty form
    await page.click('[data-testid="register-button"]');
    
    // Verify validation errors
    await expect(page.locator('[data-testid="email-error"]')).toBeVisible();
    await expect(page.locator('[data-testid="password-error"]')).toBeVisible();
    
    // Fill invalid email
    await page.fill('[data-testid="email-input"]', 'invalid-email');
    await page.click('[data-testid="register-button"]');
    
    await expect(page.locator('[data-testid="email-error"]')).toContainText('valid email');
  });

  test('handles login with invalid credentials', async ({ page }) => {
    await page.click('text=Sign In');
    
    // Try login with invalid credentials
    await page.fill('[data-testid="email-input"]', 'nonexistent@example.com');
    await page.fill('[data-testid="password-input"]', 'wrongpassword');
    await page.click('[data-testid="login-button"]');
    
    // Verify error message
    await expect(page.locator('[data-testid="error-message"]')).toBeVisible();
    await expect(page.locator('[data-testid="error-message"]')).toContainText('Invalid credentials');
  });

  test('user can logout', async ({ page }) => {
    // Login first (you might want to create a helper function for this)
    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', 'existing@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-button"]');
    
    await expect(page).toHaveURL('/dashboard');
    
    // Logout
    await page.click('[data-testid="user-menu"]');
    await page.click('[data-testid="logout-button"]');
    
    // Verify logout
    await expect(page).toHaveURL('/');
    await expect(page.locator('text=Sign In')).toBeVisible();
  });
});
```

### e2e/dashboard.spec.js
```javascript
const { test, expect } = require('@playwright/test');

test.describe('Dashboard Functionality', () => {
  test.beforeEach(async ({ page }) => {
    // Login before each test
    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-button"]');
    await expect(page).toHaveURL('/dashboard');
  });

  test('displays user dashboard correctly', async ({ page }) => {
    // Verify dashboard elements
    await expect(page.locator('[data-testid="welcome-message"]')).toBeVisible();
    await expect(page.locator('[data-testid="stats-cards"]')).toBeVisible();
    await expect(page.locator('[data-testid="recent-activity"]')).toBeVisible();
  });

  test('user can update profile', async ({ page }) => {
    // Navigate to profile
    await page.click('[data-testid="user-menu"]');
    await page.click('[data-testid="profile-link"]');
    
    await expect(page).toHaveURL('/profile');
    
    // Update profile information
    await page.fill('[data-testid="first-name-input"]', 'Jane');
    await page.fill('[data-testid="last-name-input"]', 'Smith');
    await page.click('[data-testid="save-profile-button"]');
    
    // Verify success message
    await expect(page.locator('[data-testid="success-message"]')).toBeVisible();
    
    // Verify changes are reflected
    await expect(page.locator('[data-testid="first-name-input"]')).toHaveValue('Jane');
    await expect(page.locator('[data-testid="last-name-input"]')).toHaveValue('Smith');
  });

  test('user can change password', async ({ page }) => {
    await page.click('[data-testid="user-menu"]');
    await page.click('[data-testid="profile-link"]');
    await page.click('[data-testid="change-password-tab"]');
    
    // Fill password change form
    await page.fill('[data-testid="current-password-input"]', 'password123');
    await page.fill('[data-testid="new-password-input"]', 'NewPassword123!');
    await page.fill('[data-testid="confirm-password-input"]', 'NewPassword123!');
    await page.click('[data-testid="change-password-button"]');
    
    // Verify success
    await expect(page.locator('[data-testid="success-message"]')).toBeVisible();
  });

  test('handles API errors gracefully', async ({ page }) => {
    // Mock API error
    await page.route('/api/users/me', route => {
      route.fulfill({
        status: 500,
        contentType: 'application/json',
        body: JSON.stringify({ error: 'Internal server error' })
      });
    });
    
    await page.reload();
    
    // Verify error handling
    await expect(page.locator('[data-testid="error-banner"]')).toBeVisible();
    await expect(page.locator('[data-testid="error-banner"]')).toContainText('Something went wrong');
  });
});
```

## 4. Performance Testing

### locustfile.py
```python
from locust import HttpUser, task, between
import json
import random
import string

class SaaSPlatformUser(HttpUser):
    wait_time = between(1, 3)
    
    def on_start(self):
        """Setup user session."""
        self.register_and_login()
    
    def register_and_login(self):
        """Register and login user."""
        # Generate unique user data
        timestamp = str(random.randint(100000, 999999))
        self.email = f"test{timestamp}@example.com"
        self.password = "TestPassword123!"
        
        # Register user
        user_data = {
            "email": self.email,
            "password": self.password,
            "first_name": "Test",
            "last_name": "User"
        }
        
        response = self.client.post("/api/auth/register", json=user_data)
        if response.status_code == 201:
            # Login
            login_data = {
                "email": self.email,
                "password": self.password
            }
            
            response = self.client.post("/api/auth/login", json=login_data)
            if response.status_code == 200:
                token = response.json()["access_token"]
                self.client.headers.update({"Authorization": f"Bearer {token}"})
    
    @task(3)
    def view_dashboard(self):
        """View user dashboard."""
        self.client.get("/api/dashboard")
    
    @task(2)
    def view_profile(self):
        """View user profile."""
        self.client.get("/api/users/me")
    
    @task(1)
    def update_profile(self):
        """Update user profile."""
        update_data = {
            "first_name": f"Updated{random.randint(1, 100)}",
            "last_name": "User"
        }
        self.client.put("/api/users/me", json=update_data)
    
    @task(1)
    def search_users(self):
        """Search users."""
        query = random.choice(["test", "user", "admin", "john"])
        self.client.get(f"/api/users/search?q={query}")
    
    @task(1)
    def create_organization(self):
        """Create organization."""
        org_data = {
            "name": f"Test Org {random.randint(1, 1000)}",
            "description": "Test organization"
        }
        self.client.post("/api/organizations", json=org_data)
    
    @task(2)
    def list_organizations(self):
        """List user organizations."""
        self.client.get("/api/organizations")

class AdminUser(HttpUser):
    wait_time = between(2, 5)
    weight = 1  # Lower weight for admin users
    
    def on_start(self):
        """Login as admin."""
        login_data = {
            "email": "admin@example.com",
            "password": "AdminPassword123!"
        }
        
        response = self.client.post("/api/auth/login", json=login_data)
        if response.status_code == 200:
            token = response.json()["access_token"]
            self.client.headers.update({"Authorization": f"Bearer {token}"})
    
    @task(2)
    def view_admin_dashboard(self):
        """View admin dashboard."""
        self.client.get("/api/admin/dashboard")
    
    @task(1)
    def list_all_users(self):
        """List all users."""
        self.client.get("/api/admin/users")
    
    @task(1)
    def view_system_metrics(self):
        """View system metrics."""
        self.client.get("/api/admin/metrics")
```

## 5. Code Quality Tools

### .pre-commit-config.yaml
```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: check-json
      - id: check-merge-conflict
      - id: debug-statements
      - id: check-docstring-first

  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black
        language_version: python3.11

  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort
        args: ["--profile", "black"]

  - repo: https://github.com/pycqa/flake8
    rev: 6.0.0
    hooks:
      - id: flake8
        additional_dependencies: [flake8-docstrings, flake8-import-order]
        args: [--max-line-length=88, --extend-ignore=E203,W503]

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.3.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
        args: [--ignore-missing-imports]

  - repo: https://github.com/pycqa/bandit
    rev: 1.7.5
    hooks:
      - id: bandit
        args: ["-r", "src/", "-f", "json", "-o", "reports/bandit-report.json"]

  - repo: https://github.com/Lucas-C/pre-commit-hooks-safety
    rev: v1.3.2
    hooks:
      - id: python-safety-dependencies-check

  - repo: local
    hooks:
      - id: pytest-check
        name: pytest-check
        entry: pytest
        language: system
        pass_filenames: false
        always_run: true
        args: [--tb=short, -q]
```

### pyproject.toml
```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "saas-platform"
version = "1.0.0"
description = "SAAS Platform"
authors = [{name = "Your Name", email = "your.email@example.com"}]
license = {text = "MIT"}
readme = "README.md"
requires-python = ">=3.11"
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.11",
]

dependencies = [
    "fastapi>=0.100.0",
    "uvicorn[standard]>=0.22.0",
    "pydantic>=2.0.0",
    "sqlalchemy>=2.0.0",
    "asyncpg>=0.28.0",
    "redis>=4.5.0",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "python-multipart>=0.0.6",
    "email-validator>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-asyncio>=0.21.0",
    "pytest-cov>=4.1.0",
    "pytest-mock>=3.11.0",
    "httpx>=0.24.0",
    "black>=23.3.0",
    "isort>=5.12.0",
    "flake8>=6.0.0",
    "mypy>=1.3.0",
    "bandit>=1.7.5",
    "safety>=2.3.0",
    "pre-commit>=3.3.0",
]

test = [
    "pytest>=7.4.0",
    "pytest-asyncio>=0.21.0",
    "pytest-cov>=4.1.0",
    "pytest-mock>=3.11.0",
    "httpx>=0.24.0",
    "factory-boy>=3.2.0",
    "faker>=18.0.0",
]

e2e = [
    "playwright>=1.35.0",
    "pytest-playwright>=0.3.3",
]

performance = [
    "locust>=2.15.0",
]

[tool.setuptools.packages.find]
where = ["src"]

[tool.black]
line-length = 88
target-version = ['py311']
include = '\.pyi?$'
extend-exclude = '''
/(
  # directories
  \.eggs
  | \.git
  | \.hg
  | \.mypy_cache
  | \.tox
  | \.venv
  | build
  | dist
)/
'''

[tool.isort]
profile = "black"
multi_line_output = 3
line_length = 88
known_first_party = ["src"]

[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
disallow_untyped_decorators = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_no_return = true
warn_unreachable = true
strict_equality = true

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false

[tool.pytest.ini_options]
minversion = "7.0"
addopts = [
    "-ra",
    "--strict-markers",
    "--strict-config",
    "--cov=src",
    "--cov-report=term-missing:skip-covered",
    "--cov-report=html:htmlcov",
    "--cov-report=xml",
    "--cov-fail-under=80",
]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
    "e2e: marks tests as end-to-end tests",
    "smoke: marks tests as smoke tests",
    "regression: marks tests as regression tests",
]

[tool.coverage.run]
source = ["src"]
omit = [
    "*/tests/*",
    "*/test_*.py",
    "*/conftest.py",
    "*/__init__.py",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "if self.debug:",
    "if settings.DEBUG",
    "raise AssertionError",
    "raise NotImplementedError",
    "if 0:",
    "if __name__ == .__main__.:",
    "class .*\\bProtocol\\):",
    "@(abc\\.)?abstractmethod",
]

[tool.bandit]
exclude_dirs = ["tests", "test_*.py"]
skips = ["B101", "B601"]
```

### Makefile
```makefile
.PHONY: help install install-dev test test-unit test-integration test-e2e test-performance lint format type-check security-check clean coverage

help: ## Show this help message
	@echo 'Usage: make [target]'
	@echo ''
	@echo 'Targets:'
	@awk 'BEGIN {FS = ":.*?## "} /^[a-zA-Z_-]+:.*?## / {printf "  %-20s %s\n", $$1, $$2}' $(MAKEFILE_LIST)

install: ## Install production dependencies
	pip install -e .

install-dev: ## Install development dependencies
	pip install -e ".[dev,test,e2e,performance]"
	pre-commit install

test: ## Run all tests
	pytest

test-unit: ## Run unit tests only
	pytest -m "unit" --tb=short

test-integration: ## Run integration tests only
	pytest -m "integration" --tb=short

test-e2e: ## Run end-to-end tests
	playwright install
	pytest -m "e2e" --tb=short

test-performance: ## Run performance tests
	locust --headless --users 10 --spawn-rate 2 --run-time 1m --host http://localhost:8000

lint: ## Run linting
	flake8 src tests
	black --check src tests
	isort --check-only src tests

format: ## Format code
	black src tests
	isort src tests

type-check: ## Run type checking
	mypy src

security-check: ## Run security checks
	bandit -r src/ -f json -o reports/bandit-report.json
	safety check

clean: ## Clean up generated files
	rm -rf build/
	rm -rf dist/
	rm -rf *.egg-info/
	rm -rf .pytest_cache/
	rm -rf .coverage
	rm -rf htmlcov/
	rm -rf reports/
	find . -type d -name __pycache__ -delete
	find . -type f -name "*.pyc" -delete

coverage: ## Generate coverage report
	pytest --cov=src --cov-report=html --cov-report=term-missing
	@echo "Coverage report generated in htmlcov/index.html"

ci: lint type-check security-check test ## Run all CI checks

dev-setup: install-dev ## Setup development environment
	@echo "Development environment setup complete!"
	@echo "Run 'make test' to run tests"
	@echo "Run 'make lint' to check code style"
```

## Mejores Prácticas de Testing

### 1. Estructura de Tests
- Organizar tests por funcionalidad
- Usar fixtures para setup común
- Mantener tests independientes
- Usar factories para datos de prueba

### 2. Cobertura de Código
- Mantener cobertura > 80%
- Cubrir casos edge y errores
- Probar tanto casos exitosos como fallidos
- Usar mutation testing para validar calidad

### 3. Performance Testing
- Probar bajo carga normal y pico
- Monitorear tiempos de respuesta
- Validar escalabilidad
- Probar degradación graceful

### 4. Calidad de Código
- Usar linters y formatters
- Implementar type checking
- Realizar security scanning
- Mantener documentación actualizada

Esta documentación proporciona una base completa para implementar testing y asegurar la calidad del código en aplicaciones SAAS.