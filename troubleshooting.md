# Linux + WordPress Troubleshooting Guide

How to retrain with this: read the **Symptom** and **Likely cause**, then COVER the Check and Fix sections and try to write the commands from memory (or from a quick search). Then uncover and compare. Repeat until you can do it without peeking.

---

## 0. The universal routine

1. Reproduce the symptom (what does the customer see? `curl -I http://site` shows the status code).
2. Read the error message: which layer is it pointing at?
3. Check the cheapest thing first (is the service running?).
4. Read the logs for the real reason.
5. Fix ONE thing, back up before editing (`cp file file.bak`).
6. Retest. Restart the service if you changed its config.
7. Write the customer reply.

Layer chain (follow the request): browser > DNS > Apache > PHP > MariaDB

- Page won't load at all / timeout: DNS, network, Apache down
- WordPress-styled error message: Apache + PHP are fine, look further down
- 403 / 500 / blank page: Apache or PHP error log
- "Error establishing a database connection": MariaDB or credentials

---

## 1. Command cheat sheet

### Services

- `systemctl status <service>`: running / failed / stopped
- `sudo systemctl start|stop|restart <service>`
- `sudo systemctl enable <service>`: start at boot
- Names: Debian `apache2`, `mariadb` (or `mysql`). Red Hat-style: `httpd`, `mariadb`/`mysqld`.
- Find services: `systemctl list-units --type=service | grep -Ei 'maria|mysql|apache|httpd|nginx|php'`

### Logs

- `sudo journalctl -u <service> -n 30 --no-pager`: recent log for a service
- `sudo tail -n 30 /var/log/apache2/error.log`: Apache/PHP errors
- `sudo tail -f /var/log/apache2/error.log`: watch live (Ctrl+C to quit), reload the page and watch
- Errors vs notes: `[Note]` = normal, `[Warning]`/`[ERROR]` = look here

### Files and permissions

- `ls -l <dir>`: contents; `ls -ld <dir>`: the directory itself
- Number math: r=4, w=2, x=1; digits are owner, group, others
- `rwxr-xr-x` = 755, `rw-r--r--` = 644, `rw-r-----` = 640, `rw-------` = 600
- Directories NEED x (to enter). Regular files usually don't.
- `sudo chown -R user:group path`: change owner (recursive)
- `sudo find /var/www/html -type d -exec chmod 755 {} +`: all directories
- `sudo find /var/www/html -type f -exec chmod 644 {} +`: all regular files
- `-type` takes ONE letter: `d` = directory, `f` = file

### System checks

- `df -h`: disk space; `df -i`: inodes
- `free -h`: memory
- `sudo ss -tlnp`: listening ports (80/443 web, 3306 database)
- `cat /etc/os-release`: which distro
- `ps aux | grep apache2`: which user Apache runs as (www-data)

### Network / DNS

- `curl -I http://site`: status code and headers
- `nslookup domain` or `dig domain`: what IP does it resolve to?
- `ip a`: this machine's IPs
- Private IPs (10.x, 172.16-31.x, 192.168.x) are not reachable from the internet

### Database

- `sudo mariadb`: open shell (statements end with `;`)
- `SHOW DATABASES;`
- `SELECT user, host FROM mysql.user;`
- `SHOW GRANTS FOR 'user'@'localhost';`
- Test credentials: `mariadb -u <user> -p <dbname>`
- `sudo grep DB_ /var/www/html/wp-config.php`: what WordPress is using

---

## 2. First 2 minutes on any server (discovery)

1. `cat /etc/os-release`
2. `systemctl list-units --type=service | grep -Ei 'maria|mysql|apache|httpd|nginx|php'`
3. `sudo ss -tlnp`
4. `ls -la /var/www/html` and `sudo grep DB_ /var/www/html/wp-config.php`
5. `df -h` and `free -h`

If a command or service name isn't found, search it (open book!).

---

## 3. Scenarios

### Scenario 1: "Error establishing a database connection" (service stopped)

- Likely cause: MariaDB isn't running.
- Check: `systemctl status mariadb`; `sudo journalctl -u mariadb -n 30 --no-pager`
- Read: clean "Shutdown complete / Deactivated successfully" = stopped on purpose. `[ERROR]`, `killed`, `out of memory` = it crashed, find out why before restarting.
- Fix: `sudo systemctl start mariadb`, retest.
- Tell customer: the database service had stopped, you restarted it, site is back; mention the cause if known.

### Scenario 2: Same error, but the service is running (wrong credentials)

- Likely cause: wrong DB user/password/name in `wp-config.php`.
- Check: `sudo grep DB_ /var/www/html/wp-config.php` then test them: `mariadb -u <user> -p <dbname>`. "Access denied" = credentials or host wrong.
- Also check: `SELECT user, host FROM mysql.user;` (user@host must match, e.g. `ops`@`localhost`) and `SHOW GRANTS FOR 'ops'@'localhost';`
- Fix: either correct `wp-config.php` (back it up first), or reset the DB password with `ALTER USER 'ops'@'localhost' IDENTIFIED BY 'newpass'; FLUSH PRIVILEGES;` and update `wp-config.php` to match.
- Tell customer: the site's database login didn't match; corrected it.

### Scenario 3: Site shows "403 Forbidden"

- Likely cause: Apache can't read the files/directories (wrong ownership or modes).
- Check: `ls -ld /var/www/html` and `ls -l /var/www/html`; `sudo tail -n 20 /var/log/apache2/error.log` (look for "Permission denied" or "search permissions are missing").
- Fix:
  1. `sudo chown -R www-data:www-data /var/www/html`
  2. Directories: `sudo find /var/www/html -type d -exec chmod 755 {} +`
  3. Files: `sudo find /var/www/html -type f -exec chmod 644 {} +`
  4. Tighten the secret file: `sudo chmod 640 /var/www/html/wp-config.php` (600 also works when www-data owns it). It holds the DB password, so others get no access.
- Verify: `ls -l`, reload the site, check the Apache log.
- Least privilege: Apache runs as www-data, so give it ownership; don't use chmod 777.

### Scenario 4: Page won't load at all, Apache is down

- Likely cause: Apache stopped, or failed to start (config error).
- Check: `systemctl status apache2`; `sudo apache2ctl configtest` (shows the exact file and line of a syntax error); `sudo journalctl -u apache2 -n 30 --no-pager`
- Fix: correct the bad line (back up first), re-run `configtest` until it says "Syntax OK", then `sudo systemctl restart apache2`.
- Also check: `sudo ss -tlnp | grep :80` (something else holding port 80?).

### Scenario 5: "Your PHP installation appears to be missing the MySQL extension"

- Likely cause: `php-mysql` not installed, or Apache hasn't reloaded since it was installed.
- Check: `php -m | grep -i mysql`; `dpkg -l | grep php`; compare `ls /etc/php/8.4/cli/conf.d | grep -i mysql` with `ls /etc/php/8.4/apache2/conf.d | grep -i mysql` (adjust version).
- Fix: install the missing package with `apt` if needed, then `sudo systemctl restart apache2`.
- Lesson: config can be right while the running process is stale. Restart before concluding a fix failed.

### Scenario 6: Disk full (uploads fail, DB crashes, weird errors)

- Likely cause: no free space.
- Check: `df -h` (look for 100% on `/`); `df -i` (inodes); find the culprit: `sudo du -sh /var/* | sort -h` then drill into the biggest folder.
- Usual suspects: huge logs in `/var/log`, backups, cache folders.
- Fix: remove or rotate the big file (be careful, never delete blindly; confirm what it is), then restart services that crashed (`mariadb`, `apache2`).
- Tell customer: the server ran out of space, what you cleared, and recommend monitoring or a bigger disk.

### Scenario 7: White screen or HTTP 500 after a plugin/theme change

- Likely cause: a broken plugin or theme (PHP fatal error).
- Check: `sudo tail -n 30 /var/log/apache2/error.log`, look for "PHP Fatal error" and the file path (it names the plugin).
- Fix: disable it by renaming its folder: `sudo mv /var/www/html/wp-content/plugins/<name> /var/www/html/wp-content/plugins/<name>.disabled`, reload, then restore later once it's fixed or updated.
- Optional: temporarily set `WP_DEBUG` to true in `wp-config.php` to see errors (turn it off afterwards).

### Scenario 8: "Can't upload images / install plugins"

- Likely cause: `wp-content/uploads` not writable by www-data, or disk full.
- Check: `ls -ld /var/www/html/wp-content/uploads`; `df -h`; Apache error log.
- Fix: `sudo chown -R www-data:www-data /var/www/html/wp-content/uploads` and directories 755 / files 644. Fix the narrowest thing.

### Scenario 9: Site unreachable, DNS points at the wrong place

- Likely cause: the domain resolves to the wrong or a private IP (like your SiteHost challenge).
- Check: `nslookup domain` or `dig domain`; compare the IP with the server's real public IP; `curl -I http://<correct-ip> -H "Host: domain"` to test the server directly.
- Private IP (10.x, 172.16-31.x, 192.168.x)? Unreachable from the internet, so the DNS record needs fixing.
- Quick test on your own machine: override with your hosts file.
- Tell customer: the DNS record points to the wrong address; needs to be updated at their DNS provider. Mention TTL (changes can take time).

---

## 4. Customer reply template

1. What I found (plain language, no jargon)
2. What I did to fix it
3. What you should check or do next (if anything)
4. Short, polite sign-off

Example (Scenario 1):

> Hi Maria, thanks for reporting this. Your site's database service had stopped, which caused the "database connection" error. I restarted it and confirmed your site is loading normally again. If you see this again, please let us know and we'll look into the underlying cause. Kind regards, Allen

---

## 5. Test-day checklist

- Think out loud: say what you're checking and why.
- Back up before editing: `cp file file.bak`
- One change at a time, retest after each
- Restart the service after config changes
- Open book: search error messages exactly, read docs, don't paste commands blindly
- If stuck: say what you ruled out and what you'd check next
- Finish with the customer email: found / did / next
