# Serve: The Marketing Agency Master Identity & Storage Guide

This manual defines the technical architecture of **Serve**, an enterprise-grade ecosystem tailored for 20+ users. It unifies high-performance storage (Samba) with a modern identity fabric (**LLDAP** + **Authelia OIDC**).

---

## 🏗️ Technical Architecture: The "Identity Fabric"

In this stack, **LLDAP** acts as the "Source of Truth" for all user data, while **Authelia** acts as the "Gatekeeper" that translates that data into modern web tokens (OIDC) for apps like NextExplorer and Immich.

### The Authentication Handshake
1.  **User Login:** A team member clicks "Sign-in with SSO" on NextExplorer.
2.  **OIDC Redirect:** NextExplorer redirects the user to **Authelia**.
3.  **LDAP Verification:** Authelia pings **LLDAP** via the internal `media_net` to verify the username/password.
4.  **Token Issuance:** Once verified, Authelia generates an OIDC Token containing the user's name, email, and **LDAP Groups**.
5.  **Access Granted:** NextExplorer receives the token, sees the user belongs to the `editors` group, and grants them full project access.

---

## 👥 Advanced User & Group Management

For a 20+ user marketing agency, you should never manage permissions by "user." Always manage by **Group**.

| LDAP Group | App Role | Samba Permissions | Marketing Workflow |
| :--- | :--- | :--- | :--- |
| `admins` | Superuser | Full Control (`/data`) | Server & User management |
| `editors` | Power User | R/W Access (`/data`) | Direct 4K Video/RAW editing |
| `marketing` | Web User | Read-Only | Distributing assets via NextExplorer |
| `clients` | Guest | No Samba Access | Secure web links only |

---

## 🛡️ Deep Dive: OIDC & LDAP Configuration

### 1. Connecting Authelia to LLDAP (The Backend)
Authelia must be configured to talk to the LLDAP container. In your `authelia/config/configuration.yml`, use these precise settings:

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
    user: 'uid=admin,ou=people,dc=marketing,dc=com'
    password: 'YOUR_LDAP_ADMIN_PASSWORD'
    attributes:
      username: 'uid'
      display_name: 'displayName'
      mail: 'mail'
      group_name: 'cn'
```

### 2. NextExplorer OIDC Integration
NextExplorer is pre-configured in the `docker-compose.yml` to request the `groups` scope. This is vital for **Admin Elevation**.

- **Admin Logic:** If a user in LLDAP belongs to the `admins` group, and `OIDC_ADMIN_GROUPS=admins` is set in the environment, NextExplorer will automatically promote that user to an Admin in the web UI.

---

## 🌐 Modern Auth: Google OAuth & SMTP

### Sign-in with Google
This allows your team to use their `company.com` Google accounts. 
- **Requirement:** Your server MUST be accessible via a domain with HTTPS for Google to allow the redirect.
- **Workflow:** When a user logs in via Google, Authelia verifies the email and maps it back to their LLDAP profile if the emails match.

### SMTP (The Agency Heartbeat)
- **Password Resets:** Essential for 20+ users. Without SMTP, the Admin has to manually reset every forgotten password.
- **2FA:** Enables Time-based One-Time Passwords (TOTP) or email-based 2FA for sensitive marketing data.

---

## 🚀 Step-by-Step Server Setup

### Step 1: Prepare the Identity Brain (LLDAP)
1.  Run `docker compose up -d lldap`.
2.  Access `http://<IP>:17170` (Admin / your password).
3.  **Crucial:** Create your groups (`admins`, `editors`, `marketing`) **before** adding users.

### Step 2: Unified Samba Setup
To allow 20+ users to mount the server as a drive without manual Linux account creation:
1.  Install LDAP tools: `sudo apt install libnss-ldap libpam-ldap ldap-utils -y`.
2.  Update `/etc/samba/smb.conf` to use the `ldapsam` backend pointing to `localhost:3890`.

### Step 3: Immich SSO Activation
Immich requires a manual "First-Time" link in its GUI:
1.  **Administration -> Settings -> OAuth**.
2.  **Issuer URL:** `https://auth.marketingcompany.com` (or your local IP).
3.  **Scope:** `openid email profile groups`.
4.  **Auto Register:** ON (Enables instant onboarding for the whole team).

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
- **Identity Backup:** The file `/docker/lldap/data/users.db` is the most important file on your server. Back it up daily.
- **Health Check:** If OIDC fails, check the Authelia logs: `docker logs authelia`.
- **SSO Refresh:** If you add a user to a new group in LLDAP, they must log out and back into the web apps to refresh their OIDC "claims."
