# Supabase Installation Guide

Supabase is an open-source Firebase alternative providing a PostgreSQL database, authentication, real-time subscriptions, storage, and vector embeddings. This guide covers self-hosting Supabase on your own infrastructure.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Supported Operating Systems](#supported-operating-systems)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Service Management](#service-management)
6. [Troubleshooting](#troubleshooting)
7. [Security Considerations](#security-considerations)
8. [Performance Tuning](#performance-tuning)
9. [Backup and Restore](#backup-and-restore)
10. [System Requirements](#system-requirements)
11. [Support](#support)
12. [Contributing](#contributing)
13. [License](#license)
14. [Acknowledgments](#acknowledgments)
15. [Version History](#version-history)
16. [Appendices](#appendices)

## 1. Prerequisites

- Operating system: Linux (Ubuntu/Debian recommended), macOS
- PostgreSQL 14+ 
- Node.js 16+ and npm
- Go 1.18+
- Minimum 4GB RAM (8GB+ recommended for production)
- Domain name with DNS configured
- SSL certificates (Let's Encrypt recommended)
- nginx or Caddy as reverse proxy

## Core Components

Supabase consists of multiple services:
- **PostgreSQL**: Core database with extensions
- **PostgREST**: RESTful API for PostgreSQL
- **GoTrue**: Authentication and user management
- **Realtime**: WebSocket server for real-time subscriptions
- **Storage API**: S3-compatible object storage
- **Kong**: API gateway
- **Vector**: Embeddings and similarity search


## 2. Supported Operating Systems

This guide supports installation on:
- RHEL 8/9 and derivatives (CentOS Stream, Rocky Linux, AlmaLinux)
- Debian 11/12
- Ubuntu 20.04/22.04/24.04 LTS
- Arch Linux (rolling release)
- Alpine Linux 3.18+
- openSUSE Leap 15.5+ / Tumbleweed
- SUSE Linux Enterprise Server (SLES) 15+
- macOS 12+ (Monterey and later) 
- FreeBSD 13+
- Windows 10/11/Server 2019+ (where applicable)

## 3. Installation

### System Dependencies

#### Debian/Ubuntu

```bash
# Update system
sudo apt update
sudo apt upgrade -y

# Install PostgreSQL 15
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt update
sudo apt install -y postgresql-15 postgresql-contrib-15

# Install Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install Go
wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc

# Install other dependencies
sudo apt install -y \
    git curl wget \
    build-essential \
    nginx \
    redis-server \
    certbot python3-certbot-nginx
```

#### RHEL/CentOS/Rocky Linux/AlmaLinux

```bash
# Install PostgreSQL 15
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql15-server postgresql15-contrib

# Initialize PostgreSQL
sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
sudo systemctl enable --now postgresql-15

# Install Node.js 18
curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo bash -
sudo dnf install -y nodejs

# Install Go
wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc

# Install other dependencies
sudo dnf install -y \
    git curl wget \
    gcc gcc-c++ make \
    nginx \
    redis \
    certbot python3-certbot-nginx

sudo systemctl enable --now redis nginx
```

#### Alpine Linux

```bash
# Install dependencies
apk add --no-cache \
    postgresql15 postgresql15-contrib \
    nodejs npm \
    go \
    git curl wget \
    build-base \
    nginx \
    redis \
    certbot certbot-nginx

# Start services
rc-update add postgresql default
rc-update add redis default
rc-update add nginx default
rc-service postgresql start
rc-service redis start
rc-service nginx start
```

#### macOS

```bash
# Install Homebrew dependencies
brew install \
    postgresql@15 \
    node@18 \
    go \
    redis \
    nginx

# Start services
brew services start postgresql@15
brew services start redis
brew services start nginx
```

### PostgreSQL Setup

```bash
# Create supabase user and databases
sudo -u postgres psql <<EOF
-- Create users
CREATE USER supabase_admin WITH SUPERUSER CREATEDB CREATEROLE REPLICATION BYPASSRLS PASSWORD 'your_admin_password';
CREATE USER authenticator WITH NOINHERIT LOGIN PASSWORD 'your_authenticator_password';
CREATE USER supabase_auth_admin WITH NOINHERIT CREATEROLE LOGIN PASSWORD 'your_auth_password';
CREATE USER supabase_storage_admin WITH NOINHERIT CREATEROLE LOGIN PASSWORD 'your_storage_password';
CREATE USER supabase_replication_admin WITH NOINHERIT CREATEROLE LOGIN PASSWORD 'your_replication_password';

-- Create database
CREATE DATABASE supabase OWNER supabase_admin;
\c supabase

-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS pgjwt;
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS pgaudit;
CREATE EXTENSION IF NOT EXISTS plpgsql;
CREATE EXTENSION IF NOT EXISTS plpgsql_check;
CREATE EXTENSION IF NOT EXISTS pg_cron;
CREATE EXTENSION IF NOT EXISTS pg_net;
CREATE EXTENSION IF NOT EXISTS pgsodium;
EOF
```

### Create Directory Structure

```bash
# Create base directory
sudo mkdir -p /opt/supabase/{bin,config,logs,data}
sudo useradd -r -s /bin/false -d /opt/supabase supabase
sudo chown -R supabase:supabase /opt/supabase
```

## Component Installation

### PostgREST

```bash
# Download PostgREST
cd /tmp
POSTGREST_VERSION="v11.2.0"
wget https://github.com/PostgREST/postgrest/releases/download/${POSTGREST_VERSION}/postgrest-${POSTGREST_VERSION}-linux-static-x64.tar.xz
tar xJf postgrest-${POSTGREST_VERSION}-linux-static-x64.tar.xz
sudo mv postgrest /opt/supabase/bin/
sudo chmod +x /opt/supabase/bin/postgrest

# Create configuration
sudo tee /opt/supabase/config/postgrest.conf <<EOF
db-uri = "postgres://authenticator:your_authenticator_password@localhost:5432/supabase"
db-schemas = "public,storage,graphql_public"
db-anon-role = "anon"
db-use-legacy-gucs = false

server-host = "127.0.0.1"
server-port = 3000

jwt-secret = "your-super-secret-jwt-token-with-at-least-32-characters"
jwt-aud = "authenticated"

max-rows = 1000
pre-request = "supabase.pre_request"
EOF
```

### GoTrue (Auth)

```bash
# Build GoTrue from source
cd /tmp
git clone https://github.com/supabase/gotrue.git
cd gotrue
go build -o gotrue ./cmd/gotrue
sudo mv gotrue /opt/supabase/bin/
sudo chmod +x /opt/supabase/bin/gotrue

# Create configuration
sudo tee /opt/supabase/config/gotrue.env <<EOF
GOTRUE_API_HOST=127.0.0.1
GOTRUE_API_PORT=9999
GOTRUE_DB_DRIVER=postgres
GOTRUE_DB_DATABASE_URL=postgres://supabase_auth_admin:your_auth_password@localhost:5432/supabase
GOTRUE_SITE_URL=https://your-domain.com
GOTRUE_URI_ALLOW_LIST=https://your-domain.com
GOTRUE_JWT_SECRET=your-super-secret-jwt-token-with-at-least-32-characters
GOTRUE_JWT_EXP=3600
GOTRUE_JWT_AUD=authenticated
GOTRUE_JWT_DEFAULT_GROUP_NAME=authenticated
GOTRUE_JWT_ADMIN_ROLES=service_role
GOTRUE_SMTP_HOST=smtp.gmail.com
GOTRUE_SMTP_PORT=587
GOTRUE_SMTP_USER=your-email@gmail.com
GOTRUE_SMTP_PASS=your-app-password
GOTRUE_SMTP_SENDER_NAME=Supabase
GOTRUE_MAILER_AUTOCONFIRM=false
GOTRUE_EXTERNAL_EMAIL_ENABLED=true
GOTRUE_EXTERNAL_PHONE_ENABLED=true
EOF
```

### Realtime

```bash
# Clone and build Realtime
cd /tmp
git clone https://github.com/supabase/realtime.git
cd realtime

# Install Elixir/Erlang dependencies first
# Debian/Ubuntu
wget https://packages.erlang-solutions.com/erlang-solutions_2.0_all.deb
sudo dpkg -i erlang-solutions_2.0_all.deb
sudo apt update
sudo apt install -y esl-erlang elixir

# Build Realtime
mix local.hex --force
mix local.rebar --force
MIX_ENV=prod mix do deps.get, compile
MIX_ENV=prod mix release

# Copy to supabase directory
sudo cp -r _build/prod/rel/realtime /opt/supabase/
sudo chown -R supabase:supabase /opt/supabase/realtime

# Create configuration
sudo tee /opt/supabase/config/realtime.conf <<EOF
PORT=4000
DB_HOST=localhost
DB_PORT=5432
DB_USER=supabase_admin
DB_PASSWORD=your_admin_password
DB_NAME=supabase
DB_SSL=false
SLOT_NAME=supabase_realtime_slot
TEMPORARY_SLOT=false
SECRET_KEY_BASE=your-secret-key-base-64-chars
JWT_SECRET=your-super-secret-jwt-token-with-at-least-32-characters
EOF
```

### Storage API

```bash
# Clone and build Storage API
cd /tmp
git clone https://github.com/supabase/storage-api.git
cd storage-api

# Install dependencies and build
npm install
npm run build
npm prune --production

# Move to supabase directory
sudo mv /tmp/storage-api /opt/supabase/storage
sudo chown -R supabase:supabase /opt/supabase/storage

# Create configuration
sudo tee /opt/supabase/storage/.env <<EOF
# API
PORT=5000
ANON_KEY=your-anon-key
SERVICE_KEY=your-service-key
JWT_SECRET=your-super-secret-jwt-token-with-at-least-32-characters
DATABASE_URL=postgres://supabase_storage_admin:your_storage_password@localhost:5432/supabase
PGRST_JWT_SECRET=your-super-secret-jwt-token-with-at-least-32-characters

# Storage
STORAGE_BACKEND=file
FILE_STORAGE_BACKEND_PATH=/opt/supabase/data/storage
TENANT_ID=default
REGION=us-east-1
GLOBAL_S3_BUCKET=supabase-storage

# S3 (optional)
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
EOF
```

### Kong API Gateway

```bash
# Install Kong
# Debian/Ubuntu
curl -Lo kong-3.4.2.amd64.deb "https://download.konghq.com/gateway-3.x-ubuntu-$(lsb_release -cs)/pool/all/k/kong/kong_3.4.2_amd64.deb"
sudo dpkg -i kong-3.4.2.amd64.deb

# Configure Kong
sudo tee /etc/kong/kong.conf <<EOF
prefix = /opt/supabase/kong
log_level = info
proxy_access_log = /opt/supabase/logs/kong_access.log
proxy_error_log = /opt/supabase/logs/kong_error.log
admin_access_log = /opt/supabase/logs/kong_admin_access.log
admin_error_log = /opt/supabase/logs/kong_admin_error.log
proxy_listen = 0.0.0.0:8000
admin_listen = 127.0.0.1:8001
database = postgres
pg_host = localhost
pg_port = 5432
pg_user = kong
pg_password = kong_password
pg_database = kong
EOF

# Initialize Kong database
sudo -u postgres createdb kong
sudo -u postgres psql -c "CREATE USER kong WITH PASSWORD 'kong_password';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE kong TO kong;"
sudo kong migrations bootstrap

# Start Kong
sudo kong start
```

## Service Configuration

### Create Systemd Services

#### PostgREST Service

Create `/etc/systemd/system/supabase-postgrest.service`:

```ini
[Unit]
Description=PostgREST API Server
After=postgresql.service

[Service]
Type=simple
User=supabase
Group=supabase
ExecStart=/opt/supabase/bin/postgrest /opt/supabase/config/postgrest.conf
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

#### GoTrue Service

Create `/etc/systemd/system/supabase-gotrue.service`:

```ini
[Unit]
Description=Supabase GoTrue Auth Server
After=postgresql.service

[Service]
Type=simple
User=supabase
Group=supabase
EnvironmentFile=/opt/supabase/config/gotrue.env
ExecStart=/opt/supabase/bin/gotrue serve
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

#### Realtime Service

Create `/etc/systemd/system/supabase-realtime.service`:

```ini
[Unit]
Description=Supabase Realtime Server
After=postgresql.service

[Service]
Type=simple
User=supabase
Group=supabase
EnvironmentFile=/opt/supabase/config/realtime.conf
ExecStart=/opt/supabase/realtime/bin/realtime start
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

#### Storage Service

Create `/etc/systemd/system/supabase-storage.service`:

```ini
[Unit]
Description=Supabase Storage API
After=postgresql.service

[Service]
Type=simple
User=supabase
Group=supabase
WorkingDirectory=/opt/supabase/storage
ExecStart=/usr/bin/node /opt/supabase/storage/dist/server.js
Restart=always
RestartSec=5
Environment="NODE_ENV=production"

[Install]
WantedBy=multi-user.target
```

Enable and start all services:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now supabase-postgrest supabase-gotrue supabase-realtime supabase-storage
```

## Reverse Proxy Configuration

### nginx Configuration

Create `/etc/nginx/sites-available/supabase`:

```nginx
upstream postgrest {
    server 127.0.0.1:3000;
}

upstream gotrue {
    server 127.0.0.1:9999;
}

upstream realtime {
    server 127.0.0.1:4000;
}

upstream storage {
    server 127.0.0.1:5000;
}

upstream kong {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name api.your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.your-domain.com;

    ssl_certificate /etc/letsencrypt/live/api.your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.your-domain.com/privkey.pem;

    # Main API Gateway
    location / {
        proxy_pass http://kong;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # REST API
    location /rest/v1/ {
        proxy_pass http://postgrest/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Auth API
    location /auth/v1/ {
        proxy_pass http://gotrue/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Realtime API
    location /realtime/v1/ {
        proxy_pass http://realtime/socket/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Storage API
    location /storage/v1/ {
        proxy_pass http://storage/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 5G;
    }
}
```

Enable site:
```bash
sudo ln -s /etc/nginx/sites-available/supabase /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## Database Schema Setup

```sql
-- Create schemas
CREATE SCHEMA IF NOT EXISTS auth;
CREATE SCHEMA IF NOT EXISTS storage;
CREATE SCHEMA IF NOT EXISTS graphql_public;

-- Create roles
CREATE ROLE anon NOLOGIN NOINHERIT;
CREATE ROLE authenticated NOLOGIN NOINHERIT;
CREATE ROLE service_role NOLOGIN NOINHERIT BYPASSRLS;

-- Grant permissions
GRANT USAGE ON SCHEMA public TO anon, authenticated, service_role;
GRANT USAGE ON SCHEMA auth TO anon, authenticated, service_role;
GRANT USAGE ON SCHEMA storage TO anon, authenticated, service_role;

-- Auth schema setup
GRANT ALL ON SCHEMA auth TO supabase_auth_admin;
GRANT ALL ON ALL TABLES IN SCHEMA auth TO supabase_auth_admin;
GRANT ALL ON ALL SEQUENCES IN SCHEMA auth TO supabase_auth_admin;
GRANT ALL ON ALL ROUTINES IN SCHEMA auth TO supabase_auth_admin;

-- Storage schema setup
GRANT ALL ON SCHEMA storage TO supabase_storage_admin;
GRANT ALL ON ALL TABLES IN SCHEMA storage TO supabase_storage_admin;
GRANT ALL ON ALL SEQUENCES IN SCHEMA storage TO supabase_storage_admin;
GRANT ALL ON ALL ROUTINES IN SCHEMA storage TO supabase_storage_admin;

-- Create auth tables
CREATE TABLE IF NOT EXISTS auth.users (
    instance_id uuid,
    id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    aud varchar(255),
    role varchar(255),
    email varchar(255) UNIQUE,
    encrypted_password varchar(255),
    email_confirmed_at timestamptz,
    invited_at timestamptz,
    confirmation_token varchar(255),
    confirmation_sent_at timestamptz,
    recovery_token varchar(255),
    recovery_sent_at timestamptz,
    email_change_token_new varchar(255),
    email_change varchar(255),
    email_change_sent_at timestamptz,
    last_sign_in_at timestamptz,
    raw_app_meta_data jsonb,
    raw_user_meta_data jsonb,
    is_super_admin boolean,
    created_at timestamptz,
    updated_at timestamptz,
    phone varchar(15),
    phone_confirmed_at timestamptz,
    phone_change varchar(15),
    phone_change_token varchar(255),
    phone_change_sent_at timestamptz,
    confirmed_at timestamptz GENERATED ALWAYS AS (LEAST(email_confirmed_at, phone_confirmed_at)) STORED,
    email_change_token_current varchar(255),
    email_change_confirm_status smallint,
    banned_until timestamptz,
    reauthentication_token varchar(255),
    reauthentication_sent_at timestamptz,
    is_sso_user boolean DEFAULT false NOT NULL
);

-- Enable RLS
ALTER TABLE auth.users ENABLE ROW LEVEL SECURITY;
```

## Configure Kong Routes

```bash
# Add PostgREST service
curl -i -X POST http://localhost:8001/services/ \
  --data name=postgrest \
  --data url=http://localhost:3000

# Add PostgREST route
curl -i -X POST http://localhost:8001/services/postgrest/routes \
  --data paths[]=/rest/v1 \
  --data strip_path=true

# Add GoTrue service
curl -i -X POST http://localhost:8001/services/ \
  --data name=gotrue \
  --data url=http://localhost:9999

# Add GoTrue route
curl -i -X POST http://localhost:8001/services/gotrue/routes \
  --data paths[]=/auth/v1 \
  --data strip_path=true

# Add JWT plugin
curl -i -X POST http://localhost:8001/plugins \
  --data name=jwt \
  --data config.secret_is_base64=false \
  --data config.secret=your-super-secret-jwt-token-with-at-least-32-characters
```

## Security Configuration

### JWT Configuration

Generate secure JWT secret:
```bash
openssl rand -base64 32
```

Update all services with the same JWT secret in their configurations.

### API Keys

Generate API keys:
```bash
# Generate anon key (public)
ANON_KEY=$(node -e "
const jwt = require('jsonwebtoken');
const secret = 'your-super-secret-jwt-token-with-at-least-32-characters';
const payload = {
  role: 'anon',
  iss: 'supabase',
  iat: Math.floor(Date.now() / 1000),
  exp: Math.floor(Date.now() / 1000) + (10 * 365 * 24 * 60 * 60) // 10 years
};
console.log(jwt.sign(payload, secret));
")

# Generate service key (private)
SERVICE_KEY=$(node -e "
const jwt = require('jsonwebtoken');
const secret = 'your-super-secret-jwt-token-with-at-least-32-characters';
const payload = {
  role: 'service_role',
  iss: 'supabase',
  iat: Math.floor(Date.now() / 1000),
  exp: Math.floor(Date.now() / 1000) + (10 * 365 * 24 * 60 * 60) // 10 years
};
console.log(jwt.sign(payload, secret));
")

echo "ANON_KEY: $ANON_KEY"
echo "SERVICE_KEY: $SERVICE_KEY"
```

### SSL/TLS Setup

```bash
# Obtain SSL certificates
sudo certbot --nginx -d api.your-domain.com

# Auto-renewal
sudo certbot renew --dry-run
```

## 9. Backup and Restore

### Backup Script

```bash
#!/bin/bash
# backup-supabase.sh

BACKUP_DIR="/backup/supabase/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Backup database
sudo -u postgres pg_dump supabase | gzip > "$BACKUP_DIR/supabase.sql.gz"

# Backup configuration
tar czf "$BACKUP_DIR/config.tar.gz" -C /opt/supabase config/

# Backup storage files
tar czf "$BACKUP_DIR/storage.tar.gz" -C /opt/supabase/data storage/

echo "Backup completed: $BACKUP_DIR"
```

### Restore Script

```bash
#!/bin/bash
# restore-supabase.sh

BACKUP_DIR="$1"

# Stop services
sudo systemctl stop supabase-*

# Restore database
gunzip < "$BACKUP_DIR/supabase.sql.gz" | sudo -u postgres psql supabase

# Restore configuration
tar xzf "$BACKUP_DIR/config.tar.gz" -C /opt/supabase/

# Restore storage
tar xzf "$BACKUP_DIR/storage.tar.gz" -C /opt/supabase/data/

# Fix permissions
sudo chown -R supabase:supabase /opt/supabase

# Start services
sudo systemctl start supabase-*

echo "Restore completed"
```

## Monitoring

### Health Checks

```bash
# Check all services
curl http://localhost:3000/         # PostgREST
curl http://localhost:9999/health   # GoTrue
curl http://localhost:4000/         # Realtime
curl http://localhost:5000/status   # Storage

# Check through Kong
curl https://api.your-domain.com/rest/v1/
curl https://api.your-domain.com/auth/v1/health
```

### Logs

```bash
# View service logs
sudo journalctl -u supabase-postgrest -f
sudo journalctl -u supabase-gotrue -f
sudo journalctl -u supabase-realtime -f
sudo journalctl -u supabase-storage -f

# Kong logs
tail -f /opt/supabase/logs/kong_access.log
```

## Client Configuration

### JavaScript/TypeScript

```javascript
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = 'https://api.your-domain.com'
const supabaseAnonKey = 'your-anon-key'

const supabase = createClient(supabaseUrl, supabaseAnonKey)

// Example usage
const { data, error } = await supabase
  .from('users')
  .select('*')
```

### Python

```python
from supabase import create_client

url = "https://api.your-domain.com"
key = "your-anon-key"

supabase = create_client(url, key)

# Example usage
response = supabase.table('users').select("*").execute()
```

## 6. Troubleshooting

### Common Issues

1. **Connection refused errors**:
```bash
# Check if services are running
sudo systemctl status supabase-*

# Check ports
sudo netstat -tlnp | grep -E '3000|9999|4000|5000|8000'
```

2. **JWT errors**:
```bash
# Verify JWT secret is consistent across all services
# Check PostgREST config
grep jwt-secret /opt/supabase/config/postgrest.conf

# Check GoTrue config
grep JWT_SECRET /opt/supabase/config/gotrue.env
```

3. **Database connection issues**:
```bash
# Test database connection
psql -h localhost -U supabase_admin -d supabase -c "SELECT 1;"

# Check PostgreSQL logs
sudo journalctl -u postgresql -f
```

## 8. Performance Tuning

### PostgreSQL Optimization

```sql
-- Adjust PostgreSQL settings for Supabase
ALTER SYSTEM SET shared_buffers = '2GB';
ALTER SYSTEM SET effective_cache_size = '6GB';
ALTER SYSTEM SET maintenance_work_mem = '512MB';
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_io_concurrency = 200;
ALTER SYSTEM SET max_worker_processes = 8;
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
ALTER SYSTEM SET max_parallel_workers = 8;

-- Reload configuration
SELECT pg_reload_conf();
```

### Connection Pooling

Consider using PgBouncer for connection pooling:

```bash
# Install PgBouncer
sudo apt install -y pgbouncer

# Configure PgBouncer
sudo tee /etc/pgbouncer/pgbouncer.ini <<EOF
[databases]
supabase = host=localhost port=5432 dbname=supabase

[pgbouncer]
listen_port = 6432
listen_addr = 127.0.0.1
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
EOF
```

## Additional Resources

- [Official Documentation](https://supabase.com/docs)
- [GitHub Repository](https://github.com/supabase/supabase)
- [Self-Hosting Guide](https://supabase.com/docs/guides/hosting/overview)
- [Community Discord](https://discord.supabase.com/)
- [API Reference](https://supabase.com/docs/reference)

---

**Note:** This guide is part of the [HowToMgr](https://howtomgr.github.io) collection. Always refer to official documentation for the most up-to-date information.