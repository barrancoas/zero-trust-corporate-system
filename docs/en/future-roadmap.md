# Future Roadmap & Scalability

**Project:** Zero Trust Corporate System  
**Author:** Asier Barranco  
**Date:** 12/05/2026  
**Version:** 1.0  

---

## 1. Overview

The Zero Trust Corporate System deployed in this project represents a functional, production-grade foundation. Every core component — federated identity, SSO, MFA, network segmentation, perimeter protection and security validation — is operational and documented.

This document identifies the natural evolution of the architecture: improvements, extensions and additional components that would be implemented in a real production environment or in a continuation of this project. Each item is grounded in limitations or constraints observed during the current deployment.

The roadmap is organised into three horizons:

| Horizon | Timeframe | Focus |
|---|---|---|
| Short-term | 1–3 months | Close known gaps and complete partially implemented features |
| Medium-term | 3–6 months | Strengthen resilience, observability and security depth |
| Long-term | 6–12 months | Full Zero Trust maturity and enterprise-grade scalability |

---

## 2. Short-Term Improvements

### 2.1 Mattermost SSO Integration

**Current state:** Mattermost is deployed and accessible at `https://abb-ztcs.com/mattermost/`. It operates with local accounts, independent of the Keycloak identity provider.

**Gap:** users must maintain separate credentials for Mattermost, breaking the single identity model that is core to Zero Trust.

**Next step:** configure a SAML 2.0 client for Mattermost in Keycloak, following the same pattern used for Nextcloud. Mattermost supports SAML natively in its Enterprise and Team editions. The integration would:

- Create a dedicated SAML client `https://abb-ztcs.com/mattermost` in the `zerotrust` realm
- Configure attribute mappers for username, email and display name
- Enable SAML authentication in Mattermost's System Console
- Apply the same MFA enforcement (TOTP) already active for Nextcloud

**Effort estimate:** 4–6 hours. All prerequisites are already in place.

---

### 2.2 LDAPS — Encrypted LDAP

**Current state:** Keycloak communicates with Active Directory via unencrypted LDAP on port 389. The traffic is protected because it travels through the WireGuard VPN tunnel, but LDAP itself is not encrypted.

**Gap:** this creates a dependency between LDAP security and VPN availability. If the VPN tunnel is misconfigured or the WireGuard service stops, LDAP traffic would either fail completely or — in a misconfigured scenario — travel unencrypted.

**Next step:** migrate from LDAP (389) to LDAPS (636), which encrypts the directory traffic at the protocol level using TLS, independent of the VPN tunnel.

Implementation requires:
- Installing an Active Directory Certificate Services (AD CS) role on DC01 to issue an internal certificate for the domain controller
- Configuring Keycloak's LDAP provider to use `ldaps://192.168.56.10:636` with the internal CA certificate in the Keycloak truststore
- Opening port 636 in the WireGuard routing rules

**Effort estimate:** 3–4 hours.

---

### 2.3 Automated Backup System

**Current state:** backup procedures are documented in the administrator manual but are not automated. Data backup requires manual execution of Docker and MariaDB dump commands.

**Gap:** manual backups are error-prone and easily forgotten. In a production environment, data loss due to missing backups is unacceptable.

**Next step:** implement automated backup scripts executed via cron on `ztcs-services`:

```bash
# Example cron entry — runs daily at 02:00
0 2 * * * /home/ubuntu/scripts/backup.sh >> /var/log/backup.log 2>&1
```

The backup script would:
- Dump all MariaDB databases to a timestamped SQL file
- Archive the Nextcloud data volume
- Export the Keycloak realm configuration via API
- Retain the last 7 daily backups, rotating older copies
- Transfer the backup archive to `ztcs-perimeter` or an S3 bucket for off-instance storage

**Effort estimate:** 2–3 hours for scripting and testing.

---

### 2.4 Log Monitoring Script

**Current state:** logs from Nginx, Fail2ban, Keycloak and Nextcloud are available on their respective instances but must be checked manually.

**Gap:** there is no centralised view of security events, and no automated alerting when suspicious activity occurs.

**Next step:** implement a lightweight monitoring script that aggregates and reports on key security events:

- Failed authentication attempts (Keycloak LOGIN_ERROR events via journald)
- Fail2ban ban events
- Nginx 4xx/5xx error rates
- Certificate expiry warning (< 30 days remaining)

The script would run as a daily cron job, generating a summary report committed to the repository or sent via email/webhook notification.

**Effort estimate:** 3–4 hours.

---

## 3. Medium-Term Improvements

### 3.1 High Availability — Eliminating Single Points of Failure

**Current state:** every component runs on a single instance. If `ztcs-perimeter` fails, all services become unreachable. If `ztcs-services` fails, all corporate applications go offline.

**Gap:** the architecture has no redundancy. Any instance failure causes a complete outage.

**Next step:** introduce redundancy at each layer:

| Layer | Current | Target |
|---|---|---|
| Perimeter (Nginx) | 1× EC2 t3.small | 2× EC2 behind AWS Application Load Balancer |
| Identity (Keycloak) | 1× container (dev mode) | Keycloak cluster (2+ nodes) with shared database |
| Services (Nextcloud) | 1× container | 2× containers behind internal load balancer |
| Database (MariaDB) | 1× container | MariaDB Galera Cluster (3 nodes) or AWS RDS |

This transition also requires migrating Keycloak from dev mode (embedded H2 database) to a production-grade external database, which is a prerequisite for clustering.

**Effort estimate:** significant — 2–3 weeks for a full HA deployment.

---

### 3.2 Centralised Log Management (SIEM)

**Current state:** logs are distributed across `ztcs-perimeter` and `ztcs-services` with no unified view.

**Next step:** deploy a centralised logging stack. A lightweight option for this architecture would be:

- **Loki** (log aggregation) + **Promtail** (log shipping agent) + **Grafana** (visualisation dashboard)
- Alternatively, a full **ELK stack** (Elasticsearch, Logstash, Kibana) for more powerful querying

All services would ship logs to the central stack, enabling:
- Real-time dashboards for authentication events, failed logins and ban activity
- Correlation of events across services (e.g. a Fail2ban ban linked to the Keycloak login attempt that triggered it)
- Retention policies and audit trails for compliance

**Effort estimate:** 1–2 weeks.

---

### 3.3 Internal PKI — Certificate Authority

**Current state:** TLS certificates are issued by Let's Encrypt for the public domain `abb-ztcs.com`. Internal services communicate over HTTP within the Docker network and the VPC.

**Gap:** internal service-to-service communication (Nginx → Nextcloud, Nginx → Keycloak) is unencrypted. In a Zero Trust model, all traffic — including internal — should be encrypted.

**Next step:** deploy an internal Certificate Authority using **Active Directory Certificate Services (AD CS)** on DC01, or a lightweight alternative such as **step-ca** or **Vault PKI**. This would enable:

- TLS certificates for all internal service endpoints (`10.0.2.220:8080`, `localhost:8080`)
- Mutual TLS (mTLS) between services — each service presents a certificate, not just the client
- LDAPS certificates for the domain controller (prerequisite for item 2.2)

**Effort estimate:** 1–2 weeks for full internal PKI.

---

### 3.4 Infrastructure as Code (IaC)

**Current state:** all AWS infrastructure was provisioned manually via the AWS console. The configuration is documented in Markdown but cannot be automatically reproduced.

**Gap:** manual provisioning is slow, error-prone and not repeatable. If the AWS Learner Lab resets or credits run out, the entire infrastructure must be rebuilt manually.

**Next step:** rewrite the AWS infrastructure as code using **Terraform** or **AWS CloudFormation**:

```hcl
# Example Terraform resource
resource "aws_vpc" "ztcs_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "ztcs-vpc" }
}
```

With IaC, the entire environment can be destroyed and recreated in minutes with a single command. This also enables version-controlled infrastructure changes, peer review of network modifications and disaster recovery.

**Effort estimate:** 1–2 weeks to fully Terraform the existing architecture.

---

## 4. Long-Term Vision — Full Zero Trust Maturity

### 4.1 Device Identity and Endpoint Trust

**Current state:** Zero Trust is applied at the identity layer — every user must authenticate with credentials and MFA. However, the architecture does not evaluate the trustworthiness of the device making the request.

**Limitation:** a valid user authenticating from a compromised or unmanaged device still gains access. True Zero Trust requires that both the identity and the device are verified.

**Next step:** integrate a **Mobile Device Management (MDM)** solution such as **Jamf** (macOS/iOS), **Microsoft Intune** (cross-platform) or **Fleet** (open source). Keycloak can be configured to enforce device compliance as a condition for access — only devices registered in the MDM and meeting defined security baselines (OS version, encryption enabled, no known malware) would be permitted to authenticate.

---

### 4.2 Conditional Access Policies

**Current state:** authentication is binary — a user either passes the identity check or does not. There is no context-aware access control.

**Next step:** implement context-aware access policies in Keycloak using **Authentication Flows** and **Conditions**:

- Require step-up authentication (additional MFA factor) for sensitive operations
- Block access from unknown geographic locations or high-risk IP ranges
- Enforce time-based access restrictions (e.g. no access outside business hours for standard users)
- Require re-authentication after a period of inactivity

---

### 4.3 Network Microsegmentation

**Current state:** the private subnet treats all containers as equally trusted. Nextcloud, Mattermost and MariaDB share a single Docker bridge network — any container can reach any other.

**Next step:** apply network microsegmentation within the private subnet:

- Separate Docker networks per service tier (database network, application network)
- Explicit allow rules between tiers — Nextcloud can reach MariaDB, but Mattermost cannot reach the Nextcloud database
- AWS Security Group rules restricting traffic between services at the VPC level

In a production environment, this would be extended to **service mesh** technology (Istio, Linkerd) with mTLS between all services and fine-grained traffic policies.

---

### 4.4 Secrets Management

**Current state:** service passwords and credentials are stored in `docker-compose.yml` files on the instance. Repository versions are redacted, but the live configuration files contain plaintext secrets.

**Next step:** integrate a dedicated secrets management solution such as **HashiCorp Vault** or **AWS Secrets Manager**:

- All service credentials stored in Vault, never in configuration files
- Dynamic database credentials — MariaDB passwords rotated automatically by Vault, eliminating static credentials entirely
- Audit log of every secret access
- Automatic secret rotation on a defined schedule

---

### 4.5 Security Automation and SOAR

**Current state:** incident response is manual — a ban from Fail2ban requires a human administrator to investigate, decide on further action and document the event.

**Next step:** implement a **Security Orchestration, Automation and Response (SOAR)** workflow:

- Automated enrichment of Fail2ban ban events with threat intelligence (IP reputation lookup via APIs)
- Automatic escalation of repeated ban events to a notification channel (email, Mattermost webhook)
- Self-healing playbooks — if Keycloak is unreachable, automatically restart the container and alert the administrator

---

## 5. Summary Table

| Item | Horizon | Effort | Impact |
|---|---|---|---|
| Mattermost SSO | Short | Low | Completes identity unification |
| LDAPS | Short | Low | Eliminates VPN dependency for LDAP security |
| Automated backups | Short | Low | Prevents data loss |
| Log monitoring script | Short | Low | Basic security visibility |
| High availability | Medium | High | Eliminates single points of failure |
| Centralised logging (SIEM) | Medium | Medium | Full security observability |
| Internal PKI | Medium | Medium | Encrypts all internal traffic |
| Infrastructure as Code | Medium | Medium | Reproducible, version-controlled infrastructure |
| Device trust (MDM) | Long | High | True Zero Trust — identity + device |
| Conditional access | Long | Medium | Context-aware security policies |
| Network microsegmentation | Long | High | Eliminates implicit internal trust |
| Secrets management | Long | Medium | Eliminates static credentials |
| Security automation (SOAR) | Long | High | Automated incident response |

---

## 6. Conclusion

The Zero Trust Corporate System, as deployed, implements the foundational pillars of the Zero Trust model: verified identity, enforced MFA, network segmentation, encrypted communications and active intrusion prevention. It demonstrates that a production-grade Zero Trust architecture is achievable with open source tools and modest cloud resources.

The items in this roadmap represent the natural maturity progression of the architecture — from a well-functioning single-environment deployment towards a resilient, fully automated, enterprise-grade security infrastructure. Each step builds directly on what has already been implemented, with no architectural changes required to the core design.

The principles remain constant across all horizons: **never trust, always verify** — applied progressively to every layer of the stack.