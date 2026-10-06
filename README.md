# Multi-Site Hosting with Virtual Hosts: Apache vs Nginx

A hands-on lab that hosts **three websites on one server** with both **Apache virtual hosts** and **Nginx server blocks**. Each site has its own document root and its own access and error logs. The lab ends with a practical comparison of **`.htaccess` (Apache)** and **central configuration (Nginx)**, including a real security difference between the two.

## How name-based virtual hosting works

One server, one IP address, many websites. The browser sends the site name in the `Host` header, and the web server uses it to choose the right site.

```
Client: GET /   Host: site1.local
              │
        Web server (one IP, one port)
              ├── site1.local → /var/www/site1/html → site1 logs
              ├── site2.local → /var/www/site2/html → site2 logs
              ├── site3.local → /var/www/site3/html → site3 logs
              └── anything else → default site
```

## Lab design

There are no real domains, so names are faked with `/etc/hosts`. Apache and Nginx run **at the same time on different ports** and serve the same three sites, which makes side-by-side comparison easy.

| Server | Port | Config location |
|---|---|---|
| Apache httpd | 8008 | `/etc/httpd/conf.d/vhosts.conf` |
| Nginx | 8009 | `/etc/nginx/conf.d/sites.conf` |

Ports 8008 and 8009 are already allowed for web servers by SELinux on Fedora/RHEL. For any other port, run `sudo semanage port -a -t http_port_t -p tcp PORT`. In production only one server can own port 80/443.

## What this project demonstrates

| Skill | How |
|---|---|
| Name-based virtual hosting | Apache `<VirtualHost>` and Nginx `server` blocks |
| Separate document roots | One folder per site under `/var/www` |
| Per-site logging | Separate access and error logs for each site |
| Default (catch-all) site | Handles requests for unknown host names |
| Apache vs Nginx comparison | Same sites, same tests, different configuration models |
| `.htaccess` vs central config | Live demo of rules that work on Apache and are ignored by Nginx |

## Prerequisites

- Fedora or RHEL-family Linux with `sudo` (SELinux enforcing)
- Nothing else already using ports 8008 and 8009

```bash
sudo dnf install -y httpd nginx curl
```

## Project structure

```
multi-site-hosting/
├── README.md
├── apache/vhosts.conf
├── nginx/sites.conf
├── site1/html/{index.html,404.html,.htaccess,private/{secret.html,.htaccess}}
├── site2/html/index.html
├── site3/html/index.html
└── evidence/
```

## Step-by-step setup

### Step 1: Fake domain names

```bash
echo "127.0.0.1 site1.local www.site1.local site2.local www.site2.local site3.local www.site3.local" | sudo tee -a /etc/hosts
getent hosts site1.local
```

### Step 2: Create separate document roots

```bash
for s in site1 site2 site3 default; do
  sudo mkdir -p /var/www/$s/html
  echo "<h1>Welcome to $s</h1>" | sudo tee /var/www/$s/html/index.html
done

# extra files for the .htaccess experiment
sudo mkdir -p /var/www/site1/html/private
echo "secret page" | sudo tee /var/www/site1/html/private/secret.html
echo "<h1>Custom 404 from site1</h1>" | sudo tee /var/www/site1/html/404.html
```

Files under `/var/www` automatically receive the correct SELinux label.

## Part A: Apache virtual hosts

### A1: Change the listening port

```bash
sudo sed -i 's/^Listen .*/Listen 8008/' /etc/httpd/conf/httpd.conf
echo "ServerName localhost" | sudo tee /etc/httpd/conf.d/servername.conf
```

Apache's `mod_ssl` package adds its own `Listen 443` in `ssl.conf`. If another service (for example Nginx from an HTTPS lab) already owns port 443, Apache refuses to start. This lab only needs plain HTTP, so disable that file:

```bash
sudo mv /etc/httpd/conf.d/ssl.conf ~/ssl.conf.bak 2>/dev/null
grep -rn "^Listen" /etc/httpd/conf /etc/httpd/conf.d      # expect exactly one: Listen 8008
```

### A2: Write the virtual hosts

Create `/etc/httpd/conf.d/vhosts.conf`:

```apache
# Default site: Apache uses the FIRST vhost for unknown names
<VirtualHost *:8008>
    ServerName default.invalid
    DocumentRoot /var/www/default/html
    ErrorLog  /var/log/httpd/default-error.log
    CustomLog /var/log/httpd/default-access.log combined
</VirtualHost>

<VirtualHost *:8008>
    ServerName site1.local
    ServerAlias www.site1.local
    DocumentRoot /var/www/site1/html
    ErrorLog  /var/log/httpd/site1-error.log
    CustomLog /var/log/httpd/site1-access.log combined
    <Directory /var/www/site1/html>
        # allow .htaccess on this site only (used in Part C)
        AllowOverride All
    </Directory>
</VirtualHost>

<VirtualHost *:8008>
    ServerName site2.local
    ServerAlias www.site2.local
    DocumentRoot /var/www/site2/html
    ErrorLog  /var/log/httpd/site2-error.log
    CustomLog /var/log/httpd/site2-access.log combined
</VirtualHost>

<VirtualHost *:8008>
    ServerName site3.local
    ServerAlias www.site3.local
    DocumentRoot /var/www/site3/html
    ErrorLog  /var/log/httpd/site3-error.log
    CustomLog /var/log/httpd/site3-access.log combined
</VirtualHost>
```

| Directive | Meaning |
|---|---|
| `<VirtualHost *:8008>` | One site, listening on port 8008 |
| `ServerName` / `ServerAlias` | The host names that select this site |
| `DocumentRoot` | The folder holding the site's files |
| `ErrorLog` / `CustomLog` | **Per-site** log files |
| `AllowOverride All` | Lets `.htaccess` files change settings in this folder |

Apache does not allow comments after a directive on the same line. Always put comments on their own line.

### A3: Check and start

```bash
sudo apachectl configtest        # expect: Syntax OK
sudo apachectl -S                # prints the vhost summary
sudo systemctl enable --now httpd
sudo ss -tlnp | grep httpd       # expect: *:8008
```

### A4: Test and check the per-site logs

```bash
for site in site1 site2 site3; do curl -s http://$site.local:8008; done
curl -H "Host: unknown.local" http://127.0.0.1:8008      # default site

curl -s http://site2.local:8008 > /dev/null
sudo tail -n 2 /var/log/httpd/site2-access.log
sudo wc -l /var/log/httpd/*-access.log
```

Only the log of the site you requested should grow.

## Part B: Nginx server blocks

### B1: Prepare

Make sure no configs from earlier labs are loaded:

```bash
ls /etc/nginx/conf.d/
sudo ss -tlnp | grep -E ':80 |:443 '
```

Fedora's main `nginx.conf` contains a default server on port 80. If port 80 is busy, free it or comment out that block, otherwise Nginx will not start.

### B2: Write the server blocks

Create `/etc/nginx/conf.d/sites.conf`:

```nginx
# Default site for unknown names
server {
    listen 8009 default_server;
    server_name _;
    root /var/www/default/html;
    access_log /var/log/nginx/default-access.log;
    error_log  /var/log/nginx/default-error.log;
}

server {
    listen 8009;
    server_name site1.local www.site1.local;
    root /var/www/site1/html;
    index index.html;
    access_log /var/log/nginx/site1-access.log;
    error_log  /var/log/nginx/site1-error.log;
}

server {
    listen 8009;
    server_name site2.local www.site2.local;
    root /var/www/site2/html;
    index index.html;
    access_log /var/log/nginx/site2-access.log;
    error_log  /var/log/nginx/site2-error.log;
}

server {
    listen 8009;
    server_name site3.local www.site3.local;
    root /var/www/site3/html;
    index index.html;
    access_log /var/log/nginx/site3-access.log;
    error_log  /var/log/nginx/site3-error.log;
}
```

| Directive | Meaning |
|---|---|
| `server { }` | One site |
| `listen 8009 default_server` | Port, and "use this block for unknown names" |
| `server_name` | The host names that select this block |
| `root` | Document root |
| `access_log` / `error_log` | **Per-site** logs |

### B3: Check and start

```bash
sudo nginx -t
sudo systemctl enable --now nginx
```

### B4: Test both servers side by side

```bash
for site in site1 site2 site3; do
  echo "Apache $site: $(curl -s http://$site.local:8008)"
  echo "Nginx  $site: $(curl -s http://$site.local:8009)"
done
curl -H "Host: unknown.local" http://127.0.0.1:8009
sudo tail -n 2 /var/log/nginx/site2-access.log
```

## Part C: `.htaccess` vs Nginx configuration

`.htaccess` is a small Apache config file placed **inside the website folder**. Nginx has no equivalent: all settings live in the central server config.

### C1: Create `.htaccess` rules (Apache)

```bash
sudo tee /var/www/site1/html/.htaccess <<'EOF'
Options -Indexes
ErrorDocument 404 /404.html
Redirect 301 /old-page.html /index.html
EOF

echo "Require all denied" | sudo tee /var/www/site1/html/private/.htaccess
```

They take effect immediately, with no reload:

```bash
curl -I http://site1.local:8008/old-page.html          # 301 to /index.html
curl -I http://site1.local:8008/private/secret.html    # 403 Forbidden
curl http://site1.local:8008/nope.html                 # Custom 404 from site1
```

### C2: Run the same tests against Nginx

```bash
curl -I http://site1.local:8009/old-page.html          # 404: redirect ignored
curl -I http://site1.local:8009/private/secret.html    # 200: secret file exposed
curl http://site1.local:8009/nope.html                 # default Nginx 404 page
```

**Nginx ignores `.htaccess` completely.** The "private" file that Apache protected is wide open on Nginx. When migrating a site from Apache to Nginx, every `.htaccess` rule must be translated, or protections silently disappear.

### C3: Add the Nginx equivalents

Add inside the `site1.local` server block:

```nginx
    autoindex off;
    error_page 404 /404.html;
    location = /old-page.html { return 301 /index.html; }
    location /private/ { deny all; }
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Run the tests from C2 again. The results now match Apache: 301, 403, and the custom 404.

### C4: Rule translation table

| Task | Apache (`.htaccess` or vhost) | Nginx (server block) |
|---|---|---|
| Redirect one URL | `Redirect 301 /old /new` | `location = /old { return 301 /new; }` |
| Block a folder | `Require all denied` | `location /private/ { deny all; }` |
| Custom error page | `ErrorDocument 404 /404.html` | `error_page 404 /404.html;` |
| Disable directory listing | `Options -Indexes` | `autoindex off;` |
| Password protection | `AuthType Basic` + `AuthUserFile` | `auth_basic` + `auth_basic_user_file` |
| URL rewriting | `RewriteEngine On` + `RewriteRule` | `rewrite` or `try_files` |
| Browser caching | `ExpiresActive On` / `Header set` | `expires 7d;` / `add_header` |

### C5: Key differences

| | Apache `.htaccess` | Nginx config |
|---|---|---|
| Where rules live | In each website folder | Central config files only |
| When read | On every request, for each folder in the path | Once, at start or reload |
| Applying changes | Instantly | `nginx -s reload` |
| Performance | Slower (extra file lookups) | Faster |
| Who can change it | Anyone with write access to the site files | Only the server admin |
| Safety net | A bad file can break the site immediately | `nginx -t` catches errors before applying |
| Best for | Shared hosting, apps that manage their own rules (for example WordPress) | Servers you fully control |

Best practice on Apache when you control the server: set `AllowOverride None` and put the rules in the vhost, which avoids the per-request cost.

### Results

| Check | Result |
|---|---|
| All three sites served correctly by Apache | [PASS/FAIL] |
| All three sites served correctly by Nginx | [PASS/FAIL] |
| Unknown host falls to the default site (both) | [PASS/FAIL] |
| Per-site logs only grow for the requested site | [PASS/FAIL] |
| `.htaccess` rules work on Apache | [PASS/FAIL] |
| `.htaccess` rules ignored by Nginx (secret file exposed) | [observed status code] |
| Nginx equivalents match Apache behavior | [PASS/FAIL] |

## Troubleshooting (issues hit while building this)

| Symptom | Cause | Fix |
|---|---|---|
| `AH00526: Syntax error ... Illegal override option #` | A comment placed after a directive on the same line | Put comments on their own line |
| `AH00558: Could not reliably determine the server's fully qualified domain name` | No global `ServerName` (harmless warning) | `echo "ServerName localhost" \| sudo tee /etc/httpd/conf.d/servername.conf` |
| `curl: (7) Failed to connect` on 8008 | Apache is not running | `sudo systemctl status httpd`, `sudo journalctl -xeu httpd` |
| `AH00072: could not bind to address ...:443` | `ssl.conf` adds `Listen 443` and another service owns 443 | Move `/etc/httpd/conf.d/ssl.conf` away, restart Apache |
| Nginx fails on port 80 | Fedora's default server block or an old lab config | Free port 80 or comment out the default block, move old files from `conf.d` |
| Every site shows the default page | Missing `/etc/hosts` entry or name mismatch | `getent hosts site1.local`, check `ServerName`/`server_name` spelling |
| `403 Forbidden` | Wrong permissions or SELinux label | `sudo restorecon -Rv /var/www`, keep docroots under `/var/www` |
| `.htaccess` has no effect | `AllowOverride None` | Set `AllowOverride All` for that directory and restart Apache |
| `500 Internal Server Error` after editing `.htaccess` | Bad syntax or a directive that is not allowed | `sudo tail /var/log/httpd/site1-error.log` |
| Nginx warning `conflicting server name` | Same `server_name` in two blocks | Remove the duplicate |

Debugging tools worth remembering: `apachectl -S` (Apache vhost table), `nginx -T` (Nginx active config), `ss -tlnp` (who listens where), `journalctl -xeu <service>`.

## Evidence

```bash
sudo apachectl -S > evidence/apache-vhosts.txt 2>&1
sudo nginx -T > evidence/nginx-config.txt 2>&1
for site in site1 site2 site3; do
  echo "Apache $site: $(curl -s http://$site.local:8008)"
  echo "Nginx  $site: $(curl -s http://$site.local:8009)"
done > evidence/both-servers.txt
sudo wc -l /var/log/httpd/*-access.log /var/log/nginx/*-access.log > evidence/per-site-logs.txt
# run BEFORE step C3, then again AFTER it
curl -I http://site1.local:8009/private/secret.html > evidence/nginx-private-before.txt
curl -I http://site1.local:8009/private/secret.html > evidence/nginx-private-after.txt
```

## Cleanup

```bash
sudo systemctl disable --now httpd nginx
sudo rm /etc/httpd/conf.d/vhosts.conf /etc/nginx/conf.d/sites.conf
sudo sed -i '/site[123].local/d' /etc/hosts
sudo rm -rf /var/www/site1 /var/www/site2 /var/www/site3 /var/www/default
```

## Next steps

- HTTPS for each site with SNI, using a private CA or Let's Encrypt
- Nginx on port 80 as a reverse proxy in front of Apache on 8008
- PHP-FPM on one site only
- An Ansible role that generates all vhosts and server blocks from a list of sites with a Jinja2 template
- Log rotation per site with `logrotate`

## Skills demonstrated

Name-based virtual hosting, Apache and Nginx configuration, per-site logging, SELinux-aware web hosting, port conflict diagnosis, `.htaccess` vs central configuration trade-offs, and troubleshooting with logs and `apachectl`/`nginx -T`.
