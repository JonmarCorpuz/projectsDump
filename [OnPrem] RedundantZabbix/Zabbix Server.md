# Install Zabbix Server

1. Install the latest Zabbix repository package for Ubuntu 24.04 
```Bash
sudo wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb
sudo apt -y update
```

<br>

2. Install the Zabbix server frontend
```Bash
sudo apt -y install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

<br>

3. [Install and configure](https://github.com/JonmarCorpuz/projectsDump/blob/main/%5BOnPrem%5D%20RedundantZabbix/Galera%20Cluster.md) the MariabDB databases and Galera cluster

<br>

4. Configure the Zabbix server daemon
```Bash
sudo nano /etc/zabbix/zabbix_server.conf
```
```INI
############# GENERAL PARAMETERS ################
DBHost=ip_address             # The IP address of your database server
DBName=db_name                # The name of the database that Zabbix will use
DBUser=db_user                # The username that Zabbix will use to connect to the database
DBPassword=db_password        # The passsword of the database user

############ ADVANCED PARAMETERS ################
SNMPTrapperFile=/var/log/snmptrap/snmptrap.log        # The path to the log file for captured SNMP traps
StartSNMPTrapper=1                                    # Enable the SNMP trapper process
Timeout=4                                             # Set the maximum time Zabbix will wait for agent responses and other operations
FpingLocation=/usr/bin/fping                          # The path to the 'fping' binary used for ICMP checks
LogSlowQueries=3000                                   # The log queries that take longer than 3 seconds to execute
StatsAllowedIP=127.0.0.1                              # Instruct to only allow statistics access from the localhost

########### ADDITIONAL PARAMETERS ###############
Include=/etc/zabbix/zabbix_server.d/*.conf           # Include additional configuration files from this directory
```

<br>

5. Enable and start the Zabbix server
```Bash
# Enable the Zabbix server service to start automatically at boot
sudo systemctl enable zabbix-server

# Start the Zabbix server immediately
sudo systemctl start zabbix-server
```

<br>

# Verify Zabbix Server Configuration

Verify Zabbix server is running
```Bash
sudo systemctl status zabbix-server
```
