# Serve: The Marketing Agency Master Identity & Storage Guide

This manual defines the definitive technical architecture of **Serve**, an enterprise-grade ecosystem tailored for 20+ users. It unifies high-performance storage (Samba) with a robust identity fabric using **LLDAP** and **Authelia OIDC**.

---

## 🏗️ Technical Identity Fabric: The "Handshake"

The security of your agency's assets relies on a dual-layer identity strategy. This ensures that every team member has a single identity for web apps while maintaining high-speed, secure access to the network drive.

### 1. The LLDAP Backend (Web Identity)
**LLDAP** is the "Source of Truth" for your 20+ web users.
- **Base DN:** `dc=marketing,dc=com`
- **Role:** Authelia connects to LLDAP to provide **Single Sign-On (SSO)** for NextExplorer and Immich.

### 2. The Samba Reality (Local Storage)
**Samba** requires legacy NTLM hashes to authenticate users. Because LLDAP is a modern, lightweight directory, it does **not** store these hashes.
- **The Strategy:** For maximum reliability, we use a "Dual-Management" approach. 
- **The Workflow:** When you onboard a new editor, you create their account in the **LLDAP Web UI** (for web apps) and then run a single command on the **Ubuntu Server** (for the network drive).

---

## 👥 Unified Team Roles & Scopes

To manage 20+ users, we use **OIDC Scopes**. NextExplorer requests `openid profile email groups` to automate permissions.

| LDAP Group | OIDC Claim | Samba Access | NextExplorer Role |
| :--- | :--- | :--- | :--- |
| `admins` | `groups: admins` | Full R/W | **Superuser** (Auto-Elevated) |
| `editors` | `groups: editors` | High-Speed R/W | **Editor** (Project Management) |
| `marketing` | `groups: marketing` | Read-Only | **User** (Share Creation) |
| `guests` | `groups: guests` | No Access | **Guest** (Link Access) |

---

## 🛡️ Deep Technical Configurations

### 1. Authelia ↔ LLDAP Connection
Authelia connects to LLDAP via the internal `media_net`. Use these precise settings:
```yaml
authentication_backend:
  ldap:
    implementation: custom
    address: 'ldap://lldap:3890'
    base_dn: 'dc=marketing,dc=com'
    additional_users_dn: 'ou=people'
    users_filter: '(&({username_attribute}={input})(objectClass=person))'
    additional_groups_dn: 'ou=groups'
    groups_filter: '(&(member={dn})(objectClass=groupOfNames))'
```

### 2. NextExplorer OIDC Variables
Based on industry standards, use **space-separated** scopes:
```ini
OIDC_ENABLED=true
OIDC_ISSUER=https://auth.marketingcompany.com
OIDC_CLIENT_ID=nextexplorer
OIDC_SCOPES="openid profile email groups"
OIDC_ADMIN_GROUPS=admins
OIDC_AUTO_CREATE_USERS=true
```

---

## 🚀 Step-by-Step Server Setup

### Step 1: Initialize Identity (LLDAP)
1.  Run `docker compose up -d lldap`.
2.  Access `http://<IP>:17170` (Admin / your password).
3.  Create your team groups (`admins`, `editors`, `marketing`) and add your users.

### Step 2: Samba User Management (NTLM Hashes)
To enable the network drive for a user created in LLDAP, you must sync their password to the Samba database once:
```bash
# Example: Adding 'rahul_edit' to the network drive
sudo smbpasswd -a rahul_edit
```

### Step 3: Enable Web SSO
1.  Run `docker compose up -d authelia nextexplorer`.
2.  Team members click "Continue with SSO" and log in with their LLDAP credentials.
3.  Users in the `admins` LDAP group are automatically promoted to NextExplorer Admins.

---

## ⚙️ Tdarr Nitro Boost (Mac M4 Node)
For your video editors on M4 Macs, use this configuration in `Tdarr_Node_Config.json` to leverage the M4's Media Engine over the network:

```json
{
  "nodeName": "Main-M4-Station",
  "serverIP": "<SERVER_IP>",
  "serverPort": "8266",
  "pathTranslators": [
    {
      "server": "/data",
      "node": "/Volumes/Data"
    }
  ]
}
```

---

## 🛠️ Maintenance & Backup
- **Identity Backup:** Back up `/docker/lldap/data/users.db` daily. This is the "brain" of your agency.
- **SSO Refresh:** If you add a user to a new group in LLDAP, they must log out and back into the web apps to refresh their permissions.
- **SMTP Alerts:** Ensure SMTP is configured in `.env` so you receive alerts for failed login attempts or server issues.

---

## 🔗 Connection Links Summary

| Access Method | URL / Path |
| :--- | :--- |
| **Internal High-Speed Drive** | `smb://${LOCAL_IP}/Data` |
| **Team Management (Admin)** | `http://${LOCAL_IP}:17170` |
| **Asset Distribution (Web)** | `${PRIMARY_BASE_URL}:3000` |
| **Media Gallery** | `${PRIMARY_BASE_URL}:2283` |
| **Identity Portal** | `${PRIMARY_BASE_URL}:9091` |
