# Pterodactyl Wings Troubleshooting Guide


> **Security note:** A Panel API token was included in a terminal command shared during troubleshooting. Treat that token as exposed: revoke it and create a replacement in the Pterodactyl Admin Panel. Do not include API tokens, passwords, or private keys in logs shared publicly.

## 1. Summary of the problem

Wings configuration completed successfully, but the `wings` systemd service exited with status `1/FAILURE` when starting its internal HTTPS server.

Relevant log lines:

```text
INFO: fetching list of servers from API
INFO: processing servers returned by the API total_configs=0
INFO: configuring internal webserver host_address=0.0.0.0 host_port=8080 use_auto_tls=false use_ssl=true
FATAL: failed to configure HTTPS server auto_tls=false error=open /etc/letsencrypt/live/node.underscoregang.xyz/fullchain.pem: no such file or directory
```

**Confirmed cause:** Wings is configured to use HTTPS, but the configured certificate file does not exist at:

```text
/etc/letsencrypt/live/node.underscoregang.xyz/fullchain.pem
```

Wings successfully contacted the Panel API and fetched the server list before failing. The immediate failure is certificate-file access, not evidence of a Panel API connection problem.

## 2. Collect current status and logs

Run:

```bash
sudo systemctl status wings --no-pager -l
sudo journalctl -u wings -n 100 --no-pager
sudo certbot certificates
sudo ls -la /etc/letsencrypt/live/
```

Check the SSL-related configuration lines without printing tokens or the entire configuration:

```bash
sudo grep -nE '^(api:|  ssl:|    enabled:|    cert:|    key:|remote:)' \
  /etc/pterodactyl/config.yml
```

If the output is unclear, inspect the relevant `api.ssl` section locally. Do not post API tokens, passwords, or private-key contents.

## 3. Confirm DNS and network access

Before requesting a certificate, confirm that `your-node-domain.example` resolves to this server's public IP address. Check your DNS provider and, if useful, run:

```bash
getent ahosts node.underscoregang.xyz
```

Also confirm that the server firewall and any cloud/provider firewall allow the validation traffic required by your chosen Certbot method.

- HTTP-01 validation requires inbound TCP port `80`.
- TLS-ALPN-01 validation requires inbound TCP port `443`.
- DNS-01 validation uses DNS records and can be used when inbound validation ports are unavailable.

Do not assume the domain points to the correct server merely because it exists in DNS.

## 4. Obtain or restore the node certificate

First inspect the output from `sudo certbot certificates`. If a valid certificate already exists under a different certificate name, verify its domains and file paths before changing the Wings configuration.

### Option A: Certbot standalone

Use this only if port `80` is reachable and available for Certbot's temporary web server. If Nginx is serving other sites, stopping it may interrupt those sites. Check Nginx health and dependencies first.

```bash
sudo nginx -t
sudo ss -ltnp | grep -E ':(80|443)\b'
```

If it is safe to stop Nginx temporarily and port `80` is available:

```bash
sudo systemctl stop wings
sudo systemctl stop nginx
sudo certbot certonly --standalone -d node.underscoregang.xyz
```

If Certbot succeeds, check that the expected files exist:

```bash
sudo test -s /etc/letsencrypt/live/node.underscoregang.xyz/fullchain.pem \
  && echo "Certificate file exists" \
  || echo "Certificate file is missing"

sudo test -s /etc/letsencrypt/live/node.underscoregang.xyz/privkey.pem \
  && echo "Private key file exists" \
  || echo "Private key file is missing"
```

If Certbot fails, read its error output and resolve that issue before continuing. Do not create empty certificate files or copy unrelated certificates into these paths.

### Option B: Nginx or DNS validation

If Nginx must remain running, use an appropriate Certbot Nginx/webroot method after fixing Nginx's configuration, or use DNS-01 validation if you can create the required DNS TXT record. Choose one method appropriate to the existing server setup.

**Existing complication:** A previous Nginx configuration may reference a missing Panel certificate file. If `sudo nginx -t` fails, repair the Panel certificate path/configuration before trying to start Nginx. Do not overwrite a certificate path blindly.

## 5. Verify the Wings TLS configuration

Open the configuration locally:

```bash
sudo nano /etc/pterodactyl/config.yml
```

Find the `api.ssl` settings. The configured certificate and key paths must point to existing, valid files for `your-node-domain.example`. The exact YAML structure and paths should match the configuration generated for this node.

Common Let's Encrypt paths are:

```text
/etc/letsencrypt/live/node.underscoregang.xyz/fullchain.pem
/etc/letsencrypt/live/node.underscoregang.xyz/privkey.pem
```

Do not disable TLS as a shortcut on a publicly accessible node. Do not share the full configuration file if it contains credentials.

## 6. Start and verify services

After the certificate exists and Nginx's configuration is valid, start the services that were stopped:

```bash
sudo nginx -t
sudo systemctl start nginx
sudo systemctl restart wings
sudo systemctl status wings --no-pager -l
sudo journalctl -u wings -n 80 --no-pager
```

If Nginx is not installed or was not running before, do not start it solely because it appears in this checklist; adapt the commands to the actual services on the server.

A healthy Wings startup should no longer report the missing `fullchain.pem` file. Check that the node is also shown as online in the Pterodactyl Panel.

## 7. If Wings still fails

Check the **newest** error in the logs, because it may differ from the original certificate error:

```bash
sudo journalctl -u wings -n 120 --no-pager
sudo tail -n 120 /var/log/pterodactyl/wings.log
```

Useful follow-up checks:

- Confirm the certificate and private-key paths in `/etc/pterodactyl/config.yml`.
- Confirm the certificate is valid for `your-node-domain.example`.
- Confirm DNS resolves to the correct server.
- Check for port conflicts on the configured Wings port (the observed log showed port `8080`).
- Verify the Panel's node configuration uses the intended FQDN, scheme, and port.
- If the error changes, troubleshoot the new error rather than repeating certificate commands.

## 8. Commands used during the original troubleshooting

Configuration was run with the Panel URL and node ID:

```bash
cd /etc/pterodactyl
sudo wings configure --panel-url https://panel.underscoregang.xyz \
  --token '<REDACTED_PANEL_APPLICATION_TOKEN>' \
  --node 1
```

The command reported `Successfully configured wings`, but the service then failed at HTTPS certificate initialization. This demonstrates that a successful configuration command does not guarantee that Wings can start.

To restart Wings after resolving the cause:

```bash
sudo systemctl restart wings
sudo systemctl status wings --no-pager -l
```

## 9. Important operational cautions

- Do not paste Panel API tokens, private keys, passwords, or full secret-bearing configuration files into chats or public issue trackers.
- Do not repeatedly restart Wings without checking the latest logs.
- Stopping Nginx can interrupt the Panel or other hosted websites; check impact before doing so.
- Do not delete Let's Encrypt directories or manually edit certificate symlinks as a first-line fix.
- Confirm backups and access to the server console before making network or TLS changes.
