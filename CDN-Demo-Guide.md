# Content Delivery Network (CDN) & Load Balancing — Demo Guide
**Domain:** `ozy.cns` | **Subnet:** `192.168.56.0/24`
**Repository:** https://github.com/dilithegit/PBL-DNS-CDN-Infrastructure

---

## Architecture Overview

```
                      ┌──────────────────────────────────────┐
                      │          Client Request              │
                      │   http://www.ozy.cns (or cdn.ozy.cns) │
                      └──────────────────┬───────────────────┘
                                         │
                                         ▼
                      ┌──────────────────────────────────────┐
                      │    Geo-DNS (BIND9 Views)             │
                      │    192.168.56.10                     │
                      │    Resolves www -> 192.168.56.70     │
                      └──────────────────┬───────────────────┘
                                         │
                                         ▼
                      ┌──────────────────────────────────────┐
                      │    HAProxy Load Balancer             │
                      │    192.168.56.70:80                  │
                      │    Mode: HTTP Round-Robin            │
                      └──────────────┬────────────────┬──────┘
                                     │                │
                    ┌────────────────┘                └────────────────┐
                    │                                                  │
                    ▼                                                  ▼
     ┌─────────────────────────────┐                    ┌─────────────────────────────┐
     │    Edge Cache Node 1        │                    │    Edge Cache Node 2        │
     │    192.168.56.71:80         │                    │    192.168.56.72:80         │
     │    Nginx Proxy + Cache      │                    │    Nginx Proxy + Cache      │
     └──────────────┬──────────────┘                    └──────────────┬──────────────┘
                    │                                                  │
                    └─────────────────┬────────────────────────────────┘
                                      │ (Cache Miss Fetch)
                                      ▼
                      ┌──────────────────────────────────────┐
                      │    Origin Web Server                 │
                      │    192.168.56.20:80 (Apache2)        │
                      └──────────────────────────────────────┘
```

---

## Deployed VM Roles

| Node Name | IP Address | Service | Function |
|---|---|---|---|
| **DNS Master** | `192.168.56.10` | BIND9 | Authoritative DNS with `internal` and `external` Geo-DNS views |
| **Origin Server** | `192.168.56.20` | Apache2 | Origin backend web server hosting target web pages |
| **HAProxy LB** | `192.168.56.70` | HAProxy 2.x | Layer 7 Round-Robin load balancer with active health checks |
| **Edge Cache 1** | `192.168.56.71` | Nginx 1.2x | Reverse proxy edge node with RAM/disk object cache |
| **Edge Cache 2** | `192.168.56.72` | Nginx 1.2x | Redundant edge node with independent cache storage |

---

## Demonstration Script for Lecturer Assessment

### 1. Demonstrate Geo-DNS Views
Run on **DNS Server (`192.168.56.10`)**:
```bash
dig @192.168.56.10 www.ozy.cns +short
dig @192.168.56.10 cdn.ozy.cns +short
```
* **Expected Output:** `192.168.56.70` (points to the CDN Load Balancer).

---

### 2. Demonstrate Round-Robin Load Balancing
Run consecutive requests against the load balancer to show traffic alternating 50/50 across edge servers:
```bash
curl -I http://192.168.56.70 | grep -iE 'HTTP|x-served|x-cache'
curl -I http://192.168.56.70 | grep -iE 'HTTP|x-served|x-cache'
```
* **Expected Output:**
  ```text
  HTTP/1.1 200 OK
  x-cache-status: HIT
  x-served-by: Edge-Cache-Node-2 (192.168.56.72)

  HTTP/1.1 200 OK
  x-cache-status: HIT
  x-served-by: Edge-Cache-Node-1 (192.168.56.71)
  ```

---

### 3. Demonstrate Edge Caching (`X-Cache-Status: HIT` vs `MISS`)
Clear the cache on Edge 1 to show cache lifecycle:
```bash
# On Edge Cache 1 (192.168.56.71):
sudo rm -rf /var/cache/nginx/*
sudo systemctl reload nginx

# First request: Cache MISS (must fetch from Origin 192.168.56.20)
curl -i http://192.168.56.71 | grep -i "x-cache-status"
# Output: x-cache-status: MISS

# Second request: Cache HIT (served from RAM/SSD cache)
curl -i http://192.168.56.71 | grep -i "x-cache-status"
# Output: x-cache-status: HIT
```

---

### 4. Demonstrate Automatic Failover & High Availability
Prove the load balancer handles node failures without downtime:
```bash
# 1. Stop Nginx on Edge Cache 1 (192.168.56.71)
sudo systemctl stop nginx

# 2. Query HAProxy from any client:
curl -I http://192.168.56.70 | grep -i "x-served-by"
curl -I http://192.168.56.70 | grep -i "x-served-by"
# Output: 100% of requests immediately route to Edge-Cache-Node-2! Zero dropped requests!

# 3. Bring Edge 1 back online:
sudo systemctl start nginx
# HAProxy health check automatically marks Edge 1 as UP and resumes 50/50 balance.
```

---

### 5. Live Visual Web Dashboard
Open in your browser on the host machine:
```text
http://192.168.56.70:8404/stats
```
Shows green status indicators, real-time sessions, active server weights, and packet metrics for both edge nodes.
