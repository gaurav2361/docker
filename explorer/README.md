# Serve: Enterprise Marketing Storage & Identity Stack

**Serve** is a professional-grade, self-hosted ecosystem designed for high-performance marketing agencies (20+ users). It unifies enterprise-level storage (Samba) with modern web identity (Google OAuth, SSO, and LLDAP).

---

## 🏗️ Core Architecture: How It Works

This stack is built on a "Source of Truth" model. Every asset enters the system through high-speed channels and is then automatically processed and presented to your team and clients.

### The Asset Lifecycle
1.  **Ingest (Samba):** A Senior Editor drops a 100GB 4K video directly onto the server via a native network drive.
2.  **Optimize (Tdarr):** Tdarr detects the new file and automatically converts it to H.265 (HEVC), reducing the file size by 60% without losing marketing quality.
3.  **Index (Immich):** The optimized video is instantly visible in the Immich gallery for the Social Media manager to review on their phone.
4.  **Distribute (NextExplorer):** The manager creates a secure "Guest Link" in NextExplorer and sends it to the client for final approval.
5.  **Identity (LLDAP/Authelia):** Throughout this entire process, everyone uses the same single set of credentials or their Google account.

---

## 👥 Team Structure & Permissions Matrix

With 20+ users, managing individual permissions is impossible. We use **LDAP Groups** to automate access.

| Role | Example User | Samba (Direct Edit) | NextExplorer (Sharing) | Immich (Gallery) |
| :--- | :--- | :--- | :--- | :--- |
| **Admin** | `gaurav` | Full R/W Access | Manage All Shares | Full Admin |
| **Editor** | `rahul_edit` | High-Speed R/W | Create Project Shares | View & Browse |
| **Marketing** | `priya_social` | Read-Only | Use Existing Shares | Mobile Backup/View |
| **Client** | `client_xyz` | No Access | Secure Link Only | No Access |

### Day in the Life: Adding a New User
When a new designer joins the team:
1.  Log into **LLDAP** (`http://<IP>:17170`).
2.  Create the user and add them to the `editors` group.
3.  **Result:** They can instantly mount the Samba drive on their Mac AND log into the web gallery with the same password.

---

## 🌐 Modern Authentication (Google & Email)

### 1. Sign-in with Google (OAuth)
Enable your team to skip password management by using their company Google accounts.
-   **Why:** Increases security (2FA) and simplifies onboarding for 20+ people.
-   **Setup:** 
    1.  Create a project at [Google Cloud Console](https://console.cloud.google.com/).
    2.  Create **OAuth 2.0 Credentials** (Web Application).
    3.  Set Authorized Redirect URI to: `https://auth.yourdomain.com/api/oidc/authorization/google` (if using a domain) or `http://<IP>:9091/api/oidc/authorization/google`.
    4.  Update `GOOGLE_CLIENT_ID` and `SECRET` in `.env`.

### 2. SMTP Email Notifications
The "Heartbeat" of your identity system. This allows the server to communicate with your team.
-   **Password Resets:** If an editor forgets their password, they can reset it via email without bothering the admin.
-   **2FA Verification:** Send one-time codes to emails for extra security when logging in from new devices.
-   **Welcome Invites:** Automatically notify new team members when their accounts are ready.
-   **Setup:** Use your company Gmail/Outlook SMTP settings in the `.env` file. (For Gmail, you must use an "App Password").

---

## 🚀 Step-by-Step Implementation

### Step 1: Host Preparation
```bash
# 1. Create the dual-root structure
sudo mkdir -p /docker/{lldap/data,authelia/config,nextexplorer/{config,cache},tdarr/{server,configs},immich/{upload,postgres}}
sudo mkdir -p /data

# 2. Lock down permissions for 20+ users
sudo chown -R gaurav:gaurav /docker /data
sudo find /data -type d -exec chmod 775 {} \; # Group (Team) can write
sudo find /data -type f -exec chmod 664 {} \; # Group (Team) can read
```

### Step 2: Firewall (UFW) Configuration
Marketing files are heavy; don't let a closed port slow down your editors.
```bash
sudo ufw allow 445/tcp    # Samba (Editing)
sudo ufw allow 17170/tcp  # LLDAP (Identity Admin)
sudo ufw allow 3000/tcp   # NextExplorer (Sharing)
sudo ufw allow 9091/tcp   # Authelia (Login)
sudo ufw allow 2283/tcp   # Immich (Gallery)
sudo ufw reload
```

### Step 3: Unified Samba (The "Magic" Config)
Instead of local users, Samba now asks LLDAP "Who is this?". Add this to `/etc/samba/smb.conf`:
```ini
[global]
   passdb backend = ldapsam:ldap://localhost:3890
   ldap suffix = dc=marketing,dc=com
   ldap admin dn = cn=admin,dc=marketing,dc=com

[Marketing_Assets]
   path = /data
   valid users = @editors, @admins
   force group = gaurav
   create mask = 0775
```

---

## ⚙️ Hardware Acceleration (Mac M4 & Server)

### Server Optimization
The `docker-compose.yml` is pre-configured with `/dev/dri` passthrough. If your Ubuntu server has an Intel/AMD GPU, Tdarr will use it to transcode marketing videos 10x faster than the CPU.

### Mac M4 Node (The "Nitro" Boost)
If you have a massive batch of 4K RAW footage, fire up the Tdarr Node on your Mac M4:
```json
{
  "nodeName": "Lead-Editor-M4",
  "serverIP": "<SERVER_IP>",
  "pathTranslators": [
    { "server": "/data", "node": "/Volumes/Marketing_Assets" }
  ]
}
```

---

## 🛠️ Maintenance & Troubleshooting

-   **Check Identity Logs:** `docker logs lldap`
-   **Check SSO Logs:** `docker logs authelia`
-   **Restart All:** `docker compose restart`
-   **Backups:** To backup the entire company setup, simply zip the `/docker` folder. Your `/data` assets should be backed up separately (RAID or Cloud).

---

## 🔗 Connection Links Summary

| Access Method | URL / Path |
| :--- | :--- |
| **Internal High-Speed Drive** | `smb://${LOCAL_IP}/Marketing_Assets` |
| **Team Management (Admin)** | `http://${LOCAL_IP}:17170` |
| **Asset Distribution (Web)** | `${PRIMARY_BASE_URL}:3000` |
| **Client Gallery** | `${PRIMARY_BASE_URL}:2283` |
| **Identity Portal** | `${PRIMARY_BASE_URL}:9091` |
