# Resolving Docker Network Conflict with College or VPN Network

## **Problem Overview**
When Docker is installed, it creates a default network bridge (`docker0`) with the subnet `172.17.0.0/16`. If this subnet overlaps with your college network or VPN, it can cause routing conflicts, making it impossible to access the internet or sign in to network-dependent services.

This document explains how to identify the issue, diagnose the conflict, and resolve it by changing Docker’s default bridge network.

---

## **Step 1: Understanding the Problem**
When Docker starts, it automatically configures a virtual network for containers. This network, by default, uses the `172.17.0.0/16` subnet. If your college’s network or VPN uses an overlapping subnet, network traffic may be misrouted.

For example:
- Your college network uses `10.9.0.0/21`.
- Docker creates a route for `172.17.0.0/16`.

The conflicting route can prevent proper internet access.

---

## **Step 2: Diagnosing the Conflict**

### **1. Check Current Routes**
Run the following command to display your system’s routing table:
```bash
ip route
```
Example output:
```
default via 10.9.0.1 dev eth0 proto dhcp metric 20102 
10.9.0.0/21 dev eth0 proto kernel scope link src 10.9.1.178 metric 102 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
```
Here, the route `172.17.0.0/16` corresponds to Docker’s `docker0` interface. If this overlaps with your network’s range, it will cause a problem.

### **2. Check Docker’s `docker0` Subnet**
To confirm Docker’s default subnet, run:
```bash
ip addr show docker0
```
Example output:
```
inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
```
This shows that Docker is using `172.17.0.0/16`.

---

## **Step 3: Temporary Fix**

### **Delete the Conflicting Route**
If you need immediate internet access, you can temporarily delete the Docker route:
```bash
sudo ip route del 172.17.0.0/16
```
Check the routing table again to confirm:
```bash
ip route
```
Example output:
```
default via 10.9.0.1 dev eth0 proto dhcp metric 20102 
10.9.0.0/21 dev eth0 proto kernel scope link src 10.9.1.178 metric 102
```
Now, you should be able to access the internet. However, this fix is temporary, as Docker will recreate the route on restart.

---

## **Step 4: Permanent Solution**
To permanently resolve the issue, reconfigure Docker to use a non-conflicting subnet.

### **1. Edit Docker Configuration**
Docker’s default network settings are controlled by the file `/etc/docker/daemon.json`. If the file does not exist, you can create it.

Open the file for editing:
```bash
sudo nano /etc/docker/daemon.json
```
Add the following configuration:
```json
{
    "bip": "192.168.100.1/24"
}
```
- **`bip`**: Specifies the Bridge IP for Docker’s `docker0` network.
- **`192.168.100.1/24`**: A private subnet unlikely to conflict with other networks.

### **2. Restart Docker**
Apply the changes by restarting Docker:
```bash
sudo systemctl restart docker
```

### **3. Verify the New Subnet**
Check the `docker0` interface to ensure it uses the new subnet:
```bash
ip addr show docker0
```
Example output:
```
inet 192.168.100.1/24 brd 192.168.100.255 scope global docker0
```

### **4. Confirm Routing Table**
Verify that the routing table no longer includes the conflicting subnet:
```bash
ip route
```
Example output:
```
default via 10.9.0.1 dev eth0 proto dhcp metric 20102 
10.9.0.0/21 dev eth0 proto kernel scope link src 10.9.1.178 metric 102
192.168.100.0/24 dev docker0 proto kernel scope link src 192.168.100.1
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

---

## **Troubleshooting Tips**
- **Invalid JSON Configuration**: If Docker fails to start after editing `/etc/docker/daemon.json`, check for syntax errors using:
  ```bash
  sudo systemctl status docker
  ```
- **Still Unable to Access Internet**: Verify there are no other conflicting routes in the output of `ip route`.
- **VPN Conflicts**: If using a VPN, ensure its subnet does not overlap with Docker’s new subnet.

---

## **Conclusion**
By reconfiguring Docker’s default network bridge to use a non-conflicting subnet, you can resolve routing conflicts with your college or VPN network. This ensures seamless internet access and Docker functionality.

If you encounter further issues, feel free to revisit the diagnostic steps or modify the Docker configuration as needed.
