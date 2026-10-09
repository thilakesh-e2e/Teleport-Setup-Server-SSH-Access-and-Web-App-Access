# SOP: Teleport Setup – Server, SSH Access and Web App Access

Prepared by: Thilakesh R, E2E Networks Ltd. | Type: Lab SOP | Teleport version: 18.11.3

## 1. Purpose

This SOP explains how to build a working Teleport environment from scratch on an E2E Networks server. By the end, you will be able to:

1. Create a server.
2. Install Teleport on it.
3. SSH into that same server through Teleport (browser web terminal).
4. Install a web server (nginx).
5. Open the nginx web app through Teleport.

Every step includes **what** to do, **why** we do it, and **what you should see** when it works, so a person new to Teleport can follow it without outside help.

## 2. What is Teleport (in simple words)?

Teleport is a secure access gateway. Instead of opening SSH (port 22) and many app ports to the internet, you open **one** port (443) on the Teleport proxy. Users log in to Teleport with a password plus MFA, and Teleport then gives them access to servers (SSH) and internal web apps. Every session is authenticated and can be audited.

In this lab one machine plays all roles: Auth service, Proxy service, SSH service and App service.

## 3. Scope and audience

- **Audience:** support, ops and anyone learning Teleport.
- **Scope:** a single-node **lab** setup. It uses a self-signed certificate and an insecure workaround (see Section 9), so it is **not** a production design.

## 4. Lab details (what we are using and why)

| Item | Value | Why we chose it |
| --- | --- | --- |
| Server name | Teleport-Server (`<< server name >>`) | A dedicated server keeps the lab isolated from other work. |
| OS | Ubuntu 24.04 (noble) | Teleport publishes an official apt repository for it, so install and upgrades are simple. |
| Public IP | `<< public ip >>` | Teleport's web UI must be reachable from a browser. |
| Teleport version | 18.11.3 (apt repo channel `stable/v18`) | Pinning to the v18 stable channel gives a supported release line and avoids accidental jumps to a new major version. |
| Teleport hostname | `<< public ip >>.sslip.io` | We had no domain. sslip.io is wildcard DNS, so this name resolves to our IP with no DNS setup. |
| App hostname | `nginx-app.<< public ip >>.sslip.io` | Teleport serves each app on its own subdomain. Because sslip.io resolves any `anything.<IP>.sslip.io` to the same IP, this works without extra DNS. |
| Teleport user | `<< user name >>` | Named login for the lab owner. |
| Certificate | Teleport self-signed | No domain means no public certificate. The browser shows "Not secure", which is expected. |
| Web server | nginx | Small, quick to install, and gives us a simple app to publish through Teleport. |

**How sslip.io works:** `<IP>.sslip.io` and `anything.<IP>.sslip.io` both resolve to `<IP>`. Example: `nginx-app.<< public ip >>.sslip.io` resolves to `<< public ip >>`.

## 5. Architecture

```
Browser → https://<< public ip >>.sslip.io (Teleport Proxy :443)
        → SSH service (web terminal)
        → App service → nginx (localhost:80)
```

Reading the diagram: the user only talks to port 443. The proxy forwards the request to the SSH service (for the terminal) or the App service (for nginx). nginx itself listens only on localhost, so it is never exposed directly.

## 6. Prerequisites

- Access to the E2E console to create a node.
- Root SSH access to the new server for the first install.
- A firewall or security group that allows inbound **443** (and **22** for the first login).
- A browser and an authenticator app on your phone (for MFA).
- Wherever you see `<< public ip >>` in the commands below, use your server's public IP address.

## 7. Procedure

### Step 1: Create the server

**Why:** Teleport needs a Linux host with a public IP to act as the gateway.

1. In the E2E console, create a node with a public IP (Ubuntu 24.04).
2. Allow inbound port **443** in the firewall, and port **22** for the first login. Port 443 is the only port users need once Teleport is running.
3. SSH in normally for the install:

```bash
ssh root@<< public ip >>
```

**Expected:** you get a root shell on `<< server name >>`.

### Step 2: Install Teleport

**Why:** We install from the official Teleport apt repository so the package is signed and can be upgraded with normal apt commands.

```bash
curl https://apt.releases.teleport.dev/gpg -o /usr/share/keyrings/teleport-archive-keyring.asc
source /etc/os-release
echo "deb [signed-by=/usr/share/keyrings/teleport-archive-keyring.asc] https://apt.releases.teleport.dev/${ID?} ${VERSION_CODENAME?} stable/v18" | tee /etc/apt/sources.list.d/teleport.list > /dev/null
apt-get update
apt-get install -y teleport
```

Verify:

```bash
teleport version
dpkg -l | grep -i teleport
```

**Expected:** version shows 18.11.3 and the `teleport` package is listed.

> **Common mistake:** `install-connect.sh` installs **Teleport Connect**, the desktop GUI client. It does not install the server, so `teleport` stays "command not found". Always install the apt package named `teleport`.

### Step 3: Configure Teleport

**Why:** The config file tells Teleport its cluster name, the address users will browse to, and which services to run. The public address must match the hostname in the browser, otherwise login and app launching fail.

```bash
teleport configure -o file \
  --cluster-name=<< public ip >>.sslip.io \
  --public-addr=<< public ip >>.sslip.io:443
```

By default Teleport's web UI listens on 3080. We move it to 443 so users can open the plain `https://` address.

```bash
sed -i 's/0.0.0.0:3080/0.0.0.0:443/' /etc/teleport.yaml
grep -n -E "web_listen_addr|public_addr|ssh_service|enabled" /etc/teleport.yaml
```

Confirm `/etc/teleport.yaml` contains:

- `web_listen_addr: 0.0.0.0:443`
- `public_addr: << public ip >>.sslip.io:443`
- `ssh_service` with `enabled: "yes"`

### Step 4: Start Teleport

**Why:** Enabling the service makes Teleport start automatically after a reboot.

```bash
systemctl enable --now teleport
systemctl status teleport --no-pager
journalctl -u teleport -n 30 --no-pager
```

**Expected:** `active (running)` and no errors in the log.

### Step 5: Create the Teleport user

**Why:** Teleport has its own users. This user logs in with a password and MFA, and the `--logins=root` option decides which Linux accounts the user may SSH in as.

```bash
tctl users add << user name >> --roles=editor,access --logins=root
```

1. Open the invite link it prints. It is valid for **1 hour**.
2. Click **Advanced → Proceed** on the certificate warning (expected, because of the self-signed certificate).
3. Set a password and register an authenticator app (MFA).

If the link expires, recreate the user:

```bash
tctl users rm << user name >>
tctl users add << user name >> --roles=editor,access --logins=root
```

### Step 6: SSH into the server via Teleport

**Why:** This proves the SSH service works and that you can reach the server through the proxy instead of port 22.

1. Open `https://<< public ip >>.sslip.io` and log in.
2. Go to **Resources → Servers**. `<< server name >>` should be listed.
3. Click **Connect → root**. A web terminal opens.

**Expected:** a root shell in your browser. From this step onward, run commands in this Teleport terminal.

### Step 7: Install nginx (inside the Teleport terminal)

**Why:** nginx is the sample web app we will publish through Teleport.

```bash
apt-get update && apt-get install -y nginx
echo "Hello from Teleport" > /var/www/html/index.html
systemctl status nginx --no-pager
curl -I http://localhost
```

**Expected:** `active (running)` and `HTTP/1.1 200 OK`.

### Step 8: Register nginx as a Teleport app

**Why:** Teleport only publishes apps it has been told about. The `app_service` block names the app, says where it runs locally, and sets the public hostname users will open.

Add this block to the **bottom** of `/etc/teleport.yaml`. `app_service:` must start at the left margin, and indentation must use **spaces** (not tabs):

```yaml
app_service:
  enabled: true
  apps:
  - name: "nginx-app"
    uri: "http://localhost:80"
    public_addr: "nginx-app.<< public ip >>.sslip.io"
```

Validate, restart and check:

```bash
teleport configure --test /etc/teleport.yaml
systemctl restart teleport
tctl apps ls
```

**Expected:** `tctl apps ls` lists `nginx-app`.

> The UI flow **Enroll New Resource → Web Application** is meant for adding an app on a **different** machine. It was not used here because nginx runs on the same server as Teleport.

### Step 9: Fix the reverse tunnel certificate error

**Symptom:** launching the app shows **"Access Denied: Unable to serve application requests"**.

**Cause:** `journalctl -u teleport` shows:

```
Failed to establish reverse tunnel ... x509: certificate signed by unknown authority
```

The app service connects back to the proxy over a reverse tunnel. It rejected the proxy's self-signed certificate, so the app could not register.

**Fix (lab only):** run Teleport with the `--insecure` flag.
## Fix: Teleport app shows "Access Denied" (reverse tunnel x509 error)

### Problem
Opening `nginx-app` from Teleport shows:

> Access Denied: Unable to serve application requests. Verify if the Application Services are connected to Teleport.

**Cause:** The proxy uses a self-signed certificate. The app service cannot verify it when opening the reverse tunnel, so it fails with an x509 error and never registers with the cluster.

### Steps

**1. Check how the service starts**
```bash
systemctl cat teleport | grep ExecStart
```
Note the exact binary path and flags, for example:
```
ExecStart=/usr/local/bin/teleport start --config /etc/teleport.yaml --pid-file=/run/teleport.pid
```

**2. Add `--insecure` using a systemd override**
```bash
systemctl edit teleport
```
Paste the following, using the same path and flags as above plus `--insecure`:
```ini
[Service]
ExecStart=
ExecStart=/usr/local/bin/teleport start --config /etc/teleport.yaml --pid-file=/run/teleport.pid --insecure
```
Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

**3. Reload, restart and verify**
```bash
systemctl daemon-reload
systemctl restart teleport
sleep 30
journalctl -u teleport --since "1 min ago" --no-pager | grep -i -E "reverse tunnel|certificate|error"
```
The certificate errors should no longer appear.

**4. Confirm the app is registered**
```bash
tctl apps ls
```
`nginx-app` should be listed.

**5. Launch the app**
Log in at `https://<server-ip>.sslip.io`, go to **Resources → Applications**, and click **Launch** on `nginx-app`. Accept the browser warning via **Advanced → Proceed**.

### Notes
- `--insecure` skips TLS certificate verification between Teleport components. It is acceptable for a lab or learning setup only.
- In production, use a trusted certificate (for example Let's Encrypt via ACME with a real domain).
- If the error persists, check the output of `systemctl cat teleport | grep ExecStart` and the latest `journalctl -u teleport` lines.

**Why it is acceptable here:** the lab uses a self-signed certificate and everything runs on one host. `--insecure` tells Teleport to skip certificate verification, which removes the error. It must **not** be used in production.

### Step 10: Open the web app through Teleport (final check)

1. Log in at `https://<< public ip >>.sslip.io`.
2. Go to **Resources → Applications** and click **Launch** on `nginx-app`.
3. The app opens at `nginx-app.<< public ip >>.sslip.io` (accept the browser warning again, because the certificate is self-signed).

**Expected:** the page shows "Hello from Teleport". The lab goal is complete.

## 8. Troubleshooting quick table

| Symptom | Likely cause | What to do |
| --- | --- | --- |
| `teleport: command not found` | Teleport Connect (GUI) was installed instead of the server package | Install the apt package `teleport` (Step 2). |
| Invite link not working | Link older than 1 hour | Remove and re-add the user (Step 5). |
| Browser says "Not secure" | Self-signed certificate | Expected in this lab. Click Advanced → Proceed. |
| Teleport page does not load | Port 443 blocked, or `web_listen_addr` still on 3080 | Open 443 in the firewall and recheck Step 3. |
| Config test fails | YAML indentation or tabs | Make sure `app_service:` is at the left margin and uses spaces (Step 8). |
| `nginx-app` missing in `tctl apps ls` | `app_service` block missing or Teleport not restarted | Recheck the block and restart Teleport. |
| "Access Denied: Unable to serve application requests" | Reverse tunnel rejects the self-signed certificate | Apply Step 9. |

First place to look for any problem:

```bash
journalctl -u teleport -n 30 --no-pager
```

## 9. Security notes and limits of this lab

- The self-signed certificate and `--insecure` flag are lab shortcuts only.
- The user has `--logins=root`, which allows root SSH through Teleport. This is acceptable for a lab, but production should use least-privilege logins.
- The invite link is a one-time secret. Do not share it.
- Port 22 was only needed for the first login. After Step 6 works, you can consider closing it.

## 10. Moving to production (recommended changes)

- Use a real domain and a trusted certificate (for example, Let's Encrypt through Teleport's built-in ACME support) so the browser shows no warning and `--insecure` is no longer needed.
- Use a wildcard DNS record for app subdomains.
- Use least-privilege roles and logins instead of `root`.
- Run the Auth and Proxy services separately from application hosts, and add more nodes by joining them to the cluster.

## 11. Quick reference

| Task | Command |
| --- | --- |
| Check version | `teleport version` |
| Service status | `systemctl status teleport --no-pager` |
| Recent logs | `journalctl -u teleport -n 30 --no-pager` |
| Validate config | `teleport configure --test /etc/teleport.yaml` |
| List apps | `tctl apps ls` |
| Add user | `tctl users add <name> --roles=editor,access --logins=root` |
| Remove user | `tctl users rm <name>` |
| Config file | `/etc/teleport.yaml` |
