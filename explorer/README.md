# Serve: The Ultimate Media Stack Guide

This repository contains the configuration for **Serve**, a high-performance, self-hosted media ecosystem. It is designed to run entirely on a central Ubuntu server, using a clean directory split between `/docker` (configurations) and `/data` (media storage).

---

## 🏗 Why These Tools? (The Architecture)

1.  **Samba (The Heavy Lifter):** Native file-sharing for local networks. It allows your server to appear as a native drive on your Mac/Windows machines.
2.  **NextExplorer (The Web Bridge):** A secure web interface for guest access and external sharing.
3.  **Tdarr (The Optimizer):** A background robot that automatically shrinks videos into H.265 (HEVC), saving massive amounts of storage space.
4.  **Immich (The Gallery Experience):** The ultimate Google Photos alternative with facial recognition and mobile backups.
5.  **Authelia (The Security Gatekeeper):** A Single Sign-On (SSO) provider that puts a secure "front door" on all your apps.
6.  **Docker Compose (The Orchestrator):** Isolates every app into its own container for stability and easy backups.

---

## 📁 Directory Structure

-   **/docker**: Stores all persistent application data, configuration files, and databases.
-   **/data**: Stores all actual media files (photos, videos) and the master shared folder.

---

## 🔐 The Permissions Deep Dive (PUID/PGID)

To avoid "Access Denied" errors, every app is forced to act as the same user (`UID 1000`).
- **The Solution:** We use `PUID=1000` and `PGID=1000` in Docker.
- **Samba:** We use "force user" masks to ensure any file you drop from your Mac matches the server owner instantly.

---

## 🚀 Step-by-Step Implementation

### Step 1: Host Preparation

```bash
# 1. Create root directories
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
Edit `/etc/samba/smb.conf`:

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

### Step 3: Local IP vs. Domain Connection

You can access your server via a **Local IP** (at home) or a **Domain** (anywhere).

#### Scenario A: Local-Only (At Home)
- Update `.env`: `SERVER_IP_OR_DOMAIN=192.168.1.XX`
- Access via: `http://192.168.1.XX:3000`

#### Scenario B: Domain Access (External)
- Update `.env`: `SERVER_IP_OR_DOMAIN=media.yourdomain.com`
- **Reverse Proxy:** You will need a Reverse Proxy (like Nginx Proxy Manager or Cloudflare Tunnels) to map your domain to the server's local ports.
- **SSL:** Your links will change from `http://` to `https://`. Ensure your `PUBLIC_URL` in `docker-compose.yml` matches your domain precisely.

### Step 4: Deploy the Stack

```bash
docker compose up -d
```

---

## 👥 User Management Examples

### 1. Creating a Samba User
```bash
sudo smbpasswd -a <username>
```

### 2. Creating an Authelia User (SSO)
Edit `/docker/authelia/config/users.yml`:
```yaml
users:
  gaurav:
    displayname: "Gaurav"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..." # Generate via 'authelia hash-password'
    email: gaurav@domain.com
    groups: [admins]
```

---

## 🔗 How to Connect via Samba

### Using the Server IP (Recommended)
Always use the static IP for the fastest local file transfers.

**On Mac:** `Cmd + K` -> `smb://192.168.1.XX/Shared_Media`
**On Windows:** Map Drive -> `\\192.168.1.XX\Shared_Media`

---

## 🛠 Web Access Links (Local vs Domain)

| Service | Local Access (Port) | Domain Access (Proxy) |
| :--- | :--- | :--- |
| **NextExplorer** | `http://<IP>:3000` | `https://explorer.yourdomain.com` |
| **Immich** | `http://<IP>:2283` | `https://photos.yourdomain.com` |
| **Tdarr** | `http://<IP>:8265` | `https://tdarr.yourdomain.com` |
| **Authelia** | `http://<IP>:9091` | `https://auth.yourdomain.com` |

> **Critical Note:** If you use a domain, ensure your Authelia `OIDC_ISSUER` URL in the Compose file matches your public authentication URL, or the login will fail.
