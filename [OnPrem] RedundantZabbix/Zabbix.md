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

3. [Install and configure](https://github.com/JonmarCorpuz/projectsDump/blob/main/%5BOnPrem%5D%20RedundantZabbix/Galera%20Cluster.md) the MariabDB databases and Galera cluster for Zabbix's backend

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

6. Continue the installation from a web browser by visiting `http://localhost/zabbix`

![](https://github.com/JonmarCorpuz/projectsDump/blob/main/%5BOnPrem%5D%20RedundantZabbix/Images/Screenshot_2025-10-30_at_19-36-34_Installation.png)

![](https://github.com/JonmarCorpuz/projectsDump/blob/main/%5BOnPrem%5D%20RedundantZabbix/Images/Screenshot_2025-10-30_at_19-36-49_Installation.png)

![](https://github.com/JonmarCorpuz/projectsDump/blob/main/%5BOnPrem%5D%20RedundantZabbix/Images/Screenshot_2025-10-30_at_19-37-04_Installation.png)

![](https://github.com/JonmarCorpuz/projectsDump/blob/main/%5BOnPrem%5D%20RedundantZabbix/Images/Screenshot_2025-10-30_at_19-37-12_Installation.png)

![](https://github.com/JonmarCorpuz/projectsDump/blob/main/%5BOnPrem%5D%20RedundantZabbix/Images/Screenshot_2025-10-30_at_19-37-17_Installation.png)

![](https://github.com/JonmarCorpuz/projectsDump/blob/main/%5BOnPrem%5D%20RedundantZabbix/Images/Screenshot_2025-10-30_at_19-37-22_Installation.png)

![](https://github.com/JonmarCorpuz/projectsDump/blob/main/%5BOnPrem%5D%20RedundantZabbix/Images/Screenshot_2025-10-30_at_19-37-34_zabbix1_Zabbix.png)

![](https://github.com/JonmarCorpuz/projectsDump/blob/main/%5BOnPrem%5D%20RedundantZabbix/Images/Screenshot_2025-10-30_at_19-37-40_zabbix1_Tableau_de_bord.png)

<br>

# Monitor Hosts Using Zabbix

## Zabbix Agent on Linux

1. Install the latest Zabbix repository package for Ubuntu 24.04 
```Bash
sudo wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb
sudo apt -y update
```

<br>

2. Install the Zabbix agent
```Bash
sudo apt -y install zabbix-agent
```

<br>

3. Configure the Zabbix agent daemon
```Bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```
```INI
# Specify the path to store the agent's process ID file
PidFile=/run/zabbix/zabbix_agentd.pid

# Specify the file to log agent events and activity        
LogFile=/var/log/zabbix/zabbix_agentd.log

# Specify the maximum log file size in MB (0 means unlimited)
LogFileSize=0

# Specify the IP address of the Zabbix servers for passive agent checks
Server=ZABBIX1_SERVER_IP,ZABBIX2_SERVER_IP

# Specify the IP address of the Zabbix servers for active agent checks
ServerActive=ZABBIX1_SERVER_IP,ZABBIX2_SERVER_IP

# Specify the hostname for this agent (This must match with the hostname configured on the Zabbix server)
Hostname=HOSTNAME

# Specify to include the additional agent configuration files from this directory
Include=/etc/zabbix/zabbix_agentd.d/*.conf         
```

<br>

4. Enable and start the Zabbix agent
```Bash
# Enable the Zabbix agent service to start automatically at boot
sudo systemctl enable zabbix-agent

# Start the Zabbix agent immediately
sudo systemctl start zabbix-agent
```

<br>

## SNMP

1. Configure SNMP on Cisco switches and routers
```INI
snmp-server community MyCommunity RO
snmp-server host ZABBIX1_IP_ADDRESS version 2c MyCommunity
snmp-server host ZABBIX2_IP_ADDRESS version 2c MyCommunity
snmp-server location LOCATION
snmp-server contact CONTACT_EMAIL
```

# Verify Zabbix Server Configuration

Verify Zabbix server is running
```Bash
sudo systemctl status zabbix-server
```

<br>

# Documentation

* [Download and Install Zabbix Server](https://www.zabbix.com/download?zabbix=5.2&os_distribution=ubuntu)
