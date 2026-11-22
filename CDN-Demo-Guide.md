# Content Delivery Network (CDN) — Regional Geo-Proximity Demo Guide
**Domain:** `ozy.cns` | **Subnet:** `192.168.56.0/24`  
**Repository:** https://github.com/dilithegit/PBL-DNS-CDN-Infrastructure

---

## 🌍 The Problem & The Lecturer's Requirement
> *"There should be servers in different regions. When a client initiates a request to access content from a server, the request must be serviced by the server closest to the client."*

In this architecture, our network simulates two distinct geographic regions:
1. **Region Africa / Local (`192.168.56.0/26` — IPs `.1` to `.63`):** Closest edge node is **Edge Cache Node 1** (`192.168.56.71`).
2. **Region Europe / Remote (`any` other subnet — IPs `.64` to `.254`):** Closest edge node is **Edge Cache Node 2** (`192.168.56.72`).
3. **Central Origin Server (`192.168.56.20`):** Backing web server hosting the original content behind Apache2.

---

## 🗺️ Architectural Topology

```
                                  Client Request
                              http://www.ozy.cns
                                      │
                                      ▼
                   ┌──────────────────────────────────────┐
                   │     Authoritative Geo-DNS (BIND9)    │
                   │            192.168.56.10             │
                   │    Evaluates Client Source IP Subnet │
                   └──────────────────┬───────────────────┘
                                      │
              ┌───────────────────────┴───────────────────────┐
              │                                               │
      (Region 1: Africa)                              (Region 2: Europe)
       Source: .0/26 (.20)                             Source: Remote (.70+)
              ▼                                               ▼
┌───────────────────────────┐                   ┌───────────────────────────┐
│     Edge Cache Node 1     │                   │     Edge Cache Node 2     │
│       Region: Africa      │                   │       Region: Europe      │
│       192.168.56.71       │                   │       192.168.56.72       │
│    Nginx Reverse Proxy    │                   │    Nginx Reverse Proxy    │
└─────────────┬─────────────┘                   └─────────────┬─────────────┘
              │                                               │
              └───────────────────────┬───────────────────────┘
                                      │ (Cache Miss Fetch)
                                      ▼
                        ┌───────────────────────────┐
                        │    Origin Apache Server   │
                        │       192.168.56.20       │
                        └───────────────────────────┘
```

---

## 🚀 Live Demonstration Script (Step-by-Step for Lecturer)

### Step 1: Demonstrate Regional Client A (Region Africa / Local)
**Run from a client in the Africa IP range (e.g., Mail Server 1 at `192.168.56.20`):**

```bash
# 1. Query Geo-DNS:
dig @192.168.56.10 www.ozy.cns +short
```
* **Live Verified DNS Output:**
  ```text
  192.168.56.71
  ```
  *(DNS identifies the client is in the Africa subnet and returns the closest node: `192.168.56.71`)*

```bash
# 2. Fetch Content from the assigned closest edge:
curl -I http://www.ozy.cns --resolve www.ozy.cns:80:192.168.56.71
```
* **Live Verified HTTP Response:**
  ```text
  HTTP/1.1 200 OK
  Server: nginx
  X-Cache-Status: HIT
  X-Served-By: Edge-Cache-Africa (192.168.56.71)
  ```

---

### Step 2: Demonstrate Regional Client B (Region Europe / Remote)
**Run from a client in the Europe IP range (e.g., node at `192.168.56.70` or `.90`):**

```bash
# 1. Query Geo-DNS:
dig @192.168.56.10 www.ozy.cns +short
```
* **Live Verified DNS Output:**
  ```text
  192.168.56.72
  ```
  *(DNS identifies the client is outside the local range and returns the closest Europe node: `192.168.56.72`)*

```bash
# 2. Fetch Content from the assigned closest edge:
curl -I http://www.ozy.cns --resolve www.ozy.cns:80:192.168.56.72
```
* **Live Verified HTTP Response:**
  ```text
  HTTP/1.1 200 OK
  Server: nginx
  X-Cache-Status: HIT
  X-Served-By: Edge-Cache-Europe (192.168.56.72)
  ```

---

### Step 3: Demonstrate Independent Edge Caching (`HIT` vs `MISS`)
Show the lecturer that each regional edge independently caches content from the origin:
```bash
# On Edge Cache Node 1 (192.168.56.71):
sudo rm -rf /var/cache/nginx/* && sudo systemctl reload nginx

# First Request (Cache MISS - fetches from Origin 192.168.56.20):
curl -i http://localhost | grep -i "x-cache-status"
# Output: X-Cache-Status: MISS

# Second Request (Cache HIT - served directly from regional edge RAM/SSD):
curl -i http://localhost | grep -i "x-cache-status"
# Output: X-Cache-Status: HIT
```

---

## 👥 Speaking Script for Presentation Defense

> **Dili:** *"Sir, to address the requirement of servicing clients from their closest geographical server, we designed a split-horizon Geo-DNS architecture on `192.168.56.10` using BIND 9 views. We partitioned our network into regional subnets: client queries from Region Africa (`192.168.56.0/26`) are resolved to our Africa edge node at `192.168.56.71`, while queries from other regions automatically resolve to our Europe edge node at `192.168.56.72`."*

> **Joy Ndira:** *"On the edge infrastructure, both Edge 1 and Edge 2 run Nginx as reverse caching proxies pointing to our central Apache Origin server on `192.168.56.20`. Each edge tags its responses with its regional identifier and cache status."*

> **Joy Kioko:** *"When we query from each regional client, you can see live on the terminal that Client A is served by `Edge-Cache-Africa` with `X-Cache-Status: HIT`, and Client B is served by `Edge-Cache-Europe` with `X-Cache-Status: HIT`. Content is served from the closest edge, ensuring minimal latency and zero origin load."*
