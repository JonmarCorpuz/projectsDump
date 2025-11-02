# Install and Configure HAProxy

1. Install HAProxy
```Bash
sudo apt -y update && sudo apt -y install haproxy
```

2. Enable IP forwarding
```Bash
# Enable binding to non-local IP addresses (required for HAProxy use in cluster/HA setups)
sudo bash -c "echo 'net.ipv4.ip_nonlocal_bind = 1' >> /etc/sysctl.conf"

# Enable IP forwarding (needed for routing and load balancing functions)
sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf

# Apply all kernel parameter changes from sysctl.conf
sudo sysctl -p
```

# Setting Up the HAProxy for Zabbix's Frontend

1. Configure HAProxy's main configuration file for the HAProxy load balancer
```Bash
sudo nano /etc/haproxy/haproxy.cfg
```
```INI
# Define settings for the entire HAProxy process 
global
    log /dev/log local0           # Log HAProxy events to syslog, facility local0
    log /dev/log local1 notice    # Log HAProxy notices to syslog, facility local1
    daemon                        # Run HAProxy as a background (daemon) process
    maxconn 2048                  # Maximum number of simultaneous connections allowed

# Define the baseline options that will be inherited by each section
defaults
    log global                    # Inherit global logging settings
    mode http                     # Process traffic in HTTP mode
    timeout connect 5000ms        # Timeout for establishing a backend connection (5 seconds)
    timeout client  50000ms       # Timeout for client inactivity (50 seconds)
    timeout server  50000ms       # Timeout for server inactivity (50 seconds)
    option httplog                # Use a detailed HTTP log format
    option dontlognull            # Don't log null connections (less log noise)

# Define how HAProxy will accept and handle incoming connections
frontend zabbix_frontend
    bind *:80                              # Listen for incoming HTTP requests on port 80
    default_backend zabbix_backend         # Route traffic to the 'zabbix_backend' backend group

# Define the group of servers that the frontend will forward requests to
backend zabbix_backend
    balance roundrobin                     # Distribute requests evenly across backend servers
    option httpchk GET /zabbix/            # Health check: HTTP GET request to '/zabbix/' path
    server zabbix1 ip_address:80 check     # Backend server 1 with health check enabled
    server zabbix2 ip_address:80 check     # Backend server 2 with health check enabled

# Expose the HAProxy stats dashboard (Optional)
listen stats
    bind *:8080                   # Listen for stats page requests on port 8080
    stats enable                  # Enable the statistics web interface
    stats uri /haproxy?stats      # Stats page accessible at '/haproxy?stats'
    stats refresh 5s              # Auto-refresh stats page every 5 seconds
    stats auth admin:password     # Require HTTP Basic Auth ('admin'/'password') for stats
```

2. Restart the HAProxy to apply the changes
```Bash
sudo systemctl restart haproxy
```

<br>

# Setting Up the HAProxy for Zabbix's Backend

1. Configure HAProxy's main configuration file for the HAProxy load balancer
```Bash
sudo nano /etc/haproxy/haproxy.cfg
```
```INI
# Define settings for the entire HAProxy process 
global
    log /dev/log local0           # Log HAProxy events to syslog, facility local0
    log /dev/log local1 notice    # Log HAProxy notices to syslog, facility local1
    daemon                        # Run HAProxy as a background (daemon) process
    maxconn 2048                  # Maximum number of simultaneous connections allowed

# Define the baseline options that will be inherited by each section
defaults
    log global                    # Inherit global logging settings
    mode http                     # Process traffic in HTTP mode
    timeout connect 5000ms        # Timeout for establishing a backend connection (5 seconds)
    timeout client  50000ms       # Timeout for client inactivity (50 seconds)
    timeout server  50000ms       # Timeout for server inactivity (50 seconds)
    option httplog                # Use a detailed HTTP log format
    option dontlognull            # Don't log null connections (less log noise)

# Define how HAProxy will accept and handle incoming connections
frontend galera_frontend
    bind 192.168.1.200:3306                # Bind the frontend to the virtual IP and port for the Galera cluster clients
    mode tcp                               # Use TCP mode for raw transport-layer load balancing of database traffic
    default_backend galera_backend         # Route traffic to the 'zabbix_backend' backend group

# Define the group of servers that the frontend will forward requests to
backend galera_backend
    balance roundrobin                     # Distribute requests evenly across backend servers
    option tcp-check                       # Enable TCP health checking
    server zabbix1 ip_address:80 check     # Backend server 1 with health check enabled
    server zabbix2 ip_address:80 check     # Backend server 2 with health check enabled

# Expose the HAProxy stats dashboard (Optional)
listen stats
    bind *:8080                   # Listen for stats page requests on port 8080
    stats enable                  # Enable the statistics web interface
    stats uri /haproxy?stats      # Stats page accessible at '/haproxy?stats'
    stats refresh 5s              # Auto-refresh stats page every 5 seconds
    stats auth admin:password     # Require HTTP Basic Auth ('admin'/'password') for stats
```

2. Restart the HAProxy to apply the changes
```Bash
sudo systemctl restart haproxy
```

<br>

# Verify HAProxy Configuration

Verify HAProxy's status
```Bash
sudo systemctl status haproxy
```
