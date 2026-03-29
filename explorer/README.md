# Serve: Enterprise Marketing Storage & Identity Stack

This repository contains the configuration for **Serve**, a high-performance ecosystem tailored for marketing teams (15-20+ users). It features **Unified Identity Management**—one single username/password for both the network drive (Samba) and all web applications.

---

## 🏗 Why This Architecture? (Enterprise Strategy)

### 1. Unified Identity (LLDAP + Authelia)
- **The Problem:** Managing 20 users separately in Samba and Authelia is a nightmare.
- **The Solution:** We use **LLDAP** as the single source of truth. When you add a user to LLDAP, they instantly gain access to the Samba drive and the Web SSO.
- **Authelia** connects to LLDAP to provide secure Single Sign-On (SSO) for NextExplorer and Immich.

### 2. High-Performance Storage (Samba)
- Marketing assets (4K video, high-res RAW photos) require extreme speeds. Samba provides native network drive performance, allowing your team to edit directly off the server.

### 3. Automated Asset Optimization (Tdarr)
- Automatically converts massive marketing videos to optimized H.265 (HEVC) in the background, saving terabytes of space while maintaining visual quality.

- A Read-Only gallery for clients and team members to browse assets, featuring facial recognition and object detection for quick searching.

---

## 📁 Directory Structure

- **/docker**: All configuration, databases, and application data.
- **/data**: All marketing assets (Raw, Projects, Final).

---

## 🚀 Implementation Phase 1: Host & Identity

### 1. Host Preparation
```bash
sudo mkdir -p /docker/{lldap/data,authelia/config,nextexplorer/{config,cache},tdarr/{server,configs},immich/{upload,postgres}}
sudo mkdir -p /data
sudo chown -R gaurav:gaurav /docker /data
```

### 2. Configure Firewall (UFW)
For a multi-user environment, ensuring the firewall is open but secure is critical:
```bash
sudo ufw allow 445/tcp    # Samba (Internal Drive)
sudo ufw allow 17170/tcp  # LLDAP UI (Admin only)
sudo ufw allow 3000/tcp   # NextExplorer
sudo ufw allow 9091/tcp   # Authelia SSO
sudo ufw allow 2283/tcp   # Immich Gallery
sudo ufw reload
```

### 3. Identity Setup (LLDAP)
1. Copy `.env.example` to `.env` and fill in your passwords.
2. Run `docker compose up -d lldap`.
3. Access `http://<IP>:17170`. 
4. Login with `admin` and your `LDAP_ADMIN_PASSWORD`.
5. **Create your Users:** Add your 15-20 team members here.
6. **Create a Group:** Create a group named `admins` and add yourself to it.

---

## 🚀 Implementation Phase 2: Unified Samba

To make Samba use your LLDAP users, we connect the host OS to the LDAP container.

### 1. Install LDAP Client on Ubuntu
```bash
sudo apt update && sudo apt install libnss-ldap libpam-ldap ldap-utils -y
```

### 2. Connect Samba to LLDAP
Edit `/etc/samba/smb.conf`. Instead of local users, we point to the LLDAP container:
```ini
[global]
   workgroup = WORKGROUP
   security = user
   passdb backend = ldapsam:ldap://localhost:3890
   ldap suffix = dc=marketing,dc=com
   ldap admin dn = cn=admin,dc=marketing,dc=com
   ldap ssl = off

[Marketing_Assets]
   path = /data
   valid users = @users
   read only = no
   force user = gaurav
   force group = gaurav
   create mask = 0775
   directory mask = 0775
```

---

## 🚀 Implementation Phase 3: Web Apps

### 1. Authelia OIDC Setup
Ensure your Authelia `configuration.yml` points to LLDAP as the authentication backend:
```yaml
authentication_backend:
  ldap:
    implementation: custom
    url: ldap://lldap:3890
    timeout: 5s
    base_dn: dc=marketing,dc=com
    users_filter: "(&(|(uid={input})(mail={input}))(objectClass=person))"
    groups_filter: "(member={dn})"
```

### 2. Immich SSO (Manual Step)
1. Log into Immich as admin -> **Settings -> OAuth**.
2. **Issuer URL:** `http://<SERVER_IP>:9091`
3. **Client ID:** `immich`
4. **Client Secret:** (Generate one in Authelia config)
5. **Auto Register:** Enabled (This creates their gallery profile on first login).

---

## ⚙️ Tdarr Marketing Node (Mac M4)
For marketing editors on Mac M4, use **Path Translation** in `Tdarr_Node_Config.json`:
```json
{
  "nodeName": "Editor-M4",
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

## 🔗 Connection Summary
| Access Method | Path / URL |
| :--- | :--- |
| **Samba (Internal)** | `smb://${LOCAL_IP}/Marketing_Assets` |
| **Identity Admin** | `http://${LOCAL_IP}:17170` |
| **Asset Bridge** | `${PRIMARY_BASE_URL}:3000` |
| **Media Gallery** | `${PRIMARY_BASE_URL}:2283` |
