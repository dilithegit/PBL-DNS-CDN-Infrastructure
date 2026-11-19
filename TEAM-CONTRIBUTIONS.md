# Full Team Task Allocation & Speaking Defense Guide (All 6 Members)
**Course:** Computer Networks & Internetworking (CNS3204)  
**Domain:** `ozy.cns` | **Subnet:** `192.168.56.0/24`  
**Repository:** https://github.com/dilithegit/PBL-DNS-CDN-Infrastructure

---

## 🎯 Defense Strategy (Read First)
In this project, **all 6 group members have an active, hands-on role across the entire architecture**:
1. **7-VM Hierarchical DNS Tree** (Root, TLDs, Authoritative, Resolver)
2. **Distributed Email System (DES)** (Dual MTAs, SPF, DKIM, PTR, DMARC, TLS 1.3)
3. **Content Delivery Network (CDN)** (Geo-DNS, HAProxy Round-Robin, Dual Nginx Edge Caches, Origin Apache)

If the lecturer points to ANY member and asks what they did in the Email System or CDN, they have an exact technical script and command to show.

---

## 👤 1. Okwudili Ujubuonu (Dili) — Technical Lead & DNS/Security Architect

### In the Distributed Email System (DES):
> *"As technical lead and DNS architect, I designed the core security policy infrastructure on our authoritative DNS server (`192.168.56.10`). I configured Forward-Confirmed Reverse DNS (FCrDNS) by establishing the reverse zone `56.168.192.in-addr.arpa` with PTR records for both MTAs (`.20` -> `mail.ozy.cns` and `.25` -> `mail2.ozy.cns`). I then published the SPF TXT policy (`v=spf1 ip4:192.168.56.20 ip4:192.168.56.25 mx -all`), formatted the 2048-bit DKIM public key into RFC-compliant 255-byte chunks under `default._domainkey.ozy.cns`, and published the DMARC quarantine record (`v=DMARC1; p=quarantine`). I execute the live PTR disruption demo showing the `no host name found` MTA warning when reverse lookups are broken."*
- **Primary VM:** `192.168.56.10` (Authoritative DNS)
- **Demo Commands:** `dig @192.168.56.10 -x 192.168.56.20 +short`, `dig @192.168.56.10 default._domainkey.ozy.cns TXT +short`

### In the Content Delivery Network (CDN):
> *"In the CDN, I engineered the Geo-DNS traffic steering mechanism using BIND 9 split-horizon views on `192.168.56.10`. In `/etc/bind/named.conf.local`, I divided our zone into `internal` and `external` views with `match-clients`. For client queries originating from our local subnet (`192.168.56.0/24`), DNS dynamically resolves `www.ozy.cns` and `cdn.ozy.cns` directly to our HAProxy load-balancer VIP at `192.168.56.70`, ensuring seamless client onboarding into the caching cluster."*
- **Demo Command:** `dig @192.168.56.10 www.ozy.cns +short`

---

## 👤 2. Roy — Primary MTA & DKIM Cryptography Lead

### In the Distributed Email System (DES):
> *"I led the implementation of our primary Mail Transfer Agent on Mail Server 1 (`192.168.56.20`). I deployed `exim4-daemon-heavy` and Dovecot, created user mailboxes (`dili@ozy.cns`), and implemented cryptographic outbound signing. I generated the 2048-bit RSA private key at `/etc/exim4/dkim/ozy.cns.private`, extracted the public key for DNS publishing, and configured the `remote_smtp` DKIM transport. To resolve delivery failures in our private offline tree, I disabled strict DNSSEC coercion by setting `dns_dnssec_ok = -1` in Exim's options and created the `mail2_manual` router so Mail Server 1 directly relays authenticated email to Mail Server 2 over TLS 1.3."*
- **Primary VM:** `192.168.56.20` (Mail Server 1)
- **Demo Commands:** `echo "Test" | mail -s "Demo" roy2@mail2.ozy.cns`, `sudo tail -n 10 /var/log/exim4/mainlog`

### In the Content Delivery Network (CDN):
> *"In the CDN, I configured and verified Edge Cache Node 1 (`192.168.56.71`). I deployed Nginx as an edge reverse-proxy pointing to the Apache origin (`192.168.56.20`), configured the `/var/cache/nginx` cache zone (`proxy_cache_valid 200 10m`), and injected custom response headers (`X-Served-By: Edge-Cache-Node-1` and `X-Cache-Status`) to prove that requests coming through HAProxy are cached and served from the edge."*
- **Primary VM:** `192.168.56.71` (Edge Cache 1)
- **Demo Command:** `curl -I http://localhost | grep -i x-cache-status`

---

## 👤 3. Austin — MTA Transport Security & Edge Cache 1 Node Lead

### In the Distributed Email System (DES):
> *"My focus in the email system was MTA transport security and TLS cryptographic negotiation. On Mail Server 1 (`192.168.56.20`), I verified that Exim4 advertises STARTTLS on port 25 and enforces modern cipher suites. In `/var/log/exim4/mainlog`, I monitor and prove that cross-server transactions between Server 1 and Server 2 establish an authenticated `TLS1.3:ECDHE_SECP256R1__RSA_PSS_RSAE_SHA256__AES_256_GCM:256` session, guaranteeing end-to-end transport confidentiality so cleartext credentials or message bodies can never be intercepted in transit."*
- **Primary VM:** `192.168.56.20` (Mail Server 1)
- **Demo Command:** `sudo grep "TLS1.3" /var/log/exim4/mainlog`

### In the Content Delivery Network (CDN):
> *"In the CDN, I worked on the Edge Cache 1 deployment (`192.168.56.71`) and Origin integration. I verified that Nginx correctly proxies incoming requests to the Apache backend on `192.168.56.20` and tuned cache lifecycle headers, ensuring that cache misses successfully fetch new web assets from the origin and store them locally so subsequent client hits bypass the origin entirely."*
- **Primary VM:** `192.168.56.71` (Edge Cache 1)
- **Demo Command:** `curl -s -I http://192.168.56.71 | grep -iE 'x-cache|x-served'`

---

## 👤 4. Joe — DNSSEC Debugging & Origin Web Server Lead

### In the Distributed Email System (DES):
> *"My role in the email system was DNSSEC debugging, resolver tracing, and mail routing verification. When Exim was dropping mail with the error `all relevant MX records point to non-existent hosts`, I used Exim's internal trace (`sudo exim4 -d-all+route+resolver -bt`) to diagnose that Exim's coerced DNSSEC validation was rejecting our private `.cns` TLD because our offline lab root lacks signed DS trust anchors. I traced the resolver configuration to `/var/lib/exim4/config.autogenerated`, verified the `dns_dnssec_ok = -1` fix in the split configuration, and confirmed that address routing resolves cleanly to `192.168.56.25`."*
- **Primary VM:** `192.168.56.20` (Mail Server 1 / Resolver)
- **Demo Command:** `sudo exim4 -bt roy2@mail2.ozy.cns`

### In the Content Delivery Network (CDN):
> *"In the CDN, I managed the Origin Web Server on `192.168.56.20`. I deployed and configured Apache2 to serve our web applications and Roundcube webmail assets. I verified origin responsiveness on port 80 and confirmed that during high-traffic load tests through the CDN, the origin CPU and network load remain near zero because our dual Nginx edge nodes absorb repeated requests using their cached copies."*
- **Primary VM:** `192.168.56.20` (Apache Origin Server)
- **Demo Command:** `curl -I http://localhost` (run on .20), `sudo systemctl status apache2`

---

## 👤 5. Joy Ndira — Secondary MTA Architecture & HAProxy Load Balancer Lead

### In the Distributed Email System (DES):
> *"My responsibility in the email system was the recipient mail infrastructure on Mail Server 2 (`192.168.56.25`). I configured Exim4 as our secondary MX (priority 20 in DNS), configured `/etc/exim4/update-exim4.conf.conf` for internet site delivery accepting domains `@mail2.ozy.cns` and `@ozy.cns`, and enabled Dovecot IMAP/POP3 services. I set up user `roy2` with Maildir storage at `/home/roy2/Maildir/`, configured plaintext authentication over our trusted host-only network, and verified incoming encrypted SMTP connections on port 25. During testing, I inspect the raw Maildir headers to prove that incoming messages carry valid `DKIM-Signature: v=1; a=rsa-sha256`, correct PTR IP matching, and TLS 1.3 ciphers."*
- **Primary VM:** `192.168.56.25` (Mail Server 2)
- **Demo Commands:** `sudo bash -c 'cat /home/roy2/Maildir/new/* | grep -A 15 "DKIM-Signature"'`, `sudo tail -n 10 /var/log/exim4/mainlog`

### In the Content Delivery Network (CDN):
> *"In the CDN, I architected the frontend Layer 7 Load Balancer on `192.168.56.70` using HAProxy. I configured frontend `cdn_frontend` on port 80 to distribute traffic evenly across our two edge cache nodes (`192.168.56.71` and `192.168.56.72`) using Round-Robin scheduling with HTTP health checks (`option httpchk GET /`). I also set up the live web dashboard on port 8404 (`/stats`) to monitor real-time node uptime and request rates, and demonstrated automated failover by proving that traffic transparently shifts 100% to surviving nodes if an edge goes down."*
- **Primary VM:** `192.168.56.70` (HAProxy)
- **Demo Commands:** `curl -I http://192.168.56.70 | grep -i x-served-by`, browser visit `http://192.168.56.70:8404/stats`

---

## 👤 6. Joy Kioko — Bidirectional Relay & Edge Cache 2 Node Lead

### In the Distributed Email System (DES):
> *"My role in the email system was MTA hardening, transport verification, and bidirectional relay testing. I configured Mail Server 2 (`192.168.56.25`) to relay return emails back to Mail Server 1 (`192.168.56.20`), verifying that recipient verification, envelope routing, and reverse lookup validation operate symmetrically across both servers. I handled MTA log monitoring on Server 2 during Dili's PTR disruption tests, demonstrating how Exim catches unverified senders with the `no host name found for IP address 192.168.56.20` warning, and confirmed that Dovecot properly isolates individual messages inside Maildir subdirectories (`cur`, `new`, `tmp`)."*
- **Primary VM:** `192.168.56.25` (Mail Server 2)
- **Demo Commands:** `echo "Reply Test" | mail -s "Reply" dili@ozy.cns`, `sudo tail -f /var/log/exim4/mainlog`

### In the Content Delivery Network (CDN):
> *"In the CDN, I built and tuned Edge Cache Node 2 (`192.168.56.72`) and implemented our cache validation testing. I installed and configured Nginx to cache web assets from the Apache origin server (`192.168.56.20`), configured microcaching buffers with `proxy_cache_use_stale` for high availability during origin lags, and tagged all responses with `X-Served-By: Edge-Cache-Node-2`. To prove the CDN works to the lecturer, I demonstrate both `X-Cache-Status: MISS` (on initial fetch or cache purge) and `X-Cache-Status: HIT` (on subsequent requests served entirely from edge RAM/disk)."*
- **Primary VM:** `192.168.56.72` (Edge Cache 2)
- **Demo Commands:** `curl -I http://192.168.56.72 | grep -i x-cache-status`, `sudo rm -rf /var/cache/nginx/* && sudo systemctl reload nginx`

---

## 📊 Complete 6-Member Responsibility Matrix

| Member | Distributed Email System (DES) Responsibility | Content Delivery Network (CDN) Responsibility | Live Demo Proof Command |
|---|---|---|---|
| **Dili** | Authoritative DNS: PTR reverse zone, SPF, DKIM DNS record, DMARC | Geo-DNS split-horizon views in BIND9 steering clients to CDN | `dig @192.168.56.10 -x 192.168.56.20 +short`<br/>`dig @192.168.56.10 www.ozy.cns +short` |
| **Roy** | Primary MTA (Server 1): Exim4 heavy, 2048-bit DKIM private key & signing | Edge Cache Node 1 (`.71`): Nginx proxy_cache setup & diagnostic headers | `mail -s "Test" roy2@mail2.ozy.cns`<br/>`sudo tail -n 10 /var/log/exim4/mainlog` |
| **Austin** | MTA transport security: TLS 1.3 cipher negotiation & encryption audit | Edge Cache Node 1 (`.71`): Cache tuning & origin reverse proxy integration | `sudo grep "TLS1.3" /var/log/exim4/mainlog`<br/>`curl -I http://192.168.56.71` |
| **Joe** | DNSSEC resolver debugging: diagnosing Exim DNSSEC coercion & router fixes | Origin Server (`.20`): Apache2 web hosting & origin cache offload verification | `sudo exim4 -bt roy2@mail2.ozy.cns`<br/>`curl -I http://localhost` (on .20) |
| **Joy Ndira** | Secondary MTA (Server 2): Exim4 MX-20 reception, Dovecot Maildir, user `roy2` | HAProxy Load Balancer (`.70`): Layer 7 HTTP Round-Robin, health checks, web stats | `sudo bash -c 'cat /home/roy2/Maildir/new/* \| grep DKIM'`<br/>`curl -I http://192.168.56.70` |
| **Joy Kioko** | Bidirectional mail delivery, Dovecot verification, Exim PTR log warning capture | Edge Cache Node 2 (`.72`): Nginx caching, origin offloading, `HIT/MISS` demonstration | `curl -I http://192.168.56.72 \| grep -i x-cache-status`<br/>`sudo tail -n 10 /var/log/exim4/mainlog` |
