# Templates de Seguridad

## Objetivo
Proporcionar templates estandarizados para implementar seguridad robusta en aplicaciones SAAS, incluyendo autenticación, autorización, encriptación, y protección contra vulnerabilidades comunes.

## 1. Autenticación JWT

### jwt_auth.py
```python
import jwt
import bcrypt
import secrets
from datetime import datetime, timedelta
from typing import Optional, Dict, Any
from functools import wraps
from flask import request, jsonify, current_app
import redis
import logging

logger = logging.getLogger(__name__)

class JWTManager:
    def __init__(self, secret_key: str, algorithm: str = 'HS256', 
                 access_token_expires: int = 3600, 
                 refresh_token_expires: int = 86400 * 7):
        self.secret_key = secret_key
        self.algorithm = algorithm
        self.access_token_expires = access_token_expires
        self.refresh_token_expires = refresh_token_expires
        self.redis_client = redis.Redis(host='redis', port=6379, db=0)
    
    def generate_tokens(self, user_id: str, user_data: Dict[str, Any]) -> Dict[str, str]:
        """Generate access and refresh tokens"""
        now = datetime.utcnow()
        
        # Access token payload
        access_payload = {
            'user_id': user_id,
            'email': user_data.get('email'),
            'roles': user_data.get('roles', []),
            'permissions': user_data.get('permissions', []),
            'iat': now,
            'exp': now + timedelta(seconds=self.access_token_expires),
            'type': 'access'
        }
        
        # Refresh token payload
        refresh_jti = secrets.token_urlsafe(32)
        refresh_payload = {
            'user_id': user_id,
            'jti': refresh_jti,
            'iat': now,
            'exp': now + timedelta(seconds=self.refresh_token_expires),
            'type': 'refresh'
        }
        
        # Generate tokens
        access_token = jwt.encode(access_payload, self.secret_key, algorithm=self.algorithm)
        refresh_token = jwt.encode(refresh_payload, self.secret_key, algorithm=self.algorithm)
        
        # Store refresh token in Redis
        self.redis_client.setex(
            f"refresh_token:{refresh_jti}",
            self.refresh_token_expires,
            user_id
        )
        
        logger.info(f"Tokens generated for user {user_id}")
        
        return {
            'access_token': access_token,
            'refresh_token': refresh_token,
            'expires_in': self.access_token_expires
        }
    
    def verify_token(self, token: str, token_type: str = 'access') -> Optional[Dict[str, Any]]:
        """Verify and decode JWT token"""
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            
            if payload.get('type') != token_type:
                logger.warning(f"Invalid token type: expected {token_type}, got {payload.get('type')}")
                return None
            
            # Check if refresh token is blacklisted
            if token_type == 'refresh':
                jti = payload.get('jti')
                if not self.redis_client.exists(f"refresh_token:{jti}"):
                    logger.warning(f"Refresh token {jti} not found or expired")
                    return None
            
            return payload
            
        except jwt.ExpiredSignatureError:
            logger.warning("Token has expired")
            return None
        except jwt.InvalidTokenError as e:
            logger.warning(f"Invalid token: {str(e)}")
            return None
    
    def refresh_access_token(self, refresh_token: str) -> Optional[Dict[str, str]]:
        """Generate new access token using refresh token"""
        payload = self.verify_token(refresh_token, 'refresh')
        if not payload:
            return None
        
        user_id = payload['user_id']
        
        # Get user data (you'll need to implement this)
        user_data = self.get_user_data(user_id)
        if not user_data:
            return None
        
        # Generate new access token
        now = datetime.utcnow()
        access_payload = {
            'user_id': user_id,
            'email': user_data.get('email'),
            'roles': user_data.get('roles', []),
            'permissions': user_data.get('permissions', []),
            'iat': now,
            'exp': now + timedelta(seconds=self.access_token_expires),
            'type': 'access'
        }
        
        access_token = jwt.encode(access_payload, self.secret_key, algorithm=self.algorithm)
        
        logger.info(f"Access token refreshed for user {user_id}")
        
        return {
            'access_token': access_token,
            'expires_in': self.access_token_expires
        }
    
    def revoke_refresh_token(self, refresh_token: str) -> bool:
        """Revoke refresh token"""
        payload = self.verify_token(refresh_token, 'refresh')
        if not payload:
            return False
        
        jti = payload.get('jti')
        self.redis_client.delete(f"refresh_token:{jti}")
        
        logger.info(f"Refresh token {jti} revoked")
        return True
    
    def get_user_data(self, user_id: str) -> Optional[Dict[str, Any]]:
        """Get user data from database - implement this method"""
        # This should fetch user data from your database
        pass

# Authentication decorators
jwt_manager = JWTManager(current_app.config['JWT_SECRET_KEY'])

def token_required(f):
    """Decorator to require valid JWT token"""
    @wraps(f)
    def decorated(*args, **kwargs):
        token = None
        
        # Get token from header
        if 'Authorization' in request.headers:
            auth_header = request.headers['Authorization']
            try:
                token = auth_header.split(" ")[1]  # Bearer <token>
            except IndexError:
                return jsonify({'message': 'Invalid token format'}), 401
        
        if not token:
            return jsonify({'message': 'Token is missing'}), 401
        
        # Verify token
        payload = jwt_manager.verify_token(token)
        if not payload:
            return jsonify({'message': 'Token is invalid or expired'}), 401
        
        # Add user info to request context
        request.current_user = payload
        
        return f(*args, **kwargs)
    
    return decorated

def role_required(*required_roles):
    """Decorator to require specific roles"""
    def decorator(f):
        @wraps(f)
        @token_required
        def decorated(*args, **kwargs):
            user_roles = request.current_user.get('roles', [])
            
            if not any(role in user_roles for role in required_roles):
                return jsonify({'message': 'Insufficient permissions'}), 403
            
            return f(*args, **kwargs)
        
        return decorated
    return decorator

def permission_required(*required_permissions):
    """Decorator to require specific permissions"""
    def decorator(f):
        @wraps(f)
        @token_required
        def decorated(*args, **kwargs):
            user_permissions = request.current_user.get('permissions', [])
            
            if not all(perm in user_permissions for perm in required_permissions):
                return jsonify({'message': 'Insufficient permissions'}), 403
            
            return f(*args, **kwargs)
        
        return decorated
    return decorator
```

### password_security.py
```python
import bcrypt
import secrets
import string
import re
from typing import Tuple, Optional
import logging

logger = logging.getLogger(__name__)

class PasswordManager:
    def __init__(self, rounds: int = 12):
        self.rounds = rounds
    
    def hash_password(self, password: str) -> str:
        """Hash password using bcrypt"""
        salt = bcrypt.gensalt(rounds=self.rounds)
        hashed = bcrypt.hashpw(password.encode('utf-8'), salt)
        return hashed.decode('utf-8')
    
    def verify_password(self, password: str, hashed: str) -> bool:
        """Verify password against hash"""
        try:
            return bcrypt.checkpw(password.encode('utf-8'), hashed.encode('utf-8'))
        except Exception as e:
            logger.error(f"Password verification error: {str(e)}")
            return False
    
    def generate_secure_password(self, length: int = 16) -> str:
        """Generate secure random password"""
        alphabet = string.ascii_letters + string.digits + "!@#$%^&*"
        password = ''.join(secrets.choice(alphabet) for _ in range(length))
        return password
    
    def validate_password_strength(self, password: str) -> Tuple[bool, list]:
        """Validate password strength"""
        errors = []
        
        if len(password) < 8:
            errors.append("Password must be at least 8 characters long")
        
        if len(password) > 128:
            errors.append("Password must be less than 128 characters long")
        
        if not re.search(r"[a-z]", password):
            errors.append("Password must contain at least one lowercase letter")
        
        if not re.search(r"[A-Z]", password):
            errors.append("Password must contain at least one uppercase letter")
        
        if not re.search(r"\d", password):
            errors.append("Password must contain at least one digit")
        
        if not re.search(r"[!@#$%^&*(),.?\":{}|<>]", password):
            errors.append("Password must contain at least one special character")
        
        # Check for common patterns
        if re.search(r"(.)\1{2,}", password):
            errors.append("Password cannot contain repeated characters")
        
        if re.search(r"(012|123|234|345|456|567|678|789|890)", password):
            errors.append("Password cannot contain sequential numbers")
        
        if re.search(r"(abc|bcd|cde|def|efg|fgh|ghi|hij|ijk|jkl|klm|lmn|mno|nop|opq|pqr|qrs|rst|stu|tuv|uvw|vwx|wxy|xyz)", password.lower()):
            errors.append("Password cannot contain sequential letters")
        
        return len(errors) == 0, errors

# Password reset functionality
class PasswordResetManager:
    def __init__(self, redis_client, expires_in: int = 3600):
        self.redis_client = redis_client
        self.expires_in = expires_in
    
    def generate_reset_token(self, user_id: str) -> str:
        """Generate password reset token"""
        token = secrets.token_urlsafe(32)
        
        # Store token in Redis
        self.redis_client.setex(
            f"password_reset:{token}",
            self.expires_in,
            user_id
        )
        
        logger.info(f"Password reset token generated for user {user_id}")
        return token
    
    def verify_reset_token(self, token: str) -> Optional[str]:
        """Verify password reset token and return user_id"""
        user_id = self.redis_client.get(f"password_reset:{token}")
        if user_id:
            return user_id.decode('utf-8')
        return None
    
    def consume_reset_token(self, token: str) -> Optional[str]:
        """Consume password reset token (one-time use)"""
        user_id = self.verify_reset_token(token)
        if user_id:
            self.redis_client.delete(f"password_reset:{token}")
            logger.info(f"Password reset token consumed for user {user_id}")
        return user_id
```

## 2. OAuth 2.0 / OpenID Connect

### oauth_provider.py
```python
from authlib.integrations.flask_oauth2 import AuthorizationServer, ResourceProtector
from authlib.integrations.sqla_oauth2 import (
    OAuth2ClientMixin,
    OAuth2AuthorizationCodeMixin,
    OAuth2TokenMixin,
)
from authlib.oauth2.rfc6749 import grants
from authlib.oauth2.rfc7636 import CodeChallenge
from authlib.oidc.core import UserInfo
from authlib.oidc.core.grants import OpenIDCode as _OpenIDCode
from werkzeug.security import gen_salt
import time

class AuthorizationCodeGrant(grants.AuthorizationCodeGrant):
    TOKEN_ENDPOINT_AUTH_METHODS = [
        'client_secret_basic',
        'client_secret_post',
        'none',
    ]

    def save_authorization_code(self, code, request):
        code_challenge = request.data.get('code_challenge')
        code_challenge_method = request.data.get('code_challenge_method')
        
        auth_code = AuthorizationCode(
            code=code,
            client_id=request.client.client_id,
            redirect_uri=request.redirect_uri,
            scope=request.scope,
            user_id=request.user.id,
            code_challenge=code_challenge,
            code_challenge_method=code_challenge_method,
        )
        db.session.add(auth_code)
        db.session.commit()
        return auth_code

    def query_authorization_code(self, code, client):
        auth_code = AuthorizationCode.query.filter_by(
            code=code, client_id=client.client_id).first()
        if auth_code and not auth_code.is_expired():
            return auth_code

    def delete_authorization_code(self, authorization_code):
        db.session.delete(authorization_code)
        db.session.commit()

    def authenticate_user(self, authorization_code):
        return User.query.get(authorization_code.user_id)

class PasswordGrant(grants.ResourceOwnerPasswordCredentialsGrant):
    def authenticate_user(self, username, password):
        user = User.query.filter_by(username=username).first()
        if user and user.check_password(password):
            return user

class RefreshTokenGrant(grants.RefreshTokenGrant):
    def authenticate_refresh_token(self, refresh_token):
        token = Token.query.filter_by(refresh_token=refresh_token).first()
        if token and token.is_refresh_token_active():
            return token

    def authenticate_user(self, credential):
        return User.query.get(credential.user_id)

    def revoke_old_credential(self, credential):
        credential.revoked = True
        db.session.add(credential)
        db.session.commit()

class OpenIDCode(_OpenIDCode):
    def exists_nonce(self, nonce, request):
        exists = AuthorizationCode.query.filter_by(
            client_id=request.client_id, nonce=nonce).first()
        return bool(exists)

    def get_jwt_config(self, grant):
        return {
            'key': current_app.config['JWT_SECRET_KEY'],
            'alg': 'HS256',
            'iss': current_app.config['OAUTH2_JWT_ISS'],
            'exp': 3600,
        }

    def generate_user_info(self, user, scope):
        return UserInfo(
            sub=str(user.id),
            name=user.name,
            email=user.email,
            email_verified=user.email_verified,
            preferred_username=user.username,
        )

def create_authorization_server(app):
    authorization = AuthorizationServer()
    authorization.init_app(app)

    # Register grants
    authorization.register_grant(grants.ImplicitGrant)
    authorization.register_grant(grants.ClientCredentialsGrant)
    authorization.register_grant(AuthorizationCodeGrant, [CodeChallenge(required=True)])
    authorization.register_grant(PasswordGrant)
    authorization.register_grant(RefreshTokenGrant)
    authorization.register_grant(OpenIDCode)

    return authorization

def create_resource_protector():
    require_oauth = ResourceProtector()
    
    def query_token(token_string):
        return Token.query.filter_by(access_token=token_string).first()
    
    def introspect_token(token):
        return {
            'active': not token.revoked and not token.is_expired(),
            'client_id': token.client_id,
            'username': token.user.username,
            'scope': token.scope,
            'exp': token.expires_at,
        }
    
    require_oauth.register_token_validator(query_token)
    require_oauth.register_introspect_token_validator(introspect_token)
    
    return require_oauth
```

## 3. Rate Limiting

### rate_limiter.py
```python
import time
import redis
from typing import Optional, Tuple
from functools import wraps
from flask import request, jsonify, g
import logging

logger = logging.getLogger(__name__)

class RateLimiter:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
    
    def is_allowed(self, key: str, limit: int, window: int) -> Tuple[bool, dict]:
        """
        Check if request is allowed using sliding window algorithm
        
        Args:
            key: Unique identifier for the rate limit (e.g., user_id, ip_address)
            limit: Maximum number of requests allowed
            window: Time window in seconds
        
        Returns:
            Tuple of (is_allowed, info_dict)
        """
        now = time.time()
        pipeline = self.redis.pipeline()
        
        # Remove expired entries
        pipeline.zremrangebyscore(key, 0, now - window)
        
        # Count current requests
        pipeline.zcard(key)
        
        # Add current request
        pipeline.zadd(key, {str(now): now})
        
        # Set expiration
        pipeline.expire(key, window)
        
        results = pipeline.execute()
        current_requests = results[1]
        
        if current_requests < limit:
            return True, {
                'allowed': True,
                'limit': limit,
                'remaining': limit - current_requests - 1,
                'reset_time': int(now + window)
            }
        else:
            # Remove the request we just added since it's not allowed
            self.redis.zrem(key, str(now))
            
            # Get the oldest request time to calculate reset time
            oldest = self.redis.zrange(key, 0, 0, withscores=True)
            reset_time = int(oldest[0][1] + window) if oldest else int(now + window)
            
            return False, {
                'allowed': False,
                'limit': limit,
                'remaining': 0,
                'reset_time': reset_time
            }
    
    def get_client_id(self, request) -> str:
        """Get client identifier for rate limiting"""
        # Try to get user ID from JWT token
        if hasattr(request, 'current_user') and request.current_user:
            return f"user:{request.current_user.get('user_id')}"
        
        # Fall back to IP address
        return f"ip:{request.remote_addr}"

def rate_limit(limit: int, window: int = 3600, per: str = 'hour', 
               key_func: Optional[callable] = None):
    """
    Rate limiting decorator
    
    Args:
        limit: Maximum number of requests
        window: Time window in seconds (default: 3600 for 1 hour)
        per: Human readable time period (for error messages)
        key_func: Custom function to generate rate limit key
    """
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if not hasattr(g, 'rate_limiter'):
                g.rate_limiter = RateLimiter(redis.Redis())
            
            # Generate rate limit key
            if key_func:
                key = key_func(request)
            else:
                key = g.rate_limiter.get_client_id(request)
            
            # Add endpoint to key for per-endpoint limiting
            rate_limit_key = f"rate_limit:{f.__name__}:{key}"
            
            # Check rate limit
            allowed, info = g.rate_limiter.is_allowed(rate_limit_key, limit, window)
            
            if not allowed:
                logger.warning(f"Rate limit exceeded for {key} on {f.__name__}")
                
                response = jsonify({
                    'error': 'Rate limit exceeded',
                    'message': f'Too many requests. Limit: {limit} per {per}',
                    'retry_after': info['reset_time'] - int(time.time())
                })
                response.status_code = 429
                response.headers['X-RateLimit-Limit'] = str(limit)
                response.headers['X-RateLimit-Remaining'] = str(info['remaining'])
                response.headers['X-RateLimit-Reset'] = str(info['reset_time'])
                response.headers['Retry-After'] = str(info['reset_time'] - int(time.time()))
                
                return response
            
            # Add rate limit headers to successful responses
            response = f(*args, **kwargs)
            if hasattr(response, 'headers'):
                response.headers['X-RateLimit-Limit'] = str(limit)
                response.headers['X-RateLimit-Remaining'] = str(info['remaining'])
                response.headers['X-RateLimit-Reset'] = str(info['reset_time'])
            
            return response
        
        return decorated_function
    return decorator

# Usage examples
@rate_limit(limit=100, window=3600, per='hour')
def api_endpoint():
    return jsonify({'message': 'Success'})

@rate_limit(limit=5, window=60, per='minute')
def login_endpoint():
    return jsonify({'message': 'Login successful'})

# Custom key function for API key based rate limiting
def api_key_rate_limit_key(request):
    api_key = request.headers.get('X-API-Key')
    if api_key:
        return f"api_key:{api_key}"
    return f"ip:{request.remote_addr}"

@rate_limit(limit=1000, window=3600, per='hour', key_func=api_key_rate_limit_key)
def api_with_key():
    return jsonify({'message': 'API response'})
```

## 4. Input Validation and Sanitization

### input_validation.py
```python
import re
import html
import bleach
from typing import Any, Dict, List, Optional, Union
from marshmallow import Schema, fields, validate, ValidationError, pre_load
from werkzeug.datastructures import FileStorage
import magic
import logging

logger = logging.getLogger(__name__)

class SecurityValidator:
    """Security-focused input validator"""
    
    # Allowed HTML tags and attributes for rich text
    ALLOWED_TAGS = [
        'p', 'br', 'strong', 'em', 'u', 'ol', 'ul', 'li', 'h1', 'h2', 'h3', 
        'h4', 'h5', 'h6', 'blockquote', 'code', 'pre'
    ]
    
    ALLOWED_ATTRIBUTES = {
        '*': ['class'],
        'a': ['href', 'title'],
        'img': ['src', 'alt', 'width', 'height'],
    }
    
    # File upload restrictions
    ALLOWED_IMAGE_TYPES = ['image/jpeg', 'image/png', 'image/gif', 'image/webp']
    ALLOWED_DOCUMENT_TYPES = ['application/pdf', 'text/plain', 'text/csv']
    MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB
    
    @staticmethod
    def sanitize_html(content: str) -> str:
        """Sanitize HTML content"""
        if not content:
            return ''
        
        # Clean HTML using bleach
        cleaned = bleach.clean(
            content,
            tags=SecurityValidator.ALLOWED_TAGS,
            attributes=SecurityValidator.ALLOWED_ATTRIBUTES,
            strip=True
        )
        
        return cleaned
    
    @staticmethod
    def escape_html(content: str) -> str:
        """Escape HTML entities"""
        if not content:
            return ''
        return html.escape(content)
    
    @staticmethod
    def validate_email(email: str) -> bool:
        """Validate email format"""
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return bool(re.match(pattern, email))
    
    @staticmethod
    def validate_phone(phone: str) -> bool:
        """Validate phone number format"""
        # Remove all non-digit characters
        digits_only = re.sub(r'\D', '', phone)
        
        # Check if it's a valid length (7-15 digits)
        return 7 <= len(digits_only) <= 15
    
    @staticmethod
    def validate_url(url: str) -> bool:
        """Validate URL format"""
        pattern = r'^https?://(?:[-\w.])+(?:\:[0-9]+)?(?:/(?:[\w/_.])*(?:\?(?:[\w&=%.])*)?(?:\#(?:[\w.])*)?)?$'
        return bool(re.match(pattern, url))
    
    @staticmethod
    def validate_sql_injection(value: str) -> bool:
        """Check for potential SQL injection patterns"""
        if not value:
            return True
        
        # Common SQL injection patterns
        sql_patterns = [
            r"(\b(SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|EXEC|UNION)\b)",
            r"(--|#|/\*|\*/)",
            r"(\b(OR|AND)\s+\d+\s*=\s*\d+)",
            r"(\b(OR|AND)\s+['\"]?\w+['\"]?\s*=\s*['\"]?\w+['\"]?)",
            r"(UNION\s+SELECT)",
            r"(;\s*(DROP|DELETE|INSERT|UPDATE))",
        ]
        
        for pattern in sql_patterns:
            if re.search(pattern, value, re.IGNORECASE):
                logger.warning(f"Potential SQL injection detected: {pattern}")
                return False
        
        return True
    
    @staticmethod
    def validate_xss(value: str) -> bool:
        """Check for potential XSS patterns"""
        if not value:
            return True
        
        # Common XSS patterns
        xss_patterns = [
            r"<script[^>]*>.*?</script>",
            r"javascript:",
            r"on\w+\s*=",
            r"<iframe[^>]*>.*?</iframe>",
            r"<object[^>]*>.*?</object>",
            r"<embed[^>]*>.*?</embed>",
            r"<link[^>]*>",
            r"<meta[^>]*>",
        ]
        
        for pattern in xss_patterns:
            if re.search(pattern, value, re.IGNORECASE):
                logger.warning(f"Potential XSS detected: {pattern}")
                return False
        
        return True
    
    @staticmethod
    def validate_file_upload(file: FileStorage) -> Tuple[bool, str]:
        """Validate uploaded file"""
        if not file or not file.filename:
            return False, "No file provided"
        
        # Check file size
        file.seek(0, 2)  # Seek to end
        size = file.tell()
        file.seek(0)  # Reset to beginning
        
        if size > SecurityValidator.MAX_FILE_SIZE:
            return False, f"File too large. Maximum size: {SecurityValidator.MAX_FILE_SIZE} bytes"
        
        # Check file type using python-magic
        file_content = file.read(1024)  # Read first 1KB
        file.seek(0)  # Reset
        
        mime_type = magic.from_buffer(file_content, mime=True)
        
        allowed_types = (SecurityValidator.ALLOWED_IMAGE_TYPES + 
                        SecurityValidator.ALLOWED_DOCUMENT_TYPES)
        
        if mime_type not in allowed_types:
            return False, f"File type not allowed: {mime_type}"
        
        # Check filename for malicious patterns
        filename = file.filename.lower()
        dangerous_extensions = ['.exe', '.bat', '.cmd', '.scr', '.pif', '.com', '.jar']
        
        if any(filename.endswith(ext) for ext in dangerous_extensions):
            return False, "Dangerous file extension detected"
        
        return True, "File is valid"

# Marshmallow schemas with security validation
class SecureBaseSchema(Schema):
    """Base schema with security validations"""
    
    @pre_load
    def validate_security(self, data, **kwargs):
        """Pre-load validation for security"""
        if isinstance(data, dict):
            for key, value in data.items():
                if isinstance(value, str):
                    # Check for SQL injection
                    if not SecurityValidator.validate_sql_injection(value):
                        raise ValidationError(f"Invalid input detected in field: {key}")
                    
                    # Check for XSS
                    if not SecurityValidator.validate_xss(value):
                        raise ValidationError(f"Invalid input detected in field: {key}")
        
        return data

class UserRegistrationSchema(SecureBaseSchema):
    email = fields.Email(required=True, validate=validate.Length(max=255))
    password = fields.Str(required=True, validate=validate.Length(min=8, max=128))
    first_name = fields.Str(required=True, validate=validate.Length(max=50))
    last_name = fields.Str(required=True, validate=validate.Length(max=50))
    phone = fields.Str(validate=validate.Length(max=20))
    
    @pre_load
    def sanitize_input(self, data, **kwargs):
        """Sanitize input data"""
        data = super().validate_security(data, **kwargs)
        
        # Sanitize string fields
        for field in ['first_name', 'last_name']:
            if field in data and data[field]:
                data[field] = SecurityValidator.escape_html(data[field].strip())
        
        return data

class UserProfileSchema(SecureBaseSchema):
    bio = fields.Str(validate=validate.Length(max=1000))
    website = fields.Url()
    location = fields.Str(validate=validate.Length(max=100))
    
    @pre_load
    def sanitize_input(self, data, **kwargs):
        """Sanitize input data"""
        data = super().validate_security(data, **kwargs)
        
        # Sanitize HTML in bio
        if 'bio' in data and data['bio']:
            data['bio'] = SecurityValidator.sanitize_html(data['bio'])
        
        # Escape other fields
        for field in ['location']:
            if field in data and data[field]:
                data[field] = SecurityValidator.escape_html(data[field].strip())
        
        return data

# Validation decorators
def validate_json(schema_class):
    """Decorator to validate JSON input using Marshmallow schema"""
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            try:
                schema = schema_class()
                data = schema.load(request.get_json() or {})
                request.validated_data = data
                return f(*args, **kwargs)
            except ValidationError as e:
                logger.warning(f"Validation error: {e.messages}")
                return jsonify({'errors': e.messages}), 400
        
        return decorated_function
    return decorator

# Usage examples
@validate_json(UserRegistrationSchema)
def register_user():
    data = request.validated_data
    # Process validated and sanitized data
    return jsonify({'message': 'User registered successfully'})

@validate_json(UserProfileSchema)
def update_profile():
    data = request.validated_data
    # Process validated and sanitized data
    return jsonify({'message': 'Profile updated successfully'})
```

## 5. HTTPS and TLS Configuration

### nginx-ssl.conf
```nginx
# SSL Configuration for SAAS Platform
server {
    listen 80;
    server_name saas-platform.com www.saas-platform.com;
    
    # Redirect all HTTP traffic to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name saas-platform.com www.saas-platform.com;
    
    # SSL Certificate Configuration
    ssl_certificate /etc/ssl/certs/saas-platform.com.crt;
    ssl_certificate_key /etc/ssl/private/saas-platform.com.key;
    
    # SSL Security Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/ssl/certs/ca-certificates.crt;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;
    
    # Security Headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self'; media-src 'self'; object-src 'none'; child-src 'none'; frame-ancestors 'none'; form-action 'self'; base-uri 'self';" always;
    
    # Remove Server Header
    server_tokens off;
    
    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied expired no-cache no-store private must-revalidate auth;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/javascript
        application/xml+rss
        application/json;
    
    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
    
    # API Endpoints
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeout settings
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffer settings
        proxy_buffering on;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
    }
    
    # Login endpoint with stricter rate limiting
    location /api/auth/login {
        limit_req zone=login burst=5 nodelay;
        
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Static files
    location /static/ {
        alias /var/www/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    # Frontend application
    location / {
        proxy_pass http://frontend:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

## 6. Security Middleware

### security_middleware.py
```python
from flask import Flask, request, g, abort, jsonify
import time
import hashlib
import hmac
import logging
from typing import Optional
import redis

logger = logging.getLogger(__name__)

class SecurityMiddleware:
    def __init__(self, app: Flask, redis_client: redis.Redis):
        self.app = app
        self.redis = redis_client
        self.init_app(app)
    
    def init_app(self, app: Flask):
        """Initialize security middleware"""
        app.before_request(self.before_request)
        app.after_request(self.after_request)
    
    def before_request(self):
        """Security checks before processing request"""
        # Skip security checks for health endpoints
        if request.endpoint in ['health', 'metrics']:
            return
        
        # Check for suspicious patterns
        if self.detect_suspicious_activity():
            logger.warning(f"Suspicious activity detected from {request.remote_addr}")
            abort(403)
        
        # Validate request size
        if self.check_request_size():
            logger.warning(f"Request too large from {request.remote_addr}")
            abort(413)
        
        # Check for bot/crawler patterns
        if self.detect_bot_activity():
            logger.info(f"Bot activity detected from {request.remote_addr}")
            # You might want to rate limit or block bots
        
        # Add request ID for tracing
        g.request_id = self.generate_request_id()
    
    def after_request(self, response):
        """Security headers and logging after request"""
        # Add security headers
        response.headers['X-Request-ID'] = getattr(g, 'request_id', 'unknown')
        response.headers['X-Content-Type-Options'] = 'nosniff'
        response.headers['X-Frame-Options'] = 'DENY'
        response.headers['X-XSS-Protection'] = '1; mode=block'
        
        # Log request
        self.log_request(response)
        
        return response
    
    def detect_suspicious_activity(self) -> bool:
        """Detect suspicious request patterns"""
        # Check for common attack patterns in URL
        suspicious_patterns = [
            '../', '..\\', '/etc/passwd', '/proc/', 'cmd.exe',
            'powershell', '<script', 'javascript:', 'vbscript:',
            'onload=', 'onerror=', 'eval(', 'alert(', 'document.cookie'
        ]
        
        url = request.url.lower()
        for pattern in suspicious_patterns:
            if pattern in url:
                return True
        
        # Check request headers for suspicious content
        user_agent = request.headers.get('User-Agent', '').lower()
        suspicious_agents = [
            'sqlmap', 'nikto', 'nmap', 'masscan', 'nessus',
            'openvas', 'w3af', 'burp', 'owasp'
        ]
        
        for agent in suspicious_agents:
            if agent in user_agent:
                return True
        
        return False
    
    def check_request_size(self) -> bool:
        """Check if request size exceeds limits"""
        max_content_length = 10 * 1024 * 1024  # 10MB
        
        content_length = request.content_length
        if content_length and content_length > max_content_length:
            return True
        
        return False
    
    def detect_bot_activity(self) -> bool:
        """Detect bot/crawler activity"""
        user_agent = request.headers.get('User-Agent', '').lower()
        bot_patterns = [
            'bot', 'crawler', 'spider', 'scraper', 'curl', 'wget',
            'python-requests', 'go-http-client', 'java/', 'apache-httpclient'
        ]
        
        for pattern in bot_patterns:
            if pattern in user_agent:
                return True
        
        return False
    
    def generate_request_id(self) -> str:
        """Generate unique request ID"""
        timestamp = str(time.time())
        remote_addr = request.remote_addr or 'unknown'
        data = f"{timestamp}-{remote_addr}-{request.method}-{request.path}"
        return hashlib.md5(data.encode()).hexdigest()[:16]
    
    def log_request(self, response):
        """Log request details"""
        log_data = {
            'request_id': getattr(g, 'request_id', 'unknown'),
            'method': request.method,
            'path': request.path,
            'remote_addr': request.remote_addr,
            'user_agent': request.headers.get('User-Agent', ''),
            'status_code': response.status_code,
            'response_size': len(response.get_data()),
            'timestamp': time.time()
        }
        
        # Add user info if available
        if hasattr(request, 'current_user') and request.current_user:
            log_data['user_id'] = request.current_user.get('user_id')
        
        logger.info(f"Request processed: {log_data}")

# CSRF Protection
class CSRFProtection:
    def __init__(self, app: Flask, secret_key: str):
        self.app = app
        self.secret_key = secret_key
        self.init_app(app)
    
    def init_app(self, app: Flask):
        """Initialize CSRF protection"""
        app.before_request(self.validate_csrf)
    
    def generate_csrf_token(self, session_id: str) -> str:
        """Generate CSRF token"""
        timestamp = str(int(time.time()))
        message = f"{session_id}:{timestamp}"
        signature = hmac.new(
            self.secret_key.encode(),
            message.encode(),
            hashlib.sha256
        ).hexdigest()
        
        return f"{timestamp}:{signature}"
    
    def validate_csrf_token(self, token: str, session_id: str) -> bool:
        """Validate CSRF token"""
        try:
            timestamp, signature = token.split(':', 1)
            
            # Check if token is not too old (1 hour)
            if int(time.time()) - int(timestamp) > 3600:
                return False
            
            # Verify signature
            message = f"{session_id}:{timestamp}"
            expected_signature = hmac.new(
                self.secret_key.encode(),
                message.encode(),
                hashlib.sha256
            ).hexdigest()
            
            return hmac.compare_digest(signature, expected_signature)
            
        except (ValueError, TypeError):
            return False
    
    def validate_csrf(self):
        """Validate CSRF token for state-changing requests"""
        if request.method in ['POST', 'PUT', 'DELETE', 'PATCH']:
            # Skip CSRF for API endpoints with proper authentication
            if request.path.startswith('/api/') and 'Authorization' in request.headers:
                return
            
            csrf_token = request.headers.get('X-CSRF-Token') or request.form.get('csrf_token')
            session_id = request.cookies.get('session_id')
            
            if not csrf_token or not session_id:
                logger.warning(f"Missing CSRF token or session ID from {request.remote_addr}")
                abort(403)
            
            if not self.validate_csrf_token(csrf_token, session_id):
                logger.warning(f"Invalid CSRF token from {request.remote_addr}")
                abort(403)
```

## Mejores Prácticas de Seguridad

### 1. Autenticación
- Usar JWT con expiración corta
- Implementar refresh tokens
- Hashear contraseñas con bcrypt
- Validar fortaleza de contraseñas

### 2. Autorización
- Implementar RBAC (Role-Based Access Control)
- Usar principio de menor privilegio
- Validar permisos en cada request
- Implementar OAuth 2.0 para terceros

### 3. Protección de Datos
- Encriptar datos sensibles en reposo
- Usar HTTPS para todas las comunicaciones
- Implementar PII masking en logs
- Cumplir con GDPR/CCPA

### 4. Monitoreo de Seguridad
- Implementar logging de seguridad
- Monitorear intentos de acceso fallidos
- Alertas para actividad sospechosa
- Auditorías regulares de seguridad

Esta documentación proporciona una base sólida para implementar seguridad robusta en aplicaciones SAAS.