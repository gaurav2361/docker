# Serve: The Ultimate Storage & Media Stack

This repository contains the configuration for **Serve**, a high-performance self-hosted storage and media ecosystem. It is designed for a central Ubuntu server, maintaining a strict split between `/docker` (configs) and `/data` (storage).

---

## 🏗 Why These Tools? (Architecture & Strategy)

1.  **Samba (The Heavy Lifter):** Native local file-sharing. Moves massive folders (100GB+) at maximum network speed directly from your Mac/Windows as a mounted drive.
2.  **NextExplorer (The Web Bridge):** Secure web interface for guest access. Share folders via URL without exposing your internal network or requiring complex VPNs.
3.  **Tdarr (The Optimizer):** Automated background robot that converts videos to H.265 (HEVC), saving terabytes of storage space with zero manual effort.
4.  **Immich (The Gallery Experience):** High-performance Google Photos alternative with facial recognition, mobile backup, and a beautiful timeline.
5.  **Authelia (The Security Gatekeeper):** Single Sign-On (SSO) with OIDC support. One secure "front door" for all your web apps.

---

## 📁 Directory Structure

We use a root-level separation of concerns:
- **/docker**: Stores all persistent application data, configuration files, and databases.
- **/data**: Stores all actual media files (photos, videos) and the master shared folder.

---

## 🔐 The Permissions Deep Dive (PUID/PGID)

To avoid "Access Denied" errors, every app is forced to act as the same user (`UID 1000`).
- **The Solution:** We use `PUID=1000` and `PGID=1000` in Docker.
- **Samba:** We use "force user" masks to ensure any file you drop from your Mac matches the server owner instantly.

---

## 🚀 Step-by-Step Implementation

### Step 1: Host Preparation
```bash
# 1. Create root structures
sudo mkdir -p /docker/{authelia/config,nextexplorer/{config,cache},tdarr/{server,configs},immich/{model-cache,postgres}}
sudo mkdir -p /data/{shared_media,immich_internal}

# 2. Take ownership
sudo chown -R gaurav:gaurav /docker /data

# 3. Set standard permissions
sudo find /data -type d -exec chmod 775 {} \;
sudo find /data -type f -exec chmod 664 {} \;
```

### Step 2: Install & Configure Samba
Install: `sudo apt update && sudo apt install samba -y`.
Edit `/etc/samba/smb.conf` and add to the bottom:
```ini
[Shared_Media]
   path = /data/shared_media
   valid users = gaurav
   read only = no
   browsable = yes
   force user = gaurav
   force group = gaurav
   create mask = 0775
   directory mask = 0775
```
`sudo systemctl restart smbd`.

### Step 3: Create the `.env` File (Dual Access Support)
Create a `.env` file and populate it with your credentials. You can define both your **Local IP** and your **Domain** name.

```ini
TZ=Asia/Kolkata
PUID=1000
PGID=1000

# Network Configuration
LOCAL_IP=192.168.1.XX
DOMAIN=media.yourdomain.com

# Active Identity (Which one the apps use for OIDC/Links)
# Set to http://${LOCAL_IP} for home-only access
# Set to https://${DOMAIN} for external/domain access (requires reverse proxy)
PRIMARY_BASE_URL=http://${LOCAL_IP}

# NextExplorer SSO
OIDC_CLIENT_ID=nextexplorer
OIDC_CLIENT_SECRET=your_secure_secret

# Immich Database
IMMICH_DB_USERNAME=immich
IMMICH_DB_PASSWORD=immich_secure_password
IMMICH_DB_NAME=immich
POSTGRES_USER=immich
POSTGRES_PASSWORD=immich_secure_password
POSTGRES_DB=immich
```

### Step 4: Deploy the Stack
```bash
docker compose up -d
```

---

## 🌐 Hybrid Access (IP & Domain)

You can use both your **Local IP** and **Domain** simultaneously:

1.  **Samba (Internal Only):** Always use your `LOCAL_IP` for mounting the drive on Mac/Windows. It provides the fastest speeds and doesn't rely on your internet connection.
2.  **Web Apps (Internal/External):** 
    - Use `PRIMARY_BASE_URL=http://${LOCAL_IP}` while setting things up.
    - Switch to `PRIMARY_BASE_URL=https://${DOMAIN}` when you want to share links with guests or use SSO externally.
    - **Note:** If you use a Domain, you must have a Reverse Proxy (Nginx Proxy Manager, Cloudflare Tunnels, etc.) pointing to the server's local ports.

---

## 👥 User Management Examples

### 1. Creating a Samba User
`sudo smbpasswd -a <username>`

### 2. Creating an Authelia User (SSO)
First, generate a secure Argon2 hash for your password:
`docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'YourNewPassword'`

Then, edit `/docker/authelia/config/users.yml` and paste the output:
```yaml
users:
  gaurav:
    displayname: "Gaurav"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..." 
    email: gaurav@domain.com
    groups: [admins]
```

---

## ⚙️ Tdarr Hardware Acceleration (Mac M4 Node)
To use an Apple Silicon Mac to transcode videos over the network at blazing speeds, download the macOS Tdarr Node and edit the `Tdarr_Node_Config.json` file. 

You must include **Path Translation** so your Mac knows how to map the Ubuntu server path to the local Samba mount:

```json
{
  "nodeName": "M4-MacBook",
  "serverIP": "192.168.1.XX",
  "serverPort": "8266",
  "pathTranslators": [
    {
      "server": "/data/shared_media",
      "node": "/Volumes/Shared_Media"
    }
  ]
}
```

---

## 🔗 Connection Links Summary

| Access Method | Link / Path |
| :--- | :--- |
| **Samba (Mac)** | `smb://${LOCAL_IP}/Shared_Media` |
| **NextExplorer** | `${PRIMARY_BASE_URL}:3000` |
| **Immich** | `${PRIMARY_BASE_URL}:2283` |
| **Authelia** | `${PRIMARY_BASE_URL}:9091` |
