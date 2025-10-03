# Templates de Base de Datos

## Objetivo
Proporcionar templates estandarizados para configuración, gestión y optimización de bases de datos en aplicaciones SAAS, incluyendo PostgreSQL, MongoDB, Redis y patrones de acceso a datos.

## 1. PostgreSQL Configuration

### postgresql.conf
```ini
# PostgreSQL Configuration for SAAS Platform

# Connection Settings
listen_addresses = '*'
port = 5432
max_connections = 200
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = 256MB                    # 25% of RAM for dedicated server
effective_cache_size = 1GB                # 75% of RAM
work_mem = 4MB                           # Per operation memory
maintenance_work_mem = 64MB              # Maintenance operations
dynamic_shared_memory_type = posix

# WAL Settings
wal_level = replica
wal_buffers = 16MB
checkpoint_completion_target = 0.9
max_wal_size = 1GB
min_wal_size = 80MB
checkpoint_timeout = 15min

# Query Planner
random_page_cost = 1.1                   # SSD optimized
effective_io_concurrency = 200          # SSD optimized
seq_page_cost = 1.0

# Logging
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_min_duration_statement = 1000        # Log slow queries (1 second)
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0

# Performance
shared_preload_libraries = 'pg_stat_statements'
track_activity_query_size = 2048
track_functions = all
track_io_timing = on

# Replication (for read replicas)
hot_standby = on
max_standby_streaming_delay = 30s
wal_receiver_status_interval = 10s
hot_standby_feedback = on

# Security
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
ssl_ca_file = '/etc/ssl/certs/ca.crt'
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'
ssl_prefer_server_ciphers = on
password_encryption = scram-sha-256
```

### pg_hba.conf
```ini
# PostgreSQL Client Authentication Configuration

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections
local   all             postgres                                peer
local   all             all                                     md5

# IPv4 local connections
host    all             all             127.0.0.1/32            md5
host    all             all             10.0.0.0/8              md5
host    all             all             172.16.0.0/12           md5
host    all             all             192.168.0.0/16          md5

# IPv6 local connections
host    all             all             ::1/128                 md5

# Replication connections
host    replication     replicator      10.0.0.0/8              md5
host    replication     replicator      172.16.0.0/12           md5
host    replication     replicator      192.168.0.0/16          md5

# SSL connections only for production
hostssl saas_prod       saas_user       0.0.0.0/0               md5
hostssl saas_prod       saas_readonly   0.0.0.0/0               md5
```

### Database Schema Setup
```sql
-- Database and User Setup
CREATE DATABASE saas_platform;
CREATE USER saas_user WITH ENCRYPTED PASSWORD 'secure_password_here';
CREATE USER saas_readonly WITH ENCRYPTED PASSWORD 'readonly_password_here';

-- Grant permissions
GRANT CONNECT ON DATABASE saas_platform TO saas_user;
GRANT CONNECT ON DATABASE saas_platform TO saas_readonly;

\c saas_platform;

-- Create schemas
CREATE SCHEMA IF NOT EXISTS users;
CREATE SCHEMA IF NOT EXISTS billing;
CREATE SCHEMA IF NOT EXISTS analytics;
CREATE SCHEMA IF NOT EXISTS audit;

-- Grant schema permissions
GRANT USAGE, CREATE ON SCHEMA users TO saas_user;
GRANT USAGE, CREATE ON SCHEMA billing TO saas_user;
GRANT USAGE, CREATE ON SCHEMA analytics TO saas_user;
GRANT USAGE, CREATE ON SCHEMA audit TO saas_user;

GRANT USAGE ON SCHEMA users TO saas_readonly;
GRANT USAGE ON SCHEMA billing TO saas_readonly;
GRANT USAGE ON SCHEMA analytics TO saas_readonly;
GRANT USAGE ON SCHEMA audit TO saas_readonly;

-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Users table
CREATE TABLE users.users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    is_active BOOLEAN DEFAULT true,
    is_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_login TIMESTAMP WITH TIME ZONE,
    metadata JSONB DEFAULT '{}'::jsonb
);

-- Indexes
CREATE INDEX idx_users_email ON users.users(email);
CREATE INDEX idx_users_active ON users.users(is_active);
CREATE INDEX idx_users_created_at ON users.users(created_at);
CREATE INDEX idx_users_metadata_gin ON users.users USING gin(metadata);

-- User roles
CREATE TABLE users.roles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    permissions JSONB DEFAULT '[]'::jsonb,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- User role assignments
CREATE TABLE users.user_roles (
    user_id UUID REFERENCES users.users(id) ON DELETE CASCADE,
    role_id UUID REFERENCES users.roles(id) ON DELETE CASCADE,
    assigned_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    assigned_by UUID REFERENCES users.users(id),
    PRIMARY KEY (user_id, role_id)
);

-- Billing tables
CREATE TABLE billing.subscriptions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users.users(id) ON DELETE CASCADE,
    plan_id VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    current_period_start TIMESTAMP WITH TIME ZONE NOT NULL,
    current_period_end TIMESTAMP WITH TIME ZONE NOT NULL,
    cancel_at_period_end BOOLEAN DEFAULT false,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    metadata JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_subscriptions_user_id ON billing.subscriptions(user_id);
CREATE INDEX idx_subscriptions_status ON billing.subscriptions(status);
CREATE INDEX idx_subscriptions_period_end ON billing.subscriptions(current_period_end);

-- Audit log
CREATE TABLE audit.audit_log (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users.users(id),
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50) NOT NULL,
    resource_id VARCHAR(255),
    old_values JSONB,
    new_values JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_audit_log_user_id ON audit.audit_log(user_id);
CREATE INDEX idx_audit_log_action ON audit.audit_log(action);
CREATE INDEX idx_audit_log_created_at ON audit.audit_log(created_at);
CREATE INDEX idx_audit_log_resource ON audit.audit_log(resource_type, resource_id);

-- Functions and triggers
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply updated_at trigger to relevant tables
CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users.users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_subscriptions_updated_at BEFORE UPDATE ON billing.subscriptions
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Row Level Security (RLS)
ALTER TABLE users.users ENABLE ROW LEVEL SECURITY;
ALTER TABLE billing.subscriptions ENABLE ROW LEVEL SECURITY;

-- RLS Policies
CREATE POLICY users_own_data ON users.users
    FOR ALL TO saas_user
    USING (id = current_setting('app.current_user_id')::uuid);

CREATE POLICY subscriptions_own_data ON billing.subscriptions
    FOR ALL TO saas_user
    USING (user_id = current_setting('app.current_user_id')::uuid);

-- Grant table permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA users TO saas_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA billing TO saas_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA analytics TO saas_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA audit TO saas_user;

GRANT SELECT ON ALL TABLES IN SCHEMA users TO saas_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA billing TO saas_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics TO saas_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA audit TO saas_readonly;

-- Grant sequence permissions
GRANT USAGE ON ALL SEQUENCES IN SCHEMA users TO saas_user;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA billing TO saas_user;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA analytics TO saas_user;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA audit TO saas_user;
```

## 2. MongoDB Configuration

### mongod.conf
```yaml
# MongoDB Configuration for SAAS Platform

# Network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 1000
  compression:
    compressors: snappy,zstd

# Storage
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2
      journalCompressor: snappy
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true

# Replication
replication:
  replSetName: "saas-replica-set"
  oplogSizeMB: 1024

# Sharding
sharding:
  clusterRole: shardsvr

# Security
security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile
  clusterAuthMode: keyFile

# Logging
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log
  logRotate: reopen
  component:
    accessControl:
      verbosity: 1
    command:
      verbosity: 1
    query:
      verbosity: 1
    write:
      verbosity: 1

# Process management
processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

# Profiling
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 1000
  slowOpSampleRate: 0.1

# Set parameters
setParameter:
  enableLocalhostAuthBypass: false
  authenticationMechanisms: SCRAM-SHA-1,SCRAM-SHA-256
  maxLogSizeKB: 10000
```

### MongoDB Setup Script
```javascript
// MongoDB Database Setup
use admin;

// Create admin user
db.createUser({
  user: "admin",
  pwd: "secure_admin_password",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "clusterAdmin", db: "admin" }
  ]
});

// Switch to application database
use saas_platform;

// Create application users
db.createUser({
  user: "saas_app",
  pwd: "secure_app_password",
  roles: [
    { role: "readWrite", db: "saas_platform" }
  ]
});

db.createUser({
  user: "saas_readonly",
  pwd: "secure_readonly_password",
  roles: [
    { role: "read", db: "saas_platform" }
  ]
});

// Create collections with validation
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "passwordHash", "firstName", "lastName", "createdAt"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
        },
        passwordHash: {
          bsonType: "string",
          minLength: 60
        },
        firstName: {
          bsonType: "string",
          maxLength: 100
        },
        lastName: {
          bsonType: "string",
          maxLength: 100
        },
        isActive: {
          bsonType: "bool"
        },
        isVerified: {
          bsonType: "bool"
        },
        createdAt: {
          bsonType: "date"
        },
        updatedAt: {
          bsonType: "date"
        },
        lastLogin: {
          bsonType: ["date", "null"]
        },
        metadata: {
          bsonType: "object"
        }
      }
    }
  }
});

// Create indexes
db.users.createIndex({ "email": 1 }, { unique: true });
db.users.createIndex({ "isActive": 1 });
db.users.createIndex({ "createdAt": 1 });
db.users.createIndex({ "metadata.plan": 1 });

// Organizations collection
db.createCollection("organizations", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "ownerId", "createdAt"],
      properties: {
        name: {
          bsonType: "string",
          maxLength: 200
        },
        ownerId: {
          bsonType: "objectId"
        },
        members: {
          bsonType: "array",
          items: {
            bsonType: "object",
            required: ["userId", "role", "joinedAt"],
            properties: {
              userId: { bsonType: "objectId" },
              role: { bsonType: "string" },
              joinedAt: { bsonType: "date" }
            }
          }
        },
        settings: {
          bsonType: "object"
        },
        createdAt: {
          bsonType: "date"
        },
        updatedAt: {
          bsonType: "date"
        }
      }
    }
  }
});

db.organizations.createIndex({ "ownerId": 1 });
db.organizations.createIndex({ "members.userId": 1 });
db.organizations.createIndex({ "name": "text" });

// Analytics events collection
db.createCollection("analytics_events", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["eventType", "userId", "timestamp"],
      properties: {
        eventType: {
          bsonType: "string",
          enum: ["page_view", "button_click", "form_submit", "api_call", "error"]
        },
        userId: {
          bsonType: ["objectId", "null"]
        },
        sessionId: {
          bsonType: "string"
        },
        properties: {
          bsonType: "object"
        },
        timestamp: {
          bsonType: "date"
        }
      }
    }
  }
});

// Time-series collection for analytics (MongoDB 5.0+)
db.createCollection("metrics", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "minutes"
  }
});

db.analytics_events.createIndex({ "timestamp": 1 });
db.analytics_events.createIndex({ "userId": 1, "timestamp": 1 });
db.analytics_events.createIndex({ "eventType": 1, "timestamp": 1 });
db.analytics_events.createIndex({ "sessionId": 1 });

// Set up TTL for analytics events (keep for 90 days)
db.analytics_events.createIndex(
  { "timestamp": 1 },
  { expireAfterSeconds: 7776000 }
);

// Audit log collection
db.createCollection("audit_log", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["action", "resourceType", "timestamp"],
      properties: {
        userId: {
          bsonType: ["objectId", "null"]
        },
        action: {
          bsonType: "string"
        },
        resourceType: {
          bsonType: "string"
        },
        resourceId: {
          bsonType: "string"
        },
        oldValues: {
          bsonType: "object"
        },
        newValues: {
          bsonType: "object"
        },
        ipAddress: {
          bsonType: "string"
        },
        userAgent: {
          bsonType: "string"
        },
        timestamp: {
          bsonType: "date"
        }
      }
    }
  }
});

db.audit_log.createIndex({ "userId": 1, "timestamp": 1 });
db.audit_log.createIndex({ "action": 1 });
db.audit_log.createIndex({ "resourceType": 1, "resourceId": 1 });
db.audit_log.createIndex({ "timestamp": 1 });

// Set up TTL for audit log (keep for 2 years)
db.audit_log.createIndex(
  { "timestamp": 1 },
  { expireAfterSeconds: 63072000 }
);
```

## 3. Redis Configuration

### redis.conf
```ini
# Redis Configuration for SAAS Platform

# Network
bind 0.0.0.0
port 6379
protected-mode yes
tcp-backlog 511
timeout 0
tcp-keepalive 300

# General
daemonize yes
pidfile /var/run/redis/redis-server.pid
loglevel notice
logfile /var/log/redis/redis-server.log
databases 16

# Snapshotting
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis

# Replication
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-ping-replica-period 10
repl-timeout 60
repl-disable-tcp-nodelay no
repl-backlog-size 1mb
repl-backlog-ttl 3600

# Security
requirepass your_secure_redis_password
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command KEYS ""
rename-command CONFIG "CONFIG_b835c3f8a5d2e7f1"

# Memory Management
maxmemory 2gb
maxmemory-policy allkeys-lru
maxmemory-samples 5

# Lazy Freeing
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
replica-lazy-flush yes

# Append Only File
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes

# Lua Scripting
lua-time-limit 5000

# Slow Log
slowlog-log-slower-than 10000
slowlog-max-len 128

# Latency Monitoring
latency-monitor-threshold 100

# Event Notification
notify-keyspace-events "Ex"

# Advanced Config
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes
```

## 4. Database Connection Pools

### database_manager.py
```python
import asyncio
import asyncpg
import motor.motor_asyncio
import aioredis
import logging
from typing import Optional, Dict, Any, List
from contextlib import asynccontextmanager
from dataclasses import dataclass
import json
import time

logger = logging.getLogger(__name__)

@dataclass
class DatabaseConfig:
    host: str
    port: int
    database: str
    username: str
    password: str
    min_connections: int = 5
    max_connections: int = 20
    ssl: bool = False
    ssl_cert: Optional[str] = None
    ssl_key: Optional[str] = None
    ssl_ca: Optional[str] = None

class PostgreSQLManager:
    def __init__(self, config: DatabaseConfig):
        self.config = config
        self.pool: Optional[asyncpg.Pool] = None
    
    async def initialize(self):
        """Initialize connection pool"""
        try:
            ssl_context = None
            if self.config.ssl:
                import ssl
                ssl_context = ssl.create_default_context()
                if self.config.ssl_ca:
                    ssl_context.load_verify_locations(self.config.ssl_ca)
                if self.config.ssl_cert and self.config.ssl_key:
                    ssl_context.load_cert_chain(self.config.ssl_cert, self.config.ssl_key)
            
            self.pool = await asyncpg.create_pool(
                host=self.config.host,
                port=self.config.port,
                database=self.config.database,
                user=self.config.username,
                password=self.config.password,
                min_size=self.config.min_connections,
                max_size=self.config.max_connections,
                ssl=ssl_context,
                command_timeout=60,
                server_settings={
                    'application_name': 'saas_platform',
                    'jit': 'off'
                }
            )
            
            logger.info(f"PostgreSQL pool initialized: {self.config.host}:{self.config.port}")
            
        except Exception as e:
            logger.error(f"Failed to initialize PostgreSQL pool: {str(e)}")
            raise
    
    async def close(self):
        """Close connection pool"""
        if self.pool:
            await self.pool.close()
            logger.info("PostgreSQL pool closed")
    
    @asynccontextmanager
    async def get_connection(self):
        """Get connection from pool"""
        if not self.pool:
            raise RuntimeError("Database pool not initialized")
        
        async with self.pool.acquire() as connection:
            try:
                yield connection
            except Exception as e:
                logger.error(f"Database error: {str(e)}")
                raise
    
    @asynccontextmanager
    async def get_transaction(self):
        """Get transaction from pool"""
        async with self.get_connection() as connection:
            async with connection.transaction():
                yield connection
    
    async def execute_query(self, query: str, *args) -> List[Dict[str, Any]]:
        """Execute SELECT query"""
        async with self.get_connection() as connection:
            rows = await connection.fetch(query, *args)
            return [dict(row) for row in rows]
    
    async def execute_one(self, query: str, *args) -> Optional[Dict[str, Any]]:
        """Execute SELECT query returning one row"""
        async with self.get_connection() as connection:
            row = await connection.fetchrow(query, *args)
            return dict(row) if row else None
    
    async def execute_command(self, query: str, *args) -> str:
        """Execute INSERT/UPDATE/DELETE command"""
        async with self.get_connection() as connection:
            return await connection.execute(query, *args)
    
    async def health_check(self) -> bool:
        """Check database health"""
        try:
            async with self.get_connection() as connection:
                await connection.fetchval("SELECT 1")
            return True
        except Exception as e:
            logger.error(f"PostgreSQL health check failed: {str(e)}")
            return False

class MongoDBManager:
    def __init__(self, config: DatabaseConfig):
        self.config = config
        self.client: Optional[motor.motor_asyncio.AsyncIOMotorClient] = None
        self.database: Optional[motor.motor_asyncio.AsyncIOMotorDatabase] = None
    
    async def initialize(self):
        """Initialize MongoDB client"""
        try:
            connection_string = f"mongodb://{self.config.username}:{self.config.password}@{self.config.host}:{self.config.port}/{self.config.database}"
            
            if self.config.ssl:
                connection_string += "?ssl=true"
                if self.config.ssl_ca:
                    connection_string += f"&ssl_ca_certs={self.config.ssl_ca}"
            
            self.client = motor.motor_asyncio.AsyncIOMotorClient(
                connection_string,
                maxPoolSize=self.config.max_connections,
                minPoolSize=self.config.min_connections,
                maxIdleTimeMS=30000,
                waitQueueTimeoutMS=5000,
                serverSelectionTimeoutMS=5000,
                connectTimeoutMS=10000,
                socketTimeoutMS=20000
            )
            
            self.database = self.client[self.config.database]
            
            # Test connection
            await self.client.admin.command('ping')
            
            logger.info(f"MongoDB client initialized: {self.config.host}:{self.config.port}")
            
        except Exception as e:
            logger.error(f"Failed to initialize MongoDB client: {str(e)}")
            raise
    
    async def close(self):
        """Close MongoDB client"""
        if self.client:
            self.client.close()
            logger.info("MongoDB client closed")
    
    def get_collection(self, name: str):
        """Get collection"""
        if not self.database:
            raise RuntimeError("MongoDB client not initialized")
        return self.database[name]
    
    async def health_check(self) -> bool:
        """Check MongoDB health"""
        try:
            await self.client.admin.command('ping')
            return True
        except Exception as e:
            logger.error(f"MongoDB health check failed: {str(e)}")
            return False

class RedisManager:
    def __init__(self, config: DatabaseConfig):
        self.config = config
        self.pool: Optional[aioredis.ConnectionPool] = None
        self.redis: Optional[aioredis.Redis] = None
    
    async def initialize(self):
        """Initialize Redis connection pool"""
        try:
            self.pool = aioredis.ConnectionPool.from_url(
                f"redis://:{self.config.password}@{self.config.host}:{self.config.port}/0",
                max_connections=self.config.max_connections,
                retry_on_timeout=True,
                health_check_interval=30
            )
            
            self.redis = aioredis.Redis(connection_pool=self.pool)
            
            # Test connection
            await self.redis.ping()
            
            logger.info(f"Redis pool initialized: {self.config.host}:{self.config.port}")
            
        except Exception as e:
            logger.error(f"Failed to initialize Redis pool: {str(e)}")
            raise
    
    async def close(self):
        """Close Redis connection pool"""
        if self.redis:
            await self.redis.close()
        if self.pool:
            await self.pool.disconnect()
        logger.info("Redis pool closed")
    
    async def get(self, key: str) -> Optional[str]:
        """Get value by key"""
        return await self.redis.get(key)
    
    async def set(self, key: str, value: str, ex: Optional[int] = None) -> bool:
        """Set key-value pair"""
        return await self.redis.set(key, value, ex=ex)
    
    async def delete(self, *keys: str) -> int:
        """Delete keys"""
        return await self.redis.delete(*keys)
    
    async def exists(self, key: str) -> bool:
        """Check if key exists"""
        return bool(await self.redis.exists(key))
    
    async def expire(self, key: str, seconds: int) -> bool:
        """Set key expiration"""
        return await self.redis.expire(key, seconds)
    
    async def hget(self, name: str, key: str) -> Optional[str]:
        """Get hash field value"""
        return await self.redis.hget(name, key)
    
    async def hset(self, name: str, key: str, value: str) -> int:
        """Set hash field value"""
        return await self.redis.hset(name, key, value)
    
    async def hgetall(self, name: str) -> Dict[str, str]:
        """Get all hash fields"""
        return await self.redis.hgetall(name)
    
    async def lpush(self, name: str, *values: str) -> int:
        """Push values to list"""
        return await self.redis.lpush(name, *values)
    
    async def rpop(self, name: str) -> Optional[str]:
        """Pop value from list"""
        return await self.redis.rpop(name)
    
    async def publish(self, channel: str, message: str) -> int:
        """Publish message to channel"""
        return await self.redis.publish(channel, message)
    
    async def health_check(self) -> bool:
        """Check Redis health"""
        try:
            await self.redis.ping()
            return True
        except Exception as e:
            logger.error(f"Redis health check failed: {str(e)}")
            return False

class DatabaseManager:
    """Unified database manager"""
    
    def __init__(self):
        self.postgresql: Optional[PostgreSQLManager] = None
        self.mongodb: Optional[MongoDBManager] = None
        self.redis: Optional[RedisManager] = None
    
    async def initialize_postgresql(self, config: DatabaseConfig):
        """Initialize PostgreSQL"""
        self.postgresql = PostgreSQLManager(config)
        await self.postgresql.initialize()
    
    async def initialize_mongodb(self, config: DatabaseConfig):
        """Initialize MongoDB"""
        self.mongodb = MongoDBManager(config)
        await self.mongodb.initialize()
    
    async def initialize_redis(self, config: DatabaseConfig):
        """Initialize Redis"""
        self.redis = RedisManager(config)
        await self.redis.initialize()
    
    async def close_all(self):
        """Close all database connections"""
        if self.postgresql:
            await self.postgresql.close()
        if self.mongodb:
            await self.mongodb.close()
        if self.redis:
            await self.redis.close()
    
    async def health_check_all(self) -> Dict[str, bool]:
        """Check health of all databases"""
        results = {}
        
        if self.postgresql:
            results['postgresql'] = await self.postgresql.health_check()
        
        if self.mongodb:
            results['mongodb'] = await self.mongodb.health_check()
        
        if self.redis:
            results['redis'] = await self.redis.health_check()
        
        return results

# Usage example
async def main():
    db_manager = DatabaseManager()
    
    # Initialize databases
    pg_config = DatabaseConfig(
        host="localhost",
        port=5432,
        database="saas_platform",
        username="saas_user",
        password="secure_password"
    )
    
    mongo_config = DatabaseConfig(
        host="localhost",
        port=27017,
        database="saas_platform",
        username="saas_app",
        password="secure_password"
    )
    
    redis_config = DatabaseConfig(
        host="localhost",
        port=6379,
        database="0",
        username="",
        password="secure_password"
    )
    
    await db_manager.initialize_postgresql(pg_config)
    await db_manager.initialize_mongodb(mongo_config)
    await db_manager.initialize_redis(redis_config)
    
    # Health check
    health = await db_manager.health_check_all()
    print(f"Database health: {health}")
    
    # Close connections
    await db_manager.close_all()

if __name__ == "__main__":
    asyncio.run(main())
```

## 5. Database Migrations

### migration_manager.py
```python
import asyncio
import asyncpg
import logging
from typing import List, Dict, Any, Optional
from pathlib import Path
import hashlib
import json
from datetime import datetime

logger = logging.getLogger(__name__)

class Migration:
    def __init__(self, version: str, name: str, up_sql: str, down_sql: str = ""):
        self.version = version
        self.name = name
        self.up_sql = up_sql
        self.down_sql = down_sql
        self.checksum = self._calculate_checksum()
    
    def _calculate_checksum(self) -> str:
        """Calculate migration checksum"""
        content = f"{self.version}{self.name}{self.up_sql}{self.down_sql}"
        return hashlib.sha256(content.encode()).hexdigest()

class MigrationManager:
    def __init__(self, db_manager: PostgreSQLManager, migrations_dir: str = "migrations"):
        self.db_manager = db_manager
        self.migrations_dir = Path(migrations_dir)
        self.migrations: List[Migration] = []
    
    async def initialize(self):
        """Initialize migration system"""
        await self._create_migrations_table()
        await self._load_migrations()
    
    async def _create_migrations_table(self):
        """Create migrations tracking table"""
        query = """
        CREATE TABLE IF NOT EXISTS schema_migrations (
            version VARCHAR(255) PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            checksum VARCHAR(64) NOT NULL,
            applied_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
        );
        """
        
        async with self.db_manager.get_connection() as connection:
            await connection.execute(query)
    
    async def _load_migrations(self):
        """Load migrations from files"""
        if not self.migrations_dir.exists():
            logger.warning(f"Migrations directory not found: {self.migrations_dir}")
            return
        
        migration_files = sorted(self.migrations_dir.glob("*.sql"))
        
        for file_path in migration_files:
            try:
                content = file_path.read_text()
                
                # Parse migration file
                parts = content.split("-- DOWN")
                up_sql = parts[0].replace("-- UP", "").strip()
                down_sql = parts[1].strip() if len(parts) > 1 else ""
                
                # Extract version and name from filename
                filename = file_path.stem
                if "_" in filename:
                    version, name = filename.split("_", 1)
                else:
                    version = filename
                    name = "migration"
                
                migration = Migration(version, name, up_sql, down_sql)
                self.migrations.append(migration)
                
            except Exception as e:
                logger.error(f"Failed to load migration {file_path}: {str(e)}")
    
    async def get_applied_migrations(self) -> List[Dict[str, Any]]:
        """Get list of applied migrations"""
        query = "SELECT version, name, checksum, applied_at FROM schema_migrations ORDER BY version"
        return await self.db_manager.execute_query(query)
    
    async def get_pending_migrations(self) -> List[Migration]:
        """Get list of pending migrations"""
        applied = await self.get_applied_migrations()
        applied_versions = {m['version'] for m in applied}
        
        pending = []
        for migration in self.migrations:
            if migration.version not in applied_versions:
                pending.append(migration)
        
        return pending
    
    async def apply_migration(self, migration: Migration):
        """Apply a single migration"""
        logger.info(f"Applying migration {migration.version}: {migration.name}")
        
        async with self.db_manager.get_transaction() as connection:
            try:
                # Execute migration SQL
                await connection.execute(migration.up_sql)
                
                # Record migration
                await connection.execute(
                    """
                    INSERT INTO schema_migrations (version, name, checksum)
                    VALUES ($1, $2, $3)
                    """,
                    migration.version, migration.name, migration.checksum
                )
                
                logger.info(f"Migration {migration.version} applied successfully")
                
            except Exception as e:
                logger.error(f"Failed to apply migration {migration.version}: {str(e)}")
                raise
    
    async def rollback_migration(self, version: str):
        """Rollback a specific migration"""
        # Find migration
        migration = None
        for m in self.migrations:
            if m.version == version:
                migration = m
                break
        
        if not migration:
            raise ValueError(f"Migration {version} not found")
        
        if not migration.down_sql:
            raise ValueError(f"Migration {version} has no rollback SQL")
        
        logger.info(f"Rolling back migration {version}: {migration.name}")
        
        async with self.db_manager.get_transaction() as connection:
            try:
                # Execute rollback SQL
                await connection.execute(migration.down_sql)
                
                # Remove migration record
                await connection.execute(
                    "DELETE FROM schema_migrations WHERE version = $1",
                    version
                )
                
                logger.info(f"Migration {version} rolled back successfully")
                
            except Exception as e:
                logger.error(f"Failed to rollback migration {version}: {str(e)}")
                raise
    
    async def migrate_up(self):
        """Apply all pending migrations"""
        pending = await self.get_pending_migrations()
        
        if not pending:
            logger.info("No pending migrations")
            return
        
        logger.info(f"Applying {len(pending)} pending migrations")
        
        for migration in pending:
            await self.apply_migration(migration)
        
        logger.info("All migrations applied successfully")
    
    async def migrate_down(self, target_version: Optional[str] = None):
        """Rollback migrations to target version"""
        applied = await self.get_applied_migrations()
        
        if target_version:
            # Rollback to specific version
            to_rollback = [m for m in applied if m['version'] > target_version]
        else:
            # Rollback last migration
            to_rollback = applied[-1:] if applied else []
        
        if not to_rollback:
            logger.info("No migrations to rollback")
            return
        
        # Rollback in reverse order
        for migration_record in reversed(to_rollback):
            await self.rollback_migration(migration_record['version'])
    
    async def status(self) -> Dict[str, Any]:
        """Get migration status"""
        applied = await self.get_applied_migrations()
        pending = await self.get_pending_migrations()
        
        return {
            'applied_count': len(applied),
            'pending_count': len(pending),
            'last_applied': applied[-1] if applied else None,
            'next_pending': pending[0].version if pending else None
        }

# Example migration files

# migrations/001_initial_schema.sql
INITIAL_SCHEMA = """
-- UP
CREATE SCHEMA IF NOT EXISTS users;
CREATE SCHEMA IF NOT EXISTS billing;

CREATE TABLE users.users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users.users(email);
CREATE INDEX idx_users_active ON users.users(is_active);

-- DOWN
DROP TABLE IF EXISTS users.users;
DROP SCHEMA IF EXISTS users CASCADE;
DROP SCHEMA IF EXISTS billing CASCADE;
"""

# migrations/002_add_user_verification.sql
ADD_USER_VERIFICATION = """
-- UP
ALTER TABLE users.users ADD COLUMN is_verified BOOLEAN DEFAULT false;
ALTER TABLE users.users ADD COLUMN verification_token VARCHAR(255);
ALTER TABLE users.users ADD COLUMN verification_sent_at TIMESTAMP WITH TIME ZONE;

CREATE INDEX idx_users_verification_token ON users.users(verification_token);

-- DOWN
ALTER TABLE users.users DROP COLUMN IF EXISTS is_verified;
ALTER TABLE users.users DROP COLUMN IF EXISTS verification_token;
ALTER TABLE users.users DROP COLUMN IF EXISTS verification_sent_at;
"""

# Usage example
async def run_migrations():
    from database_manager import DatabaseManager, DatabaseConfig
    
    # Initialize database manager
    db_manager = DatabaseManager()
    config = DatabaseConfig(
        host="localhost",
        port=5432,
        database="saas_platform",
        username="saas_user",
        password="secure_password"
    )
    await db_manager.initialize_postgresql(config)
    
    # Initialize migration manager
    migration_manager = MigrationManager(db_manager.postgresql)
    await migration_manager.initialize()
    
    # Check status
    status = await migration_manager.status()
    print(f"Migration status: {status}")
    
    # Apply migrations
    await migration_manager.migrate_up()
    
    # Close connections
    await db_manager.close_all()

if __name__ == "__main__":
    asyncio.run(run_migrations())
```

## Mejores Prácticas de Base de Datos

### 1. Rendimiento
- Usar índices apropiados
- Optimizar consultas complejas
- Implementar connection pooling
- Monitorear consultas lentas

### 2. Seguridad
- Usar usuarios con permisos mínimos
- Encriptar conexiones (SSL/TLS)
- Implementar Row Level Security
- Auditar accesos a datos

### 3. Backup y Recuperación
- Backups automáticos regulares
- Probar procedimientos de recuperación
- Replicación para alta disponibilidad
- Point-in-time recovery

### 4. Monitoreo
- Métricas de rendimiento
- Alertas de espacio en disco
- Monitoreo de conexiones
- Logs de errores y consultas lentas

Esta documentación proporciona una base sólida para la gestión de bases de datos en aplicaciones SAAS.