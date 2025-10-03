# Templates de Monitoreo y Observabilidad

## Objetivo
Proporcionar templates estandarizados para implementar monitoreo completo, observabilidad y alertas en aplicaciones SAAS, incluyendo métricas, logs, trazas y dashboards.

## 1. Prometheus Configuration

### prometheus.yml
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'saas-platform'
    environment: 'production'

rule_files:
  - "alert_rules.yml"
  - "recording_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter
  - job_name: 'node-exporter'
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
            - monitoring
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: node-exporter
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        action: keep
        regex: metrics

  # Kubernetes API Server
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
            - default
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

  # Kubernetes Nodes
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)

  # Kubernetes Pods
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

  # Application Services
  - job_name: 'saas-api'
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
            - saas-platform
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: api-service
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        action: keep
        regex: metrics

  # Database Monitoring
  - job_name: 'postgres-exporter'
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
            - saas-platform
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: postgres-exporter

  # Redis Monitoring
  - job_name: 'redis-exporter'
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
            - saas-platform
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: redis-exporter

  # Nginx Ingress Controller
  - job_name: 'nginx-ingress'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - ingress-nginx
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
        action: keep
        regex: ingress-nginx
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: $1:10254
```

### alert_rules.yml
```yaml
groups:
  - name: saas-platform.rules
    rules:
      # High Error Rate
      - alert: HighErrorRate
        expr: |
          (
            rate(http_requests_total{status=~"5.."}[5m]) /
            rate(http_requests_total[5m])
          ) > 0.05
        for: 5m
        labels:
          severity: critical
          service: "{{ $labels.service }}"
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} for service {{ $labels.service }}"

      # High Response Time
      - alert: HighResponseTime
        expr: |
          histogram_quantile(0.95, 
            rate(http_request_duration_seconds_bucket[5m])
          ) > 2
        for: 5m
        labels:
          severity: warning
          service: "{{ $labels.service }}"
        annotations:
          summary: "High response time detected"
          description: "95th percentile response time is {{ $value }}s for service {{ $labels.service }}"

      # Database Connection Issues
      - alert: DatabaseConnectionHigh
        expr: |
          pg_stat_activity_count > 80
        for: 2m
        labels:
          severity: warning
          service: database
        annotations:
          summary: "High database connections"
          description: "Database has {{ $value }} active connections"

      # Memory Usage High
      - alert: HighMemoryUsage
        expr: |
          (
            node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
          ) / node_memory_MemTotal_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}"

      # CPU Usage High
      - alert: HighCPUUsage
        expr: |
          100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage"
          description: "CPU usage is {{ $value }}% on {{ $labels.instance }}"

      # Disk Space Low
      - alert: DiskSpaceLow
        expr: |
          (
            node_filesystem_avail_bytes{mountpoint="/"} /
            node_filesystem_size_bytes{mountpoint="/"}
          ) < 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Low disk space"
          description: "Disk space is {{ $value | humanizePercentage }} available on {{ $labels.instance }}"

      # Pod Restart High
      - alert: PodRestartHigh
        expr: |
          increase(kube_pod_container_status_restarts_total[1h]) > 5
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Pod restarting frequently"
          description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} has restarted {{ $value }} times in the last hour"

      # Service Down
      - alert: ServiceDown
        expr: |
          up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service is down"
          description: "Service {{ $labels.job }} on {{ $labels.instance }} is down"

  - name: business.rules
    rules:
      # Low User Registration Rate
      - alert: LowUserRegistrationRate
        expr: |
          rate(user_registrations_total[1h]) < 10
        for: 30m
        labels:
          severity: warning
          team: product
        annotations:
          summary: "Low user registration rate"
          description: "User registration rate is {{ $value }} per hour"

      # High User Churn Rate
      - alert: HighUserChurnRate
        expr: |
          rate(user_cancellations_total[24h]) / rate(user_registrations_total[24h]) > 0.1
        for: 1h
        labels:
          severity: critical
          team: product
        annotations:
          summary: "High user churn rate"
          description: "User churn rate is {{ $value | humanizePercentage }}"

      # Payment Failures High
      - alert: PaymentFailuresHigh
        expr: |
          rate(payment_failures_total[5m]) > 5
        for: 5m
        labels:
          severity: critical
          team: finance
        annotations:
          summary: "High payment failure rate"
          description: "Payment failure rate is {{ $value }} per minute"
```

### recording_rules.yml
```yaml
groups:
  - name: saas-platform.recording
    interval: 30s
    rules:
      # HTTP Request Rate
      - record: saas:http_requests:rate5m
        expr: |
          rate(http_requests_total[5m])

      # HTTP Error Rate
      - record: saas:http_errors:rate5m
        expr: |
          rate(http_requests_total{status=~"5.."}[5m])

      # HTTP Success Rate
      - record: saas:http_success_rate:rate5m
        expr: |
          rate(http_requests_total{status=~"2.."}[5m]) /
          rate(http_requests_total[5m])

      # Response Time Percentiles
      - record: saas:http_request_duration:p50
        expr: |
          histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))

      - record: saas:http_request_duration:p95
        expr: |
          histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

      - record: saas:http_request_duration:p99
        expr: |
          histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

      # Database Metrics
      - record: saas:db_connections:active
        expr: |
          pg_stat_activity_count

      - record: saas:db_query_duration:p95
        expr: |
          histogram_quantile(0.95, rate(pg_stat_statements_mean_time_bucket[5m]))

      # Business Metrics
      - record: saas:user_registrations:rate1h
        expr: |
          rate(user_registrations_total[1h])

      - record: saas:revenue:rate1h
        expr: |
          rate(revenue_total[1h])
```

## 2. Grafana Dashboards

### saas-platform-overview.json
```json
{
  "dashboard": {
    "id": null,
    "title": "SAAS Platform Overview",
    "tags": ["saas", "overview"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(saas:http_requests:rate5m)",
            "legendFormat": "Requests/sec"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "reqps",
            "color": {
              "mode": "thresholds"
            },
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 100},
                {"color": "red", "value": 1000}
              ]
            }
          }
        },
        "gridPos": {"h": 8, "w": 6, "x": 0, "y": 0}
      },
      {
        "id": 2,
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(saas:http_errors:rate5m) / sum(saas:http_requests:rate5m) * 100",
            "legendFormat": "Error Rate %"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "color": {
              "mode": "thresholds"
            },
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 1},
                {"color": "red", "value": 5}
              ]
            }
          }
        },
        "gridPos": {"h": 8, "w": 6, "x": 6, "y": 0}
      },
      {
        "id": 3,
        "title": "Response Time (P95)",
        "type": "stat",
        "targets": [
          {
            "expr": "saas:http_request_duration:p95",
            "legendFormat": "P95 Response Time"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s",
            "color": {
              "mode": "thresholds"
            },
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 1},
                {"color": "red", "value": 2}
              ]
            }
          }
        },
        "gridPos": {"h": 8, "w": 6, "x": 12, "y": 0}
      },
      {
        "id": 4,
        "title": "Active Users",
        "type": "stat",
        "targets": [
          {
            "expr": "active_users_total",
            "legendFormat": "Active Users"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "short",
            "color": {
              "mode": "thresholds"
            },
            "thresholds": {
              "steps": [
                {"color": "red", "value": null},
                {"color": "yellow", "value": 100},
                {"color": "green", "value": 1000}
              ]
            }
          }
        },
        "gridPos": {"h": 8, "w": 6, "x": 18, "y": 0}
      },
      {
        "id": 5,
        "title": "Request Rate Over Time",
        "type": "timeseries",
        "targets": [
          {
            "expr": "sum by (service) (saas:http_requests:rate5m)",
            "legendFormat": "{{service}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "reqps",
            "custom": {
              "drawStyle": "line",
              "lineInterpolation": "linear",
              "lineWidth": 2,
              "fillOpacity": 10
            }
          }
        },
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8}
      },
      {
        "id": 6,
        "title": "Response Time Percentiles",
        "type": "timeseries",
        "targets": [
          {
            "expr": "saas:http_request_duration:p50",
            "legendFormat": "P50"
          },
          {
            "expr": "saas:http_request_duration:p95",
            "legendFormat": "P95"
          },
          {
            "expr": "saas:http_request_duration:p99",
            "legendFormat": "P99"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s",
            "custom": {
              "drawStyle": "line",
              "lineInterpolation": "linear",
              "lineWidth": 2,
              "fillOpacity": 10
            }
          }
        },
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 8}
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "30s"
  }
}
```

## 3. ELK Stack Configuration

### elasticsearch.yml
```yaml
cluster.name: saas-platform-logs
node.name: ${HOSTNAME}
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300

discovery.seed_hosts: ["elasticsearch-master"]
cluster.initial_master_nodes: ["elasticsearch-master-0"]

# Memory settings
bootstrap.memory_lock: true
indices.memory.index_buffer_size: 30%

# Security
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true

# Monitoring
xpack.monitoring.collection.enabled: true
```

### logstash.conf
```ruby
input {
  beats {
    port => 5044
  }
  
  http {
    port => 8080
    codec => json
  }
}

filter {
  # Parse JSON logs
  if [fields][log_type] == "application" {
    json {
      source => "message"
    }
    
    # Parse timestamp
    date {
      match => [ "timestamp", "ISO8601" ]
    }
    
    # Add environment info
    mutate {
      add_field => { "environment" => "%{[fields][environment]}" }
      add_field => { "service" => "%{[fields][service]}" }
    }
  }
  
  # Parse Nginx access logs
  if [fields][log_type] == "nginx" {
    grok {
      match => { 
        "message" => "%{NGINXACCESS}"
      }
    }
    
    # Parse response time
    mutate {
      convert => { "response_time" => "float" }
      convert => { "response" => "integer" }
    }
    
    # GeoIP lookup
    geoip {
      source => "clientip"
      target => "geoip"
    }
  }
  
  # Parse PostgreSQL logs
  if [fields][log_type] == "postgresql" {
    grok {
      match => {
        "message" => "%{TIMESTAMP_ISO8601:timestamp} \[%{DATA:pid}\] %{WORD:level}: %{GREEDYDATA:message}"
      }
      overwrite => [ "message" ]
    }
  }
  
  # Add common fields
  mutate {
    add_field => { "[@metadata][index_prefix]" => "saas-platform" }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "%{[@metadata][index_prefix]}-%{[fields][log_type]}-%{+YYYY.MM.dd}"
    template_name => "saas-platform"
    template => "/usr/share/logstash/templates/saas-platform.json"
    template_overwrite => true
  }
  
  # Debug output
  stdout {
    codec => rubydebug
  }
}
```

### filebeat.yml
```yaml
filebeat.inputs:
  # Application logs
  - type: log
    enabled: true
    paths:
      - /var/log/saas-platform/app/*.log
    fields:
      log_type: application
      service: api
      environment: ${ENVIRONMENT:production}
    fields_under_root: false
    multiline.pattern: '^\d{4}-\d{2}-\d{2}'
    multiline.negate: true
    multiline.match: after

  # Nginx access logs
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
    fields:
      log_type: nginx
      service: nginx
      environment: ${ENVIRONMENT:production}
    fields_under_root: false

  # Nginx error logs
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/error.log
    fields:
      log_type: nginx_error
      service: nginx
      environment: ${ENVIRONMENT:production}
    fields_under_root: false

  # PostgreSQL logs
  - type: log
    enabled: true
    paths:
      - /var/log/postgresql/*.log
    fields:
      log_type: postgresql
      service: database
      environment: ${ENVIRONMENT:production}
    fields_under_root: false

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~

output.logstash:
  hosts: ["logstash:5044"]

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
```

## 4. Application Metrics (Python)

### metrics.py
```python
import time
import functools
from typing import Callable, Any
from prometheus_client import Counter, Histogram, Gauge, generate_latest
from flask import Flask, Response
import logging

# Metrics definitions
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status', 'service']
)

REQUEST_DURATION = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration in seconds',
    ['method', 'endpoint', 'service']
)

ACTIVE_CONNECTIONS = Gauge(
    'active_connections',
    'Number of active connections',
    ['service']
)

DATABASE_CONNECTIONS = Gauge(
    'database_connections_active',
    'Number of active database connections'
)

USER_REGISTRATIONS = Counter(
    'user_registrations_total',
    'Total user registrations'
)

REVENUE = Counter(
    'revenue_total',
    'Total revenue in cents'
)

ACTIVE_USERS = Gauge(
    'active_users_total',
    'Number of currently active users'
)

PAYMENT_FAILURES = Counter(
    'payment_failures_total',
    'Total payment failures',
    ['reason']
)

class MetricsCollector:
    def __init__(self, service_name: str):
        self.service_name = service_name
        self.logger = logging.getLogger(__name__)
    
    def track_request(self, func: Callable) -> Callable:
        """Decorator to track HTTP requests"""
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            start_time = time.time()
            
            try:
                result = func(*args, **kwargs)
                status = getattr(result, 'status_code', 200)
                
                # Record metrics
                REQUEST_COUNT.labels(
                    method='GET',  # You might want to get this from request
                    endpoint=func.__name__,
                    status=status,
                    service=self.service_name
                ).inc()
                
                REQUEST_DURATION.labels(
                    method='GET',
                    endpoint=func.__name__,
                    service=self.service_name
                ).observe(time.time() - start_time)
                
                return result
                
            except Exception as e:
                REQUEST_COUNT.labels(
                    method='GET',
                    endpoint=func.__name__,
                    status=500,
                    service=self.service_name
                ).inc()
                
                self.logger.error(f"Request failed: {str(e)}")
                raise
        
        return wrapper
    
    def track_user_registration(self):
        """Track user registration"""
        USER_REGISTRATIONS.inc()
        self.logger.info("User registration tracked")
    
    def track_revenue(self, amount_cents: int):
        """Track revenue"""
        REVENUE.inc(amount_cents)
        self.logger.info(f"Revenue tracked: {amount_cents} cents")
    
    def update_active_users(self, count: int):
        """Update active users count"""
        ACTIVE_USERS.set(count)
    
    def track_payment_failure(self, reason: str):
        """Track payment failure"""
        PAYMENT_FAILURES.labels(reason=reason).inc()
        self.logger.warning(f"Payment failure tracked: {reason}")
    
    def update_db_connections(self, count: int):
        """Update database connections count"""
        DATABASE_CONNECTIONS.set(count)

# Flask metrics endpoint
def create_metrics_app():
    app = Flask(__name__)
    
    @app.route('/metrics')
    def metrics():
        return Response(generate_latest(), mimetype='text/plain')
    
    @app.route('/health')
    def health():
        return {'status': 'healthy', 'timestamp': time.time()}
    
    return app

# Usage example
metrics = MetricsCollector('api-service')

@metrics.track_request
def get_users():
    # Your API logic here
    return {'users': []}

# Business metrics tracking
def register_user(user_data):
    # Registration logic
    metrics.track_user_registration()
    return user_data

def process_payment(amount_cents):
    try:
        # Payment processing logic
        metrics.track_revenue(amount_cents)
        return {'status': 'success'}
    except Exception as e:
        metrics.track_payment_failure(str(e))
        raise
```

## 5. Structured Logging

### logging_config.py
```python
import logging
import json
import sys
from datetime import datetime
from typing import Dict, Any

class JSONFormatter(logging.Formatter):
    """Custom JSON formatter for structured logging"""
    
    def format(self, record: logging.LogRecord) -> str:
        log_entry = {
            'timestamp': datetime.utcnow().isoformat() + 'Z',
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno,
        }
        
        # Add extra fields
        if hasattr(record, 'user_id'):
            log_entry['user_id'] = record.user_id
        
        if hasattr(record, 'request_id'):
            log_entry['request_id'] = record.request_id
        
        if hasattr(record, 'trace_id'):
            log_entry['trace_id'] = record.trace_id
        
        if hasattr(record, 'span_id'):
            log_entry['span_id'] = record.span_id
        
        # Add exception info
        if record.exc_info:
            log_entry['exception'] = self.formatException(record.exc_info)
        
        return json.dumps(log_entry)

def setup_logging(service_name: str, log_level: str = 'INFO'):
    """Setup structured logging configuration"""
    
    # Create logger
    logger = logging.getLogger()
    logger.setLevel(getattr(logging, log_level.upper()))
    
    # Remove default handlers
    for handler in logger.handlers[:]:
        logger.removeHandler(handler)
    
    # Create console handler with JSON formatter
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setFormatter(JSONFormatter())
    logger.addHandler(console_handler)
    
    # Add service name to all log records
    old_factory = logging.getLogRecordFactory()
    
    def record_factory(*args, **kwargs):
        record = old_factory(*args, **kwargs)
        record.service = service_name
        return record
    
    logging.setLogRecordFactory(record_factory)
    
    return logger

# Context manager for request logging
class RequestContext:
    def __init__(self, request_id: str, user_id: str = None):
        self.request_id = request_id
        self.user_id = user_id
        self.old_factory = None
    
    def __enter__(self):
        self.old_factory = logging.getLogRecordFactory()
        
        def record_factory(*args, **kwargs):
            record = self.old_factory(*args, **kwargs)
            record.request_id = self.request_id
            if self.user_id:
                record.user_id = self.user_id
            return record
        
        logging.setLogRecordFactory(record_factory)
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        logging.setLogRecordFactory(self.old_factory)

# Usage example
logger = setup_logging('api-service', 'INFO')

def process_request(request_id: str, user_id: str = None):
    with RequestContext(request_id, user_id):
        logger.info("Processing request")
        try:
            # Your business logic here
            logger.info("Request processed successfully")
        except Exception as e:
            logger.error("Request processing failed", exc_info=True)
            raise
```

## 6. Alertmanager Configuration

### alertmanager.yml
```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@saas-platform.com'
  smtp_auth_username: 'alerts@saas-platform.com'
  smtp_auth_password: 'your-app-password'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'critical-alerts'
      group_wait: 0s
      repeat_interval: 5m
    
    - match:
        team: product
      receiver: 'product-team'
    
    - match:
        team: finance
      receiver: 'finance-team'

receivers:
  - name: 'default'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#alerts'
        title: 'SAAS Platform Alert'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          *Service:* {{ .Labels.service }}
          {{ end }}

  - name: 'critical-alerts'
    email_configs:
      - to: 'oncall@saas-platform.com'
        subject: 'CRITICAL: {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          Severity: {{ .Labels.severity }}
          Service: {{ .Labels.service }}
          Time: {{ .StartsAt }}
          {{ end }}
    
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#critical-alerts'
        title: 'CRITICAL ALERT'
        text: |
          🚨 CRITICAL ALERT 🚨
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Service:* {{ .Labels.service }}
          {{ end }}
    
    pagerduty_configs:
      - routing_key: 'your-pagerduty-integration-key'
        description: '{{ .GroupLabels.alertname }}'

  - name: 'product-team'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#product-alerts'
        title: 'Product Alert'
        text: |
          📊 Product Alert
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          {{ end }}

  - name: 'finance-team'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#finance-alerts'
        title: 'Finance Alert'
        text: |
          💰 Finance Alert
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          {{ end }}

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
```

## 7. Jaeger Tracing Configuration

### jaeger-config.yml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jaeger-configuration
data:
  span-storage-type: elasticsearch
  es-server-urls: http://elasticsearch:9200
  es-username: elastic
  es-password: changeme
  collector.zipkin.http-port: "9411"
  collector.grpc-port: "14250"
  collector.http-port: "14268"
  query.port: "16686"
  query.base-path: /jaeger
```

### Python Tracing Integration
```python
from jaeger_client import Config
from opentracing.ext import tags
from opentracing.propagation import Format
import opentracing
import functools

def init_tracer(service_name: str):
    """Initialize Jaeger tracer"""
    config = Config(
        config={
            'sampler': {
                'type': 'const',
                'param': 1,
            },
            'logging': True,
            'reporter_batch_size': 1,
        },
        service_name=service_name,
        validate=True,
    )
    return config.initialize_tracer()

tracer = init_tracer('api-service')

def trace_function(operation_name: str = None):
    """Decorator to trace function calls"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            op_name = operation_name or f"{func.__module__}.{func.__name__}"
            
            with tracer.start_span(op_name) as span:
                span.set_tag(tags.COMPONENT, 'api-service')
                span.set_tag('function.name', func.__name__)
                span.set_tag('function.module', func.__module__)
                
                try:
                    result = func(*args, **kwargs)
                    span.set_tag(tags.HTTP_STATUS_CODE, 200)
                    return result
                except Exception as e:
                    span.set_tag(tags.ERROR, True)
                    span.set_tag('error.message', str(e))
                    span.set_tag(tags.HTTP_STATUS_CODE, 500)
                    raise
        
        return wrapper
    return decorator

# Usage example
@trace_function('user.get_profile')
def get_user_profile(user_id: str):
    # Your business logic here
    return {'user_id': user_id, 'name': 'John Doe'}
```

## Mejores Prácticas

### 1. Métricas
- Usar etiquetas consistentes
- Evitar alta cardinalidad
- Implementar SLIs/SLOs
- Monitorear métricas de negocio

### 2. Logs
- Usar logging estructurado
- Incluir contexto de request
- Implementar niveles apropiados
- Centralizar logs

### 3. Trazas
- Trazar requests críticos
- Incluir metadatos relevantes
- Usar sampling inteligente
- Correlacionar con logs

### 4. Alertas
- Definir umbrales apropiados
- Evitar alert fatigue
- Implementar escalación
- Documentar runbooks

Esta documentación proporciona una base completa para implementar monitoreo y observabilidad en aplicaciones SAAS.