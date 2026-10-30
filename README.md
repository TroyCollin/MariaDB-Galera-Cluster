# MariaDB Galera Cluster & MaxScale Multi-Host Deployment

This repository provides production-ready configuration files to deploy a highly available, multi-host **MariaDB Galera Cluster** with an intelligent **MariaDB MaxScale** load-balancing proxy.

## Architecture Overview
- **Type:** Synchronous Multi-Master Cluster (Zero data loss, automatic failover)
- **Minimum Nodes:** 3 Database Nodes (To achieve quorum and prevent split-brain)
- **Routing:** MaxScale `readwritesplit` service routes writes to a single target master and distributes reads across all healthy cluster nodes.

---

## 🛠️ Step 1: Initialize Database User

Before spinning up MaxScale, log into **any** of your active MariaDB nodes and execute the following SQL script to create the monitoring user (changes will replicate automatically across the cluster):

```sql
CREATE USER 'maxuser'@'%' IDENTIFIED BY 'MaxScaleSecretPassword';
GRANT SELECT ON mysql.user TO 'maxuser'@'%';
GRANT SELECT ON mysql.db TO 'maxuser'@'%';
GRANT SELECT ON mysql.tables_priv TO 'maxuser'@'%';
GRANT SELECT ON mysql.roles_mapping TO 'maxuser'@'%';
GRANT SHOW DATABASES ON *.* TO 'maxuser'@'%';
FLUSH PRIVILEGES;
```

---

## 🖥️ Step 2: Database Node Configurations

Deploy each `docker-compose.yml` file to its respective physical host machine. Ensure ports `3306`, `4567` (TCP/UDP), `4568`, and `4444` are wide open between these hosts.

### Host 1 (Node A - Cluster Bootstrapper)
*File: `docker-compose.yml` on Host 1*
```yaml
version: '3.8'

services:
  mariadb-node-1:
    image: mariadb:11.4
    container_name: mariadb_node_1
    restart: always
    command: [
      "mariadbd",
      "--wsrep-on=ON",
      "--wsrep-provider=/usr/lib/galera/libgalera_smm.so",
      "--wsrep-cluster-name=my_galera_cluster",
      "--wsrep-cluster-address=gcomm://", 
      "--wsrep-node-name=node1",
      "--wsrep-node-address=192.168.1.10",
      "--wsrep-new-cluster" 
    ]
    ports:
      - "3306:3306"
      - "4567:4567"
      - "4567:4567/udp"
      - "4568:4568"
      - "4444:4444"
    environment:
      MARIADB_ROOT_PASSWORD: "SuperSecureRootPassword"
      MARIADB_DATABASE: "production_db"
    volumes:
      - mariadb_data_1:/var/lib/mysql

volumes:
  mariadb_data_1:
```

### Host 2 (Node B)
*File: `docker-compose.yml` on Host 2*
```yaml
version: '3.8'

services:
  mariadb-node-2:
    image: mariadb:11.4
    container_name: mariadb_node_2
    restart: always
    command: [
      "mariadbd",
      "--wsrep-on=ON",
      "--wsrep-provider=/usr/lib/galera/libgalera_smm.so",
      "--wsrep-cluster-name=my_galera_cluster",
      "--wsrep-cluster-address=gcomm://192.168.1.10:4567",
      "--wsrep-node-name=node2",
      "--wsrep-node-address=192.168.1.11"
    ]
    ports:
      - "3306:3306"
      - "4567:4567"
      - "4567:4567/udp"
      - "4568:4568"
      - "4444:4444"
    environment:
      MARIADB_ROOT_PASSWORD: "SuperSecureRootPassword"
    volumes:
      - mariadb_data_2:/var/lib/mysql

volumes:
  mariadb_data_2:
```

### Host 3 (Node C)
*File: `docker-compose.yml` on Host 3*
```yaml
version: '3.8'

services:
  mariadb-node-3:
    image: mariadb:11.4
    container_name: mariadb_node_3
    restart: always
    command: [
      "mariadbd",
      "--wsrep-on=ON",
      "--wsrep-provider=/usr/lib/galera/libgalera_smm.so",
      "--wsrep-cluster-name=my_galera_cluster",
      "--wsrep-cluster-address=gcomm://192.168.1.10:4567,192.168.1.11:4567",
      "--wsrep-node-name=node3",
      "--wsrep-node-address=192.168.1.12"
    ]
    ports:
      - "3306:3306"
      - "4567:4567"
      - "4567:4567/udp"
      - "4568:4568"
      - "4444:4444"
    environment:
      MARIADB_ROOT_PASSWORD: "SuperSecureRootPassword"
    volumes:
      - mariadb_data_3:/var/lib/mysql

volumes:
  mariadb_data_3:
```

⚠️ **CRUCIAL OPERATIONAL NOTE:** Once Node 2 and Node 3 have successfully joined the cluster, edit the `docker-compose.yml` file on **Host 1** to remove the `--wsrep-new-cluster` flag from the command list. If the container restarts later with this flag still present, it will break quorum by trying to initialize a brand-new, isolated cluster grid.

---

## 🚀 Step 3: MaxScale Load Balancer Setup

Place both files below into the same working directory on your proxy host (can be a standalone 4th server or co-located with a node).

### MaxScale Configuration File (`maxscale.cnf`)
```ini
[maxscale]
threads=auto

[node1]
type=server
address=192.168.1.10
port=3306
protocol=MariaDBBackend

[node2]
type=server
address=192.168.1.11
port=3306
protocol=MariaDBBackend

[node3]
type=server
address=192.168.1.12
port=3306
protocol=MariaDBBackend

[galera-monitor]
type=monitor
module=galeramon
servers=node1,node2,node3
user=maxuser
password=MaxScaleSecretPassword
monitor_interval=2000ms

[split-service]
type=service
router=readwritesplit
servers=node1,node2,node3
user=maxuser
password=MaxScaleSecretPassword

[split-listener]
type=listener
service=split-service
protocol=MariaDBClient
port=4008
```

### MaxScale Docker Compose (`docker-compose.yml`)
```yaml
version: '3.8'

services:
  maxscale-proxy:
    image: mariadb/maxscale:latest
    container_name: maxscale_proxy
    restart: always
    ports:
      - "4008:4008"
      - "8989:8989"
    volumes:
      - ./maxscale.cnf:/etc/maxscale.cnf:ro
```

Launch the proxy using:
```bash
docker compose up -d
```

---

## 🔍 Step 4: Health and Topology Verification

### Check Galera Cluster Quorum Size
Run this inside any database container to verify all 3 nodes see each other:
```bash
docker exec -it mariadb_node_1 mariadb -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

### Check MaxScale Routing Map
Run this inside your MaxScale container to see which nodes are handling reads, writes, or are offline:
```bash
docker exec -it maxscale_proxy maxctrl list servers
```

Point all your production client applications directly to the **MaxScale Proxy Host IP** on port **`4008`**.