# ACME Certificate Automation

A generic implementation demonstrating how TLS certificate issuance and
renewal can be fully automated using the ACME protocol, Certbot, and a
deployment hook — covering both the **ACME protocol layer** (communicating
with the CA) and the **automation layer** (deploying the certificate and
restarting the consuming service).

---

## Overview

TLS certificates have a finite lifetime and must be renewed periodically.
Manual renewal introduces operational risk: missed renewals, service
disruption, repetitive administrative work, and dependency on human
availability at every step.

The ACME (Automatic Certificate Management Environment) protocol lets
certificate authorities verify domain ownership and issue certificates
automatically. This project shows how to automate the full certificate
lifecycle — from issuance through to service reload — with no manual
intervention required after initial setup.

---

## Two-Part Architecture

The solution is composed of two distinct but connected layers:

```
┌─────────────────────────────────────────────────────────┐
│                   PART 1 — ACME PROTOCOL                │
│                                                         │
│  Scheduled trigger (cron / systemd timer)               │
│            │                                            │
│            ▼                                            │
│  Certbot checks renewal window (ARI or 30-day fallback) │
│            │                                            │
│            ▼                                            │
│  Certbot authenticates to CA (EAB or no credentials)   │
│            │                                            │
│            ▼                                            │
│  CA validates domain ownership                          │
│            │                                            │
│            ▼                                            │
│  CA issues signed certificate                           │
│  → fullchain.pem + privkey.pem saved to disk            │
└──────────────────────────┬──────────────────────────────┘
                           │
                           │  deploy hook fires automatically
                           ▼
┌─────────────────────────────────────────────────────────┐
│                PART 2 — AUTOMATION LAYER                │
│                                                         │
│  Deploy hook script runs                                │
│            │                                            │
│            ▼                                            │
│  Fix file permissions (service user can read cert)      │
│            │                                            │
│            ▼                                            │
│  Validate service configuration                         │
│            │                                            │
│            ▼                                            │
│  Restart / reload consuming service                     │
│            │                                            │
│            ▼                                            │
│  Log completion with timestamp and cert details         │
└─────────────────────────────────────────────────────────┘
```

Part 1 produces the certificate. Part 2 ensures it is immediately put into
service. The two layers are connected by Certbot's deploy hook mechanism —
a script that fires automatically after every successful renewal.

---

## Domain Validation

ACME requires the CA to verify domain ownership before issuing a
certificate. There are two standard methods:

**HTTP-01** — the CA fetches a token file placed at
`http://<domain>/.well-known/acme-challenge/<token>` over port 80. Requires
port 80 to be reachable from the public internet.

**DNS-01** — the CA looks for a TXT record at `_acme-challenge.<domain>`.
Requires API access to the DNS provider to create and delete that record
automatically. This method works even when port 80 is unavailable, and is
the only method that supports wildcard certificates.

**Provider-managed validation** — some CAs handle domain validation
internally as part of their ACME implementation. In this case, no port 80
exposure and no DNS API access are required. Authentication is handled
entirely via External Account Binding (EAB) credentials (a KID + HMAC key
pair) issued by the CA per domain. This is the simplest path when available.

> The validation method depends entirely on your CA. Check your CA's ACME
> documentation to determine which methods are supported and whether EAB
> credentials are required.

---

## Requirements

- Linux (RHEL/Rocky/Ubuntu or compatible)
- Python 3.8+ (in a dedicated virtual environment — do not use system Python)
- Certbot 4.1.0 or higher (required for ARI support)
- A domain name
- ACME-compatible CA credentials:
  - EAB Key ID (KID) and HMAC key — if your CA requires external account binding
  - Or no credentials — if using Let's Encrypt (DV only, no EAB required)
- Access to the service configuration consuming the certificate

---

## Configuration

Certbot reads its global configuration from `/etc/letsencrypt/cli.ini`.
Credentials must never be stored in scripts or committed to version control.

`config/cli.ini.example`:

```ini
# ACME directory URL — replace with your CA's endpoint
server = https://acme.<your-ca>.net/directory

# EAB credentials — required if your CA uses external account binding
# Leave these out if using Let's Encrypt (no EAB required)
eab-kid = CHANGE_ME
eab-hmac-key = CHANGE_ME

# Validation method
# Use standalone if port 80 is available and nothing else is bound to it
# Use dns-<provider> if DNS-01 is required
authenticator = standalone
```

`.gitignore`:

```
config/cli.ini
*.pem
*.key
.env
```

Credentials files must:
- Be excluded from version control
- Have restrictive filesystem permissions (`chmod 600`)
- Use the minimum required privileges

---

## Usage

### Python Virtual Environment Setup

If the system Python version is old or used by other services, install a
newer Python version in parallel and create a dedicated virtual environment
for Certbot. This avoids any conflict with existing system packages.

```bash
# Example: install Python 3.9 alongside system Python on RHEL/Rocky
sudo dnf install python39 -y

# Create a dedicated virtual environment
sudo python3.9 -m venv /opt/certbot
sudo /opt/certbot/bin/pip install --upgrade pip
sudo /opt/certbot/bin/pip install certbot

# Expose certbot system-wide via symlink
sudo ln -s /opt/certbot/bin/certbot /usr/bin/certbot
certbot --version
```

> The symlink ensures certbot is findable by cron, systemd, and any script
> regardless of PATH context — not just interactive shell sessions.

### Issue a Certificate

Request the certificate from your CA:

```bash
certbot certonly \
    --config /etc/letsencrypt/cli.ini \
    --agree-tos \
    --non-interactive \
    --email admin@example.com \
    --cert-name example.com \
    -d example.com \
    -v
```

Verify the certificate was issued correctly:

```bash
openssl x509 -noout -dates -issuer \
    -in /etc/letsencrypt/live/example.com/fullchain.pem
```
Check the serial number — communicate it to your CA contact to confirm which certificate to keep active:

```bash
openssl x509 -noout -serial \
    -in /etc/letsencrypt/live/example.com/fullchain.pem
```
⚠️ Run certbot certonly only once per domain during initial setup. Running it again on an existing certificate will overwrite the renewal configuration and remove any registered deploy hook. Use certbot reconfigure to change settings on existing certificates.


### Register the Deploy Hook

After issuance, register the deploy hook so Certbot runs it automatically
after every successful renewal:

```bash
certbot reconfigure \
    --cert-name example.com \
    --deploy-hook "/path/to/scripts/deploy-hook.sh"
```

Verify it was registered:

```bash
cat /etc/letsencrypt/renewal/example.com.conf | grep hook
```

Expected output:

```
deploy_hook = /path/to/scripts/deploy-hook.sh
```

### Renew Certificates

```bash

certbot renew >> /var/log/certbot-cron.log 2>&1
```

Certbot checks all managed certificates and renews any that fall within
the renewal window. If nothing is due, it exits silently. The deploy hook
runs automatically after any successful renewal — no separate trigger needed.

### Deploy Hook

`scripts/deploy-hook.sh`:

```bash
#!/usr/bin/env bash
set -e

DOMAIN="example.com"
SERVICE="nginx" #or squid
LOGFILE="/var/log/certbot-hook.log"

echo "========================================" >> "$LOGFILE"
echo "Deploy hook started at $(date)" >> "$LOGFILE"

# Check certificate file permissions
echo "Checking certificate permissions..." >> "$LOGFILE"
chmod 755 /etc/letsencrypt/live
chmod 755 /etc/letsencrypt/archive
chmod 755 /etc/letsencrypt/archive/"$DOMAIN"
chmod 644 /etc/letsencrypt/archive/"$DOMAIN"/*.pem
ls -l /etc/letsencrypt/live/"$DOMAIN"/ >> "$LOGFILE" 2>&1

# Validate service configuration
echo "Validating ${SERVICE} configuration..." >> "$LOGFILE"
${SERVICE} -t >> "$LOGFILE" 2>&1
echo "Configuration valid" >> "$LOGFILE"

# Restart service
echo "Restarting ${SERVICE}..." >> "$LOGFILE"
systemctl restart "$SERVICE" >> "$LOGFILE" 2>&1
echo "${SERVICE} restarted successfully" >> "$LOGFILE"

echo "Deploy hook completed at $(date)" >> "$LOGFILE"
echo "========================================" >> "$LOGFILE"
```

> Validating the service configuration before restarting (`nginx -t`,
> `squid -k parse`, etc.) prevents blindly restarting a service with a
> broken config. If validation fails, `set -e` stops execution
> immediately and the service keeps running with the current certificate.

---

## Automated Renewal

### Option A — Cron (simple, no extra setup)

Add to root crontab (`crontab -e`):

```
0 12 * * * certbot renew >> /var/log/certbot-cron.log 2>&1
```

This runs once daily at noon. Adjust the time to fit your operational
preferences. The `2>&1` ensures both standard output and error output land
in the log file.

### Option B — systemd timer (recommended for production)

`systemd/certbot-renew.service`:

```ini
[Unit]
Description=Renew ACME TLS certificates

[Service]
Type=oneshot
ExecStart=/usr/bin/certbot renew
```

`systemd/certbot-renew.timer`:

```ini
[Unit]
Description=Periodic ACME certificate renewal

[Timer]
OnCalendar=*-*-* 12:00:00
RandomizedDelaySec=3600
Persistent=true

[Install]
WantedBy=timers.target
```

Enable it:

```bash
sudo systemctl enable --now certbot-renew.timer
systemctl list-timers | grep certbot
```

> `RandomizedDelaySec` spreads execution across a one-hour window — useful
> in fleet deployments to avoid all hosts hitting the CA simultaneously.

---

## Log Locations

| Log | Path |
|---|---|
| Certbot general activity | `/var/log/letsencrypt/letsencrypt.log` |
| Cron / renewal output | `/var/log/certbot-cron.log` |
| Deploy hook activity | `/var/log/certbot-hook.log` |

Watch the hook log in real time:

```bash
tail -f /var/log/certbot-hook.log
```

---

## Testing

Test the renewal check without issuing a real certificate:

```bash
certbot renew --dry-run
```

Confirm Certbot is not treating a certificate as due when it should not be:

```bash
certbot renew
# Expected: Certificate not yet due for renewal
```

Check all managed certificates and their expiry dates:

```bash
certbot certificates
```

Verify the certificate currently served by the service:

```bash
openssl s_client -connect <host>:<port> -servername <domain> 2>/dev/null \
    | openssl x509 -noout -dates -issuer
```

---

## Troubleshooting

**Another instance of Certbot already running:**
```bash
ps aux | grep certbot
kill <PID>
rm -f /var/log/letsencrypt/.certbot.lock
```

**Service failed to restart after renewal:**
- Check deploy hook log: `cat /var/log/certbot-hook.log`
- Validate config manually before restarting
- Check file permissions: `namei -l /etc/letsencrypt/live/<domain>/privkey.pem`
- Directories should be `755`, `.pem` files should be `644`

**Deploy hook not firing:**
- Verify it is registered: `cat /etc/letsencrypt/renewal/<domain>.conf | grep hook`
- Re-register if missing: `certbot reconfigure --cert-name <domain> --deploy-hook "/path/to/deploy-hook.sh"`

**Rollback:**
- Restore service config backup
- Validate config
- Restart service

---

## Security Considerations

- Store EAB credentials only in `/etc/letsencrypt/cli.ini` with `chmod 600`
- Never commit credentials to version control
- Every certificate issuance may be a billable event depending on the CA —
  avoid `--force-renewal` unless necessary
- After any issuance, confirm the serial number with your CA contact to
  ensure only the intended certificate is active

---

## Replicating for Additional Domains

The same mechanism applies to any additional domain managed by the same CA.
For each new domain:

1. Obtain EAB credentials for the new domain from the CA (if required)
2. Register the deploy hook via `certbot reconfigure`
3. Update the consuming service configuration to point at the new cert paths
4. The existing cron job or systemd timer picks up the new certificate
   automatically — no additional scheduling needed

---

*Generic implementation — replace placeholder values with environment-specific
configuration before use.*
