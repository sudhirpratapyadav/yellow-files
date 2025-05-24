# Resolving Docker Network Conflict with College or VPN Network

## **Problem Overview**
When Docker is installed, it creates a default network bridge (`docker0`) with the subnet `172.17.0.0/16`. If this subnet overlaps with your college network or VPN, it can cause routing conflicts, making it impossible to access the internet or sign in to network-dependent services.

**Important Note**: Docker Compose and custom Docker networks can create additional bridges that bypass your Docker daemon configuration, causing the same conflict to reoccur even after fixing the default bridge.

This document explains how to identify the issue, diagnose the conflict, and resolve it by changing Docker's default bridge network and preventing future conflicts.

---

## **Step 1: Understanding the Problem**
When Docker starts, it automatically configures a virtual network for containers. This network, by default, uses the `172.17.0.0/16` subnet. If your college's network or VPN uses an overlapping subnet, network traffic may be misrouted.

For example:
- Your college network uses `10.9.0.0/21`.
- Docker creates a route for `172.17.0.0/16`.

The conflicting route can prevent proper internet access.

**Additional Complexity**: Docker Compose creates custom networks (like `projectname_default`) that can also use the problematic `172.17.0.0/16` subnet, even if you've fixed the main `docker0` bridge.

---

## **Step 2: Diagnosing the Conflict**

### **1. Check Current Routes**
Run the following command to display your system's routing table:
```bash
ip route
```
Example output:
```
default via 10.9.0.1 dev eth0 proto dhcp metric 20102 
10.9.0.0/21 dev eth0 proto kernel scope link src 10.9.1.178 metric 102 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.17.0.0/16 dev br-01aafac6a93e proto kernel scope link src 172.17.0.1
```
Here, you might see multiple `172.17.0.0/16` routes - one for `docker0` and possibly others for custom Docker networks (bridges starting with `br-`).

### **2. Check All Docker Networks**
List all Docker networks to identify custom networks:
```bash
docker network ls
```
Look for networks with names like `projectname_default` - these are created by Docker Compose.

### **3. Check Docker Bridge Interfaces**
To see all Docker-managed network interfaces:
```bash
ip addr show | grep -E "docker0|br-"
```
Example output:
```
inet 192.168.100.1/24 brd 192.168.100.255 scope global docker0
inet 172.17.0.1/16 brd 172.17.255.255 scope global br-01aafac6a93e
```

---

## **Step 3: Temporary Fix**

### **Delete the Conflicting Routes**
If you need immediate internet access, you can temporarily delete all Docker routes using the problematic subnet:
```bash
sudo ip route del 172.17.0.0/16 dev docker0
sudo ip route del 172.17.0.0/16 dev br-01aafac6a93e  # Replace with actual bridge name
```
Check the routing table again to confirm:
```bash
ip route
```
Now, you should be able to access the internet. However, this fix is temporary, as Docker will recreate the routes on restart.

---

## **Step 4: Permanent Solution**

### **1. Edit Docker Configuration for Comprehensive Fix**
Docker's network settings are controlled by the file `/etc/docker/daemon.json`. This configuration will fix both the default bridge and prevent future custom networks from using conflicting subnets.

Open the file for editing:
```bash
sudo nano /etc/docker/daemon.json
```
Add the following configuration:
```json
{
    "bip": "192.168.100.1/24",
    "default-address-pools": [
        {
            "base": "192.168.0.0/16",
            "size": 24
        }
    ]
}
```
- **`bip`**: Specifies the Bridge IP for Docker's `docker0` network.
- **`default-address-pools`**: Controls the IP ranges used for all Docker networks, preventing them from using `172.17.0.0/16`.
- **`base`**: The base IP range for Docker to use (`192.168.0.0/16`).
- **`size`**: Subnet size (24 = /24 networks).

### **2. Remove Existing Conflicting Networks**
**Important**: The `default-address-pools` configuration only applies to **new networks**. Existing networks will keep their old configuration.

First, stop any running containers:
```bash
docker-compose down  # If using Docker Compose
# or
docker stop $(docker ps -q)  # Stop all running containers
```

Remove existing custom networks that use conflicting subnets:
```bash
docker network ls
docker network rm netowrk_name  # Replace with actual network name
```

### **3. Restart Docker**
Apply the changes by restarting Docker:
```bash
sudo systemctl restart docker
```

### **4. Recreate Your Applications**
If you were using Docker Compose, restart your applications:
```bash
docker-compose up -d
```
Docker will recreate the networks using your new configuration.

### **5. Verify the New Configuration**
Check that all Docker interfaces now use the new subnet ranges:
```bash
ip addr show | grep -E "docker0|br-"
```
Example output (all should show 192.168.x.x ranges):
```
inet 192.168.100.1/24 brd 192.168.100.255 scope global docker0
inet 192.168.1.1/24 brd 192.168.1.255 scope global br-01aafac6a93e
```

Check the routing table:
```bash
ip route
```
Example output:
```
default via 10.9.0.1 dev eth0 proto dhcp metric 20102 
10.9.0.0/21 dev eth0 proto kernel scope link src 10.9.1.178 metric 102
192.168.100.0/24 dev docker0 proto kernel scope link src 192.168.100.1
192.168.1.0/24 dev br-01aafac6a93e proto kernel scope link src 192.168.1.1
```

---

## **Step 5: Test Docker and Internet Access**

### **Test Internet Access**
Ensure you can access the internet:
```bash
ping google.com
```
Example output:
```
PING google.com (142.250.183.78) 56(84) bytes of data.
64 bytes from bom07s27-in-f14.1e100.net (142.250.183.78): icmp_seq=1 ttl=118 time=5.02 ms
```

### **Test Docker Functionality**
Run a simple Docker container to ensure everything works:
```bash
docker run hello-world
```
Example output:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

### **Test Docker Compose Applications**
If using Docker Compose, verify your applications work correctly:
```bash
docker-compose ps
```

---

## **Troubleshooting Tips**
- **Invalid JSON Configuration**: If Docker fails to start after editing `/etc/docker/daemon.json`, check for syntax errors using:
  ```bash
  sudo systemctl status docker
  ```
- **Still Unable to Access Internet**: 
  - Check for any remaining `172.17.0.0/16` routes: `ip route | grep 172.17`
  - Verify all Docker bridges are using new subnets: `ip addr show | grep -E "docker0|br-"`
- **VPN Conflicts**: If using a VPN, ensure its subnet does not overlap with Docker's new subnet ranges.
- **Persistent Network Issues**: Some Docker networks are persistent and survive daemon restarts. Use `docker network ls` and `docker network rm <network_name>` to remove problematic networks.
- **Network Recreation Issues**: If a network won't recreate with new settings, try:
  ```bash
  docker system prune -a --volumes  # WARNING: This removes all unused Docker resources
  ```

---

## **Understanding Docker Network Types**

### **Default Bridge (`docker0`)**
- Created automatically when Docker starts
- Controlled by the `bip` setting in daemon.json
- Used by containers run with `docker run` (without `--network` flag)

### **Custom Networks (Docker Compose)**
- Created by Docker Compose for each project
- Named like `projectname_default`
- Create bridge interfaces like `br-xxxxxxxx`
- Controlled by the `default-address-pools` setting in daemon.json
- **Persistent**: Survive container and daemon restarts

### **User-Defined Networks**
- Created with `docker network create`
- Also controlled by `default-address-pools`
- Can be configured with specific subnets

---

## **Prevention Strategy**

To prevent this issue from recurring:

1. **Always use both `bip` and `default-address-pools`** in your daemon.json
2. **Regularly audit your Docker networks**: `docker network ls`
3. **Remove unused networks**: `docker network prune`
4. **When starting new projects**, verify they don't create conflicting networks
5. **Document your network configuration** for team members

---

## **Conclusion**
By reconfiguring both Docker's default network bridge and the address pools used for custom networks, you can resolve routing conflicts with your college or VPN network. The key insight is that Docker creates multiple types of networks, and a comprehensive solution must address all of them.

The `default-address-pools` configuration is crucial for preventing Docker Compose and other tools from recreating the same conflicts with custom networks.

If you encounter further issues, feel free to revisit the diagnostic steps or modify the Docker configuration as needed.
