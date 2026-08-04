# Lab Load Balancing

## 📌 Objective

The objective of this lab is to understand and practice the fundamentals of **Load Balancing** in a network and application infrastructure environment.

In this lab, you will:

* Build a network topology with clients, a load balancer, and multiple backend servers.
* Configure IP addressing and connectivity.
* Configure backend web servers.
* Create a server pool.
* Configure a Virtual IP (VIP).
* Configure a load-balancing algorithm.
* Test traffic distribution between servers.
* Configure health checks.
* Simulate server failure.
* Verify that traffic is automatically redirected to available servers.
* Use diagnostic commands and test tools to verify the configuration.

---

# 🧠 Lab Overview

**Load Balancing** is a technique used to distribute network or application traffic across multiple servers.

Instead of sending all client requests to a single server:

```text
             Client
                |
                ↓
             Server
```

a load balancer distributes requests between several backend servers:

```text
                    Client
                       |
                       ↓
                +-------------+
                | Load        |
                | Balancer    |
                +-------------+
                  /    |    \
                 /     |     \
                ↓      ↓      ↓
            Server1 Server2 Server3
```

The main objectives are:

* **Availability**
* **Scalability**
* **Performance**
* **Fault tolerance**
* **Service continuity**

---

# 🏗️ Topology

The recommended topology contains:

* 1 Client
* 1 Switch
* 1 Load Balancer
* 2 or more Web Servers

Example:

```text
                         CLIENT
                      192.168.10.10
                            |
                            |
                         +-----+
                         | SW1 |
                         +-----+
                            |
                            |
                    +---------------+
                    | Load Balancer |
                    |               |
                    | VIP           |
                    | 192.168.20.100|
                    +---------------+
                       /          \
                      /            \
                     ↓              ↓
             +-------------+  +-------------+
             | Web Server 1|  | Web Server 2|
             |192.168.20.11|  |192.168.20.12|
             |   HTTP :80  |  |   HTTP :80  |
             +-------------+  +-------------+
```

---

# 🌐 IP Addressing Plan

| Device        | Interface   | IP Address     | Mask | Role            |
| ------------- | ----------- | -------------- | ---- | --------------- |
| Client        | NIC         | 192.168.10.10  | /24  | Client          |
| Load Balancer | Client-side | 192.168.10.1   | /24  | Client Gateway  |
| Load Balancer | Server-side | 192.168.20.1   | /24  | Server Gateway  |
| Web Server 1  | NIC         | 192.168.20.11  | /24  | Backend Server  |
| Web Server 2  | NIC         | 192.168.20.12  | /24  | Backend Server  |
| Load Balancer | VIP         | 192.168.10.100 | /24  | Virtual Service |

> **Note:** The exact IP addressing depends on the platform used for the lab.

---

# 🔑 Important Concepts

## 1. Client

The client generates requests toward the application.

Example:

```text
Client
192.168.10.10
```

---

## 2. Load Balancer

The load balancer receives client requests and decides which backend server should handle each request.

```text
Client
   |
   ↓
Load Balancer
   |
   +----→ Server 1
   |
   +----→ Server 2
```

---

## 3. Virtual IP — VIP

The **Virtual IP (VIP)** is the address exposed to clients.

Example:

```text
VIP = 192.168.10.100
```

The client accesses:

```text
http://192.168.10.100
```

The client does not directly access:

```text
192.168.20.11
192.168.20.12
```

---

## 4. Real Servers

The real servers are the backend systems that actually process the requests.

Example:

```text
Server 1 → 192.168.20.11
Server 2 → 192.168.20.12
```

---

## 5. Server Pool

A **server pool** is a group of backend servers.

```text
WEB-POOL
│
├── 192.168.20.11:80
└── 192.168.20.12:80
```

---

# 🔧 Tasks

## Task 1 — Build the Topology

Create the following logical topology:

```text
Client
   |
   |
 Switch
   |
   |
Load Balancer
   |
   +----------+
   |          |
   ↓          ↓
Server 1    Server 2
```

Connect all devices and verify that the links are operational.

---

# Task 2 — Configure the Client

Configure the client:

```text
IP Address:      192.168.10.10
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.10.1
```

Test connectivity:

```bash
ping 192.168.10.1
```

Expected:

```text
Reply from 192.168.10.1
```

---

# Task 3 — Configure Web Server 1

Configure:

```text
IP Address:      192.168.20.11
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.20.1
```

Enable HTTP:

```text
Protocol: HTTP
Port: 80
```

Create a simple page identifying the server:

```html
<h1>Welcome to Web Server 1</h1>
```

---

# Task 4 — Configure Web Server 2

Configure:

```text
IP Address:      192.168.20.12
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.20.1
```

Enable HTTP:

```text
Protocol: HTTP
Port: 80
```

Create a different page:

```html
<h1>Welcome to Web Server 2</h1>
```

Using different pages allows you to determine which backend server processed the request.

---

# Task 5 — Configure the Load Balancer

Configure the load balancer with two network interfaces.

### Client-facing interface

```text
IP Address: 192.168.10.1
Mask: 255.255.255.0
```

### Server-facing interface

```text
IP Address: 192.168.20.1
Mask: 255.255.255.0
```

The logical architecture is:

```text
               Load Balancer
              /              \
             /                \
    192.168.10.1         192.168.20.1
          |                    |
       Clients             Servers
```

---

# Task 6 — Create the Virtual IP

Configure a virtual service:

```text
Virtual IP: 192.168.10.100
Protocol:   HTTP
Port:       80
```

The client will access:

```text
http://192.168.10.100
```

The traffic flow becomes:

```text
Client
192.168.10.10
      |
      | HTTP :80
      ↓
VIP 192.168.10.100
      |
      ↓
Load Balancer
      |
      +--------→ Web Server 1
      |
      +--------→ Web Server 2
```

---

# Task 7 — Create the Backend Server Pool

Create a pool called:

```text
WEB-POOL
```

Add:

```text
Server 1:
192.168.20.11:80

Server 2:
192.168.20.12:80
```

Logical configuration:

```text
WEB-POOL
│
├── WEB-SERVER-01
│   └── 192.168.20.11:80
│
└── WEB-SERVER-02
    └── 192.168.20.12:80
```

---

# Task 8 — Configure Load-Balancing Algorithm

The load balancer needs an algorithm to determine which server receives each request.

---

## Round Robin

Requests are distributed sequentially:

```text
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 1
Request 4 → Server 2
```

Example:

```text
Client
  |
  +---- Request 1 ----→ Server 1
  |
  +---- Request 2 ----→ Server 2
  |
  +---- Request 3 ----→ Server 1
  |
  +---- Request 4 ----→ Server 2
```

Round Robin is a simple and commonly used algorithm when servers have similar capacities.

---

## Least Connections

The load balancer sends new traffic to the server with fewer active connections.

Example:

```text
Server 1 → 10 connections
Server 2 → 3 connections
```

New request:

```text
New Request → Server 2
```

---

## Weighted Load Balancing

Different servers can receive different amounts of traffic.

Example:

```text
Server 1 → Weight 2
Server 2 → Weight 1
```

Server 1 will receive approximately twice as much traffic as Server 2.

---

# Task 9 — Configure Health Checks

A load balancer must verify that backend servers are available.

Configure an HTTP health check:

```text
Protocol: HTTP
Port: 80
Interval: 5 seconds
Timeout: 2 seconds
```

Logical behavior:

```text
Load Balancer
      |
      +---- Health Check ----→ Server 1
      |
      +---- Health Check ----→ Server 2
```

Expected status:

```text
Server 1 → UP
Server 2 → UP
```

---

# Task 10 — Test Basic Connectivity

From the client:

```bash
ping 192.168.10.1
```

Then test the backend network if routing allows:

```bash
ping 192.168.20.11
ping 192.168.20.12
```

Expected:

```text
SUCCESS
```

---

# Task 11 — Test Web Server 1

From the client, access:

```text
http://192.168.20.11
```

Expected:

```text
Welcome to Web Server 1
```

---

# Task 12 — Test Web Server 2

Access:

```text
http://192.168.20.12
```

Expected:

```text
Welcome to Web Server 2
```

This confirms that both backend servers are operational.

---

# Task 13 — Test the Virtual IP

Now access the VIP:

```text
http://192.168.10.100
```

The request should be processed by one of the backend servers.

Expected:

```text
Client
   |
   ↓
192.168.10.100
   |
   ↓
Load Balancer
   |
   ↓
Web Server 1 OR Web Server 2
```

---

# Task 14 — Verify Load Distribution

Send multiple requests to the VIP.

Using `curl`:

```bash
curl http://192.168.10.100
curl http://192.168.10.100
curl http://192.168.10.100
curl http://192.168.10.100
```

With Round Robin, the expected behavior is approximately:

```text
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 1
Request 4 → Server 2
```

The exact result may depend on session persistence and the load-balancer implementation.

---

# Task 15 — Test Server Failure

Stop the HTTP service on Web Server 1.

Example:

```text
WEB-SERVER-01
HTTP → OFF
```

Wait for the health check.

Expected status:

```text
Server 1 → DOWN
Server 2 → UP
```

The load balancer should remove Server 1 from the active pool.

---

# Task 16 — Verify Failover

Access the VIP again:

```text
http://192.168.10.100
```

Expected:

```text
Client
   |
   ↓
VIP
   |
   ↓
Load Balancer
   |
   X---- Server 1 (DOWN)
   |
   +----→ Server 2 (UP)
```

Traffic should continue to Server 2.

This demonstrates **fault tolerance**.

---

# Task 17 — Restore Server 1

Turn HTTP back on:

```text
WEB-SERVER-01
HTTP → ON
```

Wait for the health check.

Expected:

```text
Server 1 → UP
Server 2 → UP
```

Server 1 should automatically become available again.

---

# 🔍 Verification Commands

The exact commands depend on the load-balancing platform.

Useful commands may include:

```text
show running-config
show ip interface brief
show ip route
show arp
show mac address-table
show interfaces
```

Load-balancer-specific commands may include:

```text
show pool
show pool members
show virtual-server
show health
show statistics
show connections
show sessions
```

---

# 📊 Verification Table

| Test                   | Expected Result |
| ---------------------- | --------------- |
| Client → Load Balancer | ✅ Successful    |
| Client → Server 1      | ✅ Successful    |
| Client → Server 2      | ✅ Successful    |
| Server 1 HTTP          | ✅ UP            |
| Server 2 HTTP          | ✅ UP            |
| VIP HTTP               | ✅ Successful    |
| Server Pool            | ✅ Configured    |
| Health Checks          | ✅ UP            |
| Load Balancing         | ✅ Working       |
| Server 1 Failure       | ✅ Detected      |
| Traffic after failure  | ✅ Server 2      |
| Server 1 Recovery      | ✅ Detected      |
| Traffic Distribution   | ✅ Working       |

---

# 🧪 Testing with Curl

If the client has access to `curl`, use:

```bash
curl -i http://192.168.10.100
```

To generate multiple requests:

```bash
for i in {1..10}; do
    curl -s http://192.168.10.100
    echo
done
```

This can help observe traffic distribution.

---

# 📈 Load-Balancing Flow

The complete request flow is:

```text
                     Client
                        |
                        |
                   HTTP Request
                        |
                        ↓
                +---------------+
                | Load Balancer |
                +---------------+
                        |
                  Virtual IP
                  192.168.10.100
                        |
                        ↓
                  Server Pool
                   /         \
                  /           \
                 ↓             ↓
          Web Server 1    Web Server 2
          192.168.20.11   192.168.20.12
```

---

# 🛡️ Health Check Flow

```text
                Load Balancer
                     |
            +--------+--------+
            |                 |
            ↓                 ↓
       Health Check      Health Check
            |                 |
            ↓                 ↓
        Server 1          Server 2
           UP                UP
```

If Server 1 fails:

```text
                Load Balancer
                     |
            +--------+--------+
            |                 |
            ↓                 ↓
        Server 1          Server 2
           DOWN               UP
            X                 ✓
            |                 |
            +------→ Traffic →+
```

---

# 🚨 Failure Scenario

Before failure:

```text
Server 1 → UP
Server 2 → UP
```

Traffic:

```text
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 1
Request 4 → Server 2
```

After Server 1 fails:

```text
Server 1 → DOWN
Server 2 → UP
```

Traffic:

```text
Request 5 → Server 2
Request 6 → Server 2
Request 7 → Server 2
Request 8 → Server 2
```

After Server 1 recovers:

```text
Server 1 → UP
Server 2 → UP
```

Load balancing resumes.

---

# 🛠️ Troubleshooting

## Problem 1 — VIP is unreachable

Check:

```text
IP addressing
Interface status
Routing
VIP configuration
Client gateway
```

Use:

```text
show ip interface brief
show ip route
```

---

## Problem 2 — Backend server is DOWN

Check:

```text
Server IP
HTTP service
TCP port 80
Gateway
Health check
Firewall rules
```

Test:

```bash
ping 192.168.20.11
ping 192.168.20.12
```

---

## Problem 3 — Only one server receives traffic

Check:

```text
Load-balancing algorithm
Server pool
Health-check status
Session persistence
Server availability
```

---

## Problem 4 — Traffic stops when one server fails

Check:

```text
Health checks
Failover configuration
Server pool
Virtual service
```

The load balancer should automatically remove unavailable servers.

---

## Problem 5 — Client can access servers but not VIP

Check:

```text
VIP configuration
Virtual service
Load-balancing policy
NAT if required
Routing
```

---

# 🧠 Important Concepts

| Concept                 | Meaning                                 |
| ----------------------- | --------------------------------------- |
| Load Balancer           | Distributes traffic between servers     |
| VIP                     | Virtual IP exposed to clients           |
| Real Server             | Backend server                          |
| Server Pool             | Group of backend servers                |
| Health Check            | Verifies server availability            |
| Round Robin             | Sequential distribution                 |
| Least Connections       | Sends traffic to the least busy server  |
| Weighted Load Balancing | Distribution based on server capacity   |
| Failover                | Traffic continues after server failure  |
| Persistence             | Keeps a client associated with a server |
| Backend                 | Server-side infrastructure              |
| Frontend                | Client-facing service                   |

---

# 🎯 Learning Outcomes

After completing this lab, you should be able to:

* Explain the purpose of load balancing.
* Understand the role of a load balancer.
* Understand VIPs and real servers.
* Configure a backend server pool.
* Configure a virtual service.
* Configure load-balancing algorithms.
* Configure health checks.
* Test HTTP traffic.
* Verify traffic distribution.
* Simulate backend server failure.
* Understand automatic failover.
* Troubleshoot load-balancing problems.

---

# 📌 Key Concepts to Memorize

### Load Balancing

```text
Load Balancing = Distribution of traffic across multiple servers.
```

### VIP

```text
VIP = Virtual IP used by clients to access the service.
```

### Server Pool

```text
Server Pool = Group of backend servers providing the same service.
```

### Health Check

```text
Health Check = Mechanism used to verify server availability.
```

### Failover

```text
Failover = Redirecting traffic to available servers when one server fails.
```

---

# 📝 Final Architecture

```text
                         CLIENT
                      192.168.10.10
                            |
                            |
                         HTTP :80
                            |
                            ↓
                  +-------------------+
                  |   Virtual IP      |
                  | 192.168.10.100    |
                  +-------------------+
                            |
                            ↓
                  +-------------------+
                  |   LOAD BALANCER   |
                  +-------------------+
                            |
                       SERVER POOL
                       /          \
                      /            \
                     ↓              ↓
              +-----------+   +-----------+
              | Server 1  |   | Server 2  |
              | .20.11    |   | .20.12    |
              | HTTP :80  |   | HTTP :80  |
              +-----------+   +-----------+
```

---

# ✅ Final Verification

Before completing the lab:

```text
[✓] Topology built
[✓] IP addresses configured
[✓] Client configured
[✓] Server 1 configured
[✓] Server 2 configured
[✓] HTTP services enabled
[✓] Load balancer configured
[✓] VIP configured
[✓] Server pool created
[✓] Load-balancing algorithm configured
[✓] Health checks configured
[✓] HTTP connectivity tested
[✓] Load balancing verified
[✓] Server failure simulated
[✓] Failover verified
[✓] Server recovery tested
[✓] show commands executed
[✓] Troubleshooting completed
```

---

# 🏁 Conclusion

This lab demonstrates how **Load Balancing** improves application availability, scalability, and reliability.

The fundamental architecture is:

```text
Client
   ↓
Virtual IP (VIP)
   ↓
Load Balancer
   ↓
Server Pool
   ↓
Backend Servers
```

The load balancer distributes requests between available servers and uses health checks to detect failures.

The key principle is:

```text
             LOAD BALANCING
                    |
        +-----------+-----------+
        |                       |
   DISTRIBUTE                 MONITOR
    TRAFFIC                   SERVERS
        |                       |
        ↓                       ↓
 Multiple Servers        Health Checks
        |                       |
        +-----------+-----------+
                    |
                    ↓
             HIGH AVAILABILITY
```
