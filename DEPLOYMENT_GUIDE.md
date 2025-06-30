# MindSwarm Deployment & Operations Guide

This guide covers production deployment, monitoring, and operational aspects of the MindSwarm system.

## Table of Contents

1. [System Requirements](#system-requirements)
2. [Production Deployment](#production-deployment)
3. [Configuration Management](#configuration-management)
4. [Monitoring & Observability](#monitoring--observability)
5. [Security Hardening](#security-hardening)
6. [Backup & Recovery](#backup--recovery)
7. [Scaling Strategies](#scaling-strategies)
8. [Troubleshooting](#troubleshooting)
9. [Maintenance Procedures](#maintenance-procedures)
10. [Performance Tuning](#performance-tuning)

## System Requirements

### Minimum Hardware Requirements

**Single Instance**:
- CPU: 4 cores (8 threads)
- RAM: 8GB
- Storage: 50GB SSD
- Network: 1Gbps

**Production Cluster**:
- CPU: 8+ cores per node
- RAM: 16GB+ per node
- Storage: 100GB+ SSD per node
- Network: 10Gbps backbone

### Software Dependencies

**Operating System**:
- Ubuntu 22.04 LTS (recommended)
- CentOS 8/RHEL 8
- Docker-compatible Linux distro

**Runtime Requirements**:
- Python 3.11+
- Node.js 18+ (for frontend)
- Redis 6+ (for caching/sessions)
- PostgreSQL 14+ (for persistent data)
- Nginx (reverse proxy)

### Network Requirements

**Inbound Ports**:
- 80/443: HTTP/HTTPS traffic
- 8000: MindSwarm API (internal)
- 22: SSH (management)

**Outbound Connectivity**:
- OpenRouter API: api.openrouter.ai (HTTPS)
- GitHub API: api.github.com (HTTPS)
- Package repositories (for updates)

## Production Deployment

### Container-Based Deployment

**Directory Structure**:
```
/opt/mindswarm/
├── docker-compose.yml
├── .env
├── data/
│   ├── projects/
│   ├── logs/
│   └── backups/
├── runtime/
│   ├── agent_types/
│   ├── tools/
│   ├── templates/
│   └── knowledge/
├── config/
│   ├── nginx.conf
│   ├── ssl/
│   └── secrets/
└── scripts/
    ├── deploy.sh
    ├── backup.sh
    └── health-check.sh
```

**Production Docker Compose**:
```yaml
version: '3.8'

services:
  mindswarm-core:
    image: mindswarm/core:latest
    container_name: mindswarm-core
    restart: unless-stopped
    environment:
      - MINDSWARM_RUNTIME_PATH=/runtime
      - MINDSWARM_DATA_DIR=/data
      - LOG_LEVEL=INFO
      - REDIS_URL=redis://redis:6379/0
      - DATABASE_URL=postgresql://mindswarm:${DB_PASSWORD}@postgres:5432/mindswarm
      - OPENROUTER_API_KEY=${OPENROUTER_API_KEY}
      - JWT_SECRET=${JWT_SECRET}
    volumes:
      - ./data:/data
      - ./runtime:/runtime:ro
      - ./config/secrets:/secrets:ro
    networks:
      - mindswarm-network
    depends_on:
      - redis
      - postgres
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  nginx:
    image: nginx:alpine
    container_name: mindswarm-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./config/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./config/ssl:/etc/nginx/ssl:ro
      - ./logs/nginx:/var/log/nginx
    networks:
      - mindswarm-network
    depends_on:
      - mindswarm-core

  redis:
    image: redis:7-alpine
    container_name: mindswarm-redis
    restart: unless-stopped
    command: redis-server --appendonly yes --maxmemory 2gb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    networks:
      - mindswarm-network

  postgres:
    image: postgres:15-alpine
    container_name: mindswarm-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=mindswarm
      - POSTGRES_USER=mindswarm
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./backups:/backups
    networks:
      - mindswarm-network

  prometheus:
    image: prom/prometheus:latest
    container_name: mindswarm-prometheus
    restart: unless-stopped
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    networks:
      - mindswarm-network

  grafana:
    image: grafana/grafana:latest
    container_name: mindswarm-grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana-data:/var/lib/grafana
      - ./config/grafana:/etc/grafana/provisioning:ro
    networks:
      - mindswarm-network

networks:
  mindswarm-network:
    driver: bridge

volumes:
  redis-data:
  postgres-data:
  prometheus-data:
  grafana-data:
```

**Environment Configuration (.env)**:
```bash
# Security
JWT_SECRET=your-jwt-secret-here
DB_PASSWORD=secure-database-password
GRAFANA_PASSWORD=grafana-admin-password

# API Keys
OPENROUTER_API_KEY=your-openrouter-api-key

# Application Settings
MINDSWARM_HOST=0.0.0.0
MINDSWARM_PORT=8000
LOG_LEVEL=INFO

# Scaling
MAX_AGENTS_PER_PROJECT=50
MAX_CONCURRENT_REQUESTS=100
```

### Kubernetes Deployment

**Namespace and ConfigMap**:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mindswarm

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mindswarm-config
  namespace: mindswarm
data:
  LOG_LEVEL: "INFO"
  MINDSWARM_HOST: "0.0.0.0"
  MINDSWARM_PORT: "8000"
  MAX_AGENTS_PER_PROJECT: "50"
```

**Deployment**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mindswarm-core
  namespace: mindswarm
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: mindswarm-core
  template:
    metadata:
      labels:
        app: mindswarm-core
    spec:
      containers:
      - name: mindswarm-core
        image: mindswarm/core:latest
        ports:
        - containerPort: 8000
        env:
        - name: MINDSWARM_RUNTIME_PATH
          value: "/runtime"
        - name: MINDSWARM_DATA_DIR
          value: "/data"
        envFrom:
        - configMapRef:
            name: mindswarm-config
        - secretRef:
            name: mindswarm-secrets
        volumeMounts:
        - name: runtime-volume
          mountPath: /runtime
          readOnly: true
        - name: data-volume
          mountPath: /data
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
      volumes:
      - name: runtime-volume
        configMap:
          name: mindswarm-runtime
      - name: data-volume
        persistentVolumeClaim:
          claimName: mindswarm-data-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: mindswarm-core-service
  namespace: mindswarm
spec:
  selector:
    app: mindswarm-core
  ports:
  - protocol: TCP
    port: 8000
    targetPort: 8000
  type: ClusterIP
```

## Configuration Management

### Environment-Specific Configurations

**Development**:
```yaml
# config/dev.yaml
environment: development
debug: true
hot_reload: true
log_level: DEBUG
cors_origins: ["http://localhost:3000"]
auth:
  require_authentication: false
database:
  pool_size: 5
ai:
  default_model: "google/gemini-2.0-flash-001"
  timeout: 30
```

**Staging**:
```yaml
# config/staging.yaml
environment: staging
debug: false
hot_reload: false
log_level: INFO
cors_origins: ["https://staging.company.com"]
auth:
  require_authentication: true
database:
  pool_size: 10
ai:
  default_model: "google/gemini-2.0-flash-001"
  timeout: 60
```

**Production**:
```yaml
# config/production.yaml
environment: production
debug: false
hot_reload: false
log_level: INFO
cors_origins: ["https://app.company.com"]
auth:
  require_authentication: true
  session_timeout: 3600
database:
  pool_size: 20
  statement_timeout: 30s
ai:
  default_model: "google/gemini-2.0-flash-001"
  timeout: 120
  rate_limit:
    requests_per_minute: 100
performance:
  max_agents_per_project: 50
  context_cleanup_interval: 1800
  max_memory_per_agent: "100MB"
```

### Secret Management

**Using Kubernetes Secrets**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mindswarm-secrets
  namespace: mindswarm
type: Opaque
stringData:
  JWT_SECRET: "your-jwt-secret"
  DB_PASSWORD: "secure-password"
  OPENROUTER_API_KEY: "your-api-key"
  REDIS_PASSWORD: "redis-password"
```

**Using HashiCorp Vault**:
```python
# config/vault_integration.py
import hvac
import os

class VaultConfig:
    def __init__(self):
        self.client = hvac.Client(
            url=os.getenv('VAULT_ADDR'),
            token=os.getenv('VAULT_TOKEN')
        )
    
    def get_secret(self, path: str) -> dict:
        """Retrieve secret from Vault."""
        response = self.client.secrets.kv.v2.read_secret_version(path=path)
        return response['data']['data']
    
    def load_secrets(self):
        """Load all required secrets."""
        secrets = self.get_secret('mindswarm/production')
        
        os.environ.update({
            'JWT_SECRET': secrets['jwt_secret'],
            'DB_PASSWORD': secrets['db_password'],
            'OPENROUTER_API_KEY': secrets['openrouter_api_key']
        })
```

### Configuration Validation

```python
# config/validator.py
from pydantic import BaseModel, validator
from typing import List, Optional

class DatabaseConfig(BaseModel):
    url: str
    pool_size: int = 10
    statement_timeout: str = "30s"
    
    @validator('pool_size')
    def validate_pool_size(cls, v):
        if v < 1 or v > 100:
            raise ValueError('Pool size must be between 1 and 100')
        return v

class AIConfig(BaseModel):
    default_model: str
    timeout: int = 60
    rate_limit: Optional[dict] = None
    
    @validator('timeout')
    def validate_timeout(cls, v):
        if v < 1 or v > 300:
            raise ValueError('Timeout must be between 1 and 300 seconds')
        return v

class MindSwarmConfig(BaseModel):
    environment: str
    debug: bool = False
    hot_reload: bool = False
    log_level: str = "INFO"
    cors_origins: List[str] = ["*"]
    database: DatabaseConfig
    ai: AIConfig
    
    @validator('environment')
    def validate_environment(cls, v):
        if v not in ['development', 'staging', 'production']:
            raise ValueError('Invalid environment')
        return v
```

## Monitoring & Observability

### Metrics Collection

**Prometheus Configuration**:
```yaml
# config/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "mindswarm_rules.yml"

scrape_configs:
  - job_name: 'mindswarm-core'
    static_configs:
      - targets: ['mindswarm-core:8000']
    metrics_path: '/metrics'
    scrape_interval: 10s

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
```

**Custom Metrics in MindSwarm**:
```python
# monitoring/metrics.py
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

# Agent metrics
agent_created_total = Counter('mindswarm_agents_created_total', 'Total agents created', ['project_id', 'agent_type'])
agent_active_gauge = Gauge('mindswarm_agents_active', 'Currently active agents', ['project_id'])
agent_processing_time = Histogram('mindswarm_agent_processing_seconds', 'Agent processing time', ['agent_type'])

# Mail metrics
mail_sent_total = Counter('mindswarm_mail_sent_total', 'Total mail sent', ['from_type', 'to_type'])
mail_delivery_time = Histogram('mindswarm_mail_delivery_seconds', 'Mail delivery time')

# Tool metrics
tool_execution_total = Counter('mindswarm_tool_executions_total', 'Tool executions', ['tool_name', 'status'])
tool_execution_time = Histogram('mindswarm_tool_execution_seconds', 'Tool execution time', ['tool_name'])

# Hot reload metrics
hot_reload_total = Counter('mindswarm_hot_reloads_total', 'Hot reloads', ['component_type', 'status'])

class MetricsCollector:
    @staticmethod
    def record_agent_creation(project_id: str, agent_type: str):
        agent_created_total.labels(project_id=project_id, agent_type=agent_type).inc()
    
    @staticmethod
    def update_active_agents(project_id: str, count: int):
        agent_active_gauge.labels(project_id=project_id).set(count)
    
    @staticmethod
    def record_mail_sent(from_type: str, to_type: str, delivery_time: float):
        mail_sent_total.labels(from_type=from_type, to_type=to_type).inc()
        mail_delivery_time.observe(delivery_time)
    
    @staticmethod
    def record_tool_execution(tool_name: str, execution_time: float, success: bool):
        status = "success" if success else "error"
        tool_execution_total.labels(tool_name=tool_name, status=status).inc()
        tool_execution_time.labels(tool_name=tool_name).observe(execution_time)

# Start metrics server
def start_metrics_server(port: int = 9090):
    start_http_server(port)
```

### Alerting Rules

**Prometheus Rules**:
```yaml
# config/mindswarm_rules.yml
groups:
  - name: mindswarm.rules
    rules:
      - alert: MindSwarmHighAgentFailureRate
        expr: rate(mindswarm_agents_errors_total[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High agent failure rate"
          description: "Agent failure rate is {{ $value }} per second"

      - alert: MindSwarmHighMemoryUsage
        expr: (mindswarm_memory_usage_bytes / mindswarm_memory_limit_bytes) > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value | humanizePercentage }}"

      - alert: MindSwarmMailDeliveryLatency
        expr: histogram_quantile(0.95, rate(mindswarm_mail_delivery_seconds_bucket[5m])) > 5
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "High mail delivery latency"
          description: "95th percentile latency is {{ $value }}s"

      - alert: MindSwarmServiceDown
        expr: up{job="mindswarm-core"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MindSwarm service is down"
          description: "MindSwarm core service has been down for more than 1 minute"
```

### Logging Strategy

**Structured Logging Configuration**:
```python
# logging_config.py
import logging
import json
from datetime import datetime

class StructuredFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno
        }
        
        # Add extra fields
        if hasattr(record, 'agent_id'):
            log_entry['agent_id'] = record.agent_id
        if hasattr(record, 'project_id'):
            log_entry['project_id'] = record.project_id
        if hasattr(record, 'user_id'):
            log_entry['user_id'] = record.user_id
        
        return json.dumps(log_entry)

# Production logging config
LOGGING_CONFIG = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'structured': {
            '()': StructuredFormatter,
        },
        'simple': {
            'format': '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        }
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'structured',
            'level': 'INFO'
        },
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/data/logs/mindswarm.log',
            'maxBytes': 10485760,  # 10MB
            'backupCount': 5,
            'formatter': 'structured',
            'level': 'INFO'
        },
        'error_file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/data/logs/mindswarm_errors.log',
            'maxBytes': 10485760,  # 10MB
            'backupCount': 10,
            'formatter': 'structured',
            'level': 'ERROR'
        }
    },
    'loggers': {
        'mindswarm': {
            'handlers': ['console', 'file', 'error_file'],
            'level': 'INFO',
            'propagate': False
        },
        'uvicorn': {
            'handlers': ['console', 'file'],
            'level': 'INFO',
            'propagate': False
        }
    }
}
```

### Health Checks

**Comprehensive Health Check**:
```python
# health/health_checker.py
from dataclasses import dataclass
from typing import Dict, List
import asyncio
import time

@dataclass
class HealthStatus:
    component: str
    status: str  # "healthy", "degraded", "unhealthy"
    message: str
    response_time_ms: float
    details: Dict = None

class HealthChecker:
    def __init__(self, agent_manager, mailbox, tool_registry):
        self.agent_manager = agent_manager
        self.mailbox = mailbox
        self.tool_registry = tool_registry
    
    async def check_all(self) -> Dict[str, HealthStatus]:
        """Perform comprehensive health check."""
        checks = [
            self._check_agent_manager(),
            self._check_mailbox(),
            self._check_tool_registry(),
            self._check_database(),
            self._check_redis(),
            self._check_external_apis()
        ]
        
        results = await asyncio.gather(*checks, return_exceptions=True)
        
        health_status = {}
        for result in results:
            if isinstance(result, HealthStatus):
                health_status[result.component] = result
            else:
                # Handle exceptions
                health_status["unknown"] = HealthStatus(
                    component="unknown",
                    status="unhealthy",
                    message=str(result),
                    response_time_ms=0
                )
        
        return health_status
    
    async def _check_agent_manager(self) -> HealthStatus:
        start_time = time.time()
        try:
            states = self.agent_manager.get_agent_states()
            response_time = (time.time() - start_time) * 1000
            
            total_agents = len(states)
            active_agents = len([s for s in states.values() if s["state"] == "ACTIVE"])
            
            return HealthStatus(
                component="agent_manager",
                status="healthy",
                message=f"{total_agents} agents, {active_agents} active",
                response_time_ms=response_time,
                details={"total_agents": total_agents, "active_agents": active_agents}
            )
        except Exception as e:
            response_time = (time.time() - start_time) * 1000
            return HealthStatus(
                component="agent_manager",
                status="unhealthy",
                message=str(e),
                response_time_ms=response_time
            )
    
    async def _check_mailbox(self) -> HealthStatus:
        start_time = time.time()
        try:
            # Test mail sending
            from mindswarm.core.communication.mailbox import Mail
            
            test_mail = Mail(
                from_address="health@system.local.mindswarm.ltngt.ai",
                to_address="health@system.local.mindswarm.ltngt.ai",
                subject="Health Check",
                body="System health check"
            )
            
            result = self.mailbox.send_mail(test_mail)
            response_time = (time.time() - start_time) * 1000
            
            if result.success:
                return HealthStatus(
                    component="mailbox",
                    status="healthy",
                    message="Mail system operational",
                    response_time_ms=response_time
                )
            else:
                return HealthStatus(
                    component="mailbox",
                    status="degraded",
                    message=f"Mail system issue: {result.message}",
                    response_time_ms=response_time
                )
        except Exception as e:
            response_time = (time.time() - start_time) * 1000
            return HealthStatus(
                component="mailbox",
                status="unhealthy",
                message=str(e),
                response_time_ms=response_time
            )
    
    async def _check_tool_registry(self) -> HealthStatus:
        start_time = time.time()
        try:
            available_tools = self.tool_registry.get_available_tool_count()
            response_time = (time.time() - start_time) * 1000
            
            return HealthStatus(
                component="tool_registry",
                status="healthy",
                message=f"{available_tools} tools available",
                response_time_ms=response_time,
                details={"available_tools": available_tools}
            )
        except Exception as e:
            response_time = (time.time() - start_time) * 1000
            return HealthStatus(
                component="tool_registry",
                status="unhealthy",
                message=str(e),
                response_time_ms=response_time
            )
```

## Security Hardening

### Network Security

**Firewall Configuration (UFW)**:
```bash
# Reset firewall
sudo ufw --force reset

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH access (restrict to management network)
sudo ufw allow from 10.0.1.0/24 to any port 22

# HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Internal services (from application network)
sudo ufw allow from 10.0.2.0/24 to any port 8000  # MindSwarm API
sudo ufw allow from 10.0.2.0/24 to any port 5432  # PostgreSQL
sudo ufw allow from 10.0.2.0/24 to any port 6379  # Redis

# Enable firewall
sudo ufw enable
```

**Nginx Security Configuration**:
```nginx
# config/nginx.conf
server {
    listen 443 ssl http2;
    server_name api.company.com;
    
    # SSL Configuration
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # Security Headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';";
    
    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=ws:10m rate=5r/s;
    
    location / {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://mindswarm-core:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    location /ws {
        limit_req zone=ws burst=10 nodelay;
        proxy_pass http://mindswarm-core:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }
}
```

### Application Security

**JWT Configuration**:
```python
# security/jwt_config.py
import jwt
from datetime import datetime, timedelta
from typing import Dict, Optional

class JWTManager:
    def __init__(self, secret_key: str, algorithm: str = "HS256"):
        self.secret_key = secret_key
        self.algorithm = algorithm
        self.access_token_expire = timedelta(hours=1)
        self.refresh_token_expire = timedelta(days=7)
    
    def create_access_token(self, user_id: str, additional_claims: Dict = None) -> str:
        """Create access token with short expiration."""
        payload = {
            "user_id": user_id,
            "type": "access",
            "exp": datetime.utcnow() + self.access_token_expire,
            "iat": datetime.utcnow()
        }
        
        if additional_claims:
            payload.update(additional_claims)
        
        return jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
    
    def create_refresh_token(self, user_id: str) -> str:
        """Create refresh token with long expiration."""
        payload = {
            "user_id": user_id,
            "type": "refresh",
            "exp": datetime.utcnow() + self.refresh_token_expire,
            "iat": datetime.utcnow()
        }
        
        return jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
    
    def verify_token(self, token: str) -> Optional[Dict]:
        """Verify and decode token."""
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            return payload
        except jwt.ExpiredSignatureError:
            return None
        except jwt.InvalidTokenError:
            return None
```

**Input Validation**:
```python
# security/validation.py
from pydantic import BaseModel, validator
import re

class EmailAddress(BaseModel):
    email: str
    
    @validator('email')
    def validate_email_format(cls, v):
        # MindSwarm email format validation
        pattern = r'^[a-zA-Z0-9_-]+@[a-zA-Z0-9_-]+\.local\.mindswarm\.ltngt\.ai$'
        if not re.match(pattern, v):
            raise ValueError('Invalid MindSwarm email format')
        return v

class MailMessage(BaseModel):
    to_address: EmailAddress
    from_address: EmailAddress
    subject: str
    body: str
    
    @validator('subject')
    def validate_subject(cls, v):
        if len(v) > 200:
            raise ValueError('Subject too long')
        return v
    
    @validator('body')
    def validate_body(cls, v):
        if len(v) > 50000:  # 50KB limit
            raise ValueError('Body too long')
        return v
```

### Database Security

**PostgreSQL Hardening**:
```sql
-- Create restricted database user
CREATE USER mindswarm_app WITH PASSWORD 'secure_password';

-- Grant minimal required permissions
GRANT CONNECT ON DATABASE mindswarm TO mindswarm_app;
GRANT USAGE ON SCHEMA public TO mindswarm_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO mindswarm_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO mindswarm_app;

-- Enable SSL
ALTER SYSTEM SET ssl = on;
ALTER SYSTEM SET ssl_cert_file = '/var/lib/postgresql/server.crt';
ALTER SYSTEM SET ssl_key_file = '/var/lib/postgresql/server.key';

-- Logging
ALTER SYSTEM SET log_connections = on;
ALTER SYSTEM SET log_disconnections = on;
ALTER SYSTEM SET log_statement = 'mod';

-- Connection limits
ALTER SYSTEM SET max_connections = 100;
ALTER USER mindswarm_app CONNECTION LIMIT 20;
```

## Backup & Recovery

### Automated Backup Strategy

**Backup Script**:
```bash
#!/bin/bash
# scripts/backup.sh

set -e

BACKUP_DIR="/opt/mindswarm/backups"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# Create backup directory
mkdir -p "$BACKUP_DIR/$DATE"

echo "Starting backup at $(date)"

# Database backup
echo "Backing up PostgreSQL database..."
docker exec mindswarm-postgres pg_dump -U mindswarm mindswarm | gzip > "$BACKUP_DIR/$DATE/database.sql.gz"

# Redis backup
echo "Backing up Redis data..."
docker exec mindswarm-redis redis-cli BGSAVE
sleep 5
docker cp mindswarm-redis:/data/dump.rdb "$BACKUP_DIR/$DATE/redis.rdb"

# Application data backup
echo "Backing up application data..."
tar -czf "$BACKUP_DIR/$DATE/app_data.tar.gz" -C /opt/mindswarm/data .

# Runtime components backup
echo "Backing up runtime components..."
tar -czf "$BACKUP_DIR/$DATE/runtime.tar.gz" -C /opt/mindswarm/runtime .

# Configuration backup
echo "Backing up configuration..."
tar -czf "$BACKUP_DIR/$DATE/config.tar.gz" -C /opt/mindswarm/config .

# Cleanup old backups
echo "Cleaning up old backups..."
find "$BACKUP_DIR" -type d -mtime +$RETENTION_DAYS -exec rm -rf {} +

echo "Backup completed at $(date)"

# Verify backup
BACKUP_SIZE=$(du -sh "$BACKUP_DIR/$DATE" | cut -f1)
echo "Backup size: $BACKUP_SIZE"

# Upload to cloud storage (optional)
if [ "$CLOUD_BACKUP_ENABLED" = "true" ]; then
    echo "Uploading to cloud storage..."
    aws s3 sync "$BACKUP_DIR/$DATE" "s3://$BACKUP_BUCKET/mindswarm/$DATE/"
fi
```

### Recovery Procedures

**Database Recovery**:
```bash
#!/bin/bash
# scripts/restore_database.sh

BACKUP_DATE=$1
BACKUP_DIR="/opt/mindswarm/backups"

if [ -z "$BACKUP_DATE" ]; then
    echo "Usage: $0 <backup_date>"
    echo "Available backups:"
    ls -1 "$BACKUP_DIR"
    exit 1
fi

echo "Restoring database from backup $BACKUP_DATE"

# Stop application
docker-compose stop mindswarm-core

# Drop and recreate database
docker exec mindswarm-postgres psql -U postgres -c "DROP DATABASE IF EXISTS mindswarm;"
docker exec mindswarm-postgres psql -U postgres -c "CREATE DATABASE mindswarm OWNER mindswarm;"

# Restore database
gunzip -c "$BACKUP_DIR/$BACKUP_DATE/database.sql.gz" | docker exec -i mindswarm-postgres psql -U mindswarm mindswarm

# Restart application
docker-compose start mindswarm-core

echo "Database restore completed"
```

**Disaster Recovery Playbook**:
```bash
#!/bin/bash
# scripts/disaster_recovery.sh

echo "=== MindSwarm Disaster Recovery ==="
echo "This script will restore the system from backup"
echo "WARNING: This will overwrite current data"
read -p "Are you sure? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
    echo "Recovery cancelled"
    exit 1
fi

BACKUP_DATE=$1
if [ -z "$BACKUP_DATE" ]; then
    echo "Available backups:"
    ls -1 /opt/mindswarm/backups
    read -p "Enter backup date: " BACKUP_DATE
fi

BACKUP_PATH="/opt/mindswarm/backups/$BACKUP_DATE"

if [ ! -d "$BACKUP_PATH" ]; then
    echo "Backup not found: $BACKUP_PATH"
    exit 1
fi

echo "Starting disaster recovery with backup: $BACKUP_DATE"

# Stop all services
docker-compose down

# Restore configuration
echo "Restoring configuration..."
tar -xzf "$BACKUP_PATH/config.tar.gz" -C /opt/mindswarm/config

# Restore runtime components
echo "Restoring runtime components..."
tar -xzf "$BACKUP_PATH/runtime.tar.gz" -C /opt/mindswarm/runtime

# Restore application data
echo "Restoring application data..."
rm -rf /opt/mindswarm/data/*
tar -xzf "$BACKUP_PATH/app_data.tar.gz" -C /opt/mindswarm/data

# Start database
docker-compose up -d postgres redis
sleep 10

# Restore database
echo "Restoring database..."
docker exec mindswarm-postgres psql -U postgres -c "DROP DATABASE IF EXISTS mindswarm;"
docker exec mindswarm-postgres psql -U postgres -c "CREATE DATABASE mindswarm OWNER mindswarm;"
gunzip -c "$BACKUP_PATH/database.sql.gz" | docker exec -i mindswarm-postgres psql -U mindswarm mindswarm

# Restore Redis
echo "Restoring Redis..."
docker cp "$BACKUP_PATH/redis.rdb" mindswarm-redis:/data/dump.rdb
docker restart mindswarm-redis

# Start all services
docker-compose up -d

echo "Disaster recovery completed"
echo "Verifying system health..."
sleep 30
curl -f http://localhost:8000/health && echo "System is healthy" || echo "System health check failed"
```

## Scaling Strategies

### Horizontal Scaling

**Load Balancer Configuration**:
```yaml
# docker-compose.scale.yml
version: '3.8'

services:
  mindswarm-core:
    image: mindswarm/core:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
    environment:
      - MINDSWARM_INSTANCE_ID=${HOSTNAME}
    networks:
      - mindswarm-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./config/nginx-lb.conf:/etc/nginx/nginx.conf:ro
    deploy:
      replicas: 2
    networks:
      - mindswarm-network

networks:
  mindswarm-network:
    external: true
```

**Session Affinity with Redis**:
```python
# scaling/session_manager.py
import redis
import json
from typing import Dict, Optional

class DistributedSessionManager:
    def __init__(self, redis_url: str):
        self.redis = redis.from_url(redis_url)
        self.session_prefix = "mindswarm:session:"
        self.agent_prefix = "mindswarm:agent:"
    
    def create_session(self, session_id: str, session_data: Dict) -> None:
        """Create session in Redis."""
        key = f"{self.session_prefix}{session_id}"
        self.redis.setex(key, 3600, json.dumps(session_data))
    
    def get_session(self, session_id: str) -> Optional[Dict]:
        """Retrieve session from Redis."""
        key = f"{self.session_prefix}{session_id}"
        data = self.redis.get(key)
        return json.loads(data) if data else None
    
    def register_agent_location(self, agent_id: str, instance_id: str) -> None:
        """Register which instance an agent is running on."""
        key = f"{self.agent_prefix}{agent_id}"
        self.redis.setex(key, 7200, instance_id)
    
    def get_agent_location(self, agent_id: str) -> Optional[str]:
        """Get which instance an agent is running on."""
        key = f"{self.agent_prefix}{agent_id}"
        return self.redis.get(key)
```

### Vertical Scaling

**Resource Optimization**:
```yaml
# Resource limits for different environments
services:
  mindswarm-core:
    deploy:
      resources:
        limits:
          cpus: '4.0'
          memory: 8G
        reservations:
          cpus: '2.0'
          memory: 4G
    environment:
      - MINDSWARM_MAX_WORKERS=4
      - MINDSWARM_WORKER_CONNECTIONS=1000
      - MINDSWARM_MAX_AGENTS_PER_INSTANCE=100
```

### Database Scaling

**Read Replicas**:
```yaml
services:
  postgres-primary:
    image: postgres:15-alpine
    environment:
      - POSTGRES_REPLICATION_MODE=master
      - POSTGRES_REPLICATION_USER=replica
      - POSTGRES_REPLICATION_PASSWORD=replica_password
    
  postgres-replica:
    image: postgres:15-alpine
    environment:
      - POSTGRES_REPLICATION_MODE=slave
      - POSTGRES_REPLICATION_USER=replica
      - POSTGRES_REPLICATION_PASSWORD=replica_password
      - POSTGRES_MASTER_SERVICE=postgres-primary
    depends_on:
      - postgres-primary
```

## Performance Tuning

### Application Tuning

**FastAPI Configuration**:
```python
# Performance optimizations
app = FastAPI(
    title="MindSwarm Core",
    docs_url=None,  # Disable in production
    redoc_url=None,  # Disable in production
)

# Increase worker limits
uvicorn.run(
    app,
    host="0.0.0.0",
    port=8000,
    workers=4,
    worker_class="uvicorn.workers.UvicornWorker",
    worker_connections=1000,
    backlog=2048,
    keep_alive=5
)
```

**Memory Optimization**:
```python
# memory/optimizer.py
import gc
import asyncio
from typing import Dict

class MemoryOptimizer:
    def __init__(self, max_memory_mb: int = 1000):
        self.max_memory_mb = max_memory_mb
        self.optimization_interval = 300  # 5 minutes
    
    async def start_memory_monitoring(self):
        """Start background memory monitoring."""
        while True:
            await asyncio.sleep(self.optimization_interval)
            await self.optimize_memory()
    
    async def optimize_memory(self):
        """Perform memory optimization."""
        import psutil
        import os
        
        process = psutil.Process(os.getpid())
        memory_mb = process.memory_info().rss / 1024 / 1024
        
        if memory_mb > self.max_memory_mb:
            # Force garbage collection
            collected = gc.collect()
            
            # Log memory optimization
            new_memory_mb = process.memory_info().rss / 1024 / 1024
            
            logger.info(
                f"Memory optimization: {memory_mb:.1f}MB -> {new_memory_mb:.1f}MB "
                f"(collected {collected} objects)"
            )
```

This deployment and operations guide provides comprehensive coverage of production deployment, monitoring, security, and operational procedures for the MindSwarm system.
