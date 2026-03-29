# Serve: The Marketing Agency Master Identity & Storage Guide

**Serve** is a professional-grade ecosystem designed for high-performance marketing agencies (20+ users). It unifies enterprise-level storage with a modern identity fabric based on **LLDAP** and **Authelia OIDC**.

---

## 🏗️ Architecture & Strategy: Why These Tools?

1.  **Samba (The Heavy Lifter):** Native local file-sharing. Moves massive folders (100GB+) at maximum network speed directly from your Mac/Windows as a mounted drive. Essential for direct 4K video editing.
2.  **NextExplorer (The Web Bridge):** Secure web interface for guest access. Share folders via URL without exposing your internal network or requiring complex VPNs for clients.
3.  **Tdarr (The Optimizer):** Automated background robot that converts videos to H.265 (HEVC), saving terabytes of storage space with zero manual effort. Maintains high visual quality for marketing assets.
4.  **Immich (The Gallery Experience):** High-performance Google Photos alternative with facial recognition and object detection. Provides a beautiful, searchable timeline for clients and stakeholders.
5.  **Authelia (The Security Gatekeeper):** Single Sign-On (SSO) with OIDC support. One secure "front door" for all your web apps, eliminating password fatigue for your 20+ users.
6.  **LLDAP (The Identity Brain):** A modern, lightweight LDAP directory that acts as the single source of truth for all users and groups.

---

## 📁 Directory Structure (The Root-Split)

We maintain a strict separation of concerns to simplify backups and migrations:

- **/docker:** Stores all persistent application data, configuration files, and databases.
- **/data:** Stores all actual marketing assets (Photos, Projects, Final Renders).

---

## 👥 Team Roles & Permissions Matrix

| LDAP Group  | App Role   | Samba Access | Marketing Workflow                   |
| :---------- | :--------- | :----------- | :----------------------------------- |
| `admins`    | Superuser  | Full Control | Server & User Management             |
| `editors`   | Power User | R/W Access   | Direct 4K Video/RAW editing          |
| `marketing` | Web User   | Read-Only    | Distributing assets via NextExplorer |
| `clients`   | Guest      | No Access    | Secure web links only                |

---

## 🚀 Technical Implementation Guide (IT Runbook)

### Phase 1: Filesystem & Security Hardening

First, establish the dual-root structure and generate the cryptographic secrets required for OIDC.

```bash
# 1. Create directory structures
sudo mkdir -p /docker/{lldap/data,authelia/config,nextexplorer/{config,cache},tdarr/{server,configs},immich/{upload,postgres}}
sudo mkdir -p /data

# 2. Set ownership (Assuming PUID/PGID 1000)
sudo chown -R gaurav:gaurav /docker /data
sudo find /data -type d -exec chmod 775 {} \;
sudo find /data -type f -exec chmod 664 {} \;

# 3. Generate Authelia Secrets (Critical for OIDC)
# Run these and save the outputs for the configuration file at:
# /docker/authelia/config/configuration.yml
openssl rand -hex 64 # JWT_SECRET
openssl rand -hex 64 # SESSION_SECRET
openssl rand -hex 64 # STORAGE_ENCRYPTION_KEY
```

### Phase 2: Network & Identity Initialization

Boot the identity core and configure the system firewall to allow internal/external traffic.

```bash
# 1. Deploy LLDAP
# Configuration stored at: /docker/lldap/data/lldap_config.toml (Auto-generated on first run)
docker compose up -d lldap

# 2. Configure UFW (Ubuntu Firewall)
sudo ufw allow 445/tcp    # Samba SMB
sudo ufw allow 17170/tcp  # LLDAP Admin UI
sudo ufw allow 3000/tcp   # NextExplorer
sudo ufw allow 9091/tcp   # Authelia SSO
sudo ufw allow 2283/tcp   # Immich Gallery
sudo ufw reload
```

### Phase 3: "Zero-Touch" Samba Integration

This step connects the Host OS to the LDAP container, enabling automatic network drive access for all 20+ users.

1. **Install LDAP Client Utilities:**
   `sudo apt update && sudo apt install libnss-ldap libpam-ldap ldap-utils -y`

2. **Configure the host file at `/etc/samba/smb.conf`:**

```ini
[global]
   # Identity Backend
   passdb backend = ldapsam:ldap://localhost:3890
   ldap suffix = dc=marketing,dc=com
   ldap admin dn = cn=admin,dc=marketing,dc=com
   ldap ssl = off

   # Optimization for 4K Video Editing
   min receivefile size = 16384
   use sendfile = yes
   aio read size = 16384
   aio write size = 16384

[Drive]
   path = /data
   valid users = @editors, @admins
   force group = gaurav
   create mask = 0775
   directory mask = 0775
   browseable = yes
   writable = yes
```

### Phase 4: Web SSO & App Integration

Bridge Authelia to LLDAP and enable the OIDC handshake for the marketing apps.

1. **Authelia ↔ LLDAP Config Path:** `/docker/authelia/config/configuration.yml`

```yaml
authentication_backend:
  ldap:
    implementation: custom
    address: "ldap://lldap:3890"
    base_dn: "dc=marketing,dc=com"
    users_filter: "(&({username_attribute}={input})(objectClass=person))"
    groups_filter: "(&(member={dn})(objectClass=groupOfNames))"
```

2. **App Deployment:**

```bash
# Execute in your project root (where docker-compose.yml lives)
docker compose up -d
```

3. **Immich SSO (Manual Finalization):**
   - Log into Immich Web UI -> **Administration -> Settings -> OAuth**.
   - Set **Issuer URL** to your Authelia domain/IP.
   - Set **Scope** to `openid profile email groups`.
   - Enable **Auto Register** to allow 20+ users to onboard instantly.

---

## ⚙️ Nitro-Boost: Mac M4 Hardware Acceleration

For editors on M4 Macs, download the Tdarr Node binary and edit its local configuration file (usually `Tdarr_Node_Config.json` in the unzipped folder):

```json
{
  "nodeName": "Lead-Editor-M4",
  "serverIP": "<SERVER_IP>",
  "pathTranslators": [
    {
      "server": "/data",
      "node": "/Volumes/Drive"
    }
  ]
}
```

---

## 🛠️ Maintenance & High Availability

- **Database Backups:** Use a cron job to backup the file at: `/docker/lldap/data/users.db`. If this file is lost, all 20+ users lose access.
- **SSO Troubleshooting:** To view live authentication errors, run: `docker logs -f authelia`.
- **Reverse Proxy:** For domain access (`media.marketingcompany.com`), use a proxy to point to the server's local ports.

---

## 🔗 Connection Summary

| Service               | Protocol | URL / Path                           |
| :-------------------- | :------- | :----------------------------------- |
| **Asset Storage**     | SMB      | `smb://${LOCAL_IP}/Drive` |
| **Team Directory**    | HTTP     | `http://${LOCAL_IP}:17170`           |
| **Web Asset Bridge**  | HTTPS    | `${PRIMARY_BASE_URL}:3000`           |
| **Marketing Gallery** | HTTPS    | `${PRIMARY_BASE_URL}:2283`           |
| **Identity Portal**   | HTTPS    | `${PRIMARY_BASE_URL}:9091`           |
