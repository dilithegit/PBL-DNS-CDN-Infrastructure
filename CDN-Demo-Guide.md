# Content Delivery Network (CDN) — Master Demonstration Guide
**Course:** Computer Networks & Internetworking (CNS3204)  
**Domain:** `ozy.cns` | **Subnet:** `192.168.56.0/24`  
**Repository:** https://github.com/dilithegit/PBL-DNS-CDN-Infrastructure

---

## 🌍 The Problem & The Lecturer's Requirement
> *"There should be servers in different regions. When a client initiates a request to access content from a server, the request must be serviced by the server closest to the client."*

In this architecture, our network deploys two distinct regional client zones and two matching regional edge cache servers:
1. **Region Africa (Local Subnet `192.168.56.0/26` — IPs `.1` to `.63`):**
   - **Dedicated African Client:** `192.168.56.25`
   - **Nearest Regional Edge:** `192.168.56.71` (`Edge-Cache-Africa`)
2. **Region Europe (Remote Subnet `192.168.56.64/26` — IPs `.64` to `.254`):**
   - **Dedicated European Client:** `192.168.56.70`
   - **Nearest Regional Edge:** `192.168.56.72` (`Edge-Cache-Europe`)
3. **Central Origin Server (`192.168.56.20`):**
   - Apache2 web server hosting the master content.

---

## 🗺️ Architectural Topology

```
                  African Client (192.168.56.25)             European Client (192.168.56.70)
                         │                                                 │
                         │ Query: http://cdn.ozy.cns                       │ Query: http://cdn.ozy.cns
                         ▼                                                 ▼
             ┌──────────────────────────────────────────────────────────────────┐
             │                  Authoritative Geo-DNS (BIND9)                   │
             │                          192.168.56.10                           │
             │             Evaluates Client IP via Split-Horizon Views          │
             └──────────────────┬────────────────────────────┬──────────────────┘
                                │                            │
                (Africa Subnet: .25 in .0/26)      (Europe Subnet: .70 in .64+)
                                ▼                            ▼
                  Resolves to: 192.168.56.71    Resolves to: 192.168.56.72
                                │                            │
                                ▼                            ▼
                 ┌─────────────────────────────┐┌─────────────────────────────┐
                 │      Edge Cache Africa      ││      Edge Cache Europe      │
                 │        192.168.56.71        ││        192.168.56.72        │
                 │     Nginx Reverse Proxy     ││     Nginx Reverse Proxy     │
                 └──────────────┬──────────────┘└──────────────┬──────────────┘
                                │                              │
                                └──────────────┬───────────────┘
                                               │ (Cache Miss Fetch Only)
                                               ▼
                                 ┌─────────────────────────────┐
                                 │      Origin Web Server      │
                                 │        192.168.56.20        │
                                 │          (Apache2)          │
                                 └─────────────────────────────┘
```

---

## 🎬 Master Live Demonstration Script (Step-by-Step for Presentation)

### Step 1: Open the Three Windows Side-by-Side
- **Window 1 (Origin Monitor):** Origin Server (`192.168.56.20`) running `sudo tail -f /var/log/apache2/access.log`
- **Window 2 (African Client):** Dedicated African Client (`192.168.56.25`)
- **Window 3 (European Client):** Dedicated European Client (`192.168.56.70`)

---

### Step 2: Demonstrate African Proximity Routing
**On African Client (`192.168.56.25`):**
```bash
# Query DNS:
dig @192.168.56.10 cdn.ozy.cns +short
```
* **Verified Output:** **`192.168.56.71`** *(Geo-DNS steers client to nearest Africa edge)*

```bash
# Fetch Content:
curl -sI "http://cdn.ozy.cns/?v=africa_user" | grep -iE "x-served|x-cache"
```
* **Verified Headers:**
  ```text
  X-Cache-Status: MISS
  X-Served-By: Edge-Cache-Africa (192.168.56.71)
  ```
*Run again to prove caching:*
```bash
curl -sI "http://cdn.ozy.cns/?v=africa_user" | grep -iE "x-served|x-cache"
```
* **Verified Headers:**
  ```text
  X-Cache-Status: HIT
  X-Served-By: Edge-Cache-Africa (192.168.56.71)
  ```

---

### Step 3: Demonstrate European Proximity Routing
**On European Client (`192.168.56.70`):**
```bash
# Query DNS:
dig @192.168.56.10 cdn.ozy.cns +short
```
* **Verified Output:** **`192.168.56.72`** *(Geo-DNS steers client to nearest Europe edge)*

```bash
# Fetch Content:
curl -sI "http://cdn.ozy.cns/?v=europe_user" | grep -iE "x-served|x-cache"
```
* **Verified Headers:**
  ```text
  X-Cache-Status: MISS
  X-Served-By: Edge-Cache-Europe (192.168.56.72)
  ```
*Run again to prove caching:*
```bash
curl -sI "http://cdn.ozy.cns/?v=europe_user" | grep -iE "x-served|x-cache"
```
* **Verified Headers:**
  ```text
  X-Cache-Status: HIT
  X-Served-By: Edge-Cache-Europe (192.168.56.72)
  ```

---

### Step 4: The Ultimate Proof — Origin Server Offloading
**Show the live Apache access log on Origin (`192.168.56.20`):**
```bash
sudo tail -f /var/log/apache2/access.log
```

**Now, on the African Client (`192.168.56.25`), fire 10 requests:**
```bash
curl -sI "http://192.168.56.71/?v=proof1" | grep -i x-cache-status
for i in {1..10}; do curl -s -o /dev/null "http://192.168.56.71/?v=proof1"; done
```

**Look at what appeared in the Apache access log:**
```text
192.168.56.71 - - [24/Nov/2026:12:04:01 +0300] "GET /?v=proof1 HTTP/1.0" 200 10977 "-" "curl/8.14.1"
```
*Points to explain to lecturer:*
- The Apache log registered **EXACTLY 1 request** (the initial `MISS`).
- The other **10 requests were completely absorbed by the Edge Cache**, serving `X-Cache-Status: HIT` with **zero network traffic reaching the Origin**!

---

## 📊 Summary Assessment Matrix

| Requirement | Technical Proof | Live Verification |
|---|---|---|
| **Geographic Proximity Steering** | African client (`.25`) resolves to `.71`; European client (`.70`) resolves to `.72` | `dig @192.168.56.10 cdn.ozy.cns +short` on both clients |
| **Regional Edge Caching** | Both edges independently serve content from local RAM/SSD with `X-Cache-Status: HIT` | `curl -sI "http://cdn.ozy.cns/?v=..." \| grep -i x-cache` |
| **Origin Offloading** | Central Apache on `192.168.56.20` sees only 1 request out of 10 | `/var/log/apache2/access.log` shows 1 entry for 10 client visits |
| **Region Identification** | Headers explicitly tag `Edge-Cache-Africa` vs `Edge-Cache-Europe` | `grep -i "x-served-by"` in HTTP responses |
