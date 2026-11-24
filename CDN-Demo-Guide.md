# Content Delivery Network (CDN) — Practical 4-VM Regional Demo Guide
**Course:** Computer Networks & Internetworking (CNS3204)  
**Domain:** `ozy.cns` | **Subnet:** `192.168.56.0/24`  
**Repository:** https://github.com/dilithegit/PBL-DNS-CDN-Infrastructure

---

## 🎯 The Core Requirement Addressed
> *"There should be servers in different regions. When a client initiates a request to access content from a server, the request must be serviced by the server closest to the client."*

This practical guide requires **ONLY 4 running VMs**:
1. **Authoritative Geo-DNS Server** (`192.168.56.10`): Evaluates client source IP and routes to nearest edge.
2. **Origin Web Server** (`192.168.56.20`): Apache2 backend storing the master web pages.
3. **Edge Cache Node 1 (Region Africa)** (`192.168.56.71`): Nginx reverse proxy serving the local Africa subnet (`192.168.56.0/26`).
4. **Edge Cache Node 2 (Region Europe)** (`192.168.56.72`): Nginx reverse proxy serving remote clients (`192.168.56.64` to `.254`).

---

## 🗺️ Visual Architecture

```
                  Client Queries: http://cdn.ozy.cns
                                  │
                                  ▼
             ┌──────────────────────────────────────────┐
             │       Authoritative Geo-DNS (BIND9)      │
             │              192.168.56.10               │
             │   Evaluates Client Subnet via Views      │
             └────────────────────┬─────────────────────┘
                                  │
         ┌────────────────────────┴────────────────────────┐
         │                                                 │
 (Client in Africa Subnet: .20)            (Client in Europe Subnet: .72)
         ▼                                                 ▼
┌───────────────────────────────┐         ┌───────────────────────────────┐
│       Edge Cache Node 1       │         │       Edge Cache Node 2       │
│         Region: Africa        │         │         Region: Europe        │
│         192.168.56.71         │         │         192.168.56.72         │
│      Nginx Reverse Proxy      │         │      Nginx Reverse Proxy      │
└───────────────┬───────────────┘         └───────────────┬───────────────┘
                │                                         │
                └────────────────────┬────────────────────┘
                                     │ (Cache Miss Fetch Only)
                                     ▼
                      ┌─────────────────────────────┐
                      │     Origin Web Server       │
                      │     192.168.56.20:80        │
                      │         (Apache2)           │
                      └─────────────────────────────┘
```

---

## 🎬 Live Demonstration Script (Step-by-Step for Presentation)

### Step 1: Explain the Geography to the Lecturer
> *"Sir, we have two geographic regions simulated on our network:
> - **Region Africa:** Subnet `192.168.56.0/26` (e.g. Client VM at `.20` or host laptop)
> - **Region Europe:** Subnet `192.168.56.64/26` (e.g. Client VM at `.72`)
> 
> When clients in different regions request the exact same URL `http://cdn.ozy.cns`, Geo-DNS automatically steers them to the server closest to them."*

---

### Step 2: Show Regional Proximity Steering Live

Open two terminal windows side-by-side:

#### Terminal 1 — Client in Region Africa (`192.168.56.20`):
```bash
# 1. Ask DNS where cdn.ozy.cns is:
dig @192.168.56.10 cdn.ozy.cns +short
```
* **Output:** `192.168.56.71` *(DNS routed client to the closest Africa edge!)*

```bash
# 2. Fetch the webpage using the domain name:
curl -sI "http://cdn.ozy.cns" | grep -iE "x-served|x-cache"
```
* **Verified Header Output:**
  ```text
  X-Cache-Status: HIT
  X-Served-By: Edge-Cache-Africa (192.168.56.71)
  ```

---

#### Terminal 2 — Client in Region Europe (`192.168.56.72`):
```bash
# 1. Ask DNS where cdn.ozy.cns is:
dig @192.168.56.10 cdn.ozy.cns +short
```
* **Output:** `192.168.56.72` *(DNS routed client to the closest Europe edge!)*

```bash
# 2. Fetch the webpage using the exact same domain name:
curl -sI "http://cdn.ozy.cns" | grep -iE "x-served|x-cache"
```
* **Verified Header Output:**
  ```text
  X-Cache-Status: HIT
  X-Served-By: Edge-Cache-Europe (192.168.56.72)
  ```

---

### Step 3: Demonstrate Edge Caching (`MISS` vs `HIT`)
> *"Now we prove that the edge servers actually cache content and offload the central Origin server."*

Run on **Edge Cache Node 1 (`192.168.56.71`)**:

```bash
# Request 1: Fresh unique object -> Cache MISS (fetched from Origin .20)
curl -sI "http://192.168.56.71/?v=live_demo" | grep -iE "x-cache-status|x-served"
```
* **Output:**
  ```text
  X-Cache-Status: MISS
  X-Served-By: Edge-Cache-Africa (192.168.56.71)
  ```

```bash
# Request 2: Same object again -> Cache HIT (served in 0.002s from local RAM/disk)
curl -sI "http://192.168.56.71/?v=live_demo" | grep -iE "x-cache-status|x-served"
```
* **Output:**
  ```text
  X-Cache-Status: HIT
  X-Served-By: Edge-Cache-Africa (192.168.56.71)
  ```

---

## 📊 Summary Assessment Matrix

| Rubric Requirement | Technical Evidence | Demonstration Command |
|---|---|---|
| **Regional Proximity Routing** | BIND 9 Geo-views steer Africa client to `.71` and Europe client to `.72` | `dig @192.168.56.10 cdn.ozy.cns +short` run from both clients |
| **Origin Offloading** | Edge caches store copies of Apache web assets | Verified via `X-Served-By` headers pointing to edges, not origin |
| **Cache Lifecycle (`MISS` -> `HIT`)** | First request pulls from origin, second request served from local edge RAM | `curl -sI "http://cdn.ozy.cns/?v=demo" \| grep -i x-cache-status` |
