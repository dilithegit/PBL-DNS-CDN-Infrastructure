# Team Task Allocation & Speaking Defense Guide: DES & CDN
**Course:** Computer Networks & Internetworking (CNS3204)  
**Domain:** `ozy.cns` | **Subnet:** `192.168.56.0/24`  
**Repository:** https://github.com/dilithegit/PBL-DNS-CDN-Infrastructure

---

## 🎯 Defense Strategy (Read First)
In this project, **every member has an active, hands-on role in BOTH the Distributed Email System (DES) and the Content Delivery Network (CDN)**. If the lecturer points to ANY student and asks:
> *"What did YOU specifically configure and demonstrate in the email system?"* OR *"What did YOU specifically configure and demonstrate in the CDN?"*

Every student has an exact, technical, one-paragraph answer with their specific commands and files.

---

## 👤 1. Okwudili Ujubuonu (Dili) — DNS Core, Security Policies & Geo-Steering

### In the Distributed Email System (DES):
> *"My role in the email system was the authoritative security policies and DNS infrastructure. On our authoritative DNS server (`192.168.56.10`), I implemented Forward-Confirmed Reverse DNS (FCrDNS) by creating the reverse delegated zone `56.168.192.in-addr.arpa` with PTR records for both mail servers (`.20` -> `mail.ozy.cns` and `.25` -> `mail2.ozy.cns`). I then published the SPF TXT record (`v=spf1 ip4:192.168.56.20 ip4:192.168.56.25 mx -all`) authorizing our exact MTAs, formatted and published the 2048-bit DKIM public key into RFC-compliant 255-byte chunks under `default._domainkey.ozy.cns`, and added the DMARC enforcement policy record (`v=DMARC1; p=quarantine`). During the demo, I execute the live PTR disruption by commenting out the record to show the `no host name found` MTA warning."*
- **Primary VM:** `192.168.56.10` (DNS Server)
- **Key Files:** `/etc/bind/db.ozy.cns`, `/etc/bind/db.56.168.192.in-addr.arpa`, `/etc/bind/named.conf.local`

### In the Content Delivery Network (CDN):
> *"In the CDN, I engineered the Geo-DNS traffic steering mechanism using BIND 9 split-horizon views. In `/etc/bind/named.conf.local`, I divided our zone into `internal` and `external` views with `match-clients`. For requests originating from our local subnet (`192.168.56.0/24`), DNS dynamically steers `www.ozy.cns` and `cdn.ozy.cns` to the high-performance HAProxy Load Balancer at `192.168.56.70`. I also ensured all server zones adhered to strict view containment so query resolution remains 100% reliable across subnets."*
- **Key Commands:** `dig @192.168.56.10 www.ozy.cns +short`, `sudo named-checkzone ozy.cns /etc/bind/db.ozy.cns.internal`

---

## 👤 2. Roy — Primary MTA Architecture, DKIM Cryptography & Edge Cache 1

### In the Distributed Email System (DES):
> *"I owned the primary Mail Transfer Agent (MTA) deployment on Mail Server 1 (`192.168.56.20`). I installed `exim4-daemon-heavy` and Dovecot, created user mailboxes (`dili@ozy.cns`), and implemented cryptographic email signing. I generated the 2048-bit RSA private key under `/etc/exim4/dkim/ozy.cns.private`, extracted the public key for Dili to publish to DNS, and injected the `remote_smtp` DKIM signing transport. Furthermore, because Exim defaulted to DNSSEC enforcement on our unsigned private root tree, I identified and resolved the lookup issue by setting `dns_dnssec_ok = -1` in `/etc/exim4/conf.d/main/02_exim4-config_options` and configured the `mail2_manual` router so Mail Server 1 directly relays mail to Mail Server 2 over TLS 1.3."*
- **Primary VM:** `192.168.56.20` (Mail Server 1 / Resolver)
- **Key Files:** `/etc/exim4/dkim/ozy.cns.private`, `/etc/exim4/conf.d/router/00_manual_mail2`, `/etc/exim4/conf.d/main/02_exim4-config_options`
- **Key Commands:** `echo "Test" | mail -s "Demo" roy2@mail2.ozy.cns`, `sudo tail -n 10 /var/log/exim4/mainlog`

### In the Content Delivery Network (CDN):
> *"In the CDN, I built and verified Edge Cache Node 1 (`192.168.56.71`). I deployed Nginx as an edge reverse-proxy pointing to the Apache origin on `192.168.56.20`, established the `/var/cache/nginx` cache directory, and configured cache expiration rules (`proxy_cache_valid 200 10m`). I also injected the custom diagnostic headers `X-Cache-Status` and `X-Served-By: Edge-Cache-Node-1` so that incoming HTTP requests from the load balancer can be independently verified as cached."*
- **Primary VM:** `192.168.56.71` (Edge Cache 1)
- **Key Commands:** `curl -I http://localhost | grep -i x-cache-status`

---

## 👤 3. Joy Ndira — Secondary MTA Mailbox, Receiving Verification & HAProxy Load Balancer

### In the Distributed Email System (DES):
> *"My responsibility in the email system was the recipient mail infrastructure on Mail Server 2 (`192.168.56.25`). I configured Exim4 as our secondary MX (priority 20 in DNS), configured `/etc/exim4/update-exim4.conf.conf` for internet site delivery accepting domains `@mail2.ozy.cns` and `@ozy.cns`, and enabled Dovecot IMAP/POP3 services. I set up user `roy2` with Maildir storage at `/home/roy2/Maildir/`, configured plaintext authentication over our trusted host-only network, and verified incoming encrypted SMTP connections on port 25. During testing, I inspect the raw Maildir headers to prove that incoming messages carry valid `DKIM-Signature: v=1; a=rsa-sha256`, correct PTR IP matching, and TLS 1.3 ciphers."*
- **Primary VM:** `192.168.56.25` (Mail Server 2)
- **Key Files:** `/etc/exim4/update-exim4.conf.conf`, `/etc/dovecot/conf.d/10-mail.conf`, `/home/roy2/Maildir/new/`
- **Key Commands:** `sudo bash -c 'cat /home/roy2/Maildir/new/* | grep -A 15 "DKIM-Signature"'`, `sudo tail -n 10 /var/log/exim4/mainlog`

### In the Content Delivery Network (CDN):
> *"In the CDN, I architected the frontend Layer 7 Load Balancer on `192.168.56.70` using HAProxy. I configured frontend `cdn_frontend` on port 80 to distribute traffic evenly across our two edge cache nodes (`192.168.56.71` and `192.168.56.72`) using Round-Robin scheduling with HTTP health checks (`option httpchk GET /`). I also set up the live web dashboard on port 8404 (`/stats`) to monitor real-time node uptime and request rates, and demonstrated automated failover by proving that traffic transparently shifts 100% to surviving nodes if an edge goes down."*
- **Primary VM:** `192.168.56.70` (HAProxy)
- **Key Files:** `/etc/haproxy/haproxy.cfg`
- **Key Commands:** `curl -I http://192.168.56.70 | grep -i x-served-by`, browser visit `http://192.168.56.70:8404/stats`

---

## 👤 4. Joy Kioko — Bidirectional Email Verification, MTA Hardening & Edge Cache 2

### In the Distributed Email System (DES):
> *"My role in the email system was MTA hardening, transport verification, and bidirectional relay testing. I configured Mail Server 2 (`192.168.56.25`) to relay return emails back to Mail Server 1 (`192.168.56.20`), verifying that recipient verification, envelope routing, and reverse lookup validation operate symmetrically across both servers. I handled MTA log monitoring on Server 2 during Dili's PTR disruption tests, demonstrating how Exim catches unverified senders with the `no host name found for IP address 192.168.56.20` warning, and confirmed that Dovecot properly isolates individual messages inside Maildir subdirectories (`cur`, `new`, `tmp`)."*
- **Primary VMs:** `192.168.56.25` and `192.168.56.20`
- **Key Commands:** `echo "Reply Test" | mail -s "Reply" dili@ozy.cns`, `sudo tail -f /var/log/exim4/mainlog`

### In the Content Delivery Network (CDN):
> *"In the CDN, I built and tuned Edge Cache Node 2 (`192.168.56.72`) and implemented our cache validation testing. I installed and configured Nginx to cache web assets from the Apache origin server (`192.168.56.20`), configured microcaching buffers with `proxy_cache_use_stale` for high availability during origin lags, and tagged all responses with `X-Served-By: Edge-Cache-Node-2`. To prove the CDN works to the lecturer, I demonstrate both `X-Cache-Status: MISS` (on initial fetch or cache purge) and `X-Cache-Status: HIT` (on subsequent requests served entirely from edge RAM/disk)."*
- **Primary VM:** `192.168.56.72` (Edge Cache 2)
- **Key Files:** `/etc/nginx/sites-available/cdn`
- **Key Commands:** `curl -I http://192.168.56.72 | grep -i x-cache-status`, `sudo rm -rf /var/cache/nginx/* && sudo systemctl reload nginx`

---

## 📊 Summary Responsibility Matrix

| Team Member | Distributed Email System (DES) Responsibility | Content Delivery Network (CDN) Responsibility | Primary Demo Command |
|---|---|---|---|
| **Dili** | Authoritative DNS: PTR reverse zone, SPF, DKIM public key, DMARC policy | Geo-DNS split-horizon views in BIND9 steering clients to CDN | `dig @192.168.56.10 -x 192.168.56.20 +short`<br/>`dig @192.168.56.10 www.ozy.cns +short` |
| **Roy** | Primary MTA (Server 1): Exim4 heavy, 2048-bit DKIM private key generation & signing, DNSSEC bypass | Edge Cache Node 1 (`.71`): Nginx reverse proxy, cache setup & diagnostic headers | `mail -s "Test" roy2@mail2.ozy.cns`<br/>`sudo tail -n 10 /var/log/exim4/mainlog` |
| **Joy Ndira** | Secondary MTA (Server 2): Exim4 MX-20 reception, Dovecot Maildir setup, user `roy2` | HAProxy Load Balancer (`.70`): Layer 7 HTTP Round-Robin, health checks, web stats | `curl -I http://192.168.56.70 \| grep -i x-served-by`<br/>`http://192.168.56.70:8404/stats` |
| **Joy Kioko** | Bidirectional mail delivery, Dovecot verification, Exim PTR log warning capture | Edge Cache Node 2 (`.72`): Nginx caching, origin offloading, `HIT/MISS` demonstration | `curl -I http://192.168.56.72 \| grep -i x-cache-status`<br/>`sudo bash -c 'cat /home/roy2/Maildir/new/*'` |
