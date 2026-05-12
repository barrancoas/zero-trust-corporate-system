# Test Plan — Zero Trust Corporate System

**Project:** Zero Trust Corporate System   
**Author:** Asier Barranco  
**Date:** 12/05/2026  
**Version:** 1.0  

---

## 1. Objective

This document defines the complete test plan for the Zero Trust Corporate System. It consolidates all validation activities performed across the project, covering network isolation, identity federation, SSO authentication, MFA enforcement, perimeter security and Purple Team attack simulations.

The goal is to verify that the deployed system meets the security and functional requirements defined in the project proposal, and that all components operate correctly as an integrated whole.

---

## 2. Scope

| Area | Components tested |
|---|---|
| Network isolation | AWS VPC subnets, Security Groups, NAT Gateway absence |
| Perimeter security | Nginx TLS, security headers, HTTP→HTTPS redirect, Fail2ban, UFW |
| Identity federation | Keycloak LDAP federation with Active Directory |
| Authentication | SSO SAML 2.0 flow, MFA TOTP enforcement |
| Access control | Nextcloud SAML provisioning, session validation |
| Attack simulation | Brute force, credential theft, session hijacking |

---

## 3. Test Environment

| Component | Role | IP / URL |
|---|---|---|
| ztcs-perimeter | Nginx, Keycloak, Fail2ban, UFW | `10.0.1.9` / `32.197.108.153` |
| ztcs-services | Nextcloud, Mattermost, MariaDB | `10.0.2.220` |
| DC01 | Active Directory, DNS, DHCP | `192.168.56.10` |
| Corporate portal | Public entry point | `https://abb-ztcs.com` |
| Test user | AD federated user | `alice.smith` |
| Attacking machine | External client | `46.6.44.150` |

---

## 4. Test Cases

### 4.1 Network & Connectivity Tests

Detailed results: `tests/connectivity-tests.md`

| ID | Test | Expected | Result |
|---|---|---|---|
| TC-01 | ztcs-services has no internet access | 100% packet loss to 8.8.8.8 | ✓ PASS |
| TC-02 | Internal routing perimeter → services | nc succeeds on 8080, 8065 | ✓ PASS |
| TC-03 | Services not reachable from internet | Connection timeout | ✓ PASS |
| TC-04 | Public endpoints on 80/443 only | 302 (HTTPS), 301 (HTTP→HTTPS) | ✓ PASS |
| TC-05 | LDAP reachable via WireGuard tunnel | nc succeeds on 192.168.56.10:389 | ✓ PASS |

### 4.2 Perimeter Security Tests

| ID | Test | Expected | Result |
|---|---|---|---|
| TC-06 | HTTP redirected to HTTPS | 301 response | ✓ PASS |
| TC-07 | HSTS header present | `max-age=31536000; includeSubDomains` | ✓ PASS |
| TC-08 | Security headers present | X-Frame-Options, X-Content-Type-Options, CSP | ✓ PASS |
| TC-09 | TLS certificate valid | Let's Encrypt — expires 2026-08-07 | ✓ PASS |
| TC-10 | Fail2ban jails active | 4 jails: sshd, keycloak, nginx-http-auth, nginx-botsearch | ✓ PASS |
| TC-11 | UFW default deny incoming | Only 22, 80, 443, 51820 open | ✓ PASS |

### 4.3 SSO & Access Control Tests

Detailed results: `tests/sso-validation.md`

| ID | Test | Expected | Result |
|---|---|---|---|
| TC-12 | Keycloak realm accessible | 200 OK | ✓ PASS |
| TC-13 | Nextcloud redirects to Keycloak | SAML redirect on unauthenticated access | ✓ PASS |
| TC-14 | AD user authenticates via SSO + MFA | Dashboard accessible after TOTP | ✓ PASS |
| TC-15 | User auto-provisioned in Nextcloud | alice.smith appears in occ user:list | ✓ PASS |
| TC-16 | Unauthenticated access denied | 302 redirect to Keycloak | ✓ PASS |
| TC-17 | Local admin bypass available | 200 at /login?direct=1 | ✓ PASS |
| TC-18 | MFA API bypass blocked | unauthorized_client error | ✓ PASS |

### 4.4 Purple Team — Attack Simulations

Detailed results: `security/purple-team.md`

| ID | Test | Expected | Result |
|---|---|---|---|
| TC-19 | SSH brute force via Hydra | Password auth rejected at protocol level | ✓ PASS |
| TC-20 | Keycloak login brute force (10 attempts) | IP banned by Fail2ban after 5 attempts | ✓ PASS |
| TC-21 | Credential theft — login with stolen password | Blocked at MFA prompt | ✓ PASS |
| TC-22 | MFA bypass via direct API call | unauthorized_client — no token issued | ✓ PASS |
| TC-23 | Session hijacking — HTTP interception | 301 redirect — no cookies transmitted | ✓ PASS |
| TC-24 | Session hijacking — forged cookie | 401 Unauthorized | ✓ PASS |
| TC-25 | Session hijacking — real cookie replay | 401 Unauthorized — session context required | ✓ PASS |

---

## 5. Defence-in-Depth Validation

The following table maps each security layer to the test cases that validate it:

| Layer | Component | Validated by |
|---|---|---|
| 1 — Network | AWS Security Groups | TC-03, TC-04 |
| 2 — Host firewall | UFW | TC-11, TC-20 |
| 3 — Intrusion prevention | Fail2ban | TC-10, TC-20 |
| 4 — Perimeter | Nginx TLS + headers | TC-06, TC-07, TC-08, TC-09 |
| 5 — Authentication | Keycloak SSO + MFA | TC-13, TC-14, TC-21, TC-22 |
| 6 — Directory | Active Directory + GPO | TC-05, TC-15 |
| 7 — Application | Nextcloud session validation | TC-16, TC-24, TC-25 |

---

## 6. End-to-End Functional Test

The following sequence validates the complete system flow as a single integrated test:

1. User navigates to `https://abb-ztcs.com` → redirected to Keycloak ✓
2. User enters AD credentials → validated against DC01 via LDAP over WireGuard ✓
3. Keycloak enforces TOTP → user enters 6-digit code ✓
4. SAML assertion issued → Nextcloud validates and creates session ✓
5. User accesses dashboard → uploads a file to Nextcloud ✓
6. User logs out → Keycloak session terminated ✓

**Result:** ✓ PASS — full Zero Trust chain validated end-to-end.

A video recording of this flow is available as a demo backup for the project defence.

---

## 7. Known Limitations

| Limitation | Impact | Mitigation |
|---|---|---|
| LDAP not encrypted (port 389) | Credentials in plaintext within VPN tunnel | WireGuard encrypts all traffic between perimeter and DC01 |
| Mattermost without SSO | Users require separate local credentials | Out of scope for this project iteration |
| SSH open to 0.0.0.0/0 | Broader attack surface | Key-only authentication enforced — password auth disabled |
| Keycloak in dev mode | Not suitable for production | Acceptable for project scope — documented in recommendations |

---

## 8. Summary

| Category | Total tests | Passed | Failed |
|---|---|---|---|
| Network & Connectivity | 5 | 5 | 0 |
| Perimeter Security | 6 | 6 | 0 |
| SSO & Access Control | 7 | 7 | 0 |
| Purple Team | 7 | 7 | 0 |
| **Total** | **25** | **25** | **0** |

**All 25 test cases passed. The Zero Trust Corporate System meets the security and functional requirements defined in the project proposal.**