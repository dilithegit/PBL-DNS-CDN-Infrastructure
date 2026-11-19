# Distributed Email System (DES) — Assessment Demo Guide
**Domain:** `ozy.cns` | **Mail Server 1:** `192.168.56.20` | **Mail Server 2:** `192.168.56.25`
**Repository:** https://github.com/dilithegit/PBL-DNS-CDN-Infrastructure

---

## Team Contributions

### Dili — DNS Tree Architect
Built and verified the complete 7-VM DNS hierarchy:
- **Root DNS Primary** (`192.168.56.30`): BIND9 root zone, authoritative for `.`
- **Root DNS Secondary** (`192.168.56.90`): Zone transfer slave of root
- **TLD 1 `.cns`** (`192.168.56.40` + `192.168.56.50`): Two NS records for `.cns` TLD
- **TLD 2 `.isp`** (`192.168.56.80`): Second TLD zone
- **Authoritative DNS `ozy.cns`** (`192.168.56.10`): A, MX, TXT (SPF, DKIM, DMARC), PTR records
- **Recursive Resolver** (`192.168.56.20`): Custom `db.root.hints`, walks the full 4-tier tree

**Key config files:**
- `/etc/bind/db.ozy.cns` — Forward zone with MX, SPF, DKIM, DMARC records
- `/etc/bind/db.56.168.192.in-addr.arpa` — Reverse PTR zone
- `/etc/bind/named.conf.local` — Zone declarations

---

### Roy — Distributed Email System (DES)
Built the dual-server email system with SPF, DKIM, PTR, and DMARC verification:
- **Mail Server 1** (`192.168.56.20`): Exim4-heavy + Dovecot + Roundcube + MariaDB, user `dili`
- **Mail Server 2** (`192.168.56.25`): Exim4-heavy + Dovecot, user `roy2`
- Generated 2048-bit RSA DKIM key pair on Server 1 (`/etc/exim4/dkim/ozy.cns.private`)
- Published DKIM public key to DNS (`default._domainkey.ozy.cns`)
- Configured `mail2_manual` Exim router for cross-server delivery to `192.168.56.25`
- Disabled DNSSEC validation (`dns_dnssec_ok = -1`) for private lab DNS tree
- Verified TLS 1.3 encrypted transit between servers in Exim mainlog

---

## VM Layout

| VM | IP | Role | Key Software |
|---|---|---|---|
| DNS Authoritative | `192.168.56.10` | Zone authority for `ozy.cns` | BIND9 |
| Mail Server 1 | `192.168.56.20` | Primary MTA + Recursive Resolver | Exim4, Dovecot, BIND9 |
| Mail Server 2 | `192.168.56.25` | Secondary MTA, `roy2` mailbox | Exim4, Dovecot |

---

## Roy's Demo Commands

> Run these live during the presentation to prove each security layer.
> **Important:** Always use `sudo bash -c '...'` when reading Maildir files.

---

### 1. Verify All DNS Records (run on Mail Server 2: `192.168.56.25`)

```bash
# PTR — Reverse lookup for both mail servers
dig @192.168.56.10 -x 192.168.56.20 +short
# Expected: mail.ozy.cns.

dig @192.168.56.10 -x 192.168.56.25 +short
# Expected: mail2.ozy.cns.

# SPF — Authorized sender IPs
dig @192.168.56.10 ozy.cns TXT +short
# Expected: "v=spf1 ip4:192.168.56.20 ip4:192.168.56.25 mx -all"

# DKIM — Public key in DNS
dig @192.168.56.10 default._domainkey.ozy.cns TXT +short
# Expected: "v=DKIM1; k=rsa; p=MIIBIjAN..."

# DMARC — Policy record
dig @192.168.56.10 _dmarc.ozy.cns TXT +short
# Expected: "v=DMARC1; p=quarantine; rua=mailto:dili@ozy.cns; adkim=s; aspf=s"
```

---

### 2. Send a Signed Cross-Server Email (run on Mail Server 1: `192.168.56.20`)

```bash
echo "Live DES demo: SPF + DKIM + PTR + DMARC verification." | mail -s "PBL DES Assessment Demo" roy2@mail2.ozy.cns
```

---

### 3. Verify Delivery in Exim Log (run on Mail Server 1: `192.168.56.20`)

```bash
sudo tail -n 10 /var/log/exim4/mainlog
```

**What to point out in the log:**
```
=> roy2@mail2.ozy.cns R=mail2_manual T=remote_smtp H=192.168.56.25 [192.168.56.25]
   X=TLS1.3:ECDHE_SECP256R1__RSA_PSS_RSAE_SHA256__AES_256_GCM:256
   Completed
```
- `R=mail2_manual` → custom router delivered to Server 2
- `T=remote_smtp` → real SMTP delivery
- `H=192.168.56.25` → correct destination host
- `X=TLS1.3:...AES_256_GCM` → **TLS 1.3 encrypted transit** ✅

---

### 4. Inspect Full Email Headers (run on Mail Server 2: `192.168.56.25`)

```bash
# List arrived emails
sudo ls /home/roy2/Maildir/new/

# Read DKIM signature from the latest email
sudo bash -c 'cat /home/roy2/Maildir/new/* | grep -A 15 "DKIM-Signature"'

# Read full headers of the latest email
sudo bash -c 'cat /home/roy2/Maildir/new/* | grep -i "received\|dkim\|subject\|from\|to"'
```

**What to point out in the DKIM header:**
```
DKIM-Signature: v=1; a=rsa-sha256; q=dns/txt; c=relaxed/relaxed;
        d=ozy.cns; s=default;
        bh=Qgofv6ab8NNQZzeLLDYCV9JYWHkgWglyA8/RNnuDejA=;
        b=eEL/wkkmHUwx07hOA...YMu5brZQ==;
```
- `v=1; a=rsa-sha256` → DKIM signed with RSA-SHA256 ✅
- `d=ozy.cns; s=default` → Selector matches `default._domainkey.ozy.cns` DNS record ✅
- `bh=...` → Body hash proves content wasn't tampered with ✅

**What to point out in Received header:**
```
Received: from [192.168.56.20] (helo=vbox.strathmore.local)
        by mail2.ozy.cns with esmtps (TLS1.3) tls TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```
- `from [192.168.56.20]` → Source IP matches PTR and SPF ✅
- `TLS1.3 ... AES_256_GCM` → Encrypted transit ✅

---

### 5. PTR Disruption Demo
**Dili runs on DNS Server (`192.168.56.10`)**

**Step 1 — Comment out PTR record:**
```bash
sudo nano /etc/bind/db.56.168.192.in-addr.arpa
# Change:   20  IN  PTR  mail.ozy.cns.
# To:      ;20  IN  PTR  mail.ozy.cns.
sudo systemctl reload bind9
```

**Step 2 — Verify PTR is gone:**
```bash
dig @192.168.56.10 -x 192.168.56.20 +short
# Expected: (empty)
```

**Step 3 — Roy sends email from Server 1, Dili checks log on Server 2:**
```bash
# Roy runs on Server 1 (192.168.56.20):
echo "PTR disruption test" | mail -s "PTR Test" roy2@mail2.ozy.cns

# Dili checks log on Server 2 (192.168.56.25):
sudo tail -n 5 /var/log/exim4/mainlog
# Shows: no host name found for IP address 192.168.56.20
```

**Step 4 — Restore PTR record:**
```bash
sudo nano /etc/bind/db.56.168.192.in-addr.arpa
# Uncomment:  20  IN  PTR  mail.ozy.cns.
sudo systemctl reload bind9
dig @192.168.56.10 -x 192.168.56.20 +short
# Expected: mail.ozy.cns.  (restored)
```

---

### 6. SPF Disruption Demo
**Dili runs on DNS Server (`192.168.56.10`)**

**Step 1 — Comment out SPF record:**
```bash
sudo nano /etc/bind/db.ozy.cns
# Change:  @ IN TXT "v=spf1 ip4:192.168.56.20 ip4:192.168.56.25 mx -all"
# To:     ;@ IN TXT "v=spf1 ..."
sudo named-checkzone ozy.cns /etc/bind/db.ozy.cns
sudo systemctl reload bind9
```

**Step 2 — Verify SPF gone:**
```bash
dig @192.168.56.10 ozy.cns TXT +short
# Returns only DMARC line, no SPF
```

**Step 3 — Restore:**
```bash
sudo nano /etc/bind/db.ozy.cns
# Uncomment the SPF TXT line
sudo named-checkzone ozy.cns /etc/bind/db.ozy.cns
sudo systemctl reload bind9
dig @192.168.56.10 ozy.cns TXT +short
# Expected: "v=spf1 ip4:192.168.56.20 ip4:192.168.56.25 mx -all"
```

---

### 7. DKIM Disruption Demo
**Dili runs on DNS Server (`192.168.56.10`)**

**Step 1 — Comment out DKIM record:**
```bash
sudo nano /etc/bind/db.ozy.cns
# Comment the entire default._domainkey IN TXT ( ... ) block with ;
sudo named-checkzone ozy.cns /etc/bind/db.ozy.cns
sudo systemctl reload bind9
```

**Step 2 — Verify DKIM key gone:**
```bash
dig @192.168.56.10 default._domainkey.ozy.cns TXT +short
# Expected: (empty)
```

**Step 3 — Roy sends email and shows missing DKIM in headers:**
```bash
# Roy sends from Server 1:
echo "DKIM disruption test" | mail -s "DKIM Test" roy2@mail2.ozy.cns

# Check headers on Server 2 — DKIM signature still present (signed by Server 1)
# but recipient server cannot VERIFY it (key not in DNS)
sudo bash -c 'cat /home/roy2/Maildir/new/* | grep -i "dkim"' | tail -5
```

**Step 4 — Restore:**
```bash
sudo nano /etc/bind/db.ozy.cns
# Uncomment the full default._domainkey IN TXT ( ... ) block
sudo named-checkzone ozy.cns /etc/bind/db.ozy.cns
sudo systemctl reload bind9
dig @192.168.56.10 default._domainkey.ozy.cns TXT +short
# Expected: "v=DKIM1; k=rsa; p=MIIBIjAN..."
```

---

## Assessment Rubric Summary

| Item | Evidence | Command to prove it |
|---|---|---|
| PTR record for `.20` | `mail.ozy.cns.` | `dig @192.168.56.10 -x 192.168.56.20 +short` |
| PTR record for `.25` | `mail2.ozy.cns.` | `dig @192.168.56.10 -x 192.168.56.25 +short` |
| Removing PTR causes warning | `no host name found for IP address 192.168.56.20` in Exim log | Comment PTR, reload, resend, `sudo tail -n 5 /var/log/exim4/mainlog` |
| SPF TXT record | `v=spf1 ip4:...` | `dig @192.168.56.10 ozy.cns TXT +short` |
| DKIM public key in DNS | `v=DKIM1; k=rsa; p=...` | `dig @192.168.56.10 default._domainkey.ozy.cns TXT +short` |
| DKIM-Signature in email | `v=1; a=rsa-sha256; d=ozy.cns; s=default` | `sudo bash -c 'cat /home/roy2/Maildir/new/* \| grep -A 15 "DKIM-Signature"'` |
| DMARC policy record | `v=DMARC1; p=quarantine` | `dig @192.168.56.10 _dmarc.ozy.cns TXT +short` |
| Cross-server SMTP delivery | `=> roy2@mail2.ozy.cns ... Completed` | `sudo tail -n 10 /var/log/exim4/mainlog` on Server 2 |
| TLS 1.3 encrypted transit | `X=TLS1.3:ECDHE_SECP256R1__AES_256_GCM` | Same Exim mainlog line |
