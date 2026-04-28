# Acceptance Criteria and Performance Standards

**Project:** Zero Trust Corporate System  
**Author:** Asier Barranco  
**Date:** 27/04/2026  
**Version:** 1.1  

---

## 1. Purpose

This document defines the acceptance criteria and minimum performance standards for each component of the Zero Trust Corporate System. A component is considered functional and accepted when all its criteria are met. These criteria serve as the reference for the validation phase (Sprint 2, Phase 9) and the final test plan.

### Priority levels

Each criterion is tagged with one of the following priority levels:

- **[CORE]** — Mandatory. The project is not considered complete without this.
- **[OPTIONAL]** — Desirable. Adds value but will not be considered a failure if not achieved within the available time.

---

## 2. Acceptance Criteria by Component

### 2.1 AWS Infrastructure

| ID | Priority | Criterion | Accepted when |
|---|---|---|---|
| AWS-01 | **[CORE]** | VPC and subnets created | A public subnet and a private subnet exist within the VPC with correct CIDR ranges and routing tables |
| AWS-02 | **[CORE]** | Internet connectivity | EC2 instances in the public subnet can reach the internet outbound |
| AWS-03 | **[CORE]** | Private subnet isolation | EC2 instances in the private subnet have no direct inbound or outbound internet routing |
| AWS-04 | **[CORE]** | Security Groups | Only ports explicitly required by each service are open; all others are denied by default |
| AWS-05 | **[CORE]** | EC2 instances reachable | Public-facing instances respond to SSH from the administrator IP and to HTTPS on port 443 |

---

### 2.2 On-premise Windows Server / Active Directory

| ID | Priority | Criterion | Accepted when |
|---|---|---|---|
| AD-01 | **[CORE]** | Domain controller operational | `Get-ADDomain` and `Get-ADDomainController` return valid output without errors |
| AD-02 | **[CORE]** | DNS resolution | `Resolve-DnsName corp.zerotrust.local` resolves correctly from inside the domain |
| AD-03 | **[CORE]** | OU structure | OUs `Users`, `Groups`, `Computers`, `ServiceAccounts` and `Admins` exist under `ZeroTrust` |
| AD-04 | **[CORE]** | Test users created | Users `alice.smith`, `bob.jones` and `zt.admin` exist and can authenticate against the domain |
| AD-05 | **[CORE]** | Service account operational | `svc.keycloak` exists in `ServiceAccounts` and has LDAP read permissions on user objects |
| AD-06 | **[CORE]** | Security groups | `GRP-Users`, `GRP-Admins`, `GRP-DocMgmt` and `GRP-Comms` exist with correct members |
| AD-07 | **[CORE]** | GPO — Password policy | Minimum 12 characters and complexity requirements enforced; verified via `gpresult /r` |
| AD-08 | **[CORE]** | GPO — Account lockout | Account is locked after 5 failed attempts; verified by deliberate failed login test |
| AD-09 | **[CORE]** | LDAP connectivity | Keycloak service account can bind to `ldap://192.168.56.10:389` and query user objects |

---

### 2.3 WireGuard VPN Tunnel

| ID | Priority | Criterion | Accepted when |
|---|---|---|---|
| VPN-01 | **[CORE]** | Tunnel established | `wg show` on the Linux peer reports a valid handshake with the Windows peer |
| VPN-02 | **[CORE]** | Bidirectional connectivity | The AWS private subnet can ping `192.168.56.10` and the Windows Server can ping the AWS private subnet gateway |
| VPN-03 | **[OPTIONAL]** | Encrypted traffic verification | Traffic between on-premise and AWS is confirmed to pass through the WireGuard interface via packet capture |

---

### 2.4 Identity Provider (Keycloak)

| ID | Priority | Criterion | Accepted when |
|---|---|---|---|
| KC-01 | **[CORE]** | Keycloak operational | The Keycloak admin console is accessible at its URL and responds with HTTP 200 |
| KC-02 | **[CORE]** | AD federation via LDAP | Users from Active Directory are visible in the Keycloak user list after LDAP sync |
| KC-03 | **[CORE]** | SSO flow | A user from AD can log in to a protected application through Keycloak without re-entering credentials once authenticated |
| KC-04 | **[CORE]** | MFA enforced | After password authentication, the user is prompted for a TOTP code before access is granted |
| KC-05 | **[CORE]** | MFA repudiation | A login attempt using valid credentials but an incorrect or missing TOTP code is rejected |
| KC-06 | **[CORE]** | Token issuance | Upon successful authentication, Keycloak issues a valid OIDC/SAML token accepted by the protected application |

---

### 2.5 Reverse Proxy (Nginx)

| ID | Priority | Criterion | Accepted when |
|---|---|---|---|
| NX-01 | **[CORE]** | Single entry point | All corporate services are accessible only through the proxy — direct access to internal IPs is blocked |
| NX-02 | **[CORE]** | TLS enforced | All connections are served over HTTPS; HTTP requests are redirected to HTTPS with status 301 |
| NX-03 | **[CORE]** | Valid certificate | The TLS certificate is issued by Let's Encrypt, is not self-signed and does not trigger browser warnings |
| NX-04 | **[CORE]** | Certificate auto-renewal | Certbot renew runs without errors; certificate expiry is more than 60 days after renewal |
| NX-05 | **[CORE]** | Security headers | Response headers include at minimum: `Strict-Transport-Security`, `X-Frame-Options`, `X-Content-Type-Options` |
| NX-06 | **[CORE]** | Routing | Requests to each subdomain or path are correctly proxied to the corresponding internal service |

---

### 2.6 Active Protection (Fail2ban + UFW + Security Groups)

| ID | Priority | Criterion | Accepted when |
|---|---|---|---|
| FW-01 | **[CORE]** | UFW active | `ufw status` reports active with explicit allow rules for required ports only |
| FW-02 | **[CORE]** | Fail2ban active | `fail2ban-client status` reports active jails for SSH and Nginx |
| FW-03 | **[CORE]** | IP blocking | After 5 failed SSH or login attempts from a test IP, that IP is added to the Fail2ban ban list |
| FW-04 | **[CORE]** | Ban verification | A banned IP receives no response (connection timeout) when attempting to reach the proxy |

---

### 2.7 Relational Database (MariaDB)

| ID | Priority | Criterion | Accepted when |
|---|---|---|---|
| DB-01 | **[CORE]** | Service running | `systemctl status mariadb` (or Docker equivalent) reports active and running |
| DB-02 | **[CORE]** | No external exposure | The database port (3306) is not reachable from outside the private subnet |
| DB-03 | **[CORE]** | Dedicated users | Corporate services connect with dedicated database users with minimum required permissions |
| DB-04 | **[CORE]** | Root remote access disabled | Remote login as root is rejected |
| DB-05 | **[OPTIONAL]** | Automated backup | A backup script runs successfully and produces a valid dump file |

---

### 2.8 Document Management Service (Nextcloud)

Nextcloud is the **primary corporate service** for this project. Full SSO integration with Keycloak is a core requirement.

| ID | Priority | Criterion | Accepted when |
|---|---|---|---|
| NC-01 | **[CORE]** | Service accessible | Nextcloud is reachable through the reverse proxy at its designated URL |
| NC-02 | **[CORE]** | SSO authentication | A user from Active Directory can log in to Nextcloud via Keycloak SSO without entering a separate Nextcloud password |
| NC-03 | **[CORE]** | MFA enforced | The SSO login flow requires TOTP before granting access to Nextcloud |
| NC-04 | **[CORE]** | No direct exposure | Nextcloud is not reachable by bypassing the reverse proxy (direct IP access is blocked) |
| NC-05 | **[CORE]** | File operations | An authenticated user can upload, download and delete files without errors |

---

### 2.9 Communications Service (Mattermost)

Mattermost is a **secondary corporate service**. Deployment and SSO integration are desirable but not critical if time does not allow.

| ID | Priority | Criterion | Accepted when |
|---|---|---|---|
| MM-01 | **[OPTIONAL]** | Service accessible | Mattermost is reachable through the reverse proxy at its designated URL |
| MM-02 | **[OPTIONAL]** | SSO authentication | A user from Active Directory can log in to Mattermost via Keycloak SSO without a separate Mattermost password |
| MM-03 | **[OPTIONAL]** | MFA enforced | The SSO login flow requires TOTP before granting access to Mattermost |
| MM-04 | **[OPTIONAL]** | No direct exposure | Mattermost is not reachable by bypassing the reverse proxy |
| MM-05 | **[OPTIONAL]** | Messaging | An authenticated user can send and receive messages without errors |

---

### 2.10 Log Monitoring Script

| ID | Priority | Criterion | Accepted when |
|---|---|---|---|
| LOG-01 | **[OPTIONAL]** | Script executes | The script runs without errors via `bash` or as a cron job |
| LOG-02 | **[OPTIONAL]** | Log parsing | The script correctly identifies and reports failed authentication attempts from Nginx and SSH logs |
| LOG-03 | **[OPTIONAL]** | Scheduled execution | The cron entry is active and the script runs at the defined interval |

---

### 2.11 End-to-End System Validation

| ID | Priority | Criterion | Accepted when |
|---|---|---|---|
| E2E-01 | **[CORE]** | Full authentication flow | A user from AD authenticates via Keycloak (password + TOTP), is redirected through the proxy, and accesses Nextcloud in a single session without re-authenticating |
| E2E-02 | **[CORE]** | Access denial | A user not belonging to `GRP-DocMgmt` is denied access to Nextcloud |
| E2E-03 | **[CORE]** | Network isolation | No corporate service responds to direct requests that bypass the reverse proxy |
| E2E-04 | **[OPTIONAL]** | VPN dependency | If the WireGuard tunnel goes down, Keycloak cannot sync with AD and authentication fails |

---

## 3. Minimum Performance Standards

| Metric | Minimum standard | Priority |
|---|---|---|
| SSO authentication response time | < 3 seconds from login submission to service access | **[CORE]** |
| Proxy response time (HTTPS) | < 1 second for static requests under normal load | **[CORE]** |
| Failed login lockout time | ≤ 30 seconds after the 5th failed attempt | **[CORE]** |
| Certificate validity | > 60 days remaining at any point | **[CORE]** |
| Backup execution time | Completes without error within 5 minutes | **[OPTIONAL]** |
| VPN tunnel recovery | Tunnel re-establishes within 60 seconds of a restart | **[OPTIONAL]** |

---

## 4. Out of Scope

The following criteria are explicitly outside the scope of this project and will not be validated:

- High availability or load balancing (single-instance deployment)
- DDoS resistance at scale
- Mobile device management (MDM)
- Email service integration