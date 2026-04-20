# Market Analysis and Technology Study

**Project:** Zero Trust Corporate System  
**Author:** Asier Barranco  
**Date:** 20/04/2026  
**Version:** 1.0

---

## 1. Context and Market Need

The shift towards remote work and cloud-first infrastructure has made traditional perimeter-based security models increasingly inadequate. In the classical model, users and devices inside the corporate network are implicitly trusted. Modern threats — including credential theft, lateral movement and insider attacks — exploit precisely this assumption.

The Zero Trust security model addresses this gap by treating every access request as untrusted by default, regardless of its origin. Identity becomes the new security perimeter: every user must be explicitly verified, every access must be authorised, and all traffic must be encrypted end-to-end.

This shift is reflected in the market. According to industry reports, the global Zero Trust security market was valued at over $30 billion in 2023 and is projected to grow at a compound annual rate of over 17% through 2030, driven by regulatory pressure, cloud adoption and the rise of hybrid work environments.

---

## 2. Existing Commercial Solutions

Before selecting the technology stack for this project, a review of the main commercial Zero Trust platforms available on the market was conducted.

### 2.1 Okta Workforce Identity

Okta is one of the leading Identity-as-a-Service providers. It offers SSO, MFA, lifecycle management and deep integrations with cloud applications. Its strengths lie in its ease of use and extensive connector catalogue. However, Okta is a fully managed SaaS product with no self-hosted option, making it unsuitable for an educational project requiring hands-on infrastructure deployment. Pricing starts at approximately $6 per user per month, which becomes significant at enterprise scale.

### 2.2 Microsoft Azure Active Directory (Entra ID)

Microsoft's cloud identity platform integrates natively with Windows environments and Microsoft 365. It supports SSO, MFA and conditional access policies. While it is a powerful solution for organisations already invested in the Microsoft ecosystem, it is a proprietary closed-source platform. Its free tier is limited, and full Zero Trust features require Azure AD Premium P2 licences at significant cost.

### 2.3 Zscaler Zero Trust Exchange

Zscaler offers a complete cloud-native Zero Trust Network Access (ZTNA) platform. It operates as a globally distributed proxy, routing all corporate traffic through its cloud infrastructure. While technically mature, it is a fully managed service with no on-premise component, and its licensing model is enterprise-oriented, making it inaccessible for a project of this scope.

### 2.4 Cloudflare Zero Trust

Cloudflare offers a ZTNA solution built on its global edge network. It includes access proxy, identity federation and threat intelligence. Like Zscaler, it is a managed cloud service. It does offer a generous free tier, but its architecture does not allow the level of hands-on infrastructure work that this project requires.

### 2.5 Summary of Commercial Solutions

| Solution | Deployment | Open Source | Cost | Suitable for this project |
|----------|-----------|-------------|------|--------------------------|
| Okta | SaaS only | No | High | No |
| Azure AD / Entra ID | Cloud + hybrid | No | Medium–High | No |
| Zscaler | SaaS only | No | High | No |
| Cloudflare Zero Trust | SaaS only | No | Low–Medium | No |

None of the leading commercial solutions are suitable for this project. They are either fully managed (removing the infrastructure deployment component), proprietary (preventing deep technical integration) or prohibitively expensive. This justifies the selection of an open source stack that replicates the same architectural principles with full control over every layer.

---

## 3. Open Source Alternatives Analysis

For each component of the architecture, multiple open source options were evaluated before selecting the final tool.

### 3.1 Identity Provider

The identity provider is the most critical component of the Zero Trust stack. It must support LDAP federation with Active Directory, SSO via OIDC or SAML 2.0, and enforce MFA on every authentication.

| Tool | Protocol support | AD federation | MFA | Maturity | Selected |
|------|-----------------|---------------|-----|----------|----------|
| **Keycloak** | OIDC, SAML 2.0, OAuth 2.0 | Yes (LDAP) | Yes (TOTP, WebAuthn) | High | **Yes** |
| Authentik | OIDC, SAML 2.0 | Yes (LDAP) | Yes | Medium | No |
| Authelia | OIDC | Limited | Yes | Medium | No |
| Dex | OIDC | Yes | No (native) | Medium | No |

**Keycloak** was selected because it is the most mature and widely adopted open source identity platform. Maintained by Red Hat and the CNCF ecosystem, it has comprehensive official documentation for LDAP federation, SAML integration and TOTP-based MFA. Nextcloud and Mattermost both have documented, tested integration guides specifically for Keycloak. Authentik is a viable alternative but has a smaller community and less integration documentation for the specific services used in this project.

### 3.2 Reverse Proxy

The reverse proxy acts as the single entry point for all external traffic. It must handle TLS termination, route authenticated requests to internal services and enforce access control rules.

| Tool | TLS automation | Reverse proxy | Performance | Documentation quality | Selected |
|------|---------------|---------------|-------------|----------------------|----------|
| **Nginx** | Yes (with Certbot) | Yes | High | Excellent | **Yes** |
| Apache HTTP Server | Yes (with Certbot) | Yes | Medium | Excellent | No |
| Caddy | Yes (automatic) | Yes | High | Good | No |
| Traefik | Yes (automatic) | Yes | High | Good | No |

**Nginx** was selected over Apache despite existing familiarity with Apache. The deciding factor is that Keycloak, Nextcloud and Mattermost all provide official reverse proxy configuration examples written for Nginx. Using Apache would require translating every configuration block to its syntax, introducing unnecessary risk given the time constraints of the project. Caddy would simplify TLS management further, but Nginx offers more granular control over security headers, rate limiting and access rules, which are critical for a Zero Trust perimeter. Nginx is also the industry standard for reverse proxy in containerised and cloud environments.

### 3.3 Document Management Service

The document management service must support SSO authentication via the identity provider and run in an isolated private subnet with no direct external exposure.

| Tool | SAML / OIDC | Self-hosted | Active community | Selected |
|------|------------|-------------|-----------------|----------|
| **Nextcloud** | Yes (SAML app) | Yes | Very active | **Yes** |
| Seafile | Yes | Yes | Active | No |
| OnlyOffice | Partial | Yes | Active | No |

**Nextcloud** was selected as the corporate document management service. It is the most widely deployed self-hosted file management and collaboration platform, with a dedicated SAML authentication application that integrates directly with Keycloak. Its Docker deployment is well documented and actively maintained.

### 3.4 Communications Service

The communications platform must support team messaging and SSO authentication.

| Tool | SAML / OIDC | Self-hosted | Resource usage | Selected |
|------|------------|-------------|----------------|----------|
| **Mattermost** | Yes (SAML, LDAP) | Yes | Low | **Yes** |
| Rocket.Chat | Yes | Yes | High | No |
| Matrix / Element | Yes | Yes | Medium | No |

**Mattermost** was selected for its lightweight footprint and direct SAML integration with Keycloak. Rocket.Chat offers similar functionality but is significantly heavier in terms of resource consumption, which is relevant given the AWS budget constraints (~$40). Matrix/Element is well suited for federated communication but adds unnecessary complexity for a single-organisation deployment.

### 3.5 Relational Database

Both Nextcloud and Mattermost require a relational database backend.

| Tool | Compatibility | Known by developer | Selected |
|------|--------------|-------------------|----------|
| **MariaDB** | Nextcloud, Mattermost | Yes | **Yes** |
| PostgreSQL | Nextcloud, Mattermost | Partially | No |
| MySQL | Nextcloud, Mattermost | Yes | No |

**MariaDB** was selected because of the solid hands-on experience with it from the coursework, covering schema design, CRUD operations, user management and backup procedures. It is fully compatible with both Nextcloud and Mattermost and is the recommended database in both platforms' official documentation.

### 3.6 TLS Certificate Management

| Tool | Automation | Cost | Integration with Nginx | Selected |
|------|-----------|------|----------------------|----------|
| **Let's Encrypt + Certbot** | Yes (auto-renewal) | Free | Yes | **Yes** |
| Self-signed certificates | Manual | Free | Yes | No |
| AWS Certificate Manager | Yes | Free (with ALB) | Partial | No |

**Let's Encrypt with Certbot** was selected for automated certificate provisioning and renewal. It integrates directly with Nginx and requires minimal configuration. Self-signed certificates were discarded because they generate browser warnings and cannot be used transparently in a production-like environment.

### 3.7 VPN Tunnel (On-premise ↔ AWS)

| Tool | Performance | Setup complexity | Resource usage | Selected |
|------|------------|-----------------|----------------|----------|
| **WireGuard** | High | Low | Very low | **Yes** |
| OpenVPN | Medium | Medium | Medium | No |
| AWS Site-to-Site VPN | High | Low | N/A (managed) | No |

**WireGuard** was selected for the encrypted tunnel between the on-premise Active Directory and the AWS VPC. It is significantly simpler to configure than OpenVPN, uses modern cryptography (ChaCha20, Curve25519) and has minimal resource overhead. AWS Site-to-Site VPN was discarded because it carries an hourly cost (~$0.05/h) that would consume a significant portion of the available AWS budget.

### 3.8 Firewall and Active Protection

| Tool | Function | Known by developer | Selected |
|------|---------|-------------------|----------|
| **Fail2ban** | Dynamic IP blocking | Yes | **Yes** |
| **AWS Security Groups** | Cloud perimeter firewall | Yes | **Yes** |
| UFW | Host firewall | Yes | **Yes** (complementary) |

A layered firewall approach was adopted. AWS Security Groups control inbound and outbound traffic at the cloud perimeter. UFW manages host-level firewall rules on each EC2 instance. Fail2ban monitors authentication logs and dynamically blocks IPs after repeated failed attempts — a tool the developer has already deployed in previous practical exercises.

### 3.9 Containerisation

| Tool | Use case | Known by developer | Selected |
|------|---------|-------------------|----------|
| **Docker + Docker Compose** | Service deployment | Yes | **Yes** |

Docker Compose was selected for deploying the Layer 2 services (Nextcloud, Mattermost, MariaDB) in the private subnet. It simplifies multi-container orchestration, makes the environment reproducible and aligns with industry practice for self-hosted service deployment. The developer has prior experience with Docker from the vocational training program.

---

## 4. Final Technology Stack

| Component | Technology | Layer | Environment |
|-----------|-----------|-------|-------------|
| Domain controller | Windows Server 2022 + AD DS | Layer 1 | On-premise (VirtualBox) |
| Identity provider | Keycloak | Layer 3 | AWS EC2 (public subnet) |
| Reverse proxy | Nginx + Certbot | Layer 3 | AWS EC2 (public subnet) |
| Active protection | Fail2ban + AWS Security Groups + UFW | Layer 3 | AWS + EC2 instances |
| Document management | Nextcloud (Docker) | Layer 2 | AWS EC2 (private subnet) |
| Team communications | Mattermost (Docker) | Layer 2 | AWS EC2 (private subnet) |
| Relational database | MariaDB (Docker) | Layer 2 | AWS EC2 (private subnet) |
| TLS certificates | Let's Encrypt + Certbot | Layer 3 | AWS EC2 (public subnet) |
| VPN tunnel | WireGuard | Infrastructure | On-premise ↔ AWS |
| Cloud infrastructure | AWS (VPC, EC2, Security Groups) | Infrastructure | AWS |
| On-premise virtualisation | VirtualBox | Infrastructure | On-premise |
| Scripting | Bash + PowerShell | Automation | Linux + Windows |
| Containerisation | Docker + Docker Compose | Deployment | AWS EC2 (private subnet) |
| Version control | GitHub | Project management | Cloud |
| Project management | ProofHub | Project management | Cloud |

---

## 5. Conclusions

The selected stack is entirely composed of open source, production-grade tools that together implement a functional Zero Trust architecture. Each tool was chosen based on three criteria: technical fit with the project requirements, integration compatibility with the other components, and feasibility within the available time and budget.

The commercial alternatives reviewed — Okta, Azure AD, Zscaler and Cloudflare Zero Trust — offer comparable or superior functionality but are incompatible with the hands-on deployment goals of this project. They are either fully managed services, proprietary platforms, or require licensing costs beyond the project scope.

The open source stack selected replicates the core principles of enterprise Zero Trust implementations: centralised identity, federated authentication, enforced MFA, single entry point and network segmentation. Every component integrates with the others through open, documented protocols (LDAP, SAML 2.0, OIDC), ensuring a coherent and auditable system.