# WordPress Troubleshooting Lab (Debian VM + SSH)

Goal: a healthy WordPress site you can break on purpose and fix over SSH, like the SiteHost remote test.

## Part 1: SSH setup and connecting to the VM

Why: the SiteHost test is a remote SSH session, so do all lab work from your host terminal, not the VM console.

### 1A. Install and start the SSH server (in the VM console)

1. Debian may not have `sudo` set up. If `sudo` says "command not found", switch to root with `su -` and run the commands without `sudo`.
2. Refresh package lists: `sudo apt update`
3. Install the server: `sudo apt install openssh-server`
4. Check it: `systemctl status ssh` (on Debian the service is `ssh`, not `sshd`). You want **active (running)**.
5. If it's inactive: `sudo systemctl start ssh` and `sudo systemctl enable ssh` (enable = start at boot).
6. Confirm it's listening on port 22: `sudo ss -tlnp | grep :22`

### 1B. VirtualBox network mode (power the VM off first)

Settings > Network > Adapter 1:

- **Bridged Adapter (easiest):** attach to the host network card you're actually using right now (Wi-Fi or Ethernet). The VM gets a home-network IP, so you can SSH straight to it. Make sure the selected card matches your current connection, and that Advanced > Cable Connected is ticked.
- **NAT with port forwarding (most reliable):** Advanced > Port Forwarding, add a rule: Host IP `127.0.0.1`, Host Port `2222`, Guest Port `22`, Guest IP blank. For the website, add a second rule: Host Port `8080`, Guest Port `80`.

### 1C. Find the VM's IP (Bridged only)

- In the VM: `ip a`, then look for an `inet 192.168.x.x` (or similar) line on the main interface (e.g. `enp0s3`). Ignore `lo` (127.0.0.1).
- Only an `inet6 fe80::...` line and no IPv4 `inet`? It never got a DHCP lease. See troubleshooting below.
- The IP can change after a reboot, so re-check with `ip a` if the connection stops working.

### 1D. Connect from the host (PowerShell)

- Bridged: `ssh youruser@<vm-ip>`
- NAT with port forward: `ssh -p 2222 youruser@127.0.0.1`
- First connection asks about the host fingerprint: type `yes`, then enter your VM user's password.
- Success looks like a prompt such as `youruser@hostname:~$`. Leave with `exit`.
- Website access: Bridged `http://<vm-ip>`, NAT `http://127.0.0.1:8080`.

### 1E. SSH troubleshooting

| Symptom                                  | Likely cause                                           | What to check                                                                                 |
| ---------------------------------------- | ------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| `Connection timed out`                   | Wrong IP, wrong network mode, or VM on another network | `ip a` in the VM, confirm the adapter, ping the VM IP from the host                           |
| `Connection refused`                     | SSH server not running, or wrong port                  | `systemctl status ssh`, `sudo ss -tlnp \| grep :22`, check the NAT port-forward rule          |
| `Permission denied`                      | Wrong username or password                             | Use the VM user you created, not root; check caps lock                                        |
| No IPv4 address in `ip a`                | DHCP lease failed after reboot                         | Check the VirtualBox adapter and cable setting; `nmcli device status`; `sudo dhclient enp0s3` |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | VM was rebuilt or restored, so the host key changed    | `ssh-keygen -R <vm-ip>` (or `-R "[127.0.0.1]:2222"` for NAT), then reconnect                  |
| Works, then stops after reboot           | IP changed (Bridged)                                   | Re-run `ip a`, reconnect to the new IP                                                        |

### 1F. Checklist

- [ ] `systemctl status ssh` is active (running)
- [ ] VirtualBox network mode chosen (Bridged or NAT + port forward)
- [ ] VM IP found (Bridged) or port-forward rule added (NAT)
- [ ] Can SSH from the host PowerShell
- [ ] Snapshot taken: "ssh-ready"

## Part 2: Install the stack

Packages (install with `apt`, after `apt update`):
`apache2  mariadb-server  php  libapache2-mod-php  php-mysql  php-curl  php-xml  php-mbstring  php-zip`

Checkpoints:

- [x] `systemctl status apache2` is active (running)
- [ ] `systemctl status mariadb` is active (running)
- [ ] `http://<vm-ip>` loads from the HOST browser (not just inside the VM)

## Part 3: Database for WordPress

Open the shell: `sudo mariadb`

You need 3 statements. Fill in the blanks (search the exact phrases if stuck):

```
CREATE DATABASE ______;
CREATE USER '______'@'localhost' IDENTIFIED BY '______';
GRANT ______ ON ______.* TO '______'@'localhost';
FLUSH PRIVILEGES;
```

Checkpoint: `SHOW DATABASES;` lists yours. `EXIT;` to leave.
Write down: DB name, DB user, DB password (all 3 go into wp-config.php).

## Part 4: Install WordPress files

Hints for the sequence:

1. Download latest.tar.gz from wordpress.org into /tmp (`wget`)
2. Extract it (`tar -xzf`), you get a `wordpress/` folder
3. Copy its contents into `/var/www/html/`
4. Remove Apache's default `index.html` (Apache prefers it over index.php, so WordPress won't show otherwise)
5. Give ownership to the web server user: `www-data` (`chown -R`)

Permissions rule of thumb: directories 755, files 644.

## Part 5: Connect WordPress to the DB

1. Copy `wp-config-sample.php` to `wp-config.php`
2. Edit it (`nano`): set DB_NAME, DB_USER, DB_PASSWORD, DB_HOST (localhost)
3. Visit `http://<vm-ip>` from the host and finish the web installer
4. Log in to /wp-admin to confirm it works

## Part 6: Snapshot

Take a VirtualBox snapshot named **healthy**. Restore to it before every new ticket round.

## Troubleshooting cheat sheet

Order: outside in, cheapest check first.

| Question                   | Where to look                                    |
| -------------------------- | ------------------------------------------------ |
| Does DNS resolve?          | `nslookup` / `dig`                               |
| What does the site return? | `curl -I http://<site>` (status code)            |
| Is each service up?        | `systemctl status apache2` / `mariadb`           |
| Why did it fail?           | `journalctl -u <service> -n 50`                  |
| Web server errors          | `/var/log/apache2/error.log`                     |
| WordPress/PHP errors       | Apache error log, plus WP_DEBUG in wp-config.php |
| Config syntax OK?          | `apache2ctl configtest`                          |
| Disk full?                 | `df -h`                                          |
| Memory pressure?           | `free -h`                                        |
| Permissions/ownership?     | `ls -la /var/www/html`                           |
| DB login works?            | `mariadb -u <user> -p <dbname>`                  |

Rules: back up before editing (`cp file file.bak`), change one thing at a time, re-test after each change.

## Break-and-fix scenarios (for the practice rounds)

1. Stop the database service
2. Wrong password in wp-config.php
3. Wrong DB name or host in wp-config.php
4. Wrong ownership/permissions on /var/www/html
5. Syntax error in an Apache config or wp-config.php
6. Stop Apache
7. Rename a plugin folder or break a plugin
8. Fill the disk (advanced)

For each: note the symptom, the diagnosis steps, the fix, and the customer reply.

## Customer reply template (structure only)

1. What you found (plain language, no jargon)
2. What you did to fix it
3. What the customer should check or do next
4. Short, polite sign-off
