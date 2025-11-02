# Setting Up KeepAlived on the HAProxy

1. Install KeepAlived
```Bash
sudp apt -y update
sudo apt -y install keepalived
```

<br>

2. Set up the main configuration for Keepalived
```Bash
sudo nano /etc/keepalived/keepalived.conf
```
```INI
# Script to check if haproxy is running
vrrp_script chk_haproxy {
    script "pidof haproxy"         # Command to run for health check (0=found, non-0=not found)
    interval 2                     # Run the script every 2 seconds
    weight 2                       # Increase VRRP instance priority by 2 if script is successful
}

# Definition of the HA VRRP instance
vrrp_instance VI_1 {
    state MASTER                   # Configure this as the master node
    interface eth0                 # Specify the network interface to bind VRRP to
    virtual_router_id 51           # Specify the unique router ID for this VRRP group
    priority 101                   # Set the priority for this node
    advert_int 1                   # Configure the heartbeat for the VRRP advertisement interval in seconds
    authentication {
        auth_type PASS             # Specify the authentication type for VRRP messaging
        auth_pass MyStrongPass     # Specify the shared password for VRRP authentication
    }
    virtual_ipaddress {
        VIRTUAL_IP_ADDRESS         # Configure the Virtual IP address to be managed by VRRP and failover
    }
    track_script {
        chk_haproxy                # Specify to use the defined script to monitor HAProxy service health
    }
}
```

<br>

3. Apply the changes
```Bash
sudo systemctl restart keepalived
```

<br>

# Verify Keepalived Configuration

Verify that the virtual IP address is reachable
```Bash
ping VIRTUAL_IP_ADDRESS
```
