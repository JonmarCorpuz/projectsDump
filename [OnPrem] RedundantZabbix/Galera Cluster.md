1. Install MariaDB, Galera, and Rsync on each database node
```Bash
sudo apt -y install mariadb-server mariadb-client galera-4 rsync
```

2. Open the necessary ports
```Bash
sudo ufw allow 3306,4444,4567,4568/tcp
sudo ufw allow 4567/udp
sudo ufw reload
```

3. Create the Zabbix database and user
```Bash
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
