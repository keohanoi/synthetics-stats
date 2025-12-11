# Self-Hosted Graph Node Setup for Mantle Sepolia

This guide explains how to set up and run a self-hosted Graph Node infrastructure for indexing the GMX Synthetics subgraph on Mantle Sepolia.

## Architecture

```
[Mantle Sepolia RPC] → [Graph Node] ↔ [PostgreSQL 14]
                              ↓
                           [IPFS]
```

**Services:**
- **Graph Node v0.35.1** - Core indexing engine
- **PostgreSQL 14** - Database for indexed data
- **IPFS kubo v0.24.0** - Subgraph deployment storage

## Prerequisites

- Docker Desktop (or Docker Engine + Docker Compose)
- Node.js 16+ and Yarn
- 8GB RAM minimum (16GB recommended)
- 50GB free disk space

## Quick Start

### 1. First-Time Setup

```bash
# One command to do everything: start Docker + create + deploy
yarn setup
```

That's it! This will:
- Start all Docker services (PostgreSQL, IPFS, Graph Node)
- Wait for services to be healthy
- Create the subgraph
- Build and deploy it

### 2. Deploy Updates (After Code Changes)

```bash
yarn deploy
```

### 3. Access Endpoints

- **GraphQL Playground**: http://localhost:8000/subgraphs/name/gmx/synthetics-mantle-sepolia/graphql
- **Indexing Status**: http://localhost:8030/graphql
- **IPFS Gateway**: http://localhost:8080/ipfs/\<hash\>
- **Prometheus Metrics**: http://localhost:8040/metrics

## Available Commands

### Daily Use

```bash
yarn start    # Start Docker services
yarn stop     # Stop Docker services
yarn logs     # View Graph Node logs
yarn deploy   # Build and deploy subgraph (after code changes)
yarn status   # Check indexing progress
```

### Setup & Maintenance

```bash
yarn setup    # First-time setup (start + create + deploy)
yarn clean    # Delete all data (destructive - use with caution)
yarn codegen  # Generate TypeScript types from schema
yarn build    # Build subgraph without deploying
```

## Operations Guide

### Starting Services

```bash
yarn start
```

### Stopping Services

```bash
yarn stop
```

This stops all services but preserves data in Docker volumes.

### Viewing Logs

```bash
yarn logs
```

Press Ctrl+C to exit. To view other services:

```bash
docker-compose logs -f postgres
docker-compose logs -f ipfs
```

### Deploying Updates

After making changes to your subgraph (schema, mappings, etc.):

```bash
yarn deploy
```

This automatically runs codegen, build, and deploy.

### Complete Reset (Destructive)

```bash
# WARNING: Deletes all indexed data
yarn clean
yarn setup
```

## Monitoring

### Check Indexing Progress

```bash
yarn status
```

This shows:
- Subgraph health status
- Current indexed block
- Latest chain head block
- Sync progress

### Monitor Sync Progress

```bash
# Watch logs in real-time
yarn logs

# Look for "Scanning blocks" messages to see progress
```

### PostgreSQL Monitoring

```bash
# Connect to PostgreSQL
docker-compose exec postgres psql -U graph -d graph-node

# View subgraph deployments
SELECT * FROM subgraphs.subgraph_deployment;

# Exit psql
\q
```

### IPFS Stats

```bash
curl http://localhost:5001/api/v0/stats/repo
```

## Querying the Subgraph

### Using GraphQL Playground

Open http://localhost:8000/subgraphs/name/gmx/synthetics-mantle-sepolia/graphql in your browser.

Example query:

```graphql
{
  _meta {
    block {
      number
      timestamp
    }
  }
  orders(first: 5, orderDirection: desc) {
    id
    account
    orderType
  }
}
```

### Using curl

```bash
curl http://localhost:8000/subgraphs/name/gmx/synthetics-mantle-sepolia/graphql \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"query": "{ _meta { block { number } } }"}'
```

## Troubleshooting

### Services Won't Start

**Check Docker is running:**
```bash
docker ps
```

**Check service status:**
```bash
docker-compose ps
```

**View error logs:**
```bash
yarn logs
```

### Graph Node Not Syncing

**1. Check RPC connectivity:**
```bash
curl -X POST https://rpc.sepolia.mantle.xyz \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

**2. Check Graph Node logs for errors:**
```bash
yarn logs | grep -i error
```

**3. Verify RPC is configured correctly:**
```bash
cat .env | grep MANTLE_SEPOLIA_RPC
```

### PostgreSQL Connection Issues

**Check PostgreSQL is running:**
```bash
docker-compose ps postgres
```

**Restart PostgreSQL:**
```bash
docker-compose restart postgres
```

**Check credentials match:**
```bash
# Verify .env matches docker-compose.yml
cat .env
```

### IPFS Connection Issues

**Check IPFS is running:**
```bash
curl http://localhost:5001/api/v0/version
```

**Restart IPFS:**
```bash
docker-compose restart ipfs
```

### Out of Memory

**Increase Docker Desktop memory:**
1. Open Docker Desktop Settings
2. Resources → Memory
3. Increase to 8GB or more
4. Apply & Restart

**Check container memory usage:**
```bash
docker stats
```

### Slow Indexing

**Potential causes:**
- RPC rate limiting (use premium RPC)
- Slow RPC response times
- Insufficient system resources

**Monitor sync speed:**
```bash
yarn logs | grep "Scanning blocks"
```

### Subgraph Deployment Fails

**Error: "Failed to connect to IPFS"**
```bash
curl http://localhost:5001/api/v0/version
docker-compose restart ipfs
```

**Error: "Subgraph name already exists"**
```bash
graph remove --node http://localhost:8020 gmx/synthetics-mantle-sepolia
yarn deploy
```

**Error: "Failed to deploy to Graph Node"**
```bash
yarn status
yarn logs
```

## Network Configuration

### Mantle Sepolia Details

- **Network**: mantle-sepolia
- **RPC**: https://rpc.sepolia.mantle.xyz (public)
- **Chain ID**: 5003
- **EventEmitter**: 0x8503471902b5915A820cB6f1B6471c1Fe623d5d3
- **Start Block**: 31603039

### Using Custom RPC

Edit `.env`:

```bash
MANTLE_SEPOLIA_RPC=https://your-custom-rpc-endpoint.com
```

Then restart:

```bash
yarn stop
yarn start
```

## Port Mappings

| Port | Service | Purpose |
|------|---------|---------|
| 5432 | PostgreSQL | Database admin access |
| 5001 | IPFS | API (for deployment) |
| 8080 | IPFS | HTTP Gateway |
| 8000 | Graph Node | GraphQL HTTP queries |
| 8001 | Graph Node | GraphQL WebSocket |
| 8020 | Graph Node | Admin API (deployment) |
| 8030 | Graph Node | Indexing status |
| 8040 | Graph Node | Prometheus metrics |

## Data Persistence

All data is stored in Docker named volumes:

- `postgres_data` - Indexed subgraph data
- `ipfs_data` - IPFS repository
- `graph_data` - Graph Node internal state

### Backup Data

**PostgreSQL:**
```bash
docker-compose exec postgres pg_dump -U graph graph-node > backup-$(date +%Y%m%d).sql
```

**Restore:**
```bash
cat backup-20250101.sql | docker-compose exec -T postgres psql -U graph graph-node
```

### Clear Data (Destructive)

```bash
yarn clean
```

This removes all volumes and deletes all indexed data.

## Production Hardening

### Security

1. **Change default password** in `.env`:
   ```bash
   POSTGRES_PASSWORD=your-strong-unique-password-here
   ```

2. **Restrict port access** - Use firewall rules to limit access:
   - Expose only port 8000 (GraphQL) publicly
   - Keep 8020 (admin) internal or VPN-only
   - Keep 5432 (postgres) internal only

3. **Use dedicated RPC** with authentication for better reliability

### Performance

1. **Increase PostgreSQL resources** for large subgraphs:
   - Edit `docker-compose.yml` PostgreSQL command section
   - Increase `shared_buffers`, `effective_cache_size`

2. **Use SSD storage** for Docker volumes

3. **Monitor metrics** at http://localhost:8040/metrics

4. **Scale vertically** - Increase CPU/RAM for high-traffic subgraphs

### Monitoring

**Option 1: Prometheus + Grafana**

Add to `docker-compose.yml`:
```yaml
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

**Option 2: Cloud monitoring**

Use services like Datadog, New Relic, or Grafana Cloud.

## Resource Requirements

| Subgraph Size | RAM | CPU | Storage | Sync Time |
|---------------|-----|-----|---------|-----------|
| Small (<1M events) | 4GB | 2 cores | 20GB | 2-6 hours |
| Medium (1-10M) | 8GB | 4 cores | 50GB | 6-24 hours |
| Large (>10M) | 16GB+ | 8+ cores | 100GB+ | 1-7 days |

**Mantle Sepolia (testnet)** is likely Small-Medium size.

**Recommended minimum:**
- 8GB RAM
- 2 CPU cores
- 30GB storage
- Expected initial sync: 4-12 hours

## Switching from Satsuma

To migrate from Satsuma hosted service to self-hosted:

1. **Deploy locally** using this setup
2. **Wait for full sync** (monitor with `yarn status`)
3. **Update frontend** GraphQL endpoint to `http://localhost:8000` (or your server's IP)
4. **Verify data parity** - Compare queries between Satsuma and local
5. **Monitor performance** - Ensure local node keeps up with chain head

## Cost Analysis

**Self-hosted vs Satsuma:**

| Item | Self-hosted | Satsuma |
|------|-------------|---------|
| Infrastructure | $50-100/month (VPS) | Included in plan |
| RPC costs | Free (public) or $10-50/mo (premium) | Included |
| Maintenance | 2-4 hours/month | None |
| Query limits | Unlimited | Plan-based |
| Control | Full control | Limited |

**Benefits of self-hosting:**
- No vendor lock-in
- Unlimited queries
- Full control over data
- Custom optimizations
- No rate limits

## Environment Variables Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `POSTGRES_USER` | graph | PostgreSQL username |
| `POSTGRES_PASSWORD` | (required) | PostgreSQL password |
| `POSTGRES_DB` | graph-node | PostgreSQL database name |
| `MANTLE_SEPOLIA_RPC` | https://rpc.sepolia.mantle.xyz | RPC endpoint |
| `GRAPH_LOG_LEVEL` | info | Log level (debug, info, warn, error) |
| `GRAPH_ALLOW_NON_DETERMINISTIC_IPFS` | false | Allow non-deterministic IPFS |

## Support & Resources

- **Graph Protocol Docs**: https://thegraph.com/docs/
- **Graph Node GitHub**: https://github.com/graphprotocol/graph-node
- **Mantle Network Docs**: https://docs.mantle.xyz/
- **Discord**: The Graph Discord server

## FAQ

**Q: How long does initial sync take?**
A: For Mantle Sepolia from block 31603039, expect 4-12 hours depending on RPC performance.

**Q: Can I use this for production?**
A: Yes, but ensure you follow the Production Hardening section.

**Q: What if the RPC rate limits me?**
A: Use a paid RPC provider (Alchemy, Infura alternatives for Mantle) or reduce `GRAPH_ETHEREUM_MAX_BLOCK_RANGE_SIZE` in docker-compose.yml.

**Q: How do I update Graph Node version?**
A: Edit `docker-compose.yml`, change image version, run `yarn stop && yarn start`.

**Q: Can I run multiple subgraphs?**
A: Yes! Create additional subgraphs with `graph create --node http://localhost:8020 <org>/<subgraph-name>`.

**Q: How do I know if sync is complete?**
A: Run `yarn status` - when `latestBlock` matches `chainHeadBlock`, sync is complete.

## License

This setup is for the GMX Synthetics Stats subgraph. See main project LICENSE for details.
