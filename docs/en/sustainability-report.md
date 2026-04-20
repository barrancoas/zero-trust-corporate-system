# Sustainability Report

**Project:** Zero Trust Corporate System
**Author:** Asier Barranco
**Date:** 20/04/2026
**Version:** 1.0

---

## 1. Introduction

This report analyses the environmental, social and governance (ESG) dimensions of the Zero Trust Corporate System project. It evaluates the sustainability implications of the architectural decisions made, with particular attention to the hybrid cloud model adopted and its impact compared to traditional infrastructure approaches.

The analysis follows the ESG framework used in international sustainability standards, assessing each dimension independently before drawing overall conclusions.

---

## 2. Environmental Dimension (E)

### 2.1 Energy Consumption and Cloud Efficiency

The project adopts a hybrid architecture combining a minimal on-premise component (a single VirtualBox virtual machine acting as the Active Directory domain controller) with cloud-hosted services on AWS. This choice has direct environmental implications.

Traditional corporate infrastructure relies on permanently running physical servers in on-premise data centres, consuming energy regardless of actual usage. The AWS model operates on a shared infrastructure principle: physical hardware is shared across thousands of customers, and resources are allocated dynamically. AWS data centres consistently report Power Usage Effectiveness (PUE) values significantly below the industry average, and the company has committed to powering its global operations with 100% renewable energy.

The on-demand nature of EC2 instances means that resources can be stopped when not in use, eliminating idle energy consumption. For a project of this scope, instances can be shut down outside working hours, reducing the effective energy footprint to only the hours of active use.

### 2.2 Hardware Footprint

By deploying services on cloud infrastructure rather than dedicated physical hardware, the project avoids the manufacturing and disposal impact of additional servers. The only physical hardware involved is the existing workstation used for VirtualBox, which was already in use and requires no additional procurement.

Containerisation with Docker further improves resource efficiency: multiple services (Nextcloud, Mattermost, MariaDB) run on a single EC2 instance rather than requiring dedicated physical or virtual machines for each service, reducing both energy consumption and resource waste.

### 2.3 Circular Economy Alignment

The exclusive use of open source software eliminates licence-driven hardware replacement cycles, a common source of unnecessary equipment disposal in proprietary software environments. The architecture is designed to be reproducible and portable — if the cloud provider or instance type changes, the same Docker Compose configuration can be redeployed without infrastructure changes, extending the useful life of the setup and reducing waste.

---

## 3. Social Dimension (S)

### 3.1 Data Security and User Privacy

The primary social impact of this project is the protection of corporate user data. The Zero Trust model enforces that no user can access corporate resources without explicit, verified authentication — reducing the risk of data breaches caused by compromised credentials or unauthorised internal access.

All traffic between users and corporate services is encrypted end-to-end via TLS/HTTPS. Authentication is protected by mandatory Multi-Factor Authentication, which significantly reduces the risk of account takeover even when passwords are compromised.

### 3.2 Regulatory Compliance

The architecture is designed with data protection principles aligned to the LOPDGDD (Ley Orgánica de Protección de Datos y Garantía de los Derechos Digitales), Spain's implementation of the GDPR framework. Key compliance measures embedded in the architecture include:

- Access control based on the principle of least privilege, enforced through Active Directory group policies and Keycloak role mappings.
- Audit logging of all authentication events and access attempts, enabling traceability and incident response.
- Data stored in private subnets with no direct external exposure, minimising the attack surface for unauthorised access.
- Encryption in transit for all user-facing services and inter-service communication through the VPN tunnel.

### 3.3 Digital Skills Development

The project contributes to the social dimension of sustainability through the development of advanced digital competencies in cybersecurity, cloud infrastructure and systems administration. The skills acquired — Zero Trust architecture design, identity federation, offensive and defensive security testing — are directly aligned with growing labour market demand in the cybersecurity sector, contributing to the individual's employability and to the broader goal of reducing the cybersecurity skills gap in the European market.

---

## 4. Governance Dimension (G)

### 4.1 Open Source and Technological Sovereignty

The entire service stack is built on open source software: Keycloak, Nginx, Nextcloud, Mattermost, MariaDB and WireGuard. This choice has significant governance implications. Open source software eliminates vendor lock-in — if any component needs to be replaced, an equivalent open source alternative exists without contractual or licensing barriers. It also ensures that the software can be audited, meaning security vulnerabilities can be identified and fixed by the community rather than depending on a vendor's patch cycle.

This aligns with the European Commission's broader push for digital sovereignty and open source adoption in public and educational institutions.

### 4.2 Economic Viability

The total infrastructure cost of this project is bounded by the available AWS credit (~$40). The breakdown of estimated costs is as follows:

| Resource | Estimated monthly cost |
|----------|----------------------|
| EC2 t3.small (public subnet — proxy + Keycloak) | ~$15 |
| EC2 t3.medium (private subnet — Docker services) | ~$20 |
| Data transfer and storage | ~$3 |
| **Total estimated** | **~$38/month** |

This cost model demonstrates the economic viability of a production-grade Zero Trust architecture at minimal cost when built on open source components. The equivalent commercial solution (Okta + managed WAF + SaaS document management) would cost several hundred dollars per month at the same scale.

### 4.3 Scalability and Long-term Sustainability

The architecture is designed to scale horizontally. Additional EC2 instances can be added to the private subnet to handle increased load on Nextcloud or Mattermost without architectural changes. Keycloak supports clustering for high availability. Docker Compose configurations can be migrated to container orchestration platforms if the organisation grows.

The use of Infrastructure-as-Documentation — all deployment steps are documented in the repository — ensures that the environment can be reproduced, transferred or extended by any qualified administrator, reducing dependency on a single person and improving organisational resilience.

### 4.4 Agile Methodology and Project Governance

The project is managed using the SCRUM framework with defined sprints, planning sessions and retrospectives. This methodology ensures continuous delivery of value, early detection of technical blockers and transparent progress tracking through ProofHub and GitHub. The documentation-first approach — committing sprint records, architecture diagrams and configuration guides to the repository — creates an auditable trail of decisions and progress that supports accountability and knowledge transfer.

---

## 5. Conclusions

The Zero Trust Corporate System project demonstrates that robust, production-grade security infrastructure can be built sustainably — environmentally, socially and economically — using open source technologies on cloud infrastructure.

The hybrid cloud model reduces physical hardware requirements and energy consumption compared to traditional on-premise deployments. The Zero Trust architecture enforces strong data protection principles aligned with current regulatory frameworks. The open source stack eliminates vendor lock-in, reduces cost and supports technological sovereignty. The agile methodology ensures transparent, accountable project governance.

Together, these characteristics make the project not only technically sound but also aligned with the principles of responsible and sustainable digital infrastructure.