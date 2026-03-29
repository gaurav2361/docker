# Serve: The Marketing Agency Master Identity & Storage Guide

**Serve** is a professional-grade ecosystem designed for high-performance marketing agencies (20+ users). It unifies enterprise-level storage with a modern identity fabric based on **LLDAP** and **Authelia OIDC**.

---

## 🏗️ Architecture & Strategy: Why These Tools?

Every component in this stack has been selected to solve a specific enterprise marketing challenge:

1.  **Samba (The Heavy Lifter):** Native local file-sharing. Moves massive folders (100GB+) at maximum network speed directly from your Mac/Windows as a mounted drive. Essential for direct 4K video editing.
2.  **NextExplorer (The Web Bridge):** Secure web interface for guest access. Share folders via URL without exposing your internal network or requiring complex VPNs for clients.
3.  **Tdarr (The Optimizer):** Automated background robot that converts videos to H.265 (HEVC), saving terabytes of storage space with zero manual effort. Maintains high visual quality for marketing assets.
4.  **Immich (The Gallery Experience):** High-performance Google Photos alternative with facial recognition and object detection. Provides a beautiful, searchable timeline for clients and stakeholders.
5.  **Authelia (The Security Gatekeeper):** Single Sign-On (SSO) with OIDC support. One secure "front door" for all your web apps, eliminating password fatigue for your 20+ users.
6.  **LLDAP (The Identity Brain):** A modern, lightweight LDAP directory that acts as the single source of truth for all users and groups.

---

## 📁 Directory Structure (The Root-Split)

We maintain a strict separation of concerns to simplify backups and migrations:

-   **/docker:** Stores all persistent application data, configuration files, and databases.
-   **/data:** Stores all actual marketing assets (Photos, Projects, Final Renders).

---

## 🔐 The Identity Fabric: OIDC + LDAP Deep-Dive

Security is handled via a three-tier handshake that ensures a single set of credentials works everywhere.

### 1. The LLDAP Backend
LLDAP stores your 20+ users. 
-   **Technical Identity:** It uses a `dc=marketing,dc=com` base DN.
-   **Zero-Touch Samba:** The `ENABLE_SAMBA=true` flag forces LLDAP to generate the legacy NTLM hashes required by Samba. When you create a user in the Web UI, their network drive access is created automatically.

### 2. The Authelia OIDC Gatekeeper
Authelia translates LDAP data into modern OIDC tokens for NextExplorer and Immich.
-   **Connection Logic:** Authelia pings LLDAP via the internal `media_net` using custom filters to map LDAP groups into OIDC claims.

---

## 👥 Team Roles & Permissions Matrix

We manage access via **Groups**, not individual users.

| LDAP Group | App Role | Samba Access | Marketing Workflow |
| :--- | :--- | :--- | :--- |
| `admins` | Superuser | Full Control | Server & User Management |
| `editors` | Power User | R/W Access | Direct 4K Video/RAW editing |
| `marketing` | Web User | Read-Only | Distributing assets via NextExplorer |
| `clients` | Guest | No Access | Secure web links only |

### Onboarding Example: Senior Editor
1.  **IT Step:** Create `rahul_edit` in LLDAP (`:17170`) and add to `editors` group.
2.  **Outcome:** Rahul can instantly mount the Samba drive AND log into the media gallery with one password.

---

## 🚀 Technical Implementation Guide

### Step 1: Host Preparation & Firewall
```bash
# 1. Create directory structures
sudo mkdir -p /docker/{lldap/data,authelia/config,nextexplorer/{config,cache},tdarr/{server,configs},immich/{upload,postgres}}
sudo mkdir -p /data

# 2. Configure UFW (Ubuntu Firewall)
sudo ufw allow 445/tcp    # Samba SMB
sudo ufw allow 17170/tcp  # LLDAP Admin UI
sudo ufw allow 3000/tcp   # NextExplorer
sudo ufw allow 9091/tcp   # Authelia SSO
sudo ufw allow 2283/tcp   # Immich Gallery
sudo ufw reload
```

### Step 2: "Zero-Touch" Samba Configuration
Update `/etc/samba/smb.conf` to point to the LLDAP container. This removes the need for `smbpasswd` manual entries.

```ini
[global]
   passdb backend = ldapsam:ldap://localhost:3890
   ldap suffix = dc=marketing,dc=com
   ldap admin dn = cn=admin,dc=marketing,dc=com
   ldap ssl = off

[Marketing_Assets]
   path = /data
   valid users = @editors, @admins
   force user = gaurav
   force group = gaurav
   create mask = 0775
   directory mask = 0775
```

### Step 3: Authelia OIDC Integration
In your Authelia `configuration.yml`, use these precise settings to bridge to LLDAP:

```yaml
authentication_backend:
  ldap:
    implementation: custom
    address: 'ldap://lldap:3890'
    base_dn: 'dc=marketing,dc=com'
    users_filter: '(&({username_attribute}={input})(objectClass=person))'
    groups_filter: '(&(member={dn})(objectClass=groupOfNames))'
```

### Step 4: NextExplorer & Immich (OIDC Handshake)
NextExplorer uses **space-separated** scopes to automate permissions:
```ini
OIDC_SCOPES="openid profile email groups"
OIDC_ADMIN_GROUPS=admins
```

---

## ⚙️ Nitro-Boost: Mac M4 Tdarr Node
For video editors on M4 Macs, use **Path Translation** in `Tdarr_Node_Config.json` to leverage hardware acceleration over the network:

```json
{
  "nodeName": "Editor-Station-M4",
  "serverIP": "<SERVER_IP>",
  "serverPort": "8266",
  "pathTranslators": [
    {
      "server": "/data",
      "node": "/Volumes/Marketing_Assets"
    }
  ]
}
```

---

## 🛠️ Maintenance, Backups & Alerts

-   **The "Golden" Backup:** The file `/docker/lldap/data/users.db` contains your entire company identity. Back it up daily.
-   **SMTP Alerts:** Configure the SMTP section in `.env` so IT receives email alerts for server health and login security.
-   **Logs:** 
    - Identity issues: `docker logs lldap`
    - Login issues: `docker logs authelia`

---

## 🔗 Connection Summary

| Access Method | URL / Path |
| :--- | :--- |
| **Samba (Internal Drive)** | `smb://${LOCAL_IP}/Marketing_Assets` |
| **Identity Admin (LLDAP)** | `http://${LOCAL_IP}:17170` |
| **Asset Bridge (NextExplorer)** | `${PRIMARY_BASE_URL}:3000` |
| **Media Gallery (Immich)** | `${PRIMARY_BASE_URL}:2283` |
| **SSO Identity Portal** | `${PRIMARY_BASE_URL}:9091` |
