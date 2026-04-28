# Security Plan — Red Team & Blue Team

**Project:** Zero Trust Corporate System  
**Author:** Asier Barranco  
**Date:** 27/04/2026  
**Version:** 1.0  

---

## 1. Purpose

This document defines the offensive and defensive security activities planned for the Zero Trust Corporate System. It establishes which attacks will be simulated, which mitigations will be applied and verified, and the minimum security standards the architecture must meet to be considered solid.

The Purple Team approach means that both offensive (Red Team) and defensive (Blue Team) activities are carried out by the same person, with the explicit goal of demonstrating that the architecture can detect, resist and recover from realistic attack scenarios.

---

## 2. Scope

The security validation covers the externally exposed perimeter and the authentication layer. It does not include internal lateral movement between private subnet services, as the project scope does not include a compromised internal host scenario.

| In scope | Out of scope |
|---|---|
| Brute force against the SSO portal | Internal lateral movement |
| Credential theft simulation (phishing) | Physical access attacks |
| Session hijacking attempts | DDoS at scale |
| IP blocking and MFA repudiation | Social engineering beyond phishing simulation |
| Log analysis and incident response | Zero-day exploit simulation |

---

## 3. Red Team — Attack Simulations

### 3.1 Attack 1 — Brute Force

**Objective:** Verify that the authentication portal and SSH access are protected against automated credential guessing.

**Method:**
- Use `hydra` or `medusa` from a Kali Linux instance to launch a dictionary attack against the Keycloak login endpoint
- Separately, launch an SSH brute force attempt against the public EC2 instance

**Success condition for the attacker:** Obtain valid credentials before being blocked.

**Expected result:** The attack is blocked after 5 failed attempts. Fail2ban bans the source IP. The account is locked in Keycloak. No valid session is obtained.

**Evidence to collect:**
- Screenshot of Fail2ban ban list after the attack
- Keycloak admin console showing the locked account
- Nginx access log showing the failed attempts and subsequent connection drops

---

### 3.2 Attack 2 — Credential Theft (Phishing simulation)

**Objective:** Verify that stolen credentials alone are not sufficient to access corporate services — MFA must repudiate the access.

**Method:**
- Simulate credential compromise by deliberately using a valid username and password (`alice.smith`) to attempt login without the TOTP code
- Attempt login with valid credentials but an incorrect TOTP code
- Attempt login with valid credentials but an expired TOTP code

**Success condition for the attacker:** Access a corporate service using only the stolen password.

**Expected result:** All three attempts are rejected by Keycloak. The user never reaches Nextcloud. The events are logged in Keycloak's admin event log.

**Evidence to collect:**
- Screenshot of Keycloak rejecting the login at the MFA step
- Keycloak event log showing the failed authentication events
- Browser screenshot confirming no session was established

---

### 3.3 Attack 3 — Session Hijacking

**Objective:** Verify that session tokens cannot be reused after logout or expiry, and that token theft does not grant persistent access.

**Method:**
- Authenticate legitimately as `alice.smith` and capture the session cookie or bearer token from the browser (DevTools → Application → Cookies / Network tab)
- Log out from the session
- Attempt to reuse the captured token to access Nextcloud directly, bypassing the Keycloak login flow
- Separately, attempt to replay the token from a different browser session

**Success condition for the attacker:** Access Nextcloud using a token from a terminated session.

**Expected result:** The replayed token is rejected. Keycloak invalidates tokens on logout. Nextcloud redirects to the Keycloak login page.

**Evidence to collect:**
- Browser screenshot showing the token capture (DevTools)
- Screenshot of the rejected replay attempt (redirect to login page)
- Keycloak session management console showing no active session for the user

---

## 4. Blue Team — Mitigations and Defences

### 4.1 Mitigations in Place (by design)

These defences are built into the architecture and active before any attack is simulated:

| Defence | Implementation | Validates |
|---|---|---|
| Account lockout | AD GPO — 5 failed attempts, 30 min lockout | Brute force resistance |
| IP banning | Fail2ban — SSH and Nginx jails | Brute force resistance |
| MFA enforcement | Keycloak — TOTP required for all users | Credential theft repudiation |
| TLS everywhere | Nginx + Let's Encrypt | Prevents credential interception in transit |
| Token expiry | Keycloak session and token TTL configuration | Session hijacking resistance |
| Single entry point | Nginx reverse proxy — all traffic routed through proxy | Reduces attack surface |
| Security Groups | AWS — only ports 22 (admin IP only) and 443 open | Perimeter hardening |

### 4.2 Incident Response — Attack 1 (Brute Force)

**Detection:** Fail2ban alert in `/var/log/fail2ban.log` — repeated 401 responses in Nginx access log.

**Response actions:**
1. Confirm the ban is active: `fail2ban-client status nginx-auth`
2. Identify the source IP and check for recurrence
3. If the IP is persistent, add it permanently to UFW: `ufw deny from <IP>`
4. Review Keycloak for any successfully locked accounts and unlock if legitimate user
5. Document the event in the incident response log

**Recovery:** No recovery needed if no successful authentication occurred. If an account was locked legitimately, unlock via Keycloak admin console.

---

### 4.3 Incident Response — Attack 2 (Credential Theft)

**Detection:** Keycloak event log — repeated `LOGIN_ERROR` events with error type `INVALID_TOTP` for a specific user.

**Response actions:**
1. Review Keycloak admin events for the affected user
2. Confirm no successful session was established
3. If suspicious pattern continues, temporarily disable the user account in Keycloak
4. Notify the simulated user (document the notification in the incident log)
5. Force a TOTP secret rotation for the affected account

**Recovery:** Re-enable the user account after the TOTP secret has been rotated and the user has re-enrolled.

---

### 4.4 Incident Response — Attack 3 (Session Hijacking)

**Detection:** Keycloak session management — token presented after session invalidation triggers a rejection event.

**Response actions:**
1. Confirm in Keycloak that the session was properly invalidated at logout
2. Verify that token TTL is set to a value that minimises the window for replay attacks (recommended: access token TTL ≤ 5 minutes)
3. If a valid replay was detected, invalidate all active sessions for the user immediately
4. Review Nginx access logs for the source IP of the replay attempt

**Recovery:** Force re-authentication for the affected user. Review token TTL configuration if the replay window was too large.

---

## 5. Minimum Security Standards

The following standards define what the architecture must be able to withstand to be considered solid. These are not aspirational — they are the baseline.

| Standard | Requirement |
|---|---|
| Brute force resistance | No account is compromised after an automated dictionary attack |
| MFA repudiation | Valid credentials alone never grant access to any corporate service |
| Session invalidation | Tokens are rejected after logout with no replay window |
| Perimeter integrity | No corporate service is reachable by bypassing the reverse proxy |
| Log traceability | Every attack attempt produces a traceable log entry |
| IP blocking | Attacking IPs are blocked within 30 seconds of the 5th failed attempt |

---

## 6. Tools

| Tool | Purpose | Origin |
|---|---|---|
| Hydra | Brute force simulation | Kali Linux |
| Browser DevTools | Token capture for session hijacking test | Built-in |
| Fail2ban | Dynamic IP blocking | Ubuntu EC2 |
| Keycloak admin console | Event log review, session management | Keycloak |
| Nginx access log | Traffic analysis | Ubuntu EC2 |
| UFW | Permanent IP blocking | Ubuntu EC2 |

---

## 7. Deliverables

Each attack simulation produces the following documentation, committed to `security/`:

| File | Content |
|---|---|
| `security/red-team/attack-01-bruteforce.md` | Attack setup, execution steps, results, evidence |
| `security/red-team/attack-02-credential-theft.md` | Attack setup, execution steps, results, evidence |
| `security/red-team/attack-03-session-hijack.md` | Attack setup, execution steps, results, evidence |
| `security/blue-team/mitigation-01-bruteforce.md` | Fail2ban logs, ban list, account lockout evidence |
| `security/blue-team/mitigation-02-mfa-repudiation.md` | Keycloak event log, screenshots |
| `security/blue-team/incident-response-log.md` | Unified incident response report with timeline and lessons learned |