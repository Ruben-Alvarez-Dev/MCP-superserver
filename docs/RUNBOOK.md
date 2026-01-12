# MCP-SUPERSERVER Runbook

This runbook provides operational procedures for managing MCP-SUPERSERVER in production.

## Table of Contents

- [Incident Response](#incident-response)
- [Daily Operations](#daily-operations)
- [Maintenance Procedures](#maintenance-procedures)
- [Scaling Operations](#scaling-operations)
- [Backup and Recovery](#backup-and-recovery)
- [Monitoring and Alerting](#monitoring-and-alerting)

---

## Incident Response

### Severity Levels

| Severity | Description | Response Time |
|----------|-------------|---------------|
| P0 - Critical | Service completely down | < 15 minutes |
| P1 - High | Major functionality degraded | < 1 hour |
| P2 - Medium | Partial functionality affected | < 4 hours |
| P3 - Low | Minor issues or edge cases | < 24 hours |

### Incident Flow

```
1. Detect (Alert) → 2. Acknowledge → 3. Investigate → 4. Mitigate → 5. Resolve → 6. Post-Mortem
```

### P0 - Critical Incidents

**Definition:**
- MCP Hub completely down
- All CLI tools unable to connect
- Data loss or corruption
- Security breach

**Immediate Actions:**

1. **Acknowledge the alert** (SLA starts now)
2. **Check status page**
   ```bash
   curl http://localhost:3000/health
   ```

3. **Identify scope**
   ```bash
   # Check all services
   docker-compose ps

   # Check logs
   docker-compose logs --tail=100 mcp-hub
   docker-compose logs --tail=100 neo4j
   docker-compose logs --tail=100 ollama
   ```

4. **Communicate**
   - Update status page (if public)
   - Notify stakeholders
   - Create incident channel

5. **Mitigate** (see specific scenarios below)

6. **Verify fix**
   ```bash
   # Health check
   curl http://localhost:3000/health

   # Test tool execution
   curl -X POST http://localhost:3000/tools/call \
     -H "Content-Type: application/json" \
     -d '{"server":"neo4j-memory","tool":"count_entities","arguments":{"label":"Task"}}'
   ```

7. **Document** (see Post-Incident Review)

### Common Incident Scenarios

#### Scenario 1: MCP Hub Down

**Symptoms:**
- `curl http://localhost:3000/health` fails
- All CLI connections fail
- Alert: `MCPHubDown` firing

**Diagnosis:**
```bash
# Check container status
docker ps | grep mcp-hub

# Check recent logs
docker-compose logs --tail=200 mcp-hub

# Check resource usage
docker stats mcp-hub --no-stream
```

**Mitigation:**
```bash
# Quick restart
docker-compose restart mcp-hub

# If restart fails, check for issues
docker-compose down mcp-hub
docker-compose up -d mcp-hub

# If OOM killer, increase memory limits
# Edit docker-compose.yml:
services:
  mcp-hub:
    deploy:
      resources:
        limits:
          memory: 1G  # Increase from 512M

docker-compose up -d mcp-hub
```

**Verification:**
```bash
# Wait for health check
watch -n 5 'curl -s http://localhost:3000/health | jq "."'
```

#### Scenario 2: Neo4j Down

**Symptoms:**
- All Neo4j tool calls fail
- Alert: `Neo4jDown` firing
- "Connection refused" to bolt://localhost:7687

**Diagnosis:**
```bash
# Check Neo4j
docker ps | grep neo4j
docker-compose logs --tail=100 neo4j

# Test connection
docker exec -it mcp-neo4j cypher-shell -u neo4j -p password "RETURN 1"
```

**Mitigation:**
```bash
# Restart Neo4j
docker-compose restart neo4j

# Wait for startup (can take 30-60 seconds)
docker-compose logs -f neo4j

# If password issues, reset
docker exec -it mcp-neo4j neo4j-admin set-initial-password newpassword

# If corrupt data, check integrity
docker exec -it mcp-neo4j neo4j-admin check-consistency
```

**Note:** Neo4j can take 30-90 seconds to start. Be patient before escalating.

#### Scenario 3: High Error Rate

**Symptoms:**
- Alert: `HighErrorRate` or `CriticalErrorRate` firing
- >5% of requests returning 5xx errors
- User complaints about failed operations

**Diagnosis:**
```bash
# Check error rate
curl http://localhost:3000/metrics | grep 'mcp_requests_total{status="5"'

# Check recent errors in logs
docker-compose logs --since=5m mcp-hub | grep -i error

# Check backend health
curl http://localhost:3000/health | jq '.services'
```

**Mitigation:**
```bash
# If due to backend issues
docker-compose restart neo4j  # or ollama

# If due to resource exhaustion
docker-compose up -d --scale mcp-hub=3  # Add more replicas

# If due to bad deployment
docker-compose rollback mcp-hub  # If using rollback feature
```

#### Scenario 4: High Memory Usage

**Symptoms:**
- Alert: `HighMemoryUsage` or `CriticalMemoryUsage` firing
- Container OOM kills
- Slow response times

**Diagnosis:**
```bash
# Check memory usage
docker stats mcp-hub --no-stream

# Check for memory leaks
watch -n 5 'docker stats mcp-hub --no-stream | grep mcp-hub'

# Check Neo4j memory
docker exec -it mcp-neo4j cypher-shell -u neo4j -p password \
  "CALL dbms.queryJmx('org.neo4j:instance=kernel#0,name=Memory Pools') YIELD attributes RETURN attributes"
```

**Mitigation:**
```bash
# Quick fix: restart service
docker-compose restart mcp-hub

# Long term: increase limits
# Edit docker-compose.yml
services:
  mcp-hub:
    deploy:
      resources:
        limits:
          memory: 1G

# For Neo4j, tune heap size
# Edit docker-compose.yml
services:
  neo4j:
    environment:
      - NEO4J_dbms_memory_heap_max__size=4G
```

---

## Daily Operations

### Morning Checklist

Run daily (automated via cron recommended):

```bash
#!/bin/bash
# daily-check.sh

echo "=== MCP-SUPERSERVER Daily Health Check ==="
echo "Date: $(date)"
echo ""

# 1. Check all services are running
echo "1. Service Status:"
docker-compose ps
echo ""

# 2. Check health endpoints
echo "2. Health Check:"
curl -s http://localhost:3000/health | jq '.'
echo ""

# 3. Check disk space
echo "3. Disk Space:"
df -h | grep -E '(Filesystem|/$|/data)'
echo ""

# 4. Check memory usage
echo "4. Memory Usage:"
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}"
echo ""

# 5. Check for errors in logs
echo "5. Recent Errors:"
docker-compose logs --since=1h | grep -i error | tail -10
echo ""

# 6. Check backup status
echo "6. Latest Backup:"
ls -lth exports/ | head -5
echo ""

echo "=== Daily Check Complete ==="
```

### Weekly Tasks

1. **Review metrics and trends**
   - Check Prometheus dashboards
   - Review response time trends
   - Identify any slow queries

2. **Review and clean up logs**
   ```bash
   # Rotate logs if needed
   docker-compose logs --tail=0 > /dev/null
   ```

3. **Security scan**
   ```bash
   # Check for vulnerabilities
   docker scan mcp-superserver/mcp-hub:latest
   ```

4. **Backup verification**
   ```bash
   # Verify latest backup is valid
   ./scripts/backup/verify-backup.sh
   ```

### Monthly Tasks

1. **Review capacity planning**
   - Check resource utilization trends
   - Plan for scaling if needed
   - Review storage growth

2. **Update dependencies**
   ```bash
   # Pull latest images
   docker-compose pull

   # Test in staging first!
   ```

3. **Security audit**
   - Review access logs
   - Audit user permissions
   - Check for exposed secrets

4. **Performance review**
   - Analyze slow query logs
   - Review Neo4j query performance
   - Optimize if needed

---

## Maintenance Procedures

### Service Restart

**Planned Restart:**

```bash
# 1. Announce maintenance (minimum 15 min notice)

# 2. Stop accepting new requests
# (Configure load balancer or update DNS)

# 3. Graceful shutdown
docker-compose stop mcp-hub

# 4. Wait for in-flight requests (30 seconds)
sleep 30

# 5. Restart service
docker-compose start mcp-hub

# 6. Verify health
curl http://localhost:3000/health

# 7. Restore traffic
```

**Rolling Restart (Zero Downtime):**

```bash
# Requires multiple replicas
docker-compose up -d --scale mcp-hub=3

# Restart one at a time
for i in 1 2 3; do
  docker stop mcp-superserver_mcp-hub_$i
  sleep 10  # Wait for container to stop
  docker start mcp-superserver_mcp-hub_$i
  sleep 30  # Wait for health check
  curl http://localhost:3000/health
done
```

### Database Maintenance

**Neo4j:**

```bash
# 1. Backup first
make backup

# 2. Connect to Neo4j
docker exec -it mcp-neo4j cypher-shell -u neo4j -p password

# 3. Check database size
CALL dbms.queryJmx('org.neo4j:instance=kernel#0,name=Store sizes') YIELD attributes RETURN attributes;

# 4. Clean up old data (example: delete logs older than 90 days)
MATCH (l:LogEntry) WHERE l.timestamp < timestamp() - (90 * 24 * 3600 * 1000) DETACH DELETE l;

# 5. Rebuild indexes (if needed)
CALL db.indexes() YIELD label, properties, state
WHERE state = 'POPULATING'
RETURN *

# 6. Update statistics
CALL db.stats.retrieve('GRAPH COUNTS')

# 7. Exit
:exit
```

### Log Cleanup

```bash
# Rotate and compress old logs
#!/bin/bash
# cleanup-logs.sh

LOG_DIR="./logs"
RETENTION_DAYS=30

# Find and compress logs older than 7 days
find $LOG_DIR -name "*.log" -mtime +7 -exec gzip {} \;

# Remove compressed logs older than retention
find $LOG_DIR -name "*.gz" -mtime +$RETENTION_DAYS -delete

echo "Log cleanup complete"
```

---

## Scaling Operations

### Vertical Scaling (Increase Resources)

**MCP Hub:**
```yaml
# Edit docker-compose.yml
services:
  mcp-hub:
    deploy:
      resources:
        limits:
          cpus: '2.0'     # Increase from 1.0
          memory: 1G      # Increase from 512M
        reservations:
          cpus: '0.5'
          memory: 256M
```

**Neo4j:**
```yaml
services:
  neo4j:
    environment:
      - NEO4J_dbms_memory_heap_initial__size=1G  # Increase
      - NEO4J_dbms_memory_heap_max__size=4G      # Increase
    deploy:
      resources:
        limits:
          memory: 8G      # Increase
```

**Ollama:**
```yaml
services:
  ollama:
    deploy:
      resources:
        limits:
          memory: 32G     # Increase for larger models
          # nvidia.com/gpu: 1  # Add GPU support
```

### Horizontal Scaling (Add Replicas)

```bash
# Scale MCP Hub (stateless)
docker-compose up -d --scale mcp-hub=5

# Verify
docker-compose ps
curl http://localhost:3000/metrics | grep 'up{job="mcp-hub"}'
```

**Note:** Neo4j and Ollama typically run single instances. For HA:
- Neo4j: Use Neo4j Enterprise clustering or causal cluster
- Ollama: Run multiple instances with load balancing

### Load Balancer Setup

```nginx
# nginx.conf
upstream mcp_hub {
    least_conn;
    server mcp-hub-1:3000;
    server mcp-hub-2:3000;
    server mcp-hub-3:3000;
}

server {
    listen 80;
    server_name mcp.example.com;

    location / {
        proxy_pass http://mcp_hub;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

## Backup and Recovery

### Automated Backup

**Verify cron job is running:**
```bash
crontab -l | grep mcp-backup
```

**Expected:**
```
0 2 * * * /path/to/mcp-superserver/scripts/backup/create-backup.sh
```

### Manual Backup

```bash
# Full backup
make backup

# Or directly
./scripts/backup/create-backup.sh

# Verify backup was created
ls -lth exports/ | head -5
```

### Restore Procedure

**Scenario: Complete System Recovery**

```bash
# 1. Stop all services
docker-compose down

# 2. Restore data
./scripts/backup/restore-backup.sh

# 3. Start services
docker-compose up -d

# 4. Verify
curl http://localhost:3000/health

# 5. Test data integrity
curl -X POST http://localhost:3000/tools/call \
  -H "Content-Type: application/json" \
  -d '{"server":"neo4j-memory","tool":"count_entities","arguments":{"label":"Task"}}'
```

**Scenario: Single Service Recovery**

```bash
# Example: Restore only Neo4j
docker-compose stop neo4j

# Copy backup to temporary location
cp exports/latest-backup.tar.gz /tmp/neo4j-backup.tar.gz
cd /tmp
tar -xzf neo4j-backup.tar.gz

# Stop Neo4j and restore data
docker-compose down neo4j
rm -rf data/neo4j/data/*
cp -r neo4j-data/* data/neo4j/data/

# Start Neo4j
docker-compose up -d neo4j

# Wait for startup (30-90 seconds)
sleep 60

# Verify
docker exec -it mcp-neo4j cypher-shell -u neo4j -p password "MATCH (n) RETURN count(n) as nodeCount"
```

---

## Monitoring and Alerting

### Dashboard Metrics

**Key Metrics to Monitor:**

1. **Request Rate**
   - `rate(mcp_requests_total[5m])`
   - Alert if: drops to 0 (service down) or spikes unexpectedly (DDOS?)

2. **Response Time**
   - `histogram_quantile(0.95, mcp_request_duration_seconds_bucket)`
   - Alert if: > 1s (warning), > 5s (critical)

3. **Error Rate**
   - `rate(mcp_requests_total{status=~"5.."}[5m]) / rate(mcp_requests_total[5m])`
   - Alert if: > 5% (warning), > 15% (critical)

4. **Resource Usage**
   - `container_memory_usage_bytes / container_spec_memory_limit_bytes`
   - `rate(container_cpu_usage_seconds_total[5m])`
   - Alert if: > 90% (warning), > 95% (critical)

5. **Neo4j Specific**
   - `neo4j_cypher_execution_time_seconds`
   - `neo4j_connections_active`
   - Alert if: query time > 5s, connections > 100

### Alert Response

**When you receive an alert:**

1. **Acknowledge** in AlertManager/Prometheus
2. **Investigate** using diagnostics:
   ```bash
   # Quick diagnostics
   ./scripts/utils/collect-diagnostics.sh
   ```
3. **Mitigate** following incident procedures above
4. **Resolve** the root cause
5. **Document** in incident log

### Metrics Queries

**Common Prometheus queries:**

```promql
# Requests per second by server
sum(rate(mcp_requests_total[5m])) by (server)

# 95th percentile response time
histogram_quantile(0.95, sum(rate(mcp_request_duration_seconds_bucket[5m])) by (le, server))

# Error rate by server
sum(rate(mcp_requests_total{status=~"5.."}[5m])) by (server) /
sum(rate(mcp_requests_total[5m])) by (server)

# Memory usage by container
container_memory_usage_bytes / container_spec_memory_limit_bytes

# CPU usage by container
sum(rate(container_cpu_usage_seconds_total{container!="POD"}[5m])) by (container) * 100

# Disk space
(node_filesystem_avail_bytes{mountpoint="/data"} / node_filesystem_size_bytes{mountpoint="/data"}) * 100
```

---

## Post-Incident Review

### Template

```markdown
# Incident Report: [Short Description]

**Date:** YYYY-MM-DD
**Severity:** P0/P1/P2/P3
**Duration:** X hours
**Responder:** [Name]

## Summary
[2-3 sentence summary of what happened and impact]

## Timeline
- HH:MM - Incident detected
- HH:MM - Investigation started
- HH:MM - Root cause identified
- HH:MM - Mitigation implemented
- HH:MM - Service restored
- HH:MM - Incident closed

## Root Cause
[What actually went wrong?]

## Impact
[Who was affected? What was the user impact?]

## Resolution
[What did we do to fix it?]

## Prevention
[What will we do to prevent this from happening again?]
- [ ] Action item 1
- [ ] Action item 2

## Lessons Learned
[What did we learn? What went well? What could be improved?]
```

---

## Emergency Contacts

| Role | Name | Contact | Hours |
|------|------|---------|-------|
| On-Call Engineer | | | 24/7 |
| Engineering Lead | | | Business hours |
| DevOps Engineer | | | Business hours |
| Security Team | | | 24/7 for security issues |

---

## Runbook Maintenance

**Update this runbook when:**
- New services are added
- Procedures change
- New incident patterns emerge
- Quarterly review is due

**Version:** 1.0.0
**Last Updated:** 2024-01-15
**Next Review:** 2024-04-15
