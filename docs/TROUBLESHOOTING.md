# Troubleshooting Guide

This guide helps you diagnose and resolve common issues with MCP-SUPERSERVER.

## Table of Contents

- [Installation Issues](#installation-issues)
- [Docker Issues](#docker-issues)
- [Service Issues](#service-issues)
- [Performance Issues](#performance-issues)
- [Memory Issues](#memory-issues)
- [Network Issues](#network-issues)
- [MCP Connection Issues](#mcp-connection-issues)
- [Kubernetes Issues](#kubernetes-issues)
- [Getting Help](#getting-help)

---

## Installation Issues

### Problem: `make install` fails

**Symptoms:**
- Installation script exits with error
- Docker images fail to build
- Services fail to start

**Solutions:**

1. **Check Docker is running:**
```bash
docker ps
docker info
```

2. **Check available disk space:**
```bash
df -h
```
You need at least 50GB free.

3. **Check port availability:**
```bash
netstat -an | grep LISTEN | grep -E ':(3000|7474|7687|11434|9090)'
```

4. **Try with no cache:**
```bash
docker-compose build --no-cache
```

5. **Check system resources:**
```bash
# Memory
free -h

# CPU
nproc

# Docker resources
docker system df
```

### Problem: Permission denied errors

**Symptoms:**
- Cannot write to data directories
- Cannot create logs

**Solutions:**

1. **Check directory permissions:**
```bash
ls -la data/
```

2. **Fix permissions:**
```bash
sudo chown -R $USER:$USER data/
sudo chown -R $USER:$USER logs/
```

3. **Check Docker user:**
```bash
groups
# Ensure you're in the docker group
sudo usermod -aG docker $USER
newgrp docker
```

---

## Docker Issues

### Problem: Containers exit immediately

**Symptoms:**
- `docker ps` shows no running containers
- `docker-compose ps` shows "Exit 1" or similar

**Solutions:**

1. **Check container logs:**
```bash
docker-compose logs mcp-hub
docker-compose logs neo4j
docker-compose logs ollama
```

2. **Check for resource issues:**
```bash
docker stats
```

3. **Try starting individually:**
```bash
docker-compose up neo4j
# Wait for neo4j to be ready
docker-compose up mcp-hub
```

4. **Check health status:**
```bash
docker inspect --format='{{.State.Health.Status}}' mcp-neo4j
```

### Problem: Docker images fail to pull

**Symptoms:**
- "image not found" errors
- Network timeout during pull

**Solutions:**

1. **Check internet connection:**
```bash
ping google.com
```

2. **Try manual pull:**
```bash
docker pull neo4j:5.15-community
docker pull ollama/ollama:latest
```

3. **Check Docker registry:**
```bash
docker info | grep "Registry"
```

4. **Use mirror if in China:**
```bash
# Edit /etc/docker/daemon.json
{
  "registry-mirrors": ["https://mirror.ccs.tencentyun.com"]
}

sudo systemctl restart docker
```

### Problem: Out of disk space

**Symptoms:**
- "No space left on device" errors
- Docker cannot write to disk

**Solutions:**

1. **Check Docker disk usage:**
```bash
docker system df
```

2. **Clean up unused resources:**
```bash
docker system prune -a --volumes
```

3. **Remove specific volumes:**
```bash
docker volume rm mcp-superserver_neo4j-data
# WARNING: This deletes data!
```

4. **Configure Docker storage location:**
```bash
# Edit /etc/docker/daemon.json
{
  "data-root": "/mnt/large-disk/docker"
}

sudo systemctl restart docker
```

---

## Service Issues

### Problem: Neo4j won't start

**Symptoms:**
- Neo4j container exits immediately
- Cannot connect to Neo4j Browser

**Solutions:**

1. **Check Neo4j logs:**
```bash
docker-compose logs neo4j | tail -100
```

2. **Check if port is in use:**
```bash
lsof -i :7474
lsof -i :7687
```

3. **Verify password:**
```bash
docker exec -it mcp-neo4j cat /var/lib/neo4j/conf/neo4j.conf | grep auth
```

4. **Reset password if needed:**
```bash
docker exec -it mcp-neo4j neo4j-admin set-initial-password new_password
```

5. **Check memory settings:**
```bash
docker inspect mcp-neo4j | grep -i memory
```

### Problem: Ollama models won't download

**Symptoms:**
- Model pull hangs
- "model not found" errors

**Solutions:**

1. **Check Ollama logs:**
```bash
docker-compose logs ollama | tail -100
```

2. **Check available space:**
```bash
docker exec mcp-ollama df -h
```

3. **Test Ollama API:**
```bash
curl http://localhost:11434/api/tags
```

4. **Try manual pull:**
```bash
docker exec -it mcp-ollama ollama pull llama3.3
```

5. **Check model list:**
```bash
docker exec -it mcp-ollama ollama list
```

### Problem: MCP Hub can't connect to backend

**Symptoms:**
- Health check shows backend as "unhealthy"
- Tool calls fail with connection errors

**Solutions:**

1. **Check backend services are running:**
```bash
docker-compose ps
```

2. **Test connectivity from hub:**
```bash
docker exec -it mcp-hub ping neo4j
docker exec -it mcp-hub ping ollama
```

3. **Check environment variables:**
```bash
docker exec -it mcp-hub env | grep -E 'NEO4J|OLLAMA'
```

4. **Test direct connection:**
```bash
# From host
docker exec -it mcp-hub wget -O- http://neo4j:7474

# From hub container
docker exec -it mcp-hub curl http://neo4j:7474
```

5. **Check network:**
```bash
docker network inspect mcp-superserver_default
```

---

## Performance Issues

### Problem: Slow response times

**Symptoms:**
- API requests take >5 seconds
- Neo4j queries are slow
- Ollama responses are slow

**Solutions:**

1. **Check resource usage:**
```bash
docker stats
```

2. **Check Neo4j performance:**
```bash
# Enable query logging
docker exec -it mcp-neo4j cypher-shell -u neo4j -p password \
  "CALL dbms.queryJmx('org.neo4j:*') YIELD name, attributes WHERE name CONTAINS 'Query' RETURN name, attributes"
```

3. **Check Ollama model size:**
```bash
docker exec -it mcp-ollama ollama list
```

4. **Increase resources:**
```yaml
# In docker-compose.yml
services:
  neo4j:
    deploy:
      resources:
        limits:
          memory: 8G
```

5. **Enable caching:**
```bash
# Check cache hit rate
curl http://localhost:3000/metrics | grep cache
```

### Problem: High CPU usage

**Symptoms:**
- CPU consistently at 100%
- System becomes unresponsive

**Solutions:**

1. **Identify the culprit:**
```bash
docker stats --no-stream | sort -k2 -h
```

2. **Check for loops/busy waiting:**
```bash
docker exec -it mcp-hub ps aux
```

3. **Reduce Ollama threads:**
```bash
docker exec -it mcp-ollama OLLAMA_NUM_THREAD=4 ollama run llama3.3
```

4. **Limit CPU usage:**
```yaml
# In docker-compose.yml
services:
  ollama:
    deploy:
      resources:
        limits:
          cpus: '4.0'
```

### Problem: Memory leaks

**Symptoms:**
- Memory usage grows over time
- OOM (Out of Memory) kills

**Solutions:**

1. **Check memory growth:**
```bash
watch -n 5 'docker stats --no-stream | grep mcp-'
```

2. **Check for connection leaks:**
```bash
docker exec -it mcp-neo4j cypher-shell -u neo4j -p password \
  "CALL dbms.listConnections() YIELD connectionId, connectTime RETURN connectionId, connectTime ORDER BY connectTime DESC"
```

3. **Restart services:**
```bash
docker-compose restart mcp-hub
```

4. **Set memory limits:**
```yaml
# In docker-compose.yml
services:
  mcp-hub:
    deploy:
      resources:
        limits:
          memory: 512M
```

5. **Enable GC logging:**
```bash
# In docker-compose.yml
services:
  mcp-hub:
    environment:
      - NODE_OPTIONS=--max-old-space-size=256
```

---

## Memory Issues

### Problem: Neo4j runs out of memory

**Symptoms:**
- "Java heap space" errors
- Neo4j crashes

**Solutions:**

1. **Check Neo4j memory config:**
```bash
docker exec -it mcp-neo4j cat /var/lib/neo4j/conf/neo4j.conf | grep heap
```

2. **Increase heap size:**
```yaml
# In docker-compose.yml
services:
  neo4j:
    environment:
      - NEO4J_dbms_memory_heap_initial__size=1G
      - NEO4J_dbms_memory_heap_max__size=4G
```

3. **Check page cache:**
```bash
docker exec -it mcp-neo4j cypher-shell -u neo4j -p password \
  "CALL dbms.queryJmx('org.neo4j:instance=kernel#0,name=Page cache') YIELD attributes RETURN attributes"
```

4. **Clean up old data:**
```bash
docker exec -it mcp-neo4j cypher-shell -u neo4j -p password \
  "MATCH (n) WITH n, timestamp() AS ts WHERE ts - n.modified_at > 2592000000 DETACH DELETE n"
```

### Problem: Ollama runs out of memory

**Symptoms:**
- Model loading fails
- OOM errors

**Solutions:**

1. **Use smaller models:**
```bash
docker exec -it mcp-ollama ollama pull phi3
```

2. **Check GPU memory:**
```bash
nvidia-smi
```

3. **Reduce context size:**
```bash
docker exec -it mcp-ollama ollama run llama3.3 --num-ctx 2048
```

4. **Unload unused models:**
```bash
# Ollama automatically unloads, but you can restart
docker-compose restart ollama
```

---

## Network Issues

### Problem: Services can't communicate

**Symptoms:**
- Connection refused errors
- Timeout errors

**Solutions:**

1. **Check network:**
```bash
docker network inspect mcp-superserver_default
```

2. **Verify containers on same network:**
```bash
docker network inspect mcp-superserver_default | grep -A 10 Containers
```

3. **Test DNS:**
```bash
docker exec -it mcp-hub nslookup neo4j
```

4. **Check firewall:**
```bash
sudo ufw status
sudo iptables -L -n
```

5. **Recreate network:**
```bash
docker-compose down
docker network rm mcp-superserver_default
docker-compose up -d
```

### Problem: External access fails

**Symptoms:**
- Cannot access services from host
- Cannot access from other machines

**Solutions:**

1. **Check port bindings:**
```bash
docker-compose ps
```

2. **Verify ports are exposed:**
```bash
netstat -an | grep LISTEN | grep -E ':(3000|7474|7687|11434|9090)'
```

3. **Check firewall:**
```bash
sudo ufw allow 3000/tcp
sudo ufw allow 7474/tcp
sudo ufw allow 7687/tcp
sudo ufw allow 11434/tcp
```

4. **Check host binding:**
```yaml
# In docker-compose.yml
services:
  mcp-hub:
    ports:
      - "3000:3000"  # Binds to all interfaces
      # NOT "127.0.0.1:3000:3000"  # Only localhost
```

---

## MCP Connection Issues

### Problem: CLI can't connect to MCP Hub

**Symptoms:**
- "Connection refused" errors
- MCP tools not available

**Solutions:**

1. **Verify hub is running:**
```bash
curl http://localhost:3000/health
```

2. **Check CLI configuration:**
```bash
# For Claude Code
cat ~/.claude/config.json

# For Gemini CLI
cat ~/.gemini/extensions/mcp-hub/config.json
```

3. **Test MCP protocol:**
```bash
curl -X POST http://localhost:3000/tools/call \
  -H "Content-Type: application/json" \
  -d '{"server":"neo4j-memory","tool":"list_tools","arguments":{}}'
```

4. **Check WebSocket:**
```bash
wscat -c ws://localhost:3000/ws
```

5. **Verify API key (if enabled):**
```bash
curl -H "X-API-Key: your-key" http://localhost:3000/health
```

### Problem: MCP tools timeout

**Symptoms:**
- Tools take too long to respond
- "Request timeout" errors

**Solutions:**

1. **Check timeout settings:**
```bash
# In CLI config
{
  "timeout": 60000  // Increase to 60 seconds
}
```

2. **Check for blocking operations:**
```bash
docker exec -it mcp-hub ps aux
```

3. **Enable debug logging:**
```bash
# In .env
LOG_LEVEL=debug

docker-compose restart mcp-hub
docker-compose logs -f mcp-hub
```

4. **Check backend health:**
```bash
curl http://localhost:3000/health
```

---

## Kubernetes Issues

### Problem: Pods in CrashLoopBackOff

**Symptoms:**
- `kubectl get pods` shows CrashLoopBackOff
- Pod restarts repeatedly

**Solutions:**

1. **Check pod logs:**
```bash
kubectl logs -n mcp-superserver <pod-name>
kubectl logs -n mcp-superserver <pod-name> --previous
```

2. **Describe pod:**
```bash
kubectl describe pod -n mcp-superserver <pod-name>
```

3. **Check events:**
```bash
kubectl get events -n mcp-superserver --sort-by='.lastTimestamp'
```

4. **Common causes:**
   - Missing environment variables
   - Failed health checks
   - Missing secrets/configmaps
   - Insufficient resources

### Problem: Pods stuck in Pending state

**Symptoms:**
- Pods never start
- Status remains "Pending"

**Solutions:**

1. **Describe pod:**
```bash
kubectl describe pod -n mcp-superserver <pod-name>
```

2. **Check node resources:**
```bash
kubectl describe nodes
```

3. **Check PVCs:**
```bash
kubectl get pvc -n mcp-superserver
kubectl describe pvc neo4j-data -n mcp-superserver
```

4. **Check taints/tolerations:**
```bash
kubectl describe nodes | grep -A 5 Taints
```

### Problem: Service not reachable

**Symptoms:**
- Cannot connect to service
- DNS resolution fails

**Solutions:**

1. **Check service:**
```bash
kubectl get svc -n mcp-superserver
kubectl describe svc mcp-hub -n mcp-superserver
```

2. **Test from pod:**
```bash
kubectl run -it --rm debug --image=busybox --restart=Never -n mcp-superserver -- \
  wget -O- http://mcp-hub:3000/health
```

3. **Check endpoints:**
```bash
kubectl get endpoints -n mcp-superserver
```

4. **Verify selectors:**
```bash
kubectl get pods -n mcp-superserver --show-labels
```

---

## Backup and Restore Issues

### Problem: Backup fails

**Symptoms:**
- Backup script exits with error
- Incomplete backup files

**Solutions:**

1. **Check available space:**
```bash
df -h exports/
```

2. **Check permissions:**
```bash
ls -la exports/
```

3. **Verify services are running:**
```bash
docker-compose ps
```

4. **Check backup logs:**
```bash
./scripts/backup/create-backup.sh 2>&1 | tee backup.log
```

### Problem: Restore fails

**Symptoms:**
- Restore script exits with error
- Data not restored

**Solutions:**

1. **Verify backup file:**
```bash
tar -tzf exports/mcp-superserver-backup-*.tar.gz
```

2. **Stop services first:**
```bash
docker-compose down
```

3. **Check restore logs:**
```bash
./scripts/backup/restore-backup.sh 2>&1 | tee restore.log
```

4. **Verify data integrity:**
```bash
docker-compose up -d
curl http://localhost:3000/health
```

---

## Getting Help

### Debug Mode

Enable comprehensive logging:

```bash
# In .env
LOG_LEVEL=debug
DEBUG=true

# Restart services
docker-compose restart

# Follow logs
docker-compose logs -f
```

### Collect Diagnostics

```bash
# Create diagnostics bundle
./scripts/utils/collect-diagnostics.sh > diagnostics.txt
```

### Health Check

```bash
# Full health check
curl http://localhost:3000/health | jq '.'

# Check each service
curl http://localhost:7474  # Neo4j Browser
curl http://localhost:11434/api/tags  # Ollama
curl http://localhost:9090  # Prometheus
```

### Logs

```bash
# All logs
docker-compose logs

# Specific service
docker-compose logs mcp-hub

# Follow logs
docker-compose logs -f mcp-hub

# Last 100 lines
docker-compose logs --tail=100 mcp-hub
```

### Common Exit Codes

| Code | Meaning | Solution |
|------|---------|----------|
| 0 | Success | N/A |
| 1 | General error | Check logs |
| 125 | Docker daemon error | Restart Docker |
| 126 | Container command not found | Check image |
| 127 | File not found | Check paths |
| 137 | Killed (OOM) | Increase memory |
| 255 | Exit status out of range | Check configuration |

### Support Resources

- **GitHub Issues**: https://github.com/rubenalvarezdev/mcp-superserver/issues
- **Documentation**: https://github.com/rubenalvarezdev/mcp-superserver/wiki
- **Discord**: (Add link if available)

### Before Asking for Help

1. Check this troubleshooting guide
2. Search existing GitHub issues
3. Enable debug logging
4. Collect relevant logs
5. Include system information:
   ```bash
   uname -a
   docker --version
   docker-compose --version
   ```
6. Describe:
   - What you were trying to do
   - What happened
   - Expected vs actual behavior
   - Steps to reproduce
