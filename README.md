# Zero Trust Corporate System

> Design and implementation of a corporate Zero Trust ecosystem with federated identity, MFA enforcement and security auditing — deployed on a hybrid on-premise + AWS infrastructure.

**Author:** Asier Barranco
**Cycle:** CFGS ASIX — Cybersecurity Profile
**Centre:** Institut Tecnològic de Barcelona
**Academic year:** 2025–2026
**Project period:** 13/04/2026 → 12/05/2026
**Defence:** 20/05/2026

---

## The Problem

Traditional perimeter-based security models are no longer sufficient. In a landscape shaped by remote work and cloud migration, the concept of a trusted internal network has become obsolete. Modern threats routinely exploit compromised credentials or vulnerable internal devices to move laterally from inside the network — making the perimeter itself irrelevant.

This project responds to that reality by designing and deploying a **Zero Trust architecture**. In this model, identity becomes the new perimeter. Every access request — regardless of its origin — must be explicitly verified, authorised and encrypted before reaching any corporate resource. Nothing is trusted by default, not even traffic originating from inside the network.

---

## What This Project Builds

A **fully functional hybrid corporate infrastructure** combining on-premise resources (VirtualBox) and a public cloud environment (AWS). Corporate services — document management and team communications — are completely isolated from the public internet and accessible exclusively through a single, tightly controlled authenticated entry point.

### Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────┐
│  LAYER 3 — PERIMETER (AWS public subnet)                │
│  Nginx · Keycloak · Fail2ban · UFW · Let's Encrypt      │
│  Single entry point · TLS termination · SSO + MFA       │
└────────────────────────┬────────────────────────────────┘
                         │ authenticated traffic only
┌────────────────────────▼────────────────────────────────┐
│  LAYER 2 — SERVICES (AWS private subnet)                │
│  Nextcloud · Mattermost · MariaDB                       │
│  No public IP · No internet access · Docker isolated    │
└────────────────────────┬────────────────────────────────┘
                         │ LDAP over WireGuard VPN
┌────────────────────────▼────────────────────────────────┐
│  LAYER 1 — IDENTITY (on-premise)                        │
│  Active Directory · GPO · LDAP · DNS · DHCP             │
│  Single source of truth for all identities              │
└─────────────────────────────────────────────────────────┘
```

**Layer 1 — Identity & Control (on-premise)**
Active Directory is the trust anchor of the entire system. It stores all corporate identities, groups and access policies. No resource is accessible without going through it.

**Layer 2 — Services & Data (AWS private subnet)**
Corporate applications and their databases run in a private subnet with no public IP and no internet route. Network segmentation is enforced at the AWS infrastructure level.

**Layer 3 — Perimeter (AWS public subnet)**
A reverse proxy is the sole entry point for all external traffic. It enforces TLS, integrates with the identity federation layer via SAML 2.0, and applies dynamic firewall rules. Unauthenticated requests never reach the internal network.

---

## Authentication Flow

```
User → https://abb-ztcs.com → Nginx (TLS)
     → Keycloak (SSO redirect)
     → Active Directory (LDAP credential validation via WireGuard)
     → TOTP prompt (MFA — second factor)
     → SAML assertion issued
     → Nextcloud session created
     → Dashboard accessible
```

Every login requires:
1. Valid Active Directory credentials
2. A TOTP code from a registered authenticator device

Stolen credentials alone are insufficient. The system validates both factors before granting any session.

---

## Technology Stack

| Component | Technology | Role |
|---|---|---|
| Identity directory | Windows Server 2022 + AD DS | Central identity store, GPO enforcement |
| VPN tunnel | WireGuard | Encrypted link between AWS and on-premise |
| Identity provider | Keycloak 24 | SSO, SAML 2.0, MFA (TOTP), LDAP federation |
| Reverse proxy | Nginx + Certbot | TLS termination, routing, security headers |
| Intrusion prevention | Fail2ban + UFW | Dynamic IP banning, host firewall |
| Document management | Nextcloud 29 | Corporate file storage with SSO integration |
| Team communications | Mattermost 9.7 | Corporate messaging platform |
| Database | MariaDB 11.4 | Relational backend for Nextcloud and Mattermost |
| Cloud infrastructure | AWS (VPC, EC2, Security Groups) | Isolated network topology |
| Containers | Docker + Docker Compose | Service orchestration on AWS instances |

> No custom software is developed. The project focuses on the deployment, integration and hardening of open source solutions.

---

## Security Validation — Purple Team

The project concludes with a practical offensive/defensive validation covering three attack scenarios:

| Attack | Target | Result |
|---|---|---|
| Brute force | SSH + Keycloak login portal | SSH rejects password auth by design; Keycloak IP banned by Fail2ban after 5 attempts |
| Credential theft (phishing) | Keycloak login with stolen AD password | Blocked at MFA prompt — TOTP required, API bypass also blocked |
| Session hijacking | Nextcloud active session | All vectors blocked — HTTP→HTTPS redirect, forged cookies return 401, cookie replay rejected |

Full documentation: [`security/purple-team.md`](security/purple-team.md)

---

## Repository Structure

```
zero-trust-corporate-system/
├── architecture/                    # Network diagrams (.png, .drawio)
├── docs/
│   ├── en/                          # Technical documentation in English
│   │   ├── admin-manual.md          # Administrator reference guide
│   │   ├── user-manual.md           # End user guide
│   │   ├── market-technology-analysis.md
│   │   ├── risk-plan.md
│   │   ├── security-plan.md
│   │   └── sustainability-report.md
│   └── es/                          # Documentación técnica en español
│       ├── manual-administrador.md
│       ├── manual-usuario.md
│       └── ...
├── infrastructure/
│   ├── aws/                         # VPC, EC2, Security Groups, DNS
│   ├── domain-controller/           # Active Directory setup and configuration
│   ├── identity-provider/           # Keycloak deployment and SAML/LDAP config
│   ├── reverse-proxy/               # Nginx, TLS, Fail2ban, UFW
│   ├── services/                    # Nextcloud, Mattermost, MariaDB, SSO
│   └── vpn/                         # WireGuard tunnel configuration
├── media/                           # Screenshots and evidence
├── security/
│   └── purple-team.md               # Attack simulations and incident response
├── sprints/
│   ├── sprint-01/                   # Planning and review records
│   └── sprint-02/
└── tests/
    ├── test-plan.md                 # Complete test plan (25 test cases)
    ├── connectivity-tests.md        # Network isolation validation
    └── sso-validation.md            # SSO and access control validation
```

---

## Methodology

The project follows **SCRUM** with two sprints managed in ProofHub:

| Sprint | Period | Focus |
|---|---|---|
| Sprint 1 | 13/04/2026 → 24/04/2026 | Foundational documentation, architecture design, market analysis |
| Sprint 2 | 27/04/2026 → 12/05/2026 | Full technical implementation, SSO, Purple Team, validation |

Each sprint includes a planning record and a review/retrospective committed to this repository as Markdown files, with ProofHub board screenshots as evidence of task tracking.

---

## Key Documents

| Document | Description |
|---|---|
| [`docs/en/admin-manual.md`](docs/en/admin-manual.md) | Full administrator reference — architecture, procedures, troubleshooting |
| [`docs/en/user-manual.md`](docs/en/user-manual.md) | End user guide — login, Nextcloud, Mattermost |
| [`infrastructure/identity-provider/keycloak-setup.md`](infrastructure/identity-provider/keycloak-setup.md) | Keycloak deployment, LDAP federation, MFA, SAML |
| [`infrastructure/services/sso-nextcloud-setup.md`](infrastructure/services/sso-nextcloud-setup.md) | SSO integration between Nextcloud and Keycloak |
| [`security/purple-team.md`](security/purple-team.md) | Purple Team exercises — 3 attacks, 3 mitigations |
| [`tests/test-plan.md`](tests/test-plan.md) | 25 test cases — all passed |

---

## Test Results Summary

| Category | Tests | Passed |
|---|---|---|
| Network & Connectivity | 5 | 5 ✓ |
| Perimeter Security | 6 | 6 ✓ |
| SSO & Access Control | 7 | 7 ✓ |
| Purple Team | 7 | 7 ✓ |
| **Total** | **25** | **25 ✓** |

---

## Deployment Prerequisites

To bring the full system up from a cold start:

1. Start **DC01** VM in VirtualBox — wait ~2 minutes for AD services
2. Start **ztcs-perimeter** EC2 instance in AWS
3. Start **ztcs-services** EC2 instance in AWS
4. Verify WireGuard tunnel: `nc -zv 192.168.56.10 389` from ztcs-perimeter
5. Access corporate portal: `https://abb-ztcs.com`

Full procedure: [`docs/en/admin-manual.md`](docs/en/admin-manual.md) — Section 3.1

---

*All technical documentation is written in technical English (B2 level) with a parallel version in Spanish under `docs/es/`.*