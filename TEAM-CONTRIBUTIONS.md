# Full Team Task Allocation & Speaking Defense Guide (All 6 Members)
**Course:** Computer Networks & Internetworking (CNS3204)  
**Domain:** `ozy.cns` | **Subnet:** `192.168.56.0/24`  
**Repository:** https://github.com/dilithegit/PBL-DNS-CDN-Infrastructure

---

## 🎯 Defense Strategy (Read First)
In this project, **all 6 group members have an active, hands-on role across the entire architecture**:
1. **7-VM Hierarchical DNS Tree** (Root, TLDs, Authoritative, Resolver)
2. **Distributed Email System (DES)** (Dual MTAs, SPF, DKIM, PTR, DMARC, TLS 1.3)
3. **Content Delivery Network (CDN)** (Geo-DNS Proximity Steering, Dual Regional Edge Caches, Origin Offloading)

Every member has an exact technical script and command to show for BOTH the Email and CDN systems.

---

## 👤 1. Okwudili Ujubuonu (Dili) — Technical Lead & DNS/Security Architect

### In the Distributed Email System (DES):
> *"As technical lead and DNS architect, I designed the core security policy infrastructure on our authoritative DNS server (`192.168.56.10`). I configured Forward-Confirmed Reverse DNS (FCrDNS) by establishing the reverse zone `56.168.192.in-addr.arpa` with PTR records for both MTAs (`.20` -> `mail.ozy.cns` and `.25` -> `mail2.ozy.cns`). I then published the SPF TXT policy (`v=spf1 ip4:192.168.56.20 ip4:192.168.56.25 mx -all`), formatted the 2048-bit DKIM public key into RFC-compliant 255-byte chunks under `default._domainkey.ozy.cns`, and published the DMARC quarantine record (`v=DMARC1; p=quarantine`). I execute the live PTR disruption demo showing the `550 Mail rejected: Sender IP has no valid reverse DNS (PTR) record` hard bounce."*
- **Primary VM:** `192.168.56.10` (Authoritative DNS)
- **Demo Commands:** `dig @192.168.56.10 -x 192.168.56.20 +short`, `dig @192.168.56.10 default._domainkey.ozy.cns TXT +short`

### In the Content Delivery Network (CDN):
> *"In the CDN, I engineered the Geo-DNS traffic steering mechanism using BIND 9 split-horizon views on `192.168.56.10`. In `/etc/bind/named.conf.local`, I divided our zone into `region_africa` and `region_europe` views with `match-clients`. For client queries originating from the Africa subnet (`192.168.56.0/26`), DNS dynamically resolves `cdn.ozy.cns` directly to `192.168.56.71`, while queries from Europe resolve to `192.168.56.72`, guaranteeing that clients are routed to their geographically nearest edge server."*
- **Demo Commands:** `dig @192.168.56.10 cdn.ozy.cns +short` (run on both African and European clients)

---

## 👤 2. Roy — Primary MTA & Edge Cache Node 1 Lead

### In the Distributed Email System (DES):
> *"I led the implementation of our primary Mail Transfer Agent on Mail Server 1 (`192.168.56.20`). I deployed `exim4-daemon-heavy` and Dovecot, created user mailboxes (`dili@ozy.cns`), and implemented cryptographic outbound signing. I generated the 2048-bit RSA private key at `/etc/exim4/dkim/ozy.cns.private`, extracted the public key for DNS publishing, and configured the `remote_smtp` DKIM transport. To resolve delivery failures in our private offline tree, I disabled strict DNSSEC coercion by setting `dns_dnssec_ok = -1` in Exim's options and created the `mail2_manual` router so Mail Server 1 directly relays authenticated email to Mail Server 2 over TLS 1.3."*
- **Primary VM:** `192.168.56.20` (Mail Server 1)
- **Demo Commands:** `echo "Test" | mail -s "Demo" roy2@mail2.ozy.cns`, `sudo tail -n 10 /var/log/exim4/mainlog`

### In the Content Delivery Network (CDN):
> *"In the CDN, I built and verified Edge Cache Node 1 (`192.168.56.71`) serving Region Africa. I deployed Nginx as an edge reverse-proxy pointing to the Apache origin (`192.168.56.20`), configured the `/var/cache/nginx` cache zone (`proxy_cache_valid 200 10m`), and tagged all responses with `X-Served-By: Edge-Cache-Africa (192.168.56.71)` and `X-Cache-Status`. I demonstrate that when African clients request web content, it is served directly from Node 1's local cache with zero origin latency."*
- **Primary VM:** `192.168.56.71` (Edge Cache Africa)
- **Demo Command:** `curl -sI "http://192.168.56.71/?v=africa_user" | grep -iE "x-served|x-cache"`

---

## 👤 3. Austin — MTA Transport Security & Edge Cache 1 Node Lead

### In the Distributed Email System (DES):
> *"My focus in the email system was MTA transport security and TLS cryptographic negotiation. On Mail Server 1 (`192.168.56.20`), I verified that Exim4 advertises STARTTLS on port 25 and enforces modern cipher suites. In `/var/log/exim4/mainlog`, I monitor and prove that cross-server transactions between Server 1 and Server 2 establish an authenticated `TLS1.3:ECDHE_SECP256R1__RSA_PSS_RSAE_SHA256__AES_256_GCM:256` session, guaranteeing end-to-end transport confidentiality so cleartext credentials or message bodies can never be intercepted in transit."*
- **Primary VM:** `192.168.56.20` (Mail Server 1)
- **Demo Command:** `sudo grep "TLS1.3" /var/log/exim4/mainlog`

### In the Content Delivery Network (CDN):
> *"In the CDN, I worked on the Edge Cache 1 configuration, origin reverse proxy integration, and cache lifecycle management. I verified that Nginx properly manages cache validity windows (`proxy_cache_valid 200 10m`), handles cache expiration, and correctly fetches from the Apache backend on `192.168.56.20` only during an initial cache miss. In Step 6 of our demo, I demonstrate the speed-versus-freshness trade-off by showing how the edge continues serving valid cached objects until an administrative cache purge (`rm -rf /var/cache/nginx/* && systemctl reload nginx`) pushes updated origin content immediately to clients."*
- **Primary VM:** `192.168.56.71` (Edge Cache Africa)
- **Demo Command:** `curl -sI "http://192.168.56.71/?v=live_demo" | grep -iE "x-cache-status|x-served"`

---

## 👤 4. Joe — DNSSEC Tracing & Central Origin Web Server Lead

### In the Distributed Email System (DES):
> *"My role in the email system was DNSSEC debugging, resolver tracing, and mail routing verification. When Exim was dropping mail with the error `all relevant MX records point to non-existent hosts`, I used Exim's internal trace (`sudo exim4 -d-all+route+resolver -bt`) to diagnose that Exim's coerced DNSSEC validation was rejecting our private `.cns` TLD because our offline lab root lacks signed DS trust anchors. I traced the resolver configuration to `/var/lib/exim4/config.autogenerated`, verified the `dns_dnssec_ok = -1` fix in the split configuration, and confirmed that address routing resolves cleanly to `192.168.56.25`."*
- **Primary VM:** `192.168.56.20` (Mail Server 1 / Resolver)
- **Demo Command:** `sudo exim4 -bt roy2@mail2.ozy.cns`

### In the Content Delivery Network (CDN):
> *"In the CDN, I managed the Central Origin Web Server on `192.168.56.20`. I deployed and configured Apache2 to serve our master web content. During our live demonstration, I monitor the active access log (`sudo tail -f /var/log/apache2/access.log`) to provide indisputable proof of origin offloading: when a client fires 10 requests through the CDN, exactly ONE request hits Apache (the initial MISS), while the remaining 9 are completely absorbed by the edge cache nodes."*
- **Primary VM:** `192.168.56.20` (Apache Origin Server)
- **Demo Command:** `sudo tail -f /var/log/apache2/access.log`

---

## 👤 5. Joy Ndira — Secondary MTA & Dedicated European Client Lead

### In the Distributed Email System (DES):
> *"My responsibility in the email system was the recipient mail infrastructure on Mail Server 2 (`192.168.56.25`). I configured Exim4 as our secondary MX (priority 20 in DNS), configured `/etc/exim4/update-exim4.conf.conf` for internet site delivery accepting domains `@mail2.ozy.cns` and `@ozy.cns`, and enabled Dovecot IMAP/POP3 services. I set up user `roy2` with Maildir storage at `/home/roy2/Maildir/`, and injected the strict FCrDNS check into `/etc/exim4/exim4.conf.template` so that Server 2 actively rejects unverified sender IPs with a live `550 Mail rejected` error when PTR records are missing."*
- **Primary VM:** `192.168.56.25` (Mail Server 2)
- **Demo Commands:** `sudo bash -c 'cat /home/roy2/Maildir/new/* | grep -A 15 "DKIM-Signature"'`, `sudo tail -n 10 /var/log/exim4/mainlog`

### In the Content Delivery Network (CDN):
> *"In the CDN, I operate the Dedicated European Client on `192.168.56.70`. Because my VM resides in the European subnet (`192.168.56.64+`), when I query `cdn.ozy.cns`, Geo-DNS automatically directs me to our Europe Edge at `192.168.56.72`. I demonstrate live that my requests are serviced with `X-Served-By: Edge-Cache-Europe (192.168.56.72)` with full cache acceleration (`X-Cache-Status: HIT`), proving geographical proximity routing. In Step 5 of our demo, I demonstrate the measurable latency difference: when synthetic intercontinental delay (100ms) is applied to Europe via Linux `tc netem`, requests served from the local African edge respond in ~3ms while the distant European edge takes ~210ms, proving the tangible end-user speed benefit of proximity routing."*
- **Primary VM:** `192.168.56.70` (European Client)
- **Demo Commands:** `dig @192.168.56.10 cdn.ozy.cns +short`, `curl -sI "http://cdn.ozy.cns/?v=europe_user" | grep -iE "x-served|x-cache"`

---

## 👤 6. Joy Kioko — Dedicated African Client & Edge Cache Node 2 Lead

### In the Distributed Email System (DES):
> *"My role in the email system was MTA hardening, transport verification, and bidirectional relay testing. I configured Mail Server 2 (`192.168.56.25`) to relay return emails back to Mail Server 1 (`192.168.56.20`), verifying that recipient verification, envelope routing, and reverse lookup validation operate symmetrically across both servers. I monitor the delivery logs during Dili's PTR disruption tests to demonstrate how Exim catches unverified senders and generates non-delivery reports (NDRs) returning to the sender."*
- **Primary VM:** `192.168.56.25` (Mail Server 2)
- **Demo Commands:** `echo "Reply Test" | mail -s "Reply" dili@ozy.cns`, `sudo tail -f /var/log/exim4/mainlog`

### In the Content Delivery Network (CDN):
> *"In the CDN, I built and tuned Edge Cache Node 2 (`192.168.56.72`) serving Region Europe, and I operate the Dedicated African Client on `192.168.56.25`. As the African client, I query `cdn.ozy.cns` and prove that DNS directs me to `192.168.56.71` with `X-Served-By: Edge-Cache-Africa`. Together with Joy Ndira's European client, we demonstrate on side-by-side screens that two clients in different regions querying the exact same domain name are independently steered to their closest regional edge server."*
- **Primary VMs:** `192.168.56.72` (Edge Cache Europe) and `192.168.56.25` (African Client)
- **Demo Commands:** `curl -sI "http://cdn.ozy.cns/?v=africa_user" | grep -iE "x-served|x-cache"`

---

## 📊 Complete 6-Member Responsibility Matrix

| Member | Distributed Email System (DES) Responsibility | Content Delivery Network (CDN) Responsibility | Live Demo Proof Command |
|---|---|---|---|
| **Dili** | Authoritative DNS: PTR reverse zone, SPF, DKIM DNS record, DMARC | Geo-DNS split-horizon views in BIND9 steering clients to nearest regional edge | `dig @192.168.56.10 -x 192.168.56.20 +short`<br/>`dig @192.168.56.10 cdn.ozy.cns +short` |
| **Roy** | Primary MTA (Server 1): Exim4 heavy, 2048-bit DKIM private key & signing | Edge Cache Node 1 (`.71`): Nginx proxy_cache setup & diagnostic headers | `mail -s "Test" roy2@mail2.ozy.cns`<br/>`sudo tail -n 10 /var/log/exim4/mainlog` |
| **Austin** | MTA transport security: TLS 1.3 cipher negotiation & encryption audit | Edge Cache Node 1 (`.71`): Cache tuning & origin reverse proxy integration | `sudo grep "TLS1.3" /var/log/exim4/mainlog`<br/>`curl -sI "http://192.168.56.71" \| grep -i x-cache` |
| **Joe** | DNSSEC resolver debugging: diagnosing Exim DNSSEC coercion & router fixes | Origin Server (`.20`): Apache2 web hosting & live origin access log monitoring | `sudo exim4 -bt roy2@mail2.ozy.cns`<br/>`sudo tail -f /var/log/apache2/access.log` |
| **Joy Ndira** | Secondary MTA (Server 2): Exim4 MX-20 reception, Dovecot Maildir, user `roy2`, strict 550 PTR drop | Dedicated European Client (`.70`): Demonstrating European proximity routing | `sudo bash -c 'cat /home/roy2/Maildir/new/* \| grep DKIM'`<br/>`curl -sI "http://cdn.ozy.cns/?v=europe_user"` |
| **Joy Kioko** | Bidirectional mail delivery, Dovecot verification, Exim NDR log capture | Dedicated African Client (`.25`) & Edge Cache 2 (`.72`): Demonstrating African proximity routing | `echo "Reply" \| mail -s "Reply" dili@ozy.cns`<br/>`curl -sI "http://cdn.ozy.cns/?v=africa_user"` |
