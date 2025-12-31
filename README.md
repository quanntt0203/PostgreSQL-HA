# PostgreSQL HA Cluster - Multiple Master Replication (MMR) Setup

This directory contains a complete PostgreSQL Citus High Availability cluster setup with Patroni, HAProxy, and PgBouncer.

## Architecture

### Components
- **3 Coordinator Nodes** (Group 0) with 3 Data Definition(DD) Nodes: 
   - coord1 (Leader)
   - coord2 (Replica)
   - coord3 (Replica)
- **2 Worker Groups** with 2 Sharding nodes each:
  - Group 1: Data Nodes
      - **work1-1** (Leader)
      - **work1-2** (Replica)  
  - Group 2: Data Nodes
      - **work2-1** (Leader)
      - **work2-2** (Replica) 
- **HAProxy** for load balancing
- **PgBouncer** for connection pooling
- **etcd** for cluster coordination
- **Patroni** for high availability and automatic failover

### Network Layout
```
┌─────────────────────────────────────────────────────────────────┐
│                    Citus HA Cluster - MMR                       │
├─────────────────────────────────────────────────────────────────┤
│  Client Applications                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │   PgBouncer     │ (Port 6432)                                │
│  │ Connection Pool │                                            │
│  └─────────────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │     HAProxy     │ (Port 5433/5434)                           │
│  │ Load Balancer   │                                            │
│  └─────────────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌────────────────────────────────────────────┐                 ┤
│  │      Coordinator Cluster (Group 0)         │                 │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐     │                 │
│  │  │ coord1  │  │ coord2  │  │ coord3  │     │                 │
│  │  │ :5432   │  │ :5442   │  │ :5452   │     │                 │
│  │  └─────────┘  └─────────┘  └─────────┘     │                 │
│  └────────────────────────────────────────────┘                 ┤
│           │                         │                           │
│           ▼                         ▼                           │
│  ┌─────────────────┐       ┌─────────────────┐                  │
│  │ Worker Group 1  │       │ Worker Group 2  │                  │
│  │ ┌─────────────┐ │       │ ┌─────────────┐ │                  │
│  │ │ work1-1     │ │       │ │ work2-1     │ │                  │
│  │ │ :5462       │ │       │ │ :5482       │ │                  │
│  │ └─────────────┘ │       │ └─────────────┘ │                  │
│  │ ┌─────────────┐ │       │ ┌─────────────┐ │                  │
│  │ │ work1-2     │ │       │ │ work2-2     │ │                  │
│  │ │ :5472       │ │       │ │ :5492       │ │                  │
│  │ └─────────────┘ │       │ └─────────────┘ │                  │
│  └─────────────────┘       └─────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

## Quick Start

### 1. Start the Cluster
```powershell
# Option A: Use the management script
.\manage_cluster.sh -Action start

# Option B: Use docker-compose directly  
docker-compose up -d
```

### 2. Set up Citus Configuration
```powershell
# Wait for all services to be ready, then run setup
.\manage_cluster.sh -Action setup

# Or run setup script directly
.\setup_cluster.sh
```

### 3. Verify Cluster Status
```powershell
.\manage_cluster.sh -Action status
```

### 4. Test the Cluster
```powershell
.\manage_cluster.sh -Action test
```

## Connection Details

### Primary Access Points
- **PgBouncer (Recommended)**: `localhost:6432`
- **HAProxy Load Balanced**: `localhost:5433`  
- **HAProxy Master Only**: `localhost:5434`
- **HAProxy Stats**: `http://localhost:5000/stats`

### Direct Node Access
- **Coordinator 1**: `localhost:5432`
- **Coordinator 2**: `localhost:5442`
- **Coordinator 3**: `localhost:5452`
- **Worker 1-1**: `localhost:5462`
- **Worker 1-2**: `localhost:5472`
- **Worker 2-1**: `localhost:5482`
- **Worker 2-2**: `localhost:5492`

### Default Credentials
- **Username**: `postgres`
- **Password**: `postgrespass`
- **Database**: `citus_db`

## Management Commands

### PowerShell Management Script
```powershell
# Start cluster
.\manage_cluster.sh -Action start

# Stop cluster  
.\manage_cluster.sh -Action stop

# Restart cluster
.\manage_cluster.sh -Action restart

# Check status
.\manage_cluster.sh -Action status

# Setup Citus configuration
.\manage_cluster.sh -Action setup

# Test cluster
.\manage_cluster.sh -Action test

# View logs
.\manage_cluster.sh -Action logs
.\manage_cluster.sh -Action logs -Service coord1
```

### Direct Docker Commands
```powershell
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# View logs
docker-compose logs -f

# Check container status
docker-compose ps
```

### Database Connection Examples
```powershell
# Connect via PgBouncer (recommended)
psql -h localhost -p 6432 -U postgres -d citus_db

# Connect directly to coordinator
docker exec -it coord1 psql -U postgres -d citus_db

# Connect via HAProxy load balancer
psql -h localhost -p 5433 -U postgres -d citus_db
```

## Key Features

### High Availability
- **Automatic Failover**: Patroni automatically promotes standby nodes when primary fails
- **Quorum-based Replication**: Ensures data consistency across replicas
- **Health Monitoring**: Continuous health checks and automatic recovery

### Load Balancing  
- **HAProxy**: Distributes connections across coordinator nodes
- **Connection Pooling**: PgBouncer reduces connection overhead
- **Read/Write Splitting**: Separate endpoints for read and write operations

### Distributed Database
- **Horizontal Scaling**: Data distributed across multiple worker nodes
- **Automatic Sharding**: Citus handles data distribution automatically  
- **Cross-shard Queries**: Distributed SQL queries across all worker nodes

### Monitoring
- **HAProxy Stats**: Web interface for connection monitoring
- **Patroni REST API**: Cluster status and health information
- **PostgreSQL Logs**: Detailed logging for troubleshooting

## Troubleshooting

### Common Issues

1. **Containers not starting**
   ```powershell
   docker-compose logs
   ```

2. **Patroni cluster not forming**  
   ```powershell
   docker exec coord1 patronictl -c etcd://etcd:2379 list citus_cluster
   ```

3. **Workers not connecting to coordinators**
   ```powershell
   docker exec coord1 psql -U postgres -d citus_db -c "SELECT * FROM citus_get_active_worker_nodes();"
   ```

4. **HAProxy health checks failing**
   - Check http://localhost:5000/stats
   - Verify PostgreSQL is accepting connections

### Reset Cluster
```powershell
# Stop and remove all containers and volumes
docker-compose down -v

# Restart fresh
docker-compose up -d
.\setup_cluster.sh
```

## File Structure
```
patroni-MMA/citus/
├── docker-compose.yml         # Main orchestration file
├── Dockerfile                 # Custom Citus/Postgres-16+Patroni+pgBackRest image
├── manage_cluster.sh         # Management script
├── setup_cluster.sh          # Cluster configuration script
├── setup_cluster.sh           # Bash version of setup script  
├── patroni/                   # Patroni configurations
│   ├── coord1.yml
│   ├── coord2.yml
│   ├── coord3.yml
│   ├── work1-1.yml
│   ├── work1-2.yml
│   ├── work2-1.yml
│   └── work2-2.yml
├── haproxy/
│   └── haproxy.cfg           # HAProxy configuration
└── pgbouncer/
    ├── pgbouncer.ini         # PgBouncer configuration
    └── userlist.txt          # User authentication
```

## Initial a coord/worker node from bgbackrest repo

Step 1: Do the lastest full backup from leader

```bash
# From the pgbackup_server
pgbackrest --stanza=citus_worker1 --type=full --log-level-console=error backup
```

Step 2: Force Patroni to reinit a node from the full backup of bgbackrest (Step 1)

```bash
# From any active node in the cluster
patronictl reinit citus_cluster_mmr citus_work1-4 --force
```

## Next Steps

1. **Monitor Performance**: Set up Prometheus and Grafana for monitoring
2. **Backup Strategy**: Implement automated backup procedures
3. **Security**: Configure SSL/TLS and proper authentication
4. **Scaling**: Add more worker nodes as needed
5. **Testing**: Implement failover testing procedures