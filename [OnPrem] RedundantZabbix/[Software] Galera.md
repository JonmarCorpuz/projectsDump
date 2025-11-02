# Setting up the Galera Cluster for Zabbix's Backend

1. Install MariaDB, Galera, and Rsync on each database node
```Bash
sudo apt -y install mariadb-server mariadb-client galera-4 rsync
```

<br>

2. Open the necessary ports
```Bash
# Allow incoming MariaDB TCP connections
sudo ufw allow 3306/tcp

# Allow incoming Galera TCP communications and connections
sudo ufw allow 4444,4567,4568/tcp

# Allow incoming Galera UDP connections for replication traffic 
sudo ufw allow 4567/udp

# Reload UFW to apply the new firewall rules
sudo ufw reload
```

<br>

3. Create the Zabbix database and user
```Bash
# Open the MariabDB CLI client as the root database user
sudo mysql -u root -p
```
```SQL
-- Create a new database with utf8mb4 character set and collation for proper Unicode support
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

-- Create a new user that can connect from localhost 
CREATE USER zabbix@localhost IDENTIFIED BY 'Password123**';

-- Grant all privileges on the new database to the new user that was created
GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;

-- Enable the creation of stored functions
SET GLOBAL log_bin_trust_function_creators = 1;

-- Exit the CLI client
QUIT;
```

<br>

4. Fetch the main schema initialization script for Zabbix's MariaDB database backend
```Bash
# Download the latest Zabbix repository package for Ubuntu 24.04
sudo wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb

# Install the downloaded Zabbix repository .deb package to enable Zabbix-related apt sources
sudo dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb

# Update the package list to include Zabbix packages from the new repository
sudo apt -y update
```

<br>

5. Extract the initial schema and data for Zabbix's database backend directly into the MariaDB client
```Bash
sudo zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

<br>

6. Disable the log_bin_trust_function_creators option after importing the database schema
```Bash
# Open the MariabDB CLI client as the root database user
sudo mysql -u root -p
```
```SQL
SET GLOBAL log_bin_trust_function_creators = 0;
QUIT;
```

<br>

7. Stop MariaDB on all database nodes
```Bash
sudo systemctl stop mariadb
```

<br>

8. Define the Galera cluster-specific settings on each database node before bootstrapping the MariaDB Galera cluster
```Bash
sudo nano /etc/mysql/mariadb.conf.d/60-galera.cnf
```
```INI
[mysqld]
# Set binary log format required for Galera
binlog_format=ROW

# Use InnoDB storage engine for clustering
default_storage_engine=InnoDB

# Allow interleaved lock mode for auto-increment columns
innodb_autoinc_lock_mode=2

# Bind to all IPs (adjust to private IP if needed)
bind-address=0.0.0.0

# Enable Galera replication support
wsrep_on=ON

# Path to the Galera wsrep provider library
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Cluster name, must be the same on all nodes
wsrep_cluster_name="my_galera_cluster"

# List all cluster node IPs in gcomm URL for communication
wsrep_cluster_address="gcomm://NODE1_IP,NODE2_IP"

# Node name (must be unique per node)
wsrep_node_name="NODE_NAME"

# Node's IP address (must be unique per node)
wsrep_node_address="NODE_IP"

# Set the SST (State Snapshot Transfer) method
wsrep_sst_method=rsync

# Trust function creators (required for some Zabbix operations)
log_bin_trust_function_creators=ON
```

<br>

9. Select a primary database node and bootstrap the MariaDB Galera Cluster after ensuring that MariaDB has stopped running on all the database nodes
```Bash
sudo galera_new_cluster
```

<br>

10. Start MariaDB on the other database nodes
```Bash
sudo systemctl start mariadb
```

<br>

# Verify Cluster Configuration

Show the number of nodes that are currently connected to the cluster
```SQL
sudo mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

<br>

Show the node's current state 
```SQL
sudo mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"
```
| Node State | Description |
| --- | --- |
| **Synced** | The node is healthy, fully synchronized, and a fully operational member of the cluster |
| **Joining** | The node is in the process of joining the cluster and is synchronizing its data with the other nodes |
| **Donor** | The node is in read-only mode while providing a State Snapshot Transfer to another node |

<br>

Show if the node is able to communicate with the other nodes within the cluster
```SQL
sudo mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_connected';"
```
| Possible Value | Description |
| --- | --- |
| **ON** | The node has network connectivity with other components in the Galera cluster and can communicate |
| **OFF** | The node cannot connect to any other cluster components and is either isolated or misconfigured |

<br>

Show the cluster UUID
```SQL
sudo mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_state_uuid';"
```

<br>

Show the cluster's name
```SQL
mysql -u root -p -e "SHOW VARIABLES LIKE 'wsrep_cluster_name';"
```
