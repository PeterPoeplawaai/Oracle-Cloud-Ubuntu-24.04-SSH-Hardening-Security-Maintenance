<p align="left">
  <img src="banner_ubuntu_hardening.png" alt="Banner" width="1000" />
</p>

# Oracle Cloud Ubuntu 24.04 — SSH Hardening & Security Maintenance

A step-by-step reference guide for resolving common SSH and system security audit findings on an **Oracle Cloud Infrastructure (OCI) Free Tier Ubuntu 24.04** instance without locking yourself out.

---

## Table of Contents

1. [Overview of Resolved Issues](#1-overview-of-resolved-issues)
2. [Pre-Flight Safety & Lockout Prevention](#2-pre-flight-safety--lockout-prevention)
3. [Verify User & Sudo Privileges](#3-verify-user--sudo-privileges)
4. [Apply Pending System & Security Updates](#4-apply-pending-system--security-updates)
5. [Harden SSH Configuration & File Permissions](#5-harden-ssh-configuration--file-permissions)
6. [Validate & Restart the SSH Service](#6-validate--restart-the-ssh-service)
7. [Optional: Change the Default SSH Port](#7-optional-change-the-default-ssh-port)
8. [Emergency Recovery (If Locked Out)](#8-emergency-recovery-if-locked-out)

---

## 1. Overview of Resolved Issues

| Severity | Finding | Remediation |
| :--- | :--- | :--- |
| **CRITICAL** | SSH root login enabled | Set `PermitRootLogin no` |
| **HIGH** | SSH password authentication enabled | Set `PasswordAuthentication no` & `KbdInteractiveAuthentication no` |
| **HIGH** | Loose `sshd_config` permissions | Restrict `/etc/ssh/sshd_config` to `0600` |
| **HIGH** | Pending security updates | Run `apt update && apt upgrade -y` |
| **MEDIUM** | High SSH max auth tries | Set `MaxAuthTries 3` |
| **LOW** | SSH X11 forwarding enabled | Set `X11Forwarding no` |
| **LOW** | SSH listening on default port `22` | *(Optional)* Move SSH to a custom port after updating OCI & host firewalls |

---

## 2. Pre-Flight Safety & Lockout Prevention

Before modifying SSH settings on a remote cloud VM:

1. **Confirm SSH key login works:** Ensure you can log in as `ubuntu` using your SSH private key (via PuTTY `.ppk` or OpenSSH key).
2. **Verify `~/.ssh/authorized_keys` exists and is readable:**
   ```bash
   ls -la ~/.ssh/authorized_keys
   ```
3. **Keep an active session open:** Never close your current SSH/PuTTY window until you have opened a **second** terminal session and verified a fresh login succeeds after restarting SSH.

---

## 3. Verify User & Sudo Privileges

Confirm your active user, group memberships, and `sudo` permissions:

```bash
whoami && id
sudo -l
```

### Expected Output
- **User:** `ubuntu` (member of `sudo`)
- **Sudo rule:** `(ALL) NOPASSWD: ALL` (confirms passwordless `sudo` is active)

You can double-check non-interactive `sudo` access with:
```bash
sudo -n true && echo "Passwordless sudo OK"
```

---

## 4. Apply Pending System & Security Updates

Update the local package index and install all available security and package updates:

```bash
sudo apt update && sudo apt upgrade -y
```

### Verification
- Check if any updates remain:
  ```bash
  apt list --upgradable
  ```
- Check if a system reboot is required (e.g., after a kernel update):
  ```bash
  [ -f /var/run/reboot-required ] && echo "Reboot required" || echo "No reboot required"
  ```

---

## 5. Harden SSH Configuration & File Permissions

### 5.1 Restrict `sshd_config` Permissions
Ensure only `root` can read or modify the main SSH daemon configuration file:

```bash
sudo chmod 0600 /etc/ssh/sshd_config
```

**Verify:**
```bash
ls -l /etc/ssh/sshd_config
# Expected: -rw------- 1 root root ... /etc/ssh/sshd_config
```

### 5.2 Add a Hardening Drop-In Configuration
Ubuntu 24.04 loads drop-in files from `/etc/ssh/sshd_config.d/*.conf`. Create `/etc/ssh/sshd_config.d/99-hardening.conf`:

```bash
sudo nano /etc/ssh/sshd_config.d/99-hardening.conf
```

Paste underneath lines and save:
```bash
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
MaxAuthTries 3
X11Forwarding no
```

> **Note:** Check `/etc/ssh/sshd_config` and earlier files in `/etc/ssh/sshd_config.d/` (such as `50-cloud-init.conf`) to make sure no conflicting `PasswordAuthentication yes` or `PermitRootLogin yes` directive overrides your settings.

---

## 6. Validate & Restart the SSH Service

1. **Test the SSH configuration syntax first:**
   ```bash
   sudo sshd -t
   ```
   *(If this outputs any error, fix it before restarting the service.)*

2. **Verify the effective runtime configuration:**
   ```bash
   sudo sshd -T | grep -E "permitrootlogin|passwordauthentication|kbdinteractiveauthentication|maxauthtries|x11forwarding"
   ```
   Expected values:
   ```text
   permitrootlogin no
   passwordauthentication no
   kbdinteractiveauthentication no
   maxauthtries 3
   x11forwarding no
   ```

3. **Restart the SSH daemon:**
   ```bash
   sudo systemctl restart ssh
   ```
   *(On some images, the unit name is `sshd`: `sudo systemctl restart sshd`.)*

4. **Test in a new terminal:** Open a **new** PuTTY / SSH session while keeping your original session open. Confirm you can connect with your private key and run `sudo -n true`.

---

## 7. Optional: Change the Default SSH Port

Changing port `22` is optional when key-based authentication is enforced, as it primarily reduces automated scanner noise. If you choose to change it (e.g., to port `2525`):

1. **Allow the new port in OCI Cloud Networking first:**
   - In the **Oracle Cloud Console**, go to **Networking → Virtual Cloud Networks → Your VCN → Subnet → Security Lists** (or your instance's **Network Security Group**).
   - Add a stateful **Ingress Rule** for `TCP` destination port `2525` from your IP (or `0.0.0.0/0`).
2. **Allow the new port in your host firewall:**
   ```bash
   sudo iptables -I INPUT 1 -p tcp -m state --state NEW --dport 2525 -j ACCEPT
   sudo netfilter-persistent save
   ```
   > [!NOTE]
   > The firewall rule use `-I INPUT`, which inserts the rule at the top of the chain, so it always ends up above the REJECT rule of the Oracle Ubuntu image. Check the result at any time with: `sudo iptables -L INPUT -v -n --line-numbers`

3. **Update `/etc/ssh/sshd_config.d/99-hardening.conf`** by adding:
   ```text
   Port 2525
   ```
4. **Validate and restart SSH:**
   ```bash
   sudo sshd -t && sudo systemctl restart ssh
   ```
5. **Switch the Active Listener to Port 2525:**
   ```bash
   sudo systemctl disable --now ssh.socket
   sudo systemctl daemon-reload
   sudo systemctl enable ssh.service
   sudo systemctl restart ssh.service
   ```
6. **Confirm port 2525 is now listening and port 22 is released:**
   ```bash
   sudo ss -tlnp | grep -E ':(2525|22)\b'
   ```
   > **Note:** You see 0.0.0.0:2525 and [::]:2525 in LISTEN state for sshd, with no LISTEN lines on port 22.
7. **Test connecting on port `2525`** in a new PuTTY session before removing port `22` from your firewall rules.
8. If everything works, you can delete the firewall rule for port `22` safely:
   ```bash
   sudo iptables -D INPUT -p tcp -m state --state NEW --dport 22 -j ACCEPT
   sudo netfilter-persistent save
   ```
9. **Delete the old port `22` in OCI Cloud Networking:**
   - In the **Oracle Cloud Console**, go to **Networking → Virtual Cloud Networks → Your VCN → Subnet → Security Lists** (or your instance's **Network Security Group**).
   - Delete the stateful **Ingress Rule** for `TCP` destination port `22` from your IP (or `0.0.0.0/0`).
  
> [!TIP]
> Install [Defensia](https://defensia.cloud/) to see your real-time security hardening score. After following these steps your score should look like:
<p align="left">
  <img src="defensia_result.png" alt="Banner" width="1000" />
</p>


---

## 8. Emergency Recovery (If Locked Out)

If you ever lose SSH access to your Oracle Cloud instance:

1. Open the **Oracle Cloud Console** and navigate to **Compute → Instances → Your Instance**.
2. Scroll down to **Resources → Console connection**.
3. Click **Launch Cloud Shell connection** (or attach a local serial console SSH key).
4. Once connected to the serial console, press **Enter** to get a login prompt (or reboot the instance and edit the GRUB boot entry with `init=/bin/bash` if you need root recovery) to revert `/etc/ssh/sshd_config.d/99-hardening.conf` and restart `ssh`.
