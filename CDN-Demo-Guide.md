# Content Delivery Network (CDN) — Master Demonstration & Command Guide

**Course:** Computer Networks & Internetworking (CNS3204)  
**Domain:** `ozy.cns` | **Subnet:** `192.168.56.0/24`  
**Repository:** https://github.com/dilithegit/PBL-DNS-CDN-Infrastructure  

---

## 🎯 CDN Core Objective & Problem Statement
> *"There should be servers in different regions. When a client initiates a request to access content from a server, the request must be serviced by the server closest to the client."*

A Content Delivery Network (CDN) fundamentally performs three distinct operations:
1. **Geo-Proximity Routing:** Routes each user dynamically to the nearest regional edge server based on client IP.
2. **Edge Caching:** Caches static/dynamic web assets close to the client to eliminate latency.
3. **Origin Offloading:** Absorbs repeated requests at the edge, shielding the central origin web server from high traffic volume.

---

## 🖥️ System Architecture & Node Roles

| Role | Hostname / Label | IP Address | Subnet / Region | Technology Stack |
|---|---|---|---|---|
| **Geo-DNS Server** | `dns.ozy.cns` | `192.168.56.10` | Central Backbone | BIND 9 (Split-Horizon Views + ACLs) |
| **Origin Web Server** | `mail.ozy.cns` | `192.168.56.20` | Central Backbone | Apache 2.4 (Document Root `/var/www/html`) |
| **Edge Cache Africa** | `edge1.ozy.cns` | `192.168.56.71` | Region Africa | Nginx (Reverse Proxy + Proxy Cache) |
| **Edge Cache Europe** | `edge2.ozy.cns` | `192.168.56.72` | Region Europe | Nginx (Reverse Proxy + Proxy Cache) |
| **Dedicated African Client** | `mail2` | `192.168.56.25` | `192.168.56.0/26` (Africa Zone) | Debian Client (`curl`, `dig`) |
| **Dedicated European Client** | `vbox` | `192.168.56.70` | `192.168.56.64/26` (Europe Zone) | Debian Client (`curl`, `dig`) |

---

## 🗺️ Architectural Topology Diagram

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
                                │                            │
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

## ⚙️ Before the Demo: Setup Checks

### Check A: Confirm Clients Use the Geo-DNS Server
* **Why:** `curl` uses the local system resolver (`/etc/resolv.conf`), not the `@server` parameter passed into `dig`. If the client points elsewhere (e.g. VirtualBox NAT or home gateway), `cdn.ozy.cns` will fail to resolve.
* **Command:** *(Run on both African Client `.25` and European Client `.70`)*
  ```bash
  cat /etc/resolv.conf
  ```
  *(If incorrect, run: `sudo bash -c 'echo "nameserver 192.168.56.10" > /etc/resolv.conf'`)*
* **Meaning:** Prints the configuration file listing the recursive DNS servers queried by system libraries.
* **Investigating:** Confirms whether the client is configured to query our BIND 9 Geo-DNS server.
* **Expected Output:**
  ```text
  nameserver 192.168.56.10
  ```

---

### Check B: Confirm All CDN Services Are Active
* **Why:** A stopped service breaks the live demo flow mid-presentation.
* **Commands:**
  ```bash
  # On Geo-DNS Server (192.168.56.10):
  sudo systemctl status bind9

  # On Central Origin Server (192.168.56.20):
  sudo systemctl status apache2

  # On Edge Cache Africa (192.168.56.71) & Edge Cache Europe (192.168.56.72):
  sudo systemctl status nginx
  ```
* **Meaning:** `systemctl status` reports whether a system service daemon is currently executing without errors.
* **Investigating:** Verifies that every component of the distributed CDN is healthy and listening on ports 53 and 80.
* **Expected Output:**
  ```text
  Active: active (running)
  ```

---

### Check C: Completely Flush Edge Caches
* **Why:** A warm cache will immediately return `HIT`, preventing you from showing the transition from `MISS` to `HIT` on camera. Nginx retains in-memory keys in `keys_zone=cdn_cache:10m`, so stopping the process before wiping disk files guarantees an empty cache state.
* **Command:** *(Run on both Edge Nodes: `192.168.56.71` and `192.168.56.72`)*
  ```bash
  sudo systemctl stop nginx
  sudo rm -rf /var/cache/nginx/*
  sudo systemctl start nginx
  ```
* **Meaning:** Halts the Nginx master process, purges all stored cache files from the path defined by `proxy_cache_path`, and restarts the server with an empty in-memory cache directory table.
* **Investigating:** Resets the CDN to a pristine, predictable initial state.
* **Expected Output:** Clean restart; no error output. The immediate subsequent HTTP request is guaranteed to register as `MISS`.

---

## 🎬 Master Live Demonstration Script (Step-by-Step)

### Step 1: Geo-Routing (Same URL, Different Server)

#### 1a. From the African Client (`192.168.56.25`)
* **Why:** Demonstrates that DNS dynamically determines which regional edge node a client reaches based on the client's source IP subnet.
* **Command:**
  ```bash
  dig @192.168.56.10 cdn.ozy.cns +short
  ```
* **Meaning:** `dig` is an authoritative DNS lookup tool. `@192.168.56.10` targets our BIND 9 server directly. `cdn.ozy.cns` is the target hostname. `+short` suppresses commentary and prints only the resolved A record.
* **Investigating:** Which edge IP BIND 9 returns for a client residing in the African subnet (`192.168.56.0/26`). BIND matches `.25` to the `region_africa` view.
* **Expected Output:**
  ```text
  192.168.56.71
  ```
  *(Edge Cache Africa)*

#### 1b. From the European Client (`192.168.56.70`)
* **Why:** Runs the identical query from a different subnet to contrast against 1a and prove regional steering.
* **Command:**
  ```bash
  dig @192.168.56.10 cdn.ozy.cns +short
  ```
* **Meaning:** Identical query syntax as 1a. Only the client's network origin has changed (`.70` falls into `192.168.56.64/26`).
* **Investigating:** Confirms that the returned IP changes dynamically according to client source subnet—the textbook definition of Geo-DNS routing.
* **Expected Output:**
  ```text
  192.168.56.72
  ```
  *(Edge Cache Europe)*

---

### Step 2: Server Identification & Delivery Verification

Run from each client to demonstrate that the client connects to and receives content from the nearest edge server, rather than bypassing the CDN to the origin.

#### 2a. African Client (`192.168.56.25`)
* **Why:** Validates that the HTTP layer connects to the regional edge indicated by DNS and inspects custom tracking headers.
* **Command:**
  ```bash
  curl -sI "http://cdn.ozy.cns/?v=africa_user" | grep -iE "x-served|x-cache"
  ```
* **Meaning:**
  - `curl` fetches the URL via system DNS resolution.
  - `-s` enables silent operation.
  - `-I` requests HTTP headers only (HEAD request).
  - Pipe (`|`) redirects output to `grep`.
  - `-iE "x-served|x-cache"` matches case-insensitive diagnostic headers injected by our Nginx edge configurations.
* **Expected Output (First Request):**
  ```text
  X-Cache-Status: MISS
  X-Served-By: Edge-Cache-Africa (192.168.56.71)
  ```
* **Expected Output (Repeated Request):**
  ```text
  X-Cache-Status: HIT
  X-Served-By: Edge-Cache-Africa (192.168.56.71)
  ```

#### 2b. European Client (`192.168.56.70`)
* **Why:** Proves that the European client is simultaneously serviced by the European edge node.
* **Command:**
  ```bash
  curl -sI "http://cdn.ozy.cns/?v=europe_user" | grep -iE "x-served|x-cache"
  ```
* **Expected Output (First Request):**
  ```text
  X-Cache-Status: MISS
  X-Served-By: Edge-Cache-Europe (192.168.56.72)
  ```
* **Expected Output (Repeated Request):**
  ```text
  X-Cache-Status: HIT
  X-Served-By: Edge-Cache-Europe (192.168.56.72)
  ```

#### 2c. From Windows Host (PowerShell Check)
* **Why:** Demonstrates that testing is not restricted to Linux VM shells; the host machine or external Windows endpoints on the VirtualBox Host-Only network (`192.168.56.1`) can independently verify edge delivery and caching headers.
* **PowerShell Command (Option 1 — using `curl.exe` with `Select-String`):**
  ```powershell
  curl.exe -sI "http://192.168.56.71/" | Select-String -Pattern "X-Served|X-Cache"
  ```
* **PowerShell Command (Option 2 — native `Invoke-WebRequest`):**
  ```powershell
  $response = Invoke-WebRequest -Uri "http://192.168.56.71/" -Method Head
  $response.Headers | Select-Object "X-Cache-Status", "X-Served-By"
  ```
* **Meaning:**
  - `curl.exe` (or `Invoke-WebRequest -Method Head`) sends a standard HTTP HEAD query directly from the Windows PowerShell environment.
  - `Select-String -Pattern "X-Served|X-Cache"` is the native PowerShell equivalent to Linux `grep -iE`.
* **Expected Output:**
  ```text
  X-Cache-Status: HIT
  X-Served-By: Edge-Cache-Africa (192.168.56.71)
  ```

---

### Step 3: Edge Caching (MISS then HIT Lifecycle)

Run on **Edge Africa (`192.168.56.71`)** or African Client (`.25`).  
*Tip: Always use a unique query string (e.g. `?v=live_demo`) to ensure you are demonstrating a fresh object.*

#### 3a. Initial Request (Cold Cache)
* **Why:** Proves that when an edge server receives a request for an uncached object, it reaches back to the central origin server (`192.168.56.20`).
* **Command:**
  ```bash
  curl -sI "http://192.168.56.71/?v=live_demo" | grep -iE "x-cache-status|x-served"
  ```
* **Meaning:** Sends a HEAD request specifically targeting Edge 1 with a unique query parameter.
* **Investigating:** Confirms that an uncached object triggers an origin fetch, returning `MISS`.
* **Expected Output:**
  ```text
  X-Cache-Status: MISS
  X-Served-By: Edge-Cache-Africa (192.168.56.71)
  ```

#### 3b. Repeated Request (Warm Cache)
* **Why:** Proves that subsequent requests for the identical object are served entirely from edge storage without touching the origin.
* **Command:**
  ```bash
  curl -sI "http://192.168.56.71/?v=live_demo" | grep -iE "x-cache-status|x-served"
  ```
* **Investigating:** Confirms that the cache state immediately transitions to `HIT`.
* **Expected Output:**
  ```text
  X-Cache-Status: HIT
  X-Served-By: Edge-Cache-Africa (192.168.56.71)
  ```

---

### Step 4: The Ultimate Proof — Origin Server Offloading

This is the strongest architectural proof. Open two terminals side-by-side:
- **Terminal 1:** Central Origin Web Server (`192.168.56.20`)
- **Terminal 2:** African Client (`192.168.56.25`)

#### 4a. Watch the Origin Web Access Log in Real-Time
* **Why:** Response headers can theoretically be spoofed; the origin server's own access log provides independent, irrefutable proof of who contacted it and how many times.
* **On Origin Server (`192.168.56.20`):**
  ```bash
  sudo tail -f /var/log/apache2/access.log
  ```
* **Meaning:** `tail -f` outputs newly appended lines to the Apache access log in real-time. Leave this running in Terminal 1.
* **Expected Output:** Idle and quiet until traffic arrives.

#### 4b. Trigger Exactly One Cache MISS
* **Why:** To observe what an origin fetch looks like in the Apache logs.
* **On African Client (`192.168.56.25`):**
  ```bash
  curl -sI "http://192.168.56.71/?v=proof_origin" | grep -i x-cache-status
  ```
* **Expected Output:**
  - **On Client:** `X-Cache-Status: MISS`
  - **In Origin Log (Terminal 1):** Exactly ONE line appears:
    ```text
    192.168.56.71 - - [24/Nov/2026:12:04:01 +0300] "GET /?v=proof_origin HTTP/1.0" 200 10977 "-" "curl/8.14.1"
    ```
    *Notice the source IP is `192.168.56.71` (the Edge Cache Node), NOT the client!*

#### 4c. Send Ten More Requests (Complete Origin Offload)
* **Why:** Proves that repeated client traffic is completely shielded by the edge cache and never hits the origin backend.
* **On African Client (`192.168.56.25`):**
  ```bash
  for i in {1..10}; do curl -s -o /dev/null "http://192.168.56.71/?v=proof_origin"; done
  ```
* **Meaning:** Executes a bash loop running `curl` 10 times consecutively. `-o /dev/null` discards HTML output.
* **Investigating:** Whether any new log entries appear on the origin server.
* **Expected Output:**
  - **On Client:** Completes in milliseconds with zero errors.
  - **In Origin Log (Terminal 1):** **ABSOLUTELY NOTHING.** The log does not move.
  - **Conclusion:** 10 requests were handled; 10 were served from the edge cache (`HIT`); **0 reached Apache**. The origin was 100% offloaded.

---

### Step 5 (Advanced / Distinction): Making "Closest" Measurable via Simulated Latency

Because all virtual machines operate on the same host-only subnet (`192.168.56.0/24`), physical network propagation delay is naturally ~0.2ms. Adding synthetic latency to one edge simulates geographical distance (e.g. Europe being thousands of miles away).

#### 5a. Introduce Synthetic Delay on Edge Europe (`192.168.56.72`)
* **Why:** To simulate real-world intercontinental latency on the remote European edge.
* **On Edge Europe (`192.168.56.72`):**
  ```bash
  # Check interface name (typically enp0s8):
  ip a

  # Add 100ms artificial network delay via Linux Traffic Control:
  sudo tc qdisc add dev enp0s8 root netem delay 100ms
  ```
* **Meaning:** `tc` (Traffic Control) configures the Linux kernel packet scheduler; `netem` (Network Emulator) injects an exact 100ms delay on all outbound packets.
* **Verification:** Run `ping -c 3 192.168.56.72` from a client; RTT will jump from ~0.3ms to ~100ms.

#### 5b. Benchmark Latency Differences Between Edges
* **Why:** Empirically demonstrates the performance benefit of reaching the local, low-latency edge.
* **From African Client (`192.168.56.25`):**
  ```bash
  # Warm cache on both edges first:
  curl -s -o /dev/null "http://192.168.56.71/?v=benchmark"
  curl -s -o /dev/null "http://192.168.56.72/?v=benchmark"

  # Measure round-trip time to local Africa Edge:
  curl -s -o /dev/null -w "Africa Edge Response Time: %{time_total}s\n" "http://192.168.56.71/?v=benchmark"

  # Measure round-trip time to remote Europe Edge:
  curl -s -o /dev/null -w "Europe Edge Response Time: %{time_total}s\n" "http://192.168.56.72/?v=benchmark"
  ```
* **Expected Output:**
  - Africa Edge: `~0.003s` (3 ms — instantaneous local edge)
  - Europe Edge: `~0.210s` (210 ms — high latency due to simulated distance)
* **Cleanup afterwards on `.72`:**
  ```bash
  sudo tc qdisc del dev enp0s8 root
  ```

---

### Step 6 (Advanced / Distinction): Cache Freshness, Stale Content, and Purging

Demonstrates the core engineering trade-off of CDNs: **speed vs. freshness**.

#### 6a. Update Master Content on the Origin Server
* **Why:** Demonstrates what happens when back-end content changes while edges still hold active cache copies.
* **On Origin Server (`192.168.56.20`):**
  ```bash
  echo "<h1>Version 2 - Updated Origin Content</h1>" | sudo tee /var/www/html/index.html
  ```
* **Investigating:** Confirms that Apache immediately serves Version 2.

#### 6b. Request Content Through the Edge
* **Why:** Demonstrates that the edge protects the origin by continuing to serve its valid cached copy until TTL expires.
* **From Client:**
  ```bash
  curl -s "http://192.168.56.71/"
  ```
* **Expected Output:** The **old** content (`Version 1`). The edge honors its configured TTL (`proxy_cache_valid 200 10m`).

#### 6c. Purge Edge Cache to Force Fresh Delivery
* **Why:** Demonstrates an administrative cache purge to immediately push emergency updates out to edge users.
* **On Edge Node (`192.168.56.71`):**
  ```bash
  sudo rm -rf /var/cache/nginx/*
  sudo systemctl reload nginx
  ```
* **Refetch from Client:**
  ```bash
  curl -s "http://192.168.56.71/"
  ```
* **Expected Output:**
  ```html
  <h1>Version 2 - Updated Origin Content</h1>
  ```
  *(Content updated across the CDN instantaneously)*

---

## 📊 Summary Assessment Matrix

| Step | Capability Demonstrated | Technical Mechanism | Observable Live Proof |
|---|---|---|---|
| **Step 1** | **Geo-Proximity Routing** | BIND 9 Split-Horizon Views + ACL subnets | Client `.25` resolves to `.71`; Client `.70` resolves to `.72` |
| **Step 2** | **Edge Server Verification** | Nginx reverse proxy headers | `X-Served-By: Edge-Cache-Africa` vs `Edge-Cache-Europe` |
| **Step 3** | **Edge Cache Lifecycle** | Nginx `proxy_cache_valid` + disk cache | `X-Cache-Status: MISS` on 1st request, `HIT` on all subsequent |
| **Step 4** | **Origin Server Offloading** | Upstream reverse proxy absorption | Apache `/var/log/apache2/access.log` records 1 line for 10 client visits |
| **Step 5** | **Proximity Latency Advantage** | Linux `tc netem` latency injection | Local edge responds in ~3ms vs remote edge in ~210ms |
| **Step 6** | **Cache Freshness & Purging** | Nginx cache invalidation | Cached version persists until manual purge pushes Version 2 |

---

## 💡 Architectural Framing for Examiners
- **Subnet Emulation:** In production environments, commercial CDNs (Cloudflare, Akamai) use **BGP Anycast** and **EDNS Client Subnet (ECS, RFC 7871)**. In our virtualized university lab, geographic regions are modeled using distinct IP subnets (`192.168.56.0/26` vs `192.168.56.64/26`) evaluated by BIND 9 views.
- **Protocol Fidelity:** The underlying caching mechanisms, proxy buffering, origin offloading, and HTTP header signaling are 100% production-standard Nginx reverse proxy architectures.
