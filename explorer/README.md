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

### Step 3: Create the `.env` File
Create a `.env` file in the same directory as your `docker-compose.yml` and populate it with your secure credentials:

```ini
TZ=Asia/Kolkata
PUID=1000
PGID=1000
SERVER_IP_OR_DOMAIN=192.168.1.XX

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

## 👥 User Management Examples

### 1. Creating a Samba User
`sudo smbpasswd -a <username>`

### 2. Creating an Authelia User (SSO)
First, generate a secure Argon2 hash for your password by running:
`docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'YourNewPassword'`

Then, edit `/docker/authelia/config/users.yml` and paste the output:
```yaml
users:
  gaurav:
    displayname: "Gaurav"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..." # Paste your generated hash here
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

## 🔗 How to Connect via Samba
**On Mac:** `Cmd + K` -> `smb://<SERVER_IP>/Shared_Media`  
**On Windows:** Map Network Drive -> `\\<SERVER_IP>\Shared_Media`

---

## 🛠 Web Access Links
- **NextExplorer:** `http://<SERVER_IP>:3000`
- **Immich:** `http://<SERVER_IP>:2283`
- **Tdarr:** `http://<SERVER_IP>:8265`
- **Authelia:** `http://<SERVER_IP>:9091`
