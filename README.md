# 🔐 Centralized Authentication System with FreeIPA

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-yellow?style=flat-square)
![Stack](https://img.shields.io/badge/Stack-FreeIPA%20%7C%20LDAP%20%7C%20Kerberos%20%7C%20DNS-blue?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-Linux-informational?style=flat-square)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat-square)

> Deploy FreeIPA to establish a centralized authentication system for Linux environments — providing unified identity management, authentication, and authorization services across your infrastructure.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Technologies](#technologies)
- [Installation & Setup](#installation--setup)
  - [1. Install FreeIPA Server & Configure DNS](#1-install-freeipa-server--configure-dns)
  - [2. User Accounts, Groups & Roles](#2-user-accounts-groups--roles)
  - [3. Integrate Linux Clients](#3-integrate-linux-clients)
  - [4. Test Authentication & Access Control](#4-test-authentication--access-control)
- [Architecture Diagram](#architecture-diagram)
- [User Management Reference](#user-management-reference)
- [Delivery Report](#delivery-report)
- [Future Scalability](#future-scalability)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

This project implements a **centralized identity and access management (IAM)** layer for Linux infrastructure using [FreeIPA](https://www.freeipa.org/). By consolidating authentication through a single system, organizations eliminate per-host user management, enforce consistent access policies, and gain centralized audit trails.

**Key capabilities delivered:**

- Single Sign-On (SSO) across Linux hosts via Kerberos
- Centralized user and group directory via LDAP
- Role-Based Access Control (RBAC)
- Integrated DNS management
- Host-based access control (HBAC) policies
- Sudo rules management

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    FreeIPA Server                       │
│                                                         │
│  ┌───────────┐  ┌───────────┐  ┌──────────────────┐     │
│  │  Kerberos │  │   LDAP    │  │    DNS / NTP     │     │
│  │    KDC    │  │ Directory │  │   (BIND + ntpd)  │     │
│  └───────────┘  └───────────┘  └──────────────────┘     │
│                                                         │
│  ┌───────────┐  ┌───────────┐  ┌──────────────────┐     │
│  │  CA (PKI) │  │  HBAC     │  │   Sudo Rules     │     │
│  └───────────┘  └───────────┘  └──────────────────┘     │
└───────────────────────┬─────────────────────────────────┘
                        │  SSSD / ipa-client
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
  ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ Linux    │   │ Linux    │   │ Linux    │
  │ Client 1 │   │ Client 2 │   │ Client N │
  └──────────┘   └──────────┘   └──────────┘
```

---

## Prerequisites

Before proceeding, ensure the following:

| Requirement | Details |
|---|---|
| OS | RHEL 8/9, CentOS Stream 8/9, Rocky Linux 8/9, or Fedora |
| RAM | Minimum 2 GB (4 GB recommended for server) |
| CPU | Minimum 2 vCPUs |
| Disk | Minimum 10 GB free on `/var` |
| Network | Static IP, resolvable FQDN, ports 80/443/88/389/636/464 open |
| DNS | Ability to configure forward/reverse zones |
| Knowledge | Basic Linux sysadmin and networking concepts |

>  **Note:** FreeIPA requires exclusive use of ports 80 and 443. Ensure no web server is running on the target machine before installation.

---

## Technologies

| Technology | Role |
|---|---|
| **FreeIPA** | Central IAM platform — orchestrates all components below |
| **Kerberos** | Network authentication protocol providing SSO |
| **LDAP (389 Directory Server)** | User/group/policy directory backend |
| **DNS (BIND)** | Name resolution, SRV records for service discovery |
| **SSSD** | System Security Services Daemon — client-side auth cache |
| **Dogtag CA** | Built-in certificate authority for PKI |

---

## Installation & Setup

### 1. Install FreeIPA Server & Configure DNS

**Install packages:**

```bash
# RHEL/CentOS/Rocky
sudo dnf install -y freeipa-server freeipa-server-dns

# Set the hostname (must be FQDN)
sudo hostnamectl set-hostname ipa.example.com
```

**Run the installer:**

```bash
sudo ipa-server-install \
  --setup-dns \
  --allow-zone-overlap \
  --no-forwarders \
  --domain=example.com \
  --realm=EXAMPLE.COM \
  --ds-password='<DirectoryManagerPassword>' \
  --admin-password='<AdminPassword>' \
  --unattended
```

**Post-install — configure firewall:**

```bash
sudo firewall-cmd --permanent --add-service={freeipa-ldap,freeipa-ldaps,dns,ntp,https,http,kerberos,kpasswd}
sudo firewall-cmd --reload
```

**Verify DNS is operational:**

```bash
dig +short _kerberos._udp.example.com SRV
dig +short _ldap._tcp.example.com SRV
```

---

### 2. User Accounts, Groups & Roles

**Authenticate as admin:**

```bash
kinit admin
```

**Create a user:**

```bash
ipa user-add jdoe \
  --first="John" \
  --last="Doe" \
  --email="jdoe@example.com" \
  --shell=/bin/bash
```

**Create a group and add a user:**

```bash
ipa group-add devops --desc="DevOps Engineers"
ipa group-add-member devops --users=jdoe
```

**Create a role and assign permissions:**

```bash
ipa role-add "DevOps Role" --desc="Role for DevOps engineers"
ipa role-add-member "DevOps Role" --groups=devops
ipa role-add-privilege "DevOps Role" --privileges="Host Administrators"
```

**Create a Host-Based Access Control (HBAC) rule:**

```bash
ipa hbacrule-add allow_devops_ssh \
  --servicecat=all \
  --desc="Allow DevOps SSH to managed hosts"

ipa hbacrule-add-user allow_devops_ssh --groups=devops
ipa hbacrule-add-host allow_devops_ssh --hostgroups=managed-servers
```

**Create a Sudo rule:**

```bash
ipa sudorule-add devops_sudo --cmdcat=all --hostcat=all
ipa sudorule-add-user devops_sudo --groups=devops
```

---

### 3. Integrate Linux Clients

**On each client host — install the IPA client:**

```bash
sudo dnf install -y freeipa-client sssd sssd-tools
```

**Enroll the client:**

```bash
sudo ipa-client-install \
  --server=ipa.example.com \
  --domain=example.com \
  --realm=EXAMPLE.COM \
  --principal=admin \
  --password='<AdminPassword>' \
  --mkhomedir \
  --unattended
```

**Verify SSSD is running and the client is enrolled:**

```bash
sudo systemctl status sssd
sudo ipa host-show $(hostname)
```

---

### 4. Test Authentication & Access Control

**Test Kerberos ticket acquisition:**

```bash
kinit jdoe
klist
```

**Test SSH login from enrolled client:**

```bash
ssh jdoe@client.example.com
```

**Verify HBAC policies:**

```bash
# On the IPA server
ipa hbactest --user=jdoe --host=client.example.com --service=sshd
```

**Check SSSD cache and user resolution:**

```bash
id jdoe
getent passwd jdoe
sss_cache -E  # Flush SSSD cache if needed
```

**Test sudo access:**

```bash
sudo -l -U jdoe
```

---

## User Management Reference

| Operation | Command |
|---|---|
| List all users | `ipa user-find` |
| Disable a user | `ipa user-disable <username>` |
| Reset a password | `ipa user-mod <username> --password` |
| List groups | `ipa group-find` |
| View HBAC rules | `ipa hbacrule-find` |
| View Sudo rules | `ipa sudorule-find` |
| Check active Kerberos sessions | `klist -a` |
| Destroy a Kerberos ticket | `kdestroy` |

---

## Delivery Report

### Implemented Features

| Feature | Status | Notes |
|---|---|---|
| FreeIPA server deployment | (yes) Complete | Single-server topology |
| Integrated DNS (BIND) | (yes) Complete | Forward/reverse zones configured |
| Kerberos KDC | (yes) Complete | Realm `EXAMPLE.COM` |
| LDAP directory | (yes) Complete | 389 Directory Server backend |
| User & group provisioning | (yes) Complete | Includes role assignments |
| Host enrollment (SSSD) | (yes) Complete | Tested on 3 client hosts |
| HBAC policies | (yes) Complete | SSH access restricted by group |
| Sudo rules | (yes) Complete | Centrally managed via IPA |
| PKI / CA | (yes) Complete | Dogtag CA issuing host certificates |

### Security Considerations

- All LDAP traffic enforced over LDAPS (port 636)
- Kerberos ticket lifetimes set to 10h with 7-day renewable window
- `admin` account used only for initial setup — a dedicated `ipa-admin` service account is recommended for automation
- HBAC default policy set to **deny all** — access granted explicitly

---

## Future Scalability

The following enhancements are recommended as the environment grows:

**High Availability**
Deploy a FreeIPA replica for fault tolerance and load distribution. Replication is built into FreeIPA and can be enabled with `ipa-replica-install`.

**Active Directory Trust**
Establish a cross-forest trust with Microsoft Active Directory using `ipa trust-add` to allow AD users to authenticate against Linux systems without account duplication.

**Automation & IaC**
Manage FreeIPA configurations through Ansible using the [`freeipa` collection](https://github.com/freeipa/ansible-freeipa), enabling repeatable, version-controlled identity management.

**MFA / OTP**
Enable TOTP-based two-factor authentication natively through FreeIPA's OTP feature (`ipa otptoken-add`) for privileged accounts.

**Monitoring**
Integrate FreeIPA health checks into your observability stack. Key metrics to watch: replication status, KDC availability, LDAP response time, and certificate expiry.

---

## Contributing

1. Fork this repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit your changes: `git commit -m 'Add your feature'`
4. Push to the branch: `git push origin feature/your-feature`
5. Open a Pull Request

Please ensure all shell scripts pass `shellcheck` before submitting.

---



---

> **Resources:** [FreeIPA Official Docs](https://www.freeipa.org/page/Documentation) · [Red Hat IdM Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/installing_identity_management/) · [ansible-freeipa](https://github.com/freeipa/ansible-freeipa)
