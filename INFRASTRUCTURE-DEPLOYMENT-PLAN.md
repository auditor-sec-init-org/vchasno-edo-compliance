# Infrastructure Deployment & Setup Plan
## VCHASNO EDO System - Week 5 Execution

### Executive Summary
Comprehensive infrastructure deployment roadmap covering monitoring stack, logging infrastructure, caching layer, and database optimization for production deployment.

---

## 1. MONITORING INFRASTRUCTURE SETUP

### 1.1 Prometheus Deployment
**Installation (Ubuntu 22.04):**
```bash
# Create prometheus user
sudo useradd --no-create-home --shell /bin/false prometheus

# Download and extract
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.50.0/prometheus-2.50.0.linux-amd64.tar.gz
tar -xvf prometheus-2.50.0.linux-amd64.tar.gz

# Install binaries
sudo cp prometheus-2.50.0.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.50.0.linux-amd64/promtool /usr/local/bin/

# Create config directory
sudo mkdir -p /etc/prometheus/rules
sudo mkdir -p /var/lib/prometheus
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

**Prometheus Config:**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'api-servers'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['api1:9100', 'api2:9100']
  
  - job_name: 'database'
    static_configs:
      - targets: ['db-primary:9187']
  
  - job_name: 'redis'
    static_configs:
      - targets: ['cache-primary:9121']
```

### 1.2 Grafana Deployment
```bash
docker run -d --name=grafana --restart=always -p 3000:3000 \
  -e GF_SECURITY_ADMIN_PASSWORD=SecurePassword123! \
  -v grafana-storage:/var/lib/grafana grafana/grafana:latest
```

---

## 2. LOGGING INFRASTRUCTURE (ELK)

### 2.1 Elasticsearch + Logstash + Kibana
```yaml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
    ports: ["9200:9200"]
    volumes: ["es-data:/usr/share/elasticsearch/data"]

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports: ["5601:5601"]
    depends_on: [elasticsearch]
```

### 2.2 Log Shipping
- Filebeat: Collect logs from /var/log/vchasno/*
- Filter and parse JSON logs
- Send to Elasticsearch with index pattern: vchasno-YYYY.MM.DD

---

## 3. CACHING LAYER (REDIS)

### 3.1 Redis Setup
```bash
# Master
docker run -d --name redis-master -p 6379:6379 \
  -v redis-master:/data redis:7.2-alpine \
  redis-server --appendonly yes --requirepass SecurePassword123!

# Replica
docker run -d --name redis-replica-1 -p 6380:6379 \
  -v redis-replica-1:/data redis:7.2-alpine \
  redis-server --replicaof redis-master 6379 --requirepass SecurePassword123!
```

### 3.2 Cache Policies
- Documents list: 5 min TTL
- Document detail: 30 min TTL
- Compliance results: 10 min TTL
- User sessions: 12 hours TTL

---

## 4. DATABASE OPTIMIZATION

### 4.1 PgBouncer Connection Pooling
```ini
[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
server_lifetime = 3600
server_idle_timeout = 600
```

### 4.2 PostgreSQL Tuning
```ini
shared_buffers = 8GB
effective_cache_size = 16GB
maintenance_work_mem = 2GB
work_mem = 32MB
wal_buffers = 32MB
```

### 4.3 Critical Indexes
```sql
CREATE INDEX idx_documents_status_date ON documents(status, created_at DESC);
CREATE INDEX idx_documents_owner_status ON documents(owner_id, status);
CREATE INDEX idx_documents_search ON documents USING GIN(to_tsvector('english', title));
```

---

## DEPLOYMENT CHECKLIST - WEEK 5

### Phase 1: Foundation Infrastructure
- [ ] Deploy Prometheus + Grafana monitoring
- [ ] Set up ELK Stack (Elasticsearch + Kibana)
- [ ] Deploy Redis Master + 2 Replicas
- [ ] Install PgBouncer connection pooling
- [ ] Optimize PostgreSQL configuration
- [ ] Create all required indexes
- [ ] Test failover procedures
- [ ] Set up Alert Manager

### Phase 2: Application Integration
- [ ] Connect app to Prometheus metrics
- [ ] Enable Filebeat log shipping
- [ ] Implement Redis caching in API
- [ ] Enable PgBouncer in connection strings
- [ ] Configure alerting rules
- [ ] Test monitoring dashboards

### Phase 3: Validation & Testing
- [ ] Run baseline performance benchmarks
- [ ] Validate cache hit ratios (>85%)
- [ ] Verify database query performance (<50ms p50)
- [ ] Test alert triggering
- [ ] Load test with monitoring
- [ ] Document runbooks

---

## SUCCESS METRICS
- API Response: <100ms (p50), <250ms (p99)
- DB Queries: <50ms (p50), <200ms (p99)
- Cache Hit Ratio: >85%
- Monitoring: 100% metric collection
- Logging: 100% log aggregation
