# Content Delivery Network (CDN) — Dedicated Multi-Region Demo Guide
**Course:** Computer Networks & Internetworking (CNS3204)  
**Domain:** `ozy.cns` | **Subnet:** `192.168.56.0/24`  
**Repository:** https://github.com/dilithegit/PBL-DNS-CDN-Infrastructure

---

## 🌍 The Problem & The Lecturer's Requirement
> *"There should be servers in different regions. When a client initiates a request to access content from a server, the request must be serviced by the server closest to the client."*

In this architecture, our network deploys two distinct regional client zones and two matching regional edge cache servers:
1. **Region Africa (Local Subnet `192.168.56.0/26` — IPs `.1` to `.63`):**
   - **Dedicated African Client:** `192.168.56.25` (Client VM)
   - **Nearest Regional Edge:** `192.168.56.71` (`Edge-Cache-Africa`)
2. **Region Europe (Remote Subnet `192.168.56.64/26` — IPs `.64` to `.254`):**
   - **Dedicated European Client:** `192.168.56.70` (Client VM)
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

## 🎬 Live Demonstration Script for Lecturer Assessment

### Step 1: Open the Two Dedicated Client Terminals Side-by-Side

Show the lecturer two terminal windows:
- **Terminal 1:** Dedicated African Client (`192.168.56.25`)
- **Terminal 2:** Dedicated European Client (`192.168.56.70`)

---

### Step 2: Query from the African Client (`192.168.56.25`)
**In Terminal 1 (African Client):**
```bash
# Query Geo-DNS:
dig @192.168.56.10 cdn.ozy.cns +short
```
* **DNS Result:** **`192.168.56.71`** *(Geo-DNS automatically steers the African client to Africa Edge!)*

```bash
# Fetch Content via Domain Name:
curl -sI "http://cdn.ozy.cns/?v=africa_user" | grep -iE "x-served|x-cache"
```
* **Live Verified Header Output:**
  ```text
  X-Cache-Status: MISS
  X-Served-By: Edge-Cache-Africa (192.168.56.71)
  ```
*Run again to show caching:*
```bash
curl -sI "http://cdn.ozy.cns/?v=africa_user" | grep -iE "x-served|x-cache"
```
* **Output:** **`X-Cache-Status: HIT`** *(Served in 0.002s from Africa Edge!)*

---

### Step 3: Query from the European Client (`192.168.56.70`)
**In Terminal 2 (European Client):**
```bash
# Query Geo-DNS:
dig @192.168.56.10 cdn.ozy.cns +short
```
* **DNS Result:** **`192.168.56.72`** *(Geo-DNS automatically steers the European client to Europe Edge!)*

```bash
# Fetch Content via Domain Name:
curl -sI "http://cdn.ozy.cns/?v=europe_user" | grep -iE "x-served|x-cache"
```
* **Live Verified Header Output:**
  ```text
  X-Cache-Status: MISS
  X-Served-By: Edge-Cache-Europe (192.168.56.72)
  ```
*Run again to show caching:*
```bash
curl -sI "http://cdn.ozy.cns/?v=europe_user" | grep -iE "x-served|x-cache"
```
* **Output:** **`X-Cache-Status: HIT`** *(Served in 0.002s from Europe Edge!)*

---

## 📊 Summary Assessment Matrix

| Requirement | Technical Proof | Live Command |
|---|---|---|
| **Geographic Proximity Steering** | African client (.25) gets `.71`; European client (.70) gets `.72` | `dig @192.168.56.10 cdn.ozy.cns +short` on both clients |
| **Regional Edge Caching** | Each regional edge serves from local RAM/SSD with `X-Cache-Status: HIT` | `curl -sI "http://cdn.ozy.cns/?v=..." \| grep -i x-cache` |
| **Origin Offloading** | Central Apache on `192.168.56.20` handles 0 requests once cached | `tail -f /var/log/apache2/access.log` on Origin stays silent during HITs |
| **Region Identification** | Explicit response headers tag `Edge-Cache-Africa` vs `Edge-Cache-Europe` | `grep -i "x-served-by"` in HTTP response |
