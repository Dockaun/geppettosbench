# Deploy Geppetto's Bench to its own DigitalOcean droplet

**Host:** geppettosbench (Toronto/TOR1), public IPv4 `159.203.29.226`.
**Repository:** https://github.com/Dockaun/geppettosbench

The site is static. It does not require Node, a database, or Replit at runtime.

## Stage 1 — Install website and test by IP

The new droplet already has Caddy installed. Connect to the **geppettosbench** console, NOT the PinocchAIo droplet, then run:

```bash
apt-get update && apt-get install -y git
mkdir -p /var/www/geppettosbench
git clone https://github.com/Dockaun/geppettosbench.git /tmp/geppettosbench-site
cp /tmp/geppettosbench-site/index.html /tmp/geppettosbench-site/makeosity.html /tmp/geppettosbench-site/favicon.svg /tmp/geppettosbench-site/robots.txt /var/www/geppettosbench/
```

Create a separate Caddy site config for **HTTP testing at the droplet IP**, before changing any DNS records:

```bash
cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.before-geppettosbench
cat > /etc/caddy/Caddyfile <<'EOF'
http://159.203.29.226 {
    root * /var/www/geppettosbench
    encode zstd gzip
    file_server
}
EOF
caddy fmt --overwrite /etc/caddy/Caddyfile
caddy validate --config /etc/caddy/Caddyfile
systemctl reload caddy
```

Then test `http://159.203.29.226/` and `http://159.203.29.226/makeosity.html` in your browser. Test navigation links.

## Stage 2 — Switch GoDaddy DNS (only AFTER Stage 1 passes)

Find the **authoritative DNS host** for `geppettosbench.com`. If GoDaddy is authoritative, update the `@` A record to `159.203.29.226` and set `www` as a CNAME to `@` (or an A record to the same IP). Remove conflicting obsolete A/AAAA records only after verifying their purpose. Do not change nameservers or other domains. Allow DNS to update.

On the **geppettosbench** droplet only, replace the Caddyfile with:

```bash
cat > /etc/caddy/Caddyfile <<'EOF'
geppettosbench.com, www.geppettosbench.com {
    root * /var/www/geppettosbench
    encode zstd gzip
    file_server
}
EOF
caddy fmt --overwrite /etc/caddy/Caddyfile
caddy validate --config /etc/caddy/Caddyfile
systemctl reload caddy
```

Caddy will request HTTPS certificates after valid DNS and open ports 80/443. Ensure the droplet has public IPv4 and that firewalls allow inbound 80/443; permit SSH (22) only from trusted sources.

Test `https://geppettosbench.com/`, `https://www.geppettosbench.com/`, and `https://geppettosbench.com/makeosity.html`. **Only after those pass**, disable the Replit deployment and then cancel Replit billing.

## Updating site later

```bash
cd /tmp/geppettosbench-site && git pull --ff-only
cp index.html makeosity.html favicon.svg robots.txt /var/www/geppettosbench/
```

Avoid editing production files directly; commit changes to GitHub then redeploy.

## Rollback

If HTTP testing fails, restore the original Caddyfile:

```bash
cp /etc/caddy/Caddyfile.before-geppettosbench /etc/caddy/Caddyfile
systemctl reload caddy
```

If DNS switch fails, restore the previous DNS A/CNAME records (record the old values before editing). Keep Replit published until the new domain works.
