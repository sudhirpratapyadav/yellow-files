# Resolving Docker Network and College Network Route Conflict

When Docker is installed, it creates a default bridge network (`docker0`) with an IP range that might conflict with your college or workplace network, causing internet connectivity issues. This guide explains how to diagnose and resolve the conflict step by step.

---

## **Step 1: Understanding the Issue**

Docker's default bridge network (`docker0`) often uses the IP range `172.17.0.0/16`. If your local network overlaps with this range, routing conflicts can occur, disrupting your internet connectivity.

For example, after installing Docker, you might lose access to your college network or be unable to log in to services.

---

## **Step 2: Diagnosing the Conflict**

### **1. Check Current Routes**
Run the following command to display your system’s routing table:

~~~bash
ip route
~~~

**Example output:**
~~~
default via 10.9.0.1 dev eth0 proto dhcp metric 20102
10.9.0.0/21 dev eth0 proto kernel scope link src 10.9.1.178 metric 102
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
~~~

Here, the route `172.17.0.0/16` corresponds to Docker’s `docker0` interface. If your college network uses a conflicting IP range, this route will cause issues.

### **2. Verify Docker Bridge Network**
Run the following command to check the IP range used by the Docker bridge network:

~~~bash
ip a | grep 172.17
~~~

**Example output:**
~~~
inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
~~~

This confirms that Docker’s `docker0` bridge network is using the conflicting range `172.17.0.0/16`.

---

## **Step 3: Resolving the Conflict**

To resolve the issue, we’ll change Docker’s default bridge network to a different IP range that doesn’t conflict with your local network.

### **1. Stop Docker**
Before making changes, stop the Docker service:

~~~bash
sudo systemctl stop docker
~~~

### **2. Configure a New Bridge IP Range**
Create or edit the Docker daemon configuration file at `/etc/docker/daemon.json`. Use a text editor like `nano`:

~~~bash
sudo nano /etc/docker/daemon.json
~~~

Add the following content to specify a new IP range (e.g., `192.168.100.1/24`):

~~~json
{
  "bip": "192.168.100.1/24"
}
~~~

### **3. Restart Docker**
After saving the changes, restart Docker:

~~~bash
sudo systemctl start docker
~~~

### **4. Verify Changes**
Check the updated bridge network configuration:

~~~bash
ip route
~~~

**Example output:**
~~~
default via 10.9.0.1 dev eth0 proto dhcp metric 20102
10.9.0.0/21 dev eth0 proto kernel scope link src 10.9.1.178 metric 102
192.168.100.0/24 dev docker0 proto kernel scope link src 192.168.100.1
~~~

Now, Docker uses the `192.168.100.0/24` range, avoiding conflicts with your local network.

### **5. Test Internet Connectivity**
Try accessing the internet or logging into your college network to confirm that the issue is resolved.

---

## **Example Workflow**

Here’s a complete workflow to diagnose and solve the problem:

1. **Diagnose:**
    - Run `ip route` to identify conflicting routes.
    - Run `ip a | grep 172.17` to verify Docker’s IP range.

2. **Resolve:**
    - Stop Docker: `sudo systemctl stop docker`.
    - Edit `/etc/docker/daemon.json` to specify a new IP range.
    - Restart Docker: `sudo systemctl start docker`.
    - Verify: Run `ip route` to confirm changes.

3. **Test:**
    - Ensure internet connectivity is restored.

---

## **Key Commands Explained**

- **`ip route`**: Displays the routing table to identify conflicting routes.
- **`ip a`**: Shows network interfaces and their IP addresses.
- **`sudo systemctl stop/start docker`**: Stops or starts the Docker service.
- **`nano`**: A simple text editor to modify configuration files.

---

## **Conclusion**
Changing Docker’s default bridge network IP range is a simple yet effective way to resolve network conflicts. By following this guide, you can ensure uninterrupted internet connectivity while using Docker on networks with overlapping IP ranges.

---
