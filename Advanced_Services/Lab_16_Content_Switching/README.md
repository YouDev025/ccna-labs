# Lab Content Switching

## 📌 Objective

The objective of this lab is to understand and practice **Content Switching**, a technique used to distribute client requests across multiple backend servers based on application-level information.

In this lab, you will:

* Build a network topology with clients, a content-switching device, and multiple backend servers.
* Configure IP addressing and network connectivity.
* Configure a virtual service or virtual server.
* Configure backend servers as real servers.
* Apply traffic distribution rules.
* Verify that client requests are correctly forwarded to the appropriate backend servers.
* Test connectivity and service availability.

---

## 🧠 Lab Overview

**Content Switching** is a Layer 7 traffic-management technique that allows a network device or load balancer to make forwarding decisions based on application-level information.

For example, a content switch can inspect:

* HTTP requests
* Destination URL
* Hostname
* HTTP headers
* Application type
* Cookies
* TCP/UDP ports

Based on these parameters, the device can forward traffic to different backend servers.

### Example

A company may have:

```text
                    Internet / Client
                           |
                           |
                    +--------------+
                    |    Content   |
                    |    Switch    |
                    +--------------+
                       /          \
                      /            \
                     /              \
              +----------+     +----------+
              | Server 1 |     | Server 2 |
              |   HTTP   |     |   HTTP   |
              +----------+     +----------+
```

For example:

```text
www.example.com/app1  → Server 1
www.example.com/app2  → Server 2
```

---

# 🏗️ Topology

The lab should contain at least:

* 1 Client
* 1 Content Switch / Load Balancer
* 2 Backend Servers
* 1 or more switches
* Optional router/ISP connection

Example topology:

```text
                       CLIENT
                     192.168.10.10
                           |
                           |
                    +-------------+
                    |    SW1      |
                    +-------------+
                           |
                           |
                 +-------------------+
                 |  Content Switch   |
                 |   VIP: 10.0.0.100 |
                 +-------------------+
                     /             \
                    /               \
                   /                 \
          +---------------+   +---------------+
          |   Web Server 1|   |   Web Server 2|
          |  10.0.0.11    |   |  10.0.0.12    |
          |     HTTP      |   |      HTTP     |
          +---------------+   +---------------+
```

---

# 🌐 IP Addressing Plan

| Device         | Interface   | IP Address    | Mask | Role            |
| -------------- | ----------- | ------------- | ---- | --------------- |
| Client         | NIC         | 192.168.10.10 | /24  | Client          |
| Content Switch | Client-side | 192.168.10.1  | /24  | Client Gateway  |
| Content Switch | Server-side | 10.0.0.1      | /24  | Server Gateway  |
| Web Server 1   | NIC         | 10.0.0.11     | /24  | Backend Server  |
| Web Server 2   | NIC         | 10.0.0.12     | /24  | Backend Server  |
| Virtual IP     | VIP         | 10.0.0.100    | /24  | Virtual Service |

> **Note:** The exact addressing depends on the platform used for the lab.

---

# 🔧 Tasks

## Task 1 — Build the Topology

Create the topology according to the following logical structure:

```text
Client
   |
   |
Switch
   |
Content Switch
  / \
 /   \
Server1 Server2
```

Connect all devices using the appropriate Ethernet links.

Verify that all physical interfaces are operational.

---

# Task 2 — Configure IP Addressing

Configure the client with:

```text
IP Address:      192.168.10.10
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.10.1
```

Configure Web Server 1:

```text
IP Address:      10.0.0.11
Subnet Mask:     255.255.255.0
Default Gateway: 10.0.0.1
```

Configure Web Server 2:

```text
IP Address:      10.0.0.12
Subnet Mask:     255.255.255.0
Default Gateway: 10.0.0.1
```

---

# Task 3 — Configure the Backend Servers

Configure both servers to provide an HTTP service.

### Server 1

```text
Server Name: WEB-SERVER-01
IP Address: 10.0.0.11
Service: HTTP
Port: 80
```

### Server 2

```text
Server Name: WEB-SERVER-02
IP Address: 10.0.0.12
Service: HTTP
Port: 80
```

Use different web pages to identify the server receiving the request.

Example:

```html
<h1>Welcome to Web Server 1</h1>
```

Server 2:

```html
<h1>Welcome to Web Server 2</h1>
```

This makes it easier to verify traffic distribution.

---

# Task 4 — Configure the Content Switch

The content switch must provide a **virtual service** for clients.

Create a virtual IP:

```text
VIP: 10.0.0.100
Protocol: HTTP
Port: 80
```

The client should access:

```text
http://10.0.0.100
```

The client must not need to know the real IP addresses of the backend servers.

The content switch should receive the request and select an appropriate backend server.

---

# Task 5 — Configure Real Servers

Add the backend servers to the server pool.

Example:

```text
Server Pool: WEB-POOL

Real Server 1:
10.0.0.11:80

Real Server 2:
10.0.0.12:80
```

Logical representation:

```text
                 VIP
              10.0.0.100
                   |
                   |
          +----------------+
          | Content Switch |
          +----------------+
             /          \
            /            \
           ↓              ↓
     10.0.0.11        10.0.0.12
       HTTP              HTTP
```

---

# Task 6 — Configure Traffic Distribution

Configure a load-balancing method.

Possible algorithms include:

### Round Robin

Requests are distributed sequentially:

```text
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 1
Request 4 → Server 2
```

### Least Connections

Traffic is sent to the server with the smallest number of active connections.

```text
Server 1 → 10 connections
Server 2 → 3 connections

New request → Server 2
```

### Source IP

The client's source IP is used to determine the backend server.

---

# Task 7 — Configure Content-Based Rules

If the platform supports Layer 7 content switching, configure rules based on HTTP information.

Example:

```text
URL: /app1
      ↓
Server Pool 1
      ↓
10.0.0.11
```

and:

```text
URL: /app2
      ↓
Server Pool 2
      ↓
10.0.0.12
```

Logical flow:

```text
Client
   |
   | HTTP Request
   |
   ↓
Content Switch
   |
   +---- /app1 ----> Web Server 1
   |
   +---- /app2 ----> Web Server 2
```

---

# Task 8 — Configure Health Checks

The content switch should verify that backend servers are available.

Example health check:

```text
Protocol: HTTP
Port: 80
Interval: 5 seconds
Timeout: 2 seconds
```

The content switch periodically checks:

```text
Content Switch
      |
      +---- HTTP Check ----> Server 1
      |
      +---- HTTP Check ----> Server 2
```

If Server 1 becomes unavailable:

```text
Server 1 → DOWN
Server 2 → UP
```

The content switch should stop forwarding new requests to Server 1.

---

# 🔍 Task 9 — Verify Basic Connectivity

From the client, test connectivity with:

```bash
ping 192.168.10.1
```

Then test the backend servers if routing permits:

```bash
ping 10.0.0.11
ping 10.0.0.12
```

Expected result:

```text
Reply from 10.0.0.11
Reply from 10.0.0.12
```

---

# 🌐 Task 10 — Test HTTP Connectivity

From the client, access the virtual IP:

```text
http://10.0.0.100
```

Or use:

```bash
curl http://10.0.0.100
```

Expected behavior:

```text
Client
   |
   | HTTP request
   ↓
10.0.0.100
   |
   ↓
Content Switch
   |
   ↓
Web Server 1 / Web Server 2
```

---

# 🔄 Task 11 — Verify Load Balancing

Send multiple requests:

```bash
curl http://10.0.0.100
curl http://10.0.0.100
curl http://10.0.0.100
curl http://10.0.0.100
```

With Round Robin, the result could be:

```text
Request 1 → WEB-SERVER-01
Request 2 → WEB-SERVER-02
Request 3 → WEB-SERVER-01
Request 4 → WEB-SERVER-02
```

---

# 🛑 Task 12 — Test Server Failure

Stop the HTTP service on Web Server 1.

For example:

```text
WEB-SERVER-01 → HTTP OFF
```

Wait for the health check to detect the failure.

The content switch should show:

```text
WEB-SERVER-01 → DOWN
WEB-SERVER-02 → UP
```

Then test again:

```bash
curl http://10.0.0.100
```

The request should be forwarded to:

```text
WEB-SERVER-02
```

---

# 🔎 Verification Commands

Depending on the platform, use appropriate commands to inspect the configuration.

### Interface Status

```text
show ip interface brief
```

### Routing Table

```text
show ip route
```

### ARP Table

```text
show arp
```

### Running Configuration

```text
show running-config
```

### MAC Address Table

```text
show mac address-table
```

If supported by the content-switching platform, also verify:

```text
show virtual-server
show server
show pool
show health
show statistics
show connections
```

> Commands may vary depending on whether the lab uses Cisco IOS, an application delivery controller, or another load-balancing platform.

---

# 🧪 Verification Checklist

| Test                      | Expected Result    |
| ------------------------- | ------------------ |
| Client → Gateway          | ✅ Successful       |
| Client → Server 1         | ✅ Successful       |
| Client → Server 2         | ✅ Successful       |
| Client → VIP              | ✅ Successful       |
| HTTP service Server 1     | ✅ UP               |
| HTTP service Server 2     | ✅ UP               |
| Content Switch → Server 1 | ✅ UP               |
| Content Switch → Server 2 | ✅ UP               |
| Load balancing            | ✅ Working          |
| Server 1 failure          | ✅ Detected         |
| Traffic after failure     | ✅ Sent to Server 2 |

---

# 🧪 Troubleshooting

## Problem 1 — Client cannot reach the VIP

Check:

```text
IP addressing
Default gateway
Interface status
Routing
Virtual IP configuration
```

---

## Problem 2 — VIP is reachable but HTTP fails

Check:

```text
HTTP service
TCP port 80
Backend server status
Server pool configuration
Health checks
```

---

## Problem 3 — Only one server receives traffic

Check:

```text
Load-balancing algorithm
Server pool configuration
Health-check status
Session persistence
```

---

## Problem 4 — Server is marked DOWN

Check:

```text
Server IP address
HTTP service
TCP port 80
Routing
Firewall rules
Health-check configuration
```

---

# 📊 Expected Final Architecture

```text
                         CLIENT
                     192.168.10.10
                            |
                            |
                       HTTP : 80
                            |
                            ↓
                   +----------------+
                   | Content Switch |
                   |                |
                   | VIP            |
                   | 10.0.0.100     |
                   +----------------+
                       /        \
                      /          \
                     ↓            ↓
              +-----------+  +-----------+
              | Web       |  | Web       |
              | Server 1  |  | Server 2  |
              |10.0.0.11  |  |10.0.0.12  |
              | HTTP :80  |  | HTTP :80  |
              +-----------+  +-----------+
```

---

# 🎯 Learning Outcomes

After completing this lab, you should be able to:

* Explain the purpose of Content Switching.
* Understand the difference between a **VIP** and a **real server**.
* Configure backend server pools.
* Configure HTTP virtual services.
* Understand Layer 7 traffic distribution.
* Configure and understand health checks.
* Test load-balancing behavior.
* Detect backend server failures.
* Verify network and application connectivity.
* Use diagnostic commands to troubleshoot Content Switching.

---

# 📝 Key Concepts to Remember

```text
Client
   ↓
Virtual IP (VIP)
   ↓
Content Switch / Load Balancer
   ↓
Server Pool
   ↓
Real Servers
```

### Important terms

| Term              | Meaning                                                   |
| ----------------- | --------------------------------------------------------- |
| VIP               | Virtual IP exposed to clients                             |
| Real Server       | Actual backend server                                     |
| Server Pool       | Group of backend servers                                  |
| Health Check      | Test used to verify server availability                   |
| Load Balancing    | Distribution of requests between servers                  |
| Content Switching | Forwarding traffic according to application content       |
| Layer 7           | Application layer where HTTP information can be inspected |
| Persistence       | Keeping a client associated with the same server          |

---

# ✅ Final Verification

Before completing the lab, verify:

```text
[✓] Topology built
[✓] IP addresses configured
[✓] Interfaces operational
[✓] Backend servers configured
[✓] HTTP services enabled
[✓] Server pool configured
[✓] VIP configured
[✓] Health checks configured
[✓] Load balancing tested
[✓] Server failure tested
[✓] show commands executed
[✓] Connectivity verified
```

---

# 🏁 Conclusion

This lab demonstrates how **Content Switching** can be used to intelligently distribute application traffic between multiple backend servers.

The main traffic flow is:

```text
Client
   ↓
Virtual IP
   ↓
Content Switch
   ↓
Content-Based Decision
   ↓
Server Pool
   ↓
Backend Server
```

The use of health checks and load-balancing algorithms improves **availability, scalability, and service reliability**.
