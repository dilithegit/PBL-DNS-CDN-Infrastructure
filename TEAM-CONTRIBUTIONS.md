# Team Contributions & Assessment Breakdown: DES & CDN
**Course:** Computer Networks & Internetworking (CNS3204)  
**Domain:** `ozy.cns` | **Subnet:** `192.168.56.0/24`  
**Repository:** https://github.com/dilithegit/PBL-DNS-CDN-Infrastructure

---

## 👥 Member Role Summary (One-Paragraph Speaking Explanations)

### 👤 Okwudili Ujubuonu (Dili) — DNS Tree Architect & Geo-DNS Lead
> *"As technical lead and DNS architect, I designed and implemented the full 7-VM hierarchical DNS Tree (Root, redundant TLDs `.cns` and `.isp`, Authoritative `ozy.cns`, and Recursive Resolver) and extended it to support both the email and CDN systems. Specifically, I configured the BIND 9 split-horizon Geo-DNS views on `192.168.56.10`, wrote the forward zones directing `www.ozy.cns` and `cdn.ozy.cns` to the HAProxy cluster, established the delegated reverse zone `56.168.192.in-addr.arpa` for FCrDNS PTR lookups, and published the authoritative SPF, 2048-bit DKIM (`default._domainkey`), and DMARC TXT policies that enforce email security across our network."*

### 👤 Roy (Email Lead) — Distributed Email System (DES) & MTA Security
> *"I led the implementation of our two-node distributed email system across Mail Server 1 (`192.168.56.20`) and Mail Server 2 (`192.168.56.25`). I configured Exim4 as a heavy MTA on both nodes with Dovecot Maildir storage, created user mailboxes (`dili@ozy.cns` and `roy2@mail2.ozy.cns`), generated the 2048-bit RSA DKIM private key and signing transport on Server 1, implemented custom manual routes bypassing DNSSEC coercion in our private lab tree, and demonstrated live cross-server mail delivery over TLS 1.3 encryption where every message is cryptographically validated against SPF, DKIM, and reverse PTR records."*

### 👤 Joy Ndira (CDN Lead) — HAProxy Layer 7 Load Balancing & High Availability
> *"I designed and deployed the CDN frontend load-balancing tier using HAProxy on `192.168.56.70`. I configured Layer 7 HTTP round-robin scheduling with active health checking (`option httpchk GET /`) across our edge cache nodes, established the administrative live statistics dashboard on port 8404, and demonstrated automatic failover—proving that if any edge cache node goes down, HAProxy transparently reroutes 100% of user traffic to surviving nodes without a single HTTP error or dropped session."*

### 👤 Joy Kioko (Edge Caching Lead) — Nginx Edge Cache Cluster & Reverse Proxy
> *"I built and optimized the dual edge caching tier using Nginx on Edge Cache Node 1 (`192.168.56.71`) and Edge Cache Node 2 (`192.168.56.72`). I configured Nginx reverse proxying to our Origin Apache server (`192.168.56.20`), implemented memory- and disk-backed microcaching (`proxy_cache cdn_cache`) with stale-while-revalidate protection, and configured custom diagnostic response headers (`X-Cache-Status: HIT/MISS` and `X-Served-By`) so we can verify that repeated client requests are absorbed at the edge with zero origin load."*

---

## 📋 Comprehensive Assessment Verification Table

| Member | System | Component Tested | Live Command / Proof | Expected Outcome |
|---|---|---|---|---|
| **Dili** | DNS / DES | PTR Reverse Lookups | `dig @192.168.56.10 -x 192.168.56.20 +short` | `mail.ozy.cns.` |
| **Dili** | DNS / DES | SPF Authorization | `dig @192.168.56.10 ozy.cns TXT +short` | `"v=spf1 ip4:192.168.56.20 ip4:192.168.56.25 mx -all"` |
| **Dili** | DNS / DES | DKIM Public Key | `dig @192.168.56.10 default._domainkey.ozy.cns TXT +short` | `"v=DKIM1; k=rsa; p=MIIBIjAN..."` |
| **Dili** | DNS / DES | DMARC Policy | `dig @192.168.56.10 _dmarc.ozy.cns TXT +short` | `"v=DMARC1; p=quarantine; ..."` |
| **Dili** | DNS / CDN | Geo-DNS Views | `dig @192.168.56.10 www.ozy.cns +short` | `192.168.56.70` (HAProxy VIP) |
| **Roy** | DES | Cross-Server Delivery | Send from Server 1: `mail -s "Test" roy2@mail2.ozy.cns` | Exim log: `=> roy2@mail2.ozy.cns ... Completed` |
| **Roy** | DES | TLS 1.3 Transit | `sudo tail -n 10 /var/log/exim4/mainlog` | `X=TLS1.3:ECDHE_SECP256R1__AES_256_GCM:256` |
| **Roy** | DES | Cryptographic DKIM Header | `sudo bash -c 'cat /home/roy2/Maildir/new/* \| grep -A 15 "DKIM-Signature"'` | `DKIM-Signature: v=1; a=rsa-sha256; d=ozy.cns; s=default; bh=...` |
| **Roy** | DES | PTR Disruption Demo | Comment PTR line in `db.56.168.192.in-addr.arpa`, resend email | Server 2 Exim log: `no host name found for IP address 192.168.56.20` |
| **Joy Ndira** | CDN | Round-Robin Balancing | `curl -I http://192.168.56.70` (run consecutively) | Alternates `x-served-by: Node-1` and `Node-2` |
| **Joy Ndira** | CDN | Live Health Check Failover | Stop Nginx on Node 1 (`sudo systemctl stop nginx`), curl HAProxy | 100% of traffic automatically shifts to Node-2 |
| **Joy Ndira** | CDN | Web Stats Dashboard | Browser visit `http://192.168.56.70:8404/stats` | Visual status, session rates, uptime |
| **Joy Kioko** | CDN | Edge Cache MISS | Clear cache (`sudo rm -rf /var/cache/nginx/*`), curl edge | `x-cache-status: MISS` |
| **Joy Kioko** | CDN | Edge Cache HIT | Immediately re-run curl against edge | `x-cache-status: HIT` |
| **Joy Kioko** | CDN | Origin Offload | Monitor Apache on `.20` while caching | Zero new requests reach Apache during cache window |
