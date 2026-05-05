# AWS Infrastructure Deployment

**Project:** Zero Trust Corporate System  
**Author:** Asier Barranco  
**Date:** 28/04/2026  
**Region:** us-east-1 (N. Virginia)  
**Version:** 1.1

---

## 1. Cost Management — Read Before Starting

This project runs on AWS Academy Learner Lab credits. The available balance at the time of writing is approximately $30. The following rules must be observed throughout the entire project lifecycle to ensure credit lasts until the defence on 20/05/2026.

**Mandatory rules:**

- Stop all EC2 instances at the end of every work session. A stopped instance incurs no compute charges — only EBS storage is billed at ~$0.08/GB/month, which is negligible.
- Never leave instances running overnight, over weekends, or during non-working hours.
- Do not allocate Elastic IPs unless they are immediately attached to a running instance. An unattached Elastic IP incurs charges.
- Do not create a NAT Gateway. It costs approximately $1/day and is not needed for this architecture.

**Estimated costs:**

| Scenario | Approximate daily cost |
|---|---|
| Both instances running (active work) | ~$1.50–2.00 |
| Both instances stopped (idle) | ~$0.05 |
| Elastic IP attached to running instance | $0.00 |
| Elastic IP unattached or on stopped instance | ~$0.005/h |

At $1.50–2.00/day of active use, the remaining $30 of credit provides 15–20 full working days, which comfortably covers the project period if instances are stopped when idle.

---

## 2. Architecture Overview

The AWS deployment creates an isolated virtual network (VPC) with two subnets that correspond directly to the project's architectural layers:

| Component | Name | Type | Subnet | Purpose |
|---|---|---|---|---|
| Virtual network | `ztcs-vpc` | VPC | — | Isolated network boundary for the entire project |
| Public subnet | `ztcs-public-subnet` | Subnet | 10.0.1.0/24 | Hosts the reverse proxy (Nginx) and identity provider (Keycloak) |
| Private subnet | `ztcs-private-subnet` | Subnet | 10.0.2.0/24 | Hosts Docker services: Nextcloud, Mattermost, MariaDB |
| Internet gateway | `ztcs-igw` | Gateway | — | Provides internet routing for the public subnet |
| Perimeter instance | `ztcs-perimeter` | EC2 t3.small | Public | Layer 3 — reverse proxy, TLS termination, identity provider |
| Services instance | `ztcs-services` | EC2 t3.medium | Private | Layer 2 — containerised corporate services and database |
| Perimeter firewall | `ztcs-sg-perimeter` | Security Group | — | Inbound rules for the public-facing instance |
| Services firewall | `ztcs-sg-services` | Security Group | — | Inbound rules restricted to perimeter traffic only |
| SSH key | `ztcs-keypair` | Key Pair | — | Administrative access to both instances |

**Design decisions:**

- No NAT Gateway is provisioned. The private subnet has no internet routing by design. Docker images for Nextcloud, Mattermost and MariaDB will be transferred to the services instance during initial setup via the perimeter instance. After that, the services instance operates fully air-gapped from the public internet.
- Both instances are placed in the same Availability Zone (`us-east-1a`) to minimise inter-AZ data transfer costs and latency.
- The private instance is only accessible via SSH jump through the perimeter instance — there is no direct administrative path from the internet to the services tier.

---

## 3. Create the VPC

1. Open the AWS console → search **VPC** in the top bar → open the VPC Dashboard
2. Click **Create VPC**
3. Select **VPC only** (not "VPC and more")
4. Configure:

| Field | Value |
|---|---|
| Name tag | `ztcs-vpc` |
| IPv4 CIDR block | `10.0.0.0/16` |
| IPv6 CIDR block | No IPv6 CIDR block |
| Tenancy | Default |

5. Click **Create VPC**

After creation, note the VPC ID — it will be referenced in subsequent steps.

---

## 4. Create the Subnets

### 4.1 Public Subnet

1. In the VPC console, go to **Subnets → Create subnet**
2. Select VPC: `ztcs-vpc`
3. Configure:

| Field | Value |
|---|---|
| Subnet name | `ztcs-public-subnet` |
| Availability Zone | `us-east-1a` |
| IPv4 CIDR block | `10.0.1.0/24` |

4. Click **Create subnet**

### 4.2 Private Subnet

1. Click **Create subnet** again
2. Select VPC: `ztcs-vpc`
3. Configure:

| Field | Value |
|---|---|
| Subnet name | `ztcs-private-subnet` |
| Availability Zone | `us-east-1a` |
| IPv4 CIDR block | `10.0.2.0/24` |

4. Click **Create subnet**

### 4.3 Enable Auto-assign Public IP on the Public Subnet

1. Go to **Subnets** → select `ztcs-public-subnet`
2. Click **Actions → Edit subnet settings**
3. Check **Enable auto-assign public IPv4 address**
4. Click **Save**

This ensures that any EC2 instance launched in the public subnet automatically receives a public IPv4 address.

---

## 5. Create and Attach the Internet Gateway

1. In the VPC console, go to **Internet Gateways → Create internet gateway**
2. Configure:

| Field | Value |
|---|---|
| Name tag | `ztcs-igw` |

3. Click **Create internet gateway**
4. On the confirmation page, click **Actions → Attach to VPC**
5. Select `ztcs-vpc` → click **Attach internet gateway**

---

## 6. Configure Route Tables

### 6.1 Public Route Table

1. Go to **Route Tables → Create route table**
2. Configure:

| Field | Value |
|---|---|
| Name | `ztcs-public-rt` |
| VPC | `ztcs-vpc` |

3. Click **Create route table**
4. Select `ztcs-public-rt` → go to the **Routes** tab → click **Edit routes**
5. Click **Add route** and set:

| Destination | Target |
|---|---|
| `0.0.0.0/0` | Internet Gateway → `ztcs-igw` |

6. Click **Save changes**
7. Go to the **Subnet associations** tab → click **Edit subnet associations**
8. Select `ztcs-public-subnet` → click **Save associations**

### 6.2 Private Route Table — Verification Only

The private subnet uses the VPC's default route table, which contains only the local route (`10.0.0.0/16 → local`). This is correct — the private subnet must have no route to the internet gateway.

To verify:

1. Go to **Route Tables** → identify the default route table for `ztcs-vpc` (the one not named `ztcs-public-rt`)
2. Select it → go to the **Routes** tab
3. Confirm it contains only the local route and no `0.0.0.0/0` entry

If a `0.0.0.0/0` route exists pointing to the internet gateway, remove it immediately — this would compromise the network isolation of the private subnet.

---

## 7. Create Security Groups

### 7.1 Perimeter Security Group (public instance)

1. In the VPC console, go to **Security Groups → Create security group**
2. Configure:

| Field | Value |
|---|---|
| Security group name | `ztcs-sg-perimeter` |
| Description | Layer 3 — Reverse proxy and identity provider. Public facing. |
| VPC | `ztcs-vpc` |

3. Add the following **Inbound rules**:

| Type | Protocol | Port | Source | Description |
|---|---|---|---|---|
| SSH | TCP | 22 | My IP | Admin SSH — restricted to current IP |
| HTTP | TCP | 80 | 0.0.0.0/0 | Let's Encrypt ACME validation + HTTPS redirect |
| HTTPS | TCP | 443 | 0.0.0.0/0 | Public HTTPS — all corporate traffic |
| Custom UDP | UDP | 51820 | 0.0.0.0/0 | WireGuard VPN tunnel endpoint |

4. **Outbound rules:** leave the default rule (all traffic, all ports, 0.0.0.0/0). The perimeter instance needs outbound access for package updates, Let's Encrypt validation and communication with the private subnet.

5. Click **Create security group**

> **Note on SSH source "My IP":** AWS auto-fills your current public IP. If you change network (e.g. home vs. school), you will need to update this rule with your new IP to regain SSH access.

### 7.2 Services Security Group (private instance)

1. Click **Create security group** again
2. Configure:

| Field | Value |
|---|---|
| Security group name | `ztcs-sg-services` |
| Description | Layer 2 — Docker services. Private subnet, no public exposure. |
| VPC | `ztcs-vpc` |

3. Add the following **Inbound rules**:

| Type | Protocol | Port | Source | Description |
|---|---|---|---|---|
| SSH | TCP | 22 | sg-xxxx (`ztcs-sg-perimeter`) | Admin SSH via jump host only |
| Custom TCP | TCP | 80 | sg-xxxx (`ztcs-sg-perimeter`) | Nginx proxy → Nextcloud HTTP |
| Custom TCP | TCP | 8065 | sg-xxxx (`ztcs-sg-perimeter`) | Nginx proxy → Mattermost |
| Custom TCP | TCP | 8080 | sg-xxxx (`ztcs-sg-perimeter`) | Nginx proxy → Keycloak (if backend is here) |
| MySQL/Aurora | TCP | 3306 | sg-yyyy (`ztcs-sg-services`) | MariaDB — internal access only (self-referencing) |
| Custom UDP | UDP | 51820 | 0.0.0.0/0 | WireGuard VPN tunnel endpoint |

> **How to reference a Security Group as source:** in the Source field, select **Custom**, then start typing `sg-` — AWS will show a dropdown with available security groups. Select `ztcs-sg-perimeter` for rules that should only accept traffic from the perimeter instance. For the MariaDB rule, select `ztcs-sg-services` itself — this is a self-referencing rule that allows containers on the same instance to communicate over port 3306.

4. **Outbound rules:** leave the default (all traffic allowed). Although the private subnet has no internet route, the instance still needs outbound access to communicate with the perimeter instance within the VPC.

5. Click **Create security group**

---

## 8. Create the Key Pair

1. Go to **EC2** in the AWS console → **Key Pairs** (under Network & Security) → **Create key pair**
2. Configure:

| Field | Value |
|---|---|
| Name | `ztcs-keypair` |
| Key pair type | RSA |
| Private key file format | `.pem` |

3. Click **Create key pair** — the `.pem` file downloads automatically to your browser's download folder

4. Secure the key file immediately. Open a terminal on your local machine:

```bash
mkdir -p ~/.ssh
mv ~/Downloads/ztcs-keypair.pem ~/.ssh/
chmod 400 ~/.ssh/ztcs-keypair.pem
```

This key file cannot be downloaded again. If lost, a new key pair must be created and the instances relaunched or reconfigured.

> **AWS Academy note:** the Learner Lab environment may provide a pre-existing key pair called `vockey`. If so, you can use it instead of creating a new one. Check **AWS Details** in the Learner Lab interface to see if a `vockey.pem` is available for download. If you use `vockey`, replace all references to `ztcs-keypair` with `vockey` throughout this guide.

---

## 9. Launch EC2 Instances

### 9.1 Perimeter Instance (public subnet — Layer 3)

1. Go to **EC2 → Instances → Launch instances**
2. Configure:

| Field | Value |
|---|---|
| Name | `ztcs-perimeter` |
| AMI | Ubuntu Server 24.04 LTS (HVM, SSD Volume Type) |
| Architecture | 64-bit (x86) |
| Instance type | `t3.small` (2 vCPU, 2 GB RAM) |
| Key pair | `ztcs-keypair` (or `vockey` if using the lab default) |

3. Under **Network settings**, click **Edit** and configure:

| Field | Value |
|---|---|
| VPC | `ztcs-vpc` |
| Subnet | `ztcs-public-subnet` |
| Auto-assign public IP | Enable |
| Firewall (security groups) | Select existing security group → `ztcs-sg-perimeter` |

4. Under **Configure storage**: set **20 GB**, volume type **gp3**

5. Click **Launch instance**

### 9.2 Services Instance (private subnet — Layer 2)

1. Click **Launch instances** again
2. Configure:

| Field | Value |
|---|---|
| Name | `ztcs-services` |
| AMI | Ubuntu Server 24.04 LTS (HVM, SSD Volume Type) |
| Architecture | 64-bit (x86) |
| Instance type | `t3.medium` (2 vCPU, 4 GB RAM) |
| Key pair | `ztcs-keypair` (or `vockey`) |

3. Under **Network settings**, click **Edit** and configure:

| Field | Value |
|---|---|
| VPC | `ztcs-vpc` |
| Subnet | `ztcs-private-subnet` |
| Auto-assign public IP | Disable |
| Firewall (security groups) | Select existing security group → `ztcs-sg-services` |

4. Under **Configure storage**: set **30 GB**, volume type **gp3**

5. Click **Launch instance**

---

## 10. Verify the Infrastructure

Wait until both instances show **Instance state: Running** and **Status checks: 2/2 checks passed** in the EC2 console.

### 10.1 Record Instance Details

From the EC2 console, note the following values for later use:

| Instance | Public IPv4 | Private IPv4 |
|---|---|---|
| `ztcs-perimeter` | (visible in console) | 10.0.1.x |
| `ztcs-services` | (none — no public IP) | 10.0.2.x |

### 10.2 Verify the Perimeter Instance

1. Open a terminal on your local machine
2. Connect via SSH:

```bash
ssh -i ~/.ssh/ztcs-keypair.pem ubuntu@<PERIMETER_PUBLIC_IP>
```

3. Accept the host key fingerprint if prompted
4. Once connected, verify internet connectivity:

```bash
ping -c 4 8.8.8.8
```

Expected result: 4 packets transmitted, 4 received. This confirms that the public subnet routing and internet gateway are configured correctly.

5. Verify DNS resolution:

```bash
ping -c 4 google.com
```

Expected result: successful replies. This confirms outbound DNS works, which will be needed for package installation and Let's Encrypt.

### 10.3 Verify the Services Instance

The services instance has no public IP and cannot be reached directly from the internet. Access it via SSH jump through the perimeter instance:

```bash
ssh -i ~/.ssh/ztcs-keypair.pem -J ubuntu@<PERIMETER_PUBLIC_IP> ubuntu@<SERVICES_PRIVATE_IP>
```

Once connected, run the following verifications:

**1. Verify VPC internal connectivity:**

```bash
ping -c 4 10.0.1.1
```

Expected result: 4 packets received. This confirms that the private subnet can reach the public subnet gateway within the VPC.

**2. Verify internet isolation:**

```bash
ping -c 4 8.8.8.8
```

Expected result: **no response** (100% packet loss). This is the correct behaviour — it confirms that the private subnet has no internet routing, which is a core requirement of the network architecture.

**3. Verify internal connectivity to the perimeter instance:**

```bash
ping -c 4 <PERIMETER_PRIVATE_IP>
```

Expected result: 4 packets received. This confirms that the two instances can communicate within the VPC, which is essential for Nginx to proxy traffic to the Docker services.

---

## 11. Stopping and Starting Instances

### End of every work session

1. Go to **EC2 → Instances**
2. Select both `ztcs-perimeter` and `ztcs-services`
3. Click **Instance state → Stop instance** → confirm

### Start of every work session

1. Select both instances
2. Click **Instance state → Start instance**
3. Wait approximately 1 minute for both to reach **Running** state with 2/2 status checks passed
4. **Important:** the public IP of `ztcs-perimeter` changes on every start. Note the new IP before connecting via SSH.

> The public IP will become static once an Elastic IP is assigned during the reverse proxy and domain configuration phase. Until then, the changing IP is expected behaviour and does not affect the deployment.

---

## 12. Connection Reference

Quick reference for SSH connections throughout the project.

**Important:** the `labsuser.pem` key must be added to the SSH agent before using jump host connections. Run this at the start of every work session:

```bash
eval $(ssh-agent)
ssh-add /media/asier.barranco.7e6/ASIER/labsuser.pem
```

**Connect to the perimeter instance (direct):**

```bash
ssh -i /media/asier.barranco.7e6/ASIER/labsuser.pem ubuntu@<PERIMETER_PUBLIC_IP>
```

**Connect to the services instance (via jump host):**

```bash
ssh -A -J ubuntu@<PERIMETER_PUBLIC_IP> ubuntu@10.0.2.220
```

**Copy a file to the perimeter instance:**

```bash
scp -i /media/asier.barranco.7e6/ASIER/labsuser.pem ./file.tar.gz ubuntu@<PERIMETER_PUBLIC_IP>:~/
```

**Copy a file from the perimeter to the services instance (run from perimeter):**

```bash
scp ./file.tar.gz ubuntu@10.0.2.220:~/
```

> Note: the public IP of `ztcs-perimeter` changes every time the lab session starts or the instance is restarted. The private IPs are permanent: perimeter `10.0.1.9`, services `10.0.2.220`.

This two-hop transfer method (local → perimeter → services) is the standard procedure for deploying Docker images and configuration files to the air-gapped services instance.

---

## 13. Summary

At the end of this phase, the following infrastructure is operational:

| Component | Details |
|---|---|
| VPC | `ztcs-vpc` — CIDR `10.0.0.0/16` |
| Public subnet | `ztcs-public-subnet` — `10.0.1.0/24` — internet routable via `ztcs-igw` |
| Private subnet | `ztcs-private-subnet` — `10.0.2.0/24` — isolated, no internet routing |
| Internet Gateway | `ztcs-igw` — attached to VPC, routed for public subnet only |
| Route tables | Public: `0.0.0.0/0 → ztcs-igw`. Private: local only (no internet route) |
| Security Group (perimeter) | SSH (0.0.0.0/0), HTTP (80), HTTPS (443), WireGuard (51820/UDP) |
| Security Group (services) | SSH, HTTP, 8065, 8080, 3306 — restricted to perimeter SG or self-referencing. WireGuard (51820/UDP) |
| EC2 perimeter | `ztcs-perimeter` — t3.small — Ubuntu 24.04 LTS — public subnet — 20 GB gp3 — `10.0.1.9` |
| EC2 services | `ztcs-services` — t3.medium — Ubuntu 24.04 LTS — private subnet — 30 GB gp3 — `10.0.2.220` |
| SSH access | Verified — direct to perimeter, jump host to services via ssh-agent forwarding |
| Internal connectivity | Verified — TCP port 22 reachable between instances (ICMP blocked by Security Groups by design) |
| Internet isolation | Verified — private subnet has no outbound internet connectivity |

**Next step:** WireGuard VPN tunnel between `ztcs-perimeter` and the on-premise Windows Server (`192.168.56.10`).