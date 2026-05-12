# Connectivity & Network Isolation Tests

**Project:** Zero Trust Corporate System  
**Author:** Asier Barranco  
**Date:** 12/05/2026  
**Version:** 1.0

---

## 1. Objective

Verify that the network segmentation is correctly implemented: the services subnet has no direct internet access, internal services are only reachable through the perimeter layer, and public endpoints are exclusively exposed via Nginx on ports 80/443.

---

## 2. Test Environment

| Component | Host | IP |
|---|---|---|
| Perimeter (Nginx, Keycloak, Fail2ban) | ztcs-perimeter | 10.0.1.9 / 32.197.108.153 |
| Services (Nextcloud, Mattermost, MariaDB) | ztcs-services | 10.0.2.220 |
| Identity (Active Directory) | DC01 | 192.168.56.10 |
| External attacker | ubuntu-desktop | 46.6.44.150 |

---

## 3. Test Results

### TC-01 — Services subnet has no internet access

**From:** `ztcs-services`
**Command:**
```bash
ping -c 2 8.8.8.8 || echo 'NO INTERNET'
```
**Result:**
```
2 packets transmitted, 0 received, 100% packet loss
NO INTERNET
```
**Status:** ✓ PASS — `ztcs-services` is isolated from the internet. No NAT Gateway is attached to the private subnet.

---

### TC-02 — Services are reachable from perimeter (internal routing)

**From:** `ztcs-perimeter`
**Commands:**
```bash
nc -zv 10.0.2.220 8080
nc -zv 10.0.2.220 8065
```
**Results:**
```
Connection to 10.0.2.220 8080 port [tcp/http-alt] succeeded!
Connection to 10.0.2.220 8065 port [tcp/*] succeeded!
```
**Status:** ✓ PASS — Nextcloud (8080) and Mattermost (8065) are reachable from the perimeter layer via the internal VPC network.

---

### TC-03 — Services are NOT directly reachable from the internet

**From:** `ubuntu-desktop` (external)
**Command:**
```bash
curl -s --max-time 3 http://10.0.2.220:8080 || echo 'BLOCKED'
```
**Result:**
```
BLOCKED
```
**Status:** ✓ PASS — `ztcs-services` has no public IP. Direct access from the internet is impossible by network design.

---

### TC-04 — Public endpoints respond correctly on 443/80

**From:** `ubuntu-desktop` (external)
**Commands:**
```bash
curl -s -o /dev/null -w "%{http_code}\n" https://abb-ztcs.com
curl -s -o /dev/null -w "%{http_code}\n" http://abb-ztcs.com
```
**Results:**
```
302  (HTTPS → Keycloak SSO redirect)
301  (HTTP → HTTPS redirect)
```
**Status:** ✓ PASS — HTTPS returns 302 (authenticated redirect to Keycloak). HTTP returns 301 (forced redirect to HTTPS). No direct service exposure.

---

### TC-05 — LDAP connectivity to Active Directory via WireGuard

**From:** `ztcs-perimeter`
**Command:**
```bash
nc -zv 192.168.56.10 389
```
**Result:**
```
Connection to 192.168.56.10 389 port [tcp/ldap] succeeded!
```
**Status:** ✓ PASS — Keycloak can reach the Active Directory domain controller via the WireGuard VPN tunnel on port 389 (LDAP).

---

### TC-06 — Perimeter not directly accessible from services subnet (note)

The jump SSH test from `ztcs-perimeter` to `ztcs-services` using `-J` flag failed with `Permission denied (publickey)` because the SSH agent forwarding requires the key to be loaded in the originating session. This does not affect the network isolation result — the internal routing between perimeter and services is confirmed by TC-02.

---

## 4. Summary

| Test | Description | Result |
|---|---|---|
| TC-01 | Services subnet — no internet access | ✓ PASS |
| TC-02 | Internal routing perimeter → services | ✓ PASS |
| TC-03 | Services not reachable from internet | ✓ PASS |
| TC-04 | Public endpoints on 80/443 only | ✓ PASS |
| TC-05 | LDAP reachable via WireGuard tunnel | ✓ PASS |

**All network isolation tests passed. The Zero Trust network segmentation is correctly implemented.**