# Deploying chromeplasma.space — EC2 + nginx + Cloudflare

Complete reference for the current setup: an app on port 3000 of an Ubuntu EC2
instance, served publicly at `https://chromeplasma.space` through Cloudflare.

**Environment**

| Piece | Value |
|---|---|
| Server | AWS EC2, Ubuntu, ap-south-1 |
| Public IP | 3.108.220.241 |
| App port | 3000 |
| Web server | nginx 1.28.3 |
| Domain registrar | Hostinger |
| DNS + CDN + TLS | Cloudflare |

---

## 1. Who is actually doing what

This is the question that prompted this document: HTTPS started working right
after Cloudflare was enabled, so which component made it work?

**Cloudflare made HTTPS work. nginx makes the site work at all.** They do
different jobs and both are required.

```
Browser
   │  https://chromeplasma.space          ← TLS terminated here, by Cloudflare
   ▼
Cloudflare edge
   │  http://3.108.220.241:80             ← this hop's encryption depends on SSL mode
   ▼
nginx  (listen 80)
   │  http://localhost:3000               ← always plaintext, but never leaves the box
   ▼
Your app
```

### Why it's definitely Cloudflare

- certbot was never run, so there is no TLS certificate on the EC2 instance.
- The nginx config contains only `listen 80;` — nothing on 443. nginx is
  physically incapable of completing an HTTPS handshake right now.
- Cloudflare issues a free **Universal SSL** certificate automatically once a
  domain uses their nameservers and the DNS record is proxied (orange cloud).

The padlock in the browser is Cloudflare's certificate.

Prove it:

```bash
curl -sI https://chromeplasma.space | grep -i -E 'server|cf-ray'
```

`server: cloudflare` plus a `cf-ray` header confirms the response came from
Cloudflare's edge.

### Why nginx still matters

Cloudflare forwards requests to port 80 on the origin. If nothing listens on
port 80, Cloudflare returns **error 521 — Web server is down**. nginx is the
component that accepts on 80 and proxies to 3000. Stop nginx and the site goes
down regardless of Cloudflare.

### And the earlier iptables rule

```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3000
```

That was the original approach. It works but has two fatal flaws: it vanishes
on reboot unless saved with `iptables-persistent`, and it can never do TLS,
headers, or multiple sites. It has been removed — `PREROUTING` is empty, which
is what nginx needs. **Never run both.** An iptables redirect on port 80
intercepts traffic before nginx sees it.

---

## 2. The part that needs fixing: SSL/TLS mode

Go to the Cloudflare dashboard → **SSL/TLS → Overview** and check the encryption
mode.

| Mode | Browser → Cloudflare | Cloudflare → your server | Verdict |
|---|---|---|---|
| Off | plaintext | plaintext | no |
| **Flexible** | encrypted | **plaintext** | insecure — likely your current state |
| Full | encrypted | encrypted, cert not validated | better |
| **Full (strict)** | encrypted | encrypted and validated | correct |

If you are on **Flexible**, the hop from Cloudflare to your EC2 box crosses the
open internet unencrypted. Visitors see a padlock that is telling them something
untrue, and passwords or session cookies travel in the clear on that leg.

Flexible also causes **redirect loops** with apps that redirect http→https
themselves: your app sees `X-Forwarded-Proto: http`, redirects to https,
Cloudflare receives it, forwards as http again, forever.

The fix is to put a certificate on the origin and switch to Full (strict).

---

## 3. Fixing it with a Cloudflare Origin Certificate

Preferred over certbot here: 15-year validity instead of 90 days, no renewal
automation to maintain, and no dependency on port-80 challenge validation
(which Cloudflare's proxying can interfere with).

### 3.1 Generate the certificate

Cloudflare dashboard → **SSL/TLS → Origin Server → Create Certificate**.

Accept the defaults (RSA, covers `chromeplasma.space` and `*.chromeplasma.space`,
15 years). Cloudflare then shows the **certificate** and the **private key**.

> Copy the private key immediately. It is displayed exactly once and cannot be
> retrieved later — you would have to generate a new certificate.

### 3.2 Install it on the server

```bash
# Install nginx (no-op if already present — keeps this block self-contained)
sudo apt update && sudo apt install nginx -y
sudo systemctl status nginx --no-pager

# Create the certificate directory
sudo mkdir -p /etc/ssl/cloudflare

# Paste the certificate, then the private key
sudo nano /etc/ssl/cloudflare/chromeplasma.pem
sudo nano /etc/ssl/cloudflare/chromeplasma.key

# Lock down permissions
sudo chmod 600 /etc/ssl/cloudflare/chromeplasma.key
sudo chmod 644 /etc/ssl/cloudflare/chromeplasma.pem
sudo chown root:root /etc/ssl/cloudflare/*
```

In nano: paste, `Ctrl+O` then `Enter` to save, `Ctrl+X` to exit. Include the
`-----BEGIN ...-----` and `-----END ...-----` lines in both files.

Verify before going further:

```bash
sudo openssl x509 -in /etc/ssl/cloudflare/chromeplasma.pem -noout -subject -dates
sudo openssl rsa -in /etc/ssl/cloudflare/chromeplasma.key -check -noout
```

The first prints the domain and a roughly 15-year expiry; the second prints
`RSA key ok`. A malformed paste here is the usual cause of nginx failing to
start with an `SSL_CTX_use_PrivateKey_file` error.

### 3.3 Rewrite the nginx config

Full contents of `/etc/nginx/sites-available/chromeplasma`:

```nginx
# Redirect all plain HTTP to HTTPS
server {
    listen 80;
    server_name chromeplasma.space www.chromeplasma.space;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    http2 on;
    server_name chromeplasma.space www.chromeplasma.space;

    ssl_certificate     /etc/ssl/cloudflare/chromeplasma.pem;
    ssl_certificate_key /etc/ssl/cloudflare/chromeplasma.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

What each proxy header is for:

- `Upgrade` / `Connection` — WebSocket support. Next.js hot reload and any
  realtime feature breaks without these.
- `Host $host` — your app sees the real hostname, not `localhost`.
- `X-Real-IP` / `X-Forwarded-For` — visitor IP. Without these every request
  looks like it came from 127.0.0.1. (See §5 — behind Cloudflare these need
  extra work.)
- `X-Forwarded-Proto` — whether the original request was http or https. Apps
  use this to build absolute URLs and decide on redirects.

### 3.4 Apply

```bash
sudo ufw allow 'Nginx Full'
sudo nginx -t
sudo systemctl reload nginx
```

`nginx -t` before every reload, always. A syntax error means nginx refuses to
start and the whole site goes dark.

### 3.5 Switch Cloudflare to Full (strict)

SSL/TLS → Overview → **Full (strict)**. Then enable **Always Use HTTPS** under
SSL/TLS → Edge Certificates.

### 3.6 Verify

```bash
# origin serving TLS correctly (bypasses Cloudflare)
curl -kIv https://3.108.220.241 --resolve chromeplasma.space:443:3.108.220.241

# end to end
curl -sI https://chromeplasma.space
curl -sI http://chromeplasma.space     # should be 301 to https
```

---

## 4. Lock the origin down

Right now anyone can hit `http://3.108.220.241` directly and bypass Cloudflare
entirely — no DDoS protection, no WAF, no caching, and your real IP is exposed.

In the EC2 security group, replace the `0.0.0.0/0` rules on ports 80 and 443
with Cloudflare's published IP ranges (https://www.cloudflare.com/ips/). Keep
SSH on 22 restricted to your own IP.

Optionally enforce it at the nginx level too — `deny all;` with `allow` entries
for the Cloudflare ranges.

Do not lock yourself out: keep SSH access working and test in a second terminal
before closing your current session.

---

## 5. Getting the real visitor IP

With Cloudflare in front, `X-Forwarded-For` as your app receives it will show
Cloudflare's edge IP, not the visitor. The true client IP arrives in the
**`CF-Connecting-IP`** header.

Either read `CF-Connecting-IP` directly in your app, or restore it at the nginx
level so the standard headers become correct again — nginx's `real_ip` module
with `set_real_ip_from` entries for each Cloudflare range and
`real_ip_header CF-Connecting-IP`.

Matters for rate limiting, audit logs, geolocation, and abuse blocking. If you
do none of these, you can ignore it.

---

## 6. Keeping the app alive

nginx returns **502 Bad Gateway** when nothing is listening on 3000. If the app
was started with `npm start` in an SSH session, it died when you disconnected.

```bash
sudo npm install -g pm2
pm2 start npm --name "app" -- start
pm2 startup      # prints a command to copy and run
pm2 save
```

`pm2 startup` + `pm2 save` together are what make the app come back after a
reboot. Running only one of them is a common mistake.

Day to day:

```bash
pm2 list          # what's running
pm2 logs app      # tail output
pm2 restart app   # after a deploy
pm2 monit         # live resource usage
```

nginx already survives reboots — the apt install registered its systemd unit.

---

## 7. Command reference

```bash
# nginx
sudo nginx -t                          # validate config — before every reload
sudo systemctl reload nginx            # apply config, no dropped connections
sudo systemctl restart nginx           # hard restart
sudo systemctl status nginx
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log

# app
pm2 list && pm2 logs app

# what is listening where
sudo ss -tlnp

# firewalls
sudo ufw status
sudo iptables -t nat -L PREROUTING -n --line-numbers

# dns — nameservers should be Cloudflare's now, not Hostinger's
dig +short chromeplasma.space
dig NS chromeplasma.space +short

# test the chain, innermost outward
curl -I http://localhost:3000          # app
curl -I http://localhost               # nginx
curl -I https://chromeplasma.space     # full path through Cloudflare
```

Those last three are the diagnostic backbone. Whichever fails first is where
the break is:

| Fails at | Meaning | Where to look |
|---|---|---|
| app | not running / wrong port | `pm2 logs app`, `sudo ss -tlnp` |
| nginx | config or service problem | `sudo nginx -t`, `systemctl status nginx` |
| public | DNS, security group, ufw, or Cloudflare | dashboard, `sudo ufw status`, AWS console |

---

## 8. Errors and what they mean

**Cloudflare 521 — web server is down.** Cloudflare reached your IP, nothing
answered. nginx is stopped, or the security group blocks Cloudflare's IPs.

**Cloudflare 522 — connection timed out.** Packets are being dropped rather than
refused. Almost always a firewall: security group or ufw.

**Cloudflare 526 — invalid SSL certificate.** You are on Full (strict) but the
origin certificate is missing, expired, or doesn't match. Check §3.

**Cloudflare 520 — unknown error.** Origin returned something malformed. Look at
`/var/log/nginx/error.log`.

**502 Bad Gateway (from nginx, not Cloudflare).** nginx is fine; the app on 3000
is not. `pm2 list`.

**Redirect loop / ERR_TOO_MANY_REDIRECTS.** Classic Flexible-mode symptom, or an
http→https redirect in both nginx and the app. Move to Full (strict).

**nginx welcome page instead of the app.** The default site is still enabled:
`sudo rm /etc/nginx/sites-enabled/default && sudo systemctl reload nginx`.

**Changes to DNS not taking effect.** With Cloudflare proxying on, records are
served from their edge. Also check you actually changed nameservers at
Hostinger to Cloudflare's — otherwise Hostinger's DNS is still authoritative and
Cloudflare does nothing.

**Everything died after a reboot.** The signature of the iptables approach. With
nginx + `pm2 startup` + `pm2 save`, both layers return automatically.

---

## 9. Doing this again from scratch

1. Launch the instance; security group allows 22 (your IP only), 80, 443.
2. Get the app running and listening on 3000. Confirm with `curl -I http://localhost:3000`.
3. `sudo apt update && sudo apt install nginx -y`
4. Write `/etc/nginx/sites-available/<name>` with the proxy block.
5. `sudo ln -s /etc/nginx/sites-available/<name> /etc/nginx/sites-enabled/`
6. `sudo rm /etc/nginx/sites-enabled/default`
7. `sudo nginx -t && sudo systemctl reload nginx`
8. `sudo ufw allow 'Nginx Full'`
9. pm2: `pm2 start`, `pm2 startup`, `pm2 save`
10. Confirm plain HTTP works over the public IP.
11. Add the site to Cloudflare; change nameservers at the registrar; wait for
    activation; set the A record to the instance IP with the proxy **on**.
12. Create an Origin Certificate, install it, add the 443 server block.
13. Set SSL mode to **Full (strict)** and enable **Always Use HTTPS**.
14. Restrict the security group to Cloudflare IP ranges.

Steps 8, 11, and 14 are the ones people forget. The first two produce identical
useless timeouts; the third leaves the origin wide open with no visible symptom
at all.

---

## 10. Mental model to keep

Four independent layers, each of which can break the site on its own:

1. **The app** — is the process running and bound to 3000?
2. **nginx** — is it listening on 80/443 and proxying correctly?
3. **Firewalls** — AWS security group *and* ufw both have to allow the traffic.
4. **DNS + Cloudflare** — does the name resolve, is the proxy on, is the SSL
   mode right?

Debug them in that order, innermost first. Testing from the outside in only
tells you that something is broken; testing from the inside out tells you what.
