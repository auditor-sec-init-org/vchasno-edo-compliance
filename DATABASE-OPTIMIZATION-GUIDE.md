# Database Optimization & Connection Pooling

## PgBouncer Setup

```ini
[databases]
vchasno = host=db-primary port=5432 user=vchasno_app password=secure_password

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
min_pool_size = 10
reserve_pool_size = 5
server_lifetime = 3600
server_idle_timeout = 600
connect_timeout = 15
```

## PostgreSQL Configuration

```ini
shared_buffers = 8GB
effective_cache_size = 16GB
maintenance_work_mem = 2GB
work_mem = 32MB
wal_buffers = 32MB
```

## Critical Indexes

```sql
CREATE INDEX idx_documents_status_date ON documents(status, created_at DESC);
CREATE INDEX idx_documents_owner_status ON documents(owner_id, status);
CREATE INDEX idx_documents_search ON documents USING GIN(to_tsvector('english', title));
ANALYZE documents;
```

## Deployment Steps

- [ ] Install PgBouncer (apt-get install pgbouncer)
- [ ] Configure pgbouncer.ini
- [ ] Create system user and directories
- [ ] Start pgbouncer service
- [ ] Update application connection strings
- [ ] Optimize PostgreSQL parameters
- [ ] Create all required indexes
- [ ] Run ANALYZE
- [ ] Monitor connection pool
- [ ] Load test and verify
