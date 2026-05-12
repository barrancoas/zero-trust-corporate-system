# Administrator Manual — Zero Trust Corporate System

**Project:** Zero Trust Corporate System  
**Author:** Asier Barranco  
**Date:** 12/05/2026  
**Version:** 1.0  
**Audience:** System administrators  

---

## 1. System Overview

The Zero Trust Corporate System is a hybrid infrastructure implementing the Zero Trust security model. No user, device or network is trusted by default — every access request is authenticated, authorised and encrypted regardless of origin.

### 1.1 Architecture Summary

The system is organised in three layers:

| Layer | Role | Components |
|---|---|---|
| Layer 1 — Identity & Control | Centralised identity directory | Active Directory (DC01) |
| Layer 2 — Services & Data | Corporate applications and databases | Nextcloud, Mattermost, MariaDB |
| Layer 3 — Perimeter | Single entry point, authentication gateway | Nginx, Keycloak, Fail2ban, UFW |

**Infrastructure:**
- **On-premise:** Windows Server 2022 running Active Directory — hosted in VirtualBox
- **Cloud (AWS):** two EC2 instances in a VPC with public/private subnet segmentation
- **Connectivity:** WireGuard VPN tunnel connecting AWS perimeter to on-premise DC01

### 1.2 Network Topology

| Host | Role | Private IP | Public IP |
|---|---|---|---|
| DC01 | Active Directory, DNS, DHCP | 192.168.56.10 | None |
| ztcs-perimeter | Nginx, Keycloak, Fail2ban, UFW | 10.0.1.9 | 32.197.108.153 |
| ztcs-services | Nextcloud, Mattermost, MariaDB | 10.0.2.220 | None |

**Domain:** `abb-ztcs.com` → `32.197.108.153` (IONOS DNS)
**VPN subnet:** `10.10.0.0/24` (WireGuard)
**AD domain:** `corp.zerotrust.local`

### 1.3 Repository Structure

```
zero-trust-corporate-system/
├── architecture/          # Network diagrams
├── docs/                  # Project documentation (en + es)
├── infrastructure/        # Deployment guides per component
│   ├── aws/
│   ├── domain-controller/
│   ├── identity-provider/
│   ├── reverse-proxy/
│   ├── services/
│   └── vpn/
├── media/                 # Screenshots and evidence
├── security/              # Purple Team exercises
├── sprints/               # SCRUM sprint documentation
└── tests/                 # Test plans and validation results
```

---

## 2. Component Reference

### 2.1 Active Directory (DC01)

**Setup guide:** `infrastructure/domain-controller/dc-setup.md`

| Parameter | Value |
|---|---|
| OS | Windows Server 2022 |
| Domain | `corp.zerotrust.local` |
| IP | `192.168.56.10` |
| Administrator | `CORP\zt-administrator` |
| LDAP port | 389 |

**Organisational Units:**
- `OU=ZeroTrust Users` — corporate user accounts
- `OU=ZeroTrust Computers` — workstations
- `OU=ZeroTrust Groups` — security groups

**Key users:**

| Username | Role |
|---|---|
| alice.smith | Standard corporate user |
| bob.jones | Standard corporate user |
| zt.admin | IT administrator |
| svc.keycloak | Keycloak LDAP service account (read-only) |

**GPO policies applied:**
- Password policy: min 12 characters, complexity required, 90-day expiry
- Account lockout: 5 failed attempts → 30 minutes lockout
- Security audit: logon events, account management, privilege use

**Start/stop DC01:**
- Start VirtualBox VM `DC01`
- Log in as `CORP\zt-administrator`
- Verify AD services: `Get-Service NTDS, DNS, Netlogon`

---

### 2.2 WireGuard VPN

**Setup guide:** `infrastructure/vpn/wireguard-setup.md`

The WireGuard tunnel connects `ztcs-perimeter` (AWS) to `DC01` (on-premise), enabling Keycloak to reach Active Directory via LDAP.

**Perimeter config:** `infrastructure/vpn/wg0-perimeter.conf`
**DC01 config:** `infrastructure/vpn/wg0-dc01.conf`

**Verify tunnel status (ztcs-perimeter):**
```bash
sudo wg show
```

**Verify LDAP reachability:**
```bash
nc -zv 192.168.56.10 389
```

**Restart WireGuard (ztcs-perimeter):**
```bash
sudo systemctl restart wg-quick@wg0
```

> **Important:** the WireGuard tunnel must be active at all times for authentication to work. If the tunnel is down, Keycloak cannot validate credentials against AD and all logins will fail.

---

### 2.3 AWS Infrastructure

**Setup guide:** `infrastructure/aws/aws-setup.md`

**VPC:** `10.0.0.0/16`

| Subnet | CIDR | Type | Hosts |
|---|---|---|---|
| Public subnet | `10.0.1.0/24` | Public (Internet Gateway) | ztcs-perimeter |
| Private subnet | `10.0.2.0/24` | Private (no internet) | ztcs-services |

**Security Groups:**

| Group | Inbound rules |
|---|---|
| sg-perimeter | 22/tcp, 80/tcp, 443/tcp, 51820/udp from 0.0.0.0/0 |
| sg-services | All traffic from sg-perimeter only |

**SSH access to ztcs-perimeter:**
```bash
ssh -i /path/to/labsuser.pem ubuntu@32.197.108.153
```

**SSH access to ztcs-services (via jump host):**
```bash
eval $(ssh-agent) && ssh-add /path/to/labsuser.pem
ssh -A -J ubuntu@32.197.108.153 ubuntu@10.0.2.220
```

> **Cost management:** stop EC2 instances when not in use to preserve AWS credits. Always stop `ztcs-services` first, then `ztcs-perimeter`.

---

### 2.4 Nginx — Reverse Proxy

**Setup guide:** `infrastructure/reverse-proxy/nginx-setup.md`
**Config file:** `infrastructure/reverse-proxy/abb-ztcs.com.conf`

Nginx is the single entry point for all external traffic. It terminates TLS and proxies to internal services.

**Routing table:**

| Path | Proxied to | Service |
|---|---|---|
| `https://abb-ztcs.com/` | `http://10.0.2.220:8080` | Nextcloud |
| `https://abb-ztcs.com/auth/` | `http://localhost:8080` | Keycloak |
| `https://abb-ztcs.com/mattermost/` | `http://10.0.2.220:8065` | Mattermost |

**Key commands:**
```bash
sudo systemctl status nginx
sudo systemctl restart nginx
sudo nginx -t                          # Test configuration syntax
sudo certbot renew --dry-run           # Test certificate auto-renewal
```

**TLS Certificate:**
- Provider: Let's Encrypt (Certbot)
- Domain: `abb-ztcs.com`
- Expiry: 2026-08-07
- Auto-renewal: configured via systemd timer

---

### 2.5 Keycloak — Identity Provider

**Setup guide:** `infrastructure/identity-provider/keycloak-setup.md`
**Docker Compose:** `infrastructure/identity-provider/docker-compose.yml`

| Parameter | Value |
|---|---|
| Version | 24.0.4 |
| Admin URL | `https://abb-ztcs.com/auth/` (browser) or `http://localhost:8080/auth/` (SSH tunnel) |
| Admin credentials | `admin` / `KeycloakAdmin2026!` |
| Realm | `zerotrust` |
| LDAP provider | `corp.zerotrust.local` via `192.168.56.10:389` |
| SAML client | `https://abb-ztcs.com` (Nextcloud) |

**Key commands (ztcs-perimeter):**
```bash
docker ps | grep keycloak
docker logs keycloak --tail 50
docker restart keycloak
cd ~/keycloak && docker compose up -d
```

**Access admin console via SSH tunnel (avoids CORS issues):**
```bash
ssh -i /path/to/labsuser.pem -L 8080:localhost:8080 ubuntu@32.197.108.153
# Then open http://localhost:8080/auth/ in browser
```

**Sync LDAP users manually:**
Admin console → realm `zerotrust` → Identity Providers → User Federation → `ldap` → **Sync all users**

**Reset user MFA:**
Admin console → realm `zerotrust` → Users → select user → Credentials → delete OTP entry

---

### 2.6 Fail2ban & UFW — Perimeter Protection

**Setup guide:** `infrastructure/reverse-proxy/perimeter-protection-setup.md`
**Config:** `infrastructure/reverse-proxy/jail.local` and `infrastructure/reverse-proxy/keycloak.conf`

**Active jails:**

| Jail | Trigger | Ban duration |
|---|---|---|
| sshd | 3 failed SSH attempts | 1 hour |
| keycloak | 5 LOGIN_ERROR events via journald | 30 minutes |
| nginx-http-auth | Failed HTTP auth | 1 hour |
| nginx-botsearch | Bot scanning patterns | 1 hour |

**Key commands:**
```bash
sudo fail2ban-client status
sudo fail2ban-client status keycloak
sudo fail2ban-client set keycloak unbanip <IP>
sudo grep "Ban" /var/log/fail2ban.log | tail -20
```

**UFW open ports:**
```
22/tcp    SSH
80/tcp    HTTP (redirected to HTTPS)
443/tcp   HTTPS
51820/udp WireGuard VPN
```

---

### 2.7 Nextcloud & Mattermost — Corporate Services

**Setup guide:** `infrastructure/services/services-setup.md`
**SSO guide:** `infrastructure/services/sso-nextcloud-setup.md`
**Docker Compose:** `infrastructure/services/docker-compose.yml`

All services run as Docker containers on `ztcs-services`.

**Key commands (ztcs-services):**
```bash
docker ps
docker compose -f ~/services/docker-compose.yml up -d
docker compose -f ~/services/docker-compose.yml down
docker logs nextcloud --tail 50
docker logs mattermost --tail 50
docker logs mariadb --tail 50
```

**Nextcloud occ commands:**
```bash
docker exec -u 33 -it nextcloud php occ user:list
docker exec -u 33 -it nextcloud php occ app:list
docker exec -u 33 -it nextcloud php occ saml:config:get --providerId=1
```

**Emergency — disable SAML (if locked out):**
```bash
docker exec -u 33 -it nextcloud php occ app:disable user_saml
# Access restored via standard login at https://abb-ztcs.com/login?direct=1
```

**Local admin access (bypasses SSO):**
```
https://abb-ztcs.com/login?direct=1
Credentials: nc_admin / AdminZeroTrust2026!
```

---

### 2.8 MariaDB — Database

MariaDB runs as a Docker container on `ztcs-services` and is used exclusively as a backend by Nextcloud and Mattermost. It has no public exposure.

```bash
# Access MariaDB shell
docker exec -it mariadb mariadb -u root -p
# Password: MariaZeroTrust2026!
```

**Databases:**

| Database | User | Used by |
|---|---|---|
| nextcloud | nextcloud_user | Nextcloud |
| mattermost | mattermost_user | Mattermost |

---

## 3. Operational Procedures

### 3.1 Starting the Full System

Follow this order to bring the system up from a cold start:

1. **Start DC01** in VirtualBox — wait for AD services to initialise (~2 minutes)
2. **Start ztcs-perimeter** EC2 instance in AWS console
3. **Start ztcs-services** EC2 instance in AWS console
4. **Verify WireGuard tunnel** from ztcs-perimeter: `nc -zv 192.168.56.10 389`
5. **Verify all services:** `docker ps` on both instances
6. **Verify public endpoint:** `curl -s -o /dev/null -w "%{http_code}\n" https://abb-ztcs.com`

### 3.2 Stopping the Full System

1. Stop ztcs-services EC2 instance
2. Stop ztcs-perimeter EC2 instance
3. Shut down DC01 VM in VirtualBox

### 3.3 Adding a New User

1. Create user in Active Directory (DC01):
```powershell
New-ADUser -Name "John Doe" -SamAccountName "john.doe" `
  -UserPrincipalName "john.doe@corp.zerotrust.local" `
  -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
  -Enabled $true -Path "OU=ZeroTrust Users,DC=corp,DC=zerotrust,DC=local"
```

2. Sync users in Keycloak: Admin console → User Federation → `ldap` → **Sync all users**

3. User can now log in at `https://abb-ztcs.com` — Nextcloud account is created automatically on first login.

### 3.4 Renewing the TLS Certificate

Certbot renews automatically via systemd timer. To renew manually:

```bash
sudo certbot renew
sudo systemctl reload nginx
```

Verify expiry:
```bash
echo | openssl s_client -connect abb-ztcs.com:443 2>/dev/null | openssl x509 -noout -dates
```

### 3.5 Viewing Logs

| Component | Command |
|---|---|
| Nginx access | `sudo tail -f /var/log/nginx/access.log` |
| Nginx errors | `sudo tail -f /var/log/nginx/error.log` |
| Keycloak | `docker logs keycloak -f` or `journalctl CONTAINER_TAG=keycloak -f` |
| Fail2ban | `sudo tail -f /var/log/fail2ban.log` |
| Nextcloud | `docker exec -it nextcloud cat data/nextcloud.log \| tail -50` |
| MariaDB | `docker logs mariadb --tail 50` |

### 3.6 Backing Up Data

**Nextcloud data:**
```bash
docker exec -it nextcloud tar czf /tmp/nextcloud-backup.tar.gz /var/www/html/data
docker cp nextcloud:/tmp/nextcloud-backup.tar.gz ~/backups/
```

**MariaDB:**
```bash
docker exec mariadb mariadb-dump -u root -pMariaZeroTrust2026! --all-databases > ~/backups/mariadb-$(date +%Y%m%d).sql
```

**Keycloak realm config:**
```bash
# Export realm via API
TOKEN=$(curl -s -X POST http://localhost:8080/auth/realms/master/protocol/openid-connect/token \
  -d "username=admin&password=KeycloakAdmin2026%21&grant_type=password&client_id=admin-cli" \
  | grep -o '"access_token":"[^"]*"' | cut -d'"' -f4)

curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/auth/admin/realms/zerotrust > ~/backups/keycloak-realm-$(date +%Y%m%d).json
```

---

## 4. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Login fails with "Unexpected error" | DC01 offline or WireGuard tunnel down | Start DC01, verify `nc -zv 192.168.56.10 389` |
| `https://abb-ztcs.com` unreachable | Nginx stopped or EC2 instance down | `sudo systemctl restart nginx` or start EC2 |
| Keycloak 502 Bad Gateway | Keycloak container not running | `docker ps` → `docker compose up -d` in `~/keycloak/` |
| Nextcloud shows blank page | Container down or out of memory | `docker restart nextcloud` |
| IP blocked by Fail2ban | Too many failed attempts | `sudo fail2ban-client set <jail> unbanip <IP>` |
| User cannot log in — "Account not provisioned" | SAML uid mapping issue | Check `occ saml:config:get --providerId=1` |
| User locked out of AD | Too many failed login attempts | `Unlock-ADAccount -Identity <username>` on DC01 |
| Certificate expired | Auto-renewal failed | `sudo certbot renew && sudo systemctl reload nginx` |

---

## 5. Security Considerations

| Area | Implementation | Notes |
|---|---|---|
| SSH authentication | Key-only — password auth disabled | Prevents brute force at protocol level |
| Network isolation | ztcs-services has no public IP or internet | Services only reachable via perimeter |
| TLS | Let's Encrypt — auto-renewed | All traffic encrypted in transit |
| MFA | TOTP enforced for all users | Stolen credentials alone are insufficient |
| Intrusion prevention | Fail2ban with 4 jails | Dynamic IP banning via UFW integration |
| LDAP traffic | Transmitted over WireGuard tunnel | Encrypted point-to-point VPN |
| Session security | HttpOnly, Secure, SameSite cookies | Mitigates XSS and CSRF attacks |

---

## 6. Documentation Index

| Document | Path | Content |
|---|---|---|
| AWS setup | `infrastructure/aws/aws-setup.md` | VPC, subnets, EC2, Security Groups |
| DNS & Elastic IP | `infrastructure/aws/dns-elastic-ip-setup.md` | Domain configuration |
| Domain controller | `infrastructure/domain-controller/dc-setup.md` | AD, GPO, users, groups |
| WireGuard VPN | `infrastructure/vpn/wireguard-setup.md` | VPN tunnel configuration |
| Keycloak | `infrastructure/identity-provider/keycloak-setup.md` | IdP, LDAP federation, MFA |
| Nginx | `infrastructure/reverse-proxy/nginx-setup.md` | Reverse proxy, TLS |
| Perimeter protection | `infrastructure/reverse-proxy/perimeter-protection-setup.md` | Fail2ban, UFW |
| Services deployment | `infrastructure/services/services-setup.md` | Nextcloud, Mattermost, MariaDB |
| SSO integration | `infrastructure/services/sso-nextcloud-setup.md` | SAML SSO configuration |
| Network tests | `tests/connectivity-tests.md` | Isolation and routing validation |
| SSO tests | `tests/sso-validation.md` | Authentication validation |
| Test plan | `tests/test-plan.md` | Complete test summary |
| Purple Team | `security/purple-team.md` | Attack simulations and incident response |
| User manual | `docs/en/user-manual.md` | End user guide |