# tirreno for administrators

System administration guide for tirreno

---

## About this guide

Welcome to the tirreno administration guide. This document covers installation, configuration, security hardening, updating, and troubleshooting for tirreno deployments.

tirreno is available in three editions:

- **Community Edition** (open-source): For developer and product teams that want to add a security layer to self-hosted applications. Get started today without getting into complex business relationships. Licensed under GNU Affero General Public License v3 (AGPL-3.0).

- **Application Edition**: Protects your organization's internal applications from account threats, ensures audit trails and field history for compliance, detects data exfiltration and insider threats.

- **Platform Edition**: Built for client portals, SaaS, public sector portals, and digital platforms. Multi-application support, fraud and abuse prevention, and dedicated assistance for your SOC, product, and risk teams.

For Application and Platform editions, contact team@tirreno.com.

```
     Community ──────────────► Application ────────────► Platform
       Edition                    Edition                  Edition
         │                         │                         │
         ▼                         ▼                         ▼
    Personal apps             Internal apps            External-facing
    + Basic security          + Compliance             + Multi-app
    + Development             + Audit trails           + Fraud/abuse
    + No support              + Insider threats        + Dedicated support
```

### Target audience

This guide is for system administrators who want to install, configure, secure, update, and maintain tirreno deployments.

---

## Table of contents

1. [Installation](#installation)
   - [System requirements](#system-requirements)
   - [Download](#download)
   - [Git clone](#git-clone)
   - [Composer installation](#composer-installation)
   - [Docker installation](#docker-installation)
   - [Heroku deployment](#heroku-deployment)
   - [Web installer](#web-installer)
   - [Post-installation steps](#post-installation-steps)

2. [Configuration](#configuration)
   - [Database connection](#database-connection)
   - [Environment variables](#environment-variables)
   - [Cronjob setup](#cronjob-setup)

3. [Security](#security)
   - [Post-installation security](#post-installation-security)
   - [Network security](#network-security)
   - [Access control](#access-control)
   - [Monitoring and logging](#monitoring-and-logging)
   - [Security checklist](#security-checklist)

4. [Migration](#migration)
   - [Changing the site URL](#changing-the-site-url)
   - [Database migration](#database-migration)

5. [Updating](#updating)
   - [Check for updates](#check-for-updates)
   - [Before updating](#before-updating)
   - [Git](#git)
   - [Composer](#composer)
   - [Docker](#docker)
   - [After updating](#after-updating)

6. [Troubleshooting](#troubleshooting)
   - [Installation issues](#installation-issues)
   - [Sending data issues](#sending-data-issues)
   - [Logbook review](#logbook-review)
   - [Cronjob issues](#cronjob-issues)

7. [Resources](#resources)

8. [Found a mistake?](#found-a-mistake)

---

## Installation

### System requirements

- **PHP** 8.0–8.3 with PDO_PGSQL, cURL, mbstring
- **PostgreSQL** 12+
- **Apache** with mod_rewrite

Hardware: 512 MB RAM for PostgreSQL (4 GB recommended), ~3 GB storage per 1M events.

### Download

1. Download the latest version of tirreno: [tirreno-master.zip](https://www.tirreno.com/download.php)
2. Extract the ZIP file to the location where you want it installed on your web server
3. Configure your web server to point to the tirreno directory
4. Run the [web installer](#web-installer)

### Git clone

```bash
git clone https://github.com/tirrenotechnologies/tirreno.git
cd tirreno
```

After cloning, configure your web server to point to the tirreno directory and run the [web installer](#web-installer).

### Composer installation

tirreno is published on Packagist and can be installed with Composer.

**Create a new project:**

```bash
composer create-project tirreno/tirreno
```

**Or add to an existing project:**

```bash
composer require tirreno/tirreno
```

After installation, configure your web server to point to the tirreno directory and run the [web installer](#web-installer).

### Docker installation

```bash
# Create network
docker network create tirreno-network

# Start PostgreSQL
docker run -d --name tirreno-db \
  --network tirreno-network \
  -e POSTGRES_DB=tirreno \
  -e POSTGRES_USER=tirreno \
  -e POSTGRES_PASSWORD=secret \
  -v ./db:/var/lib/postgresql/data \
  postgres:15

# Start tirreno
docker run --name tirreno-app \
  --network tirreno-network \
  -p 8585:80 \
  -v tirreno:/var/www/html \
  -d tirreno/tirreno:latest
```

### Heroku deployment

Click [here](https://heroku.com/deploy?template=https://github.com/tirrenotechnologies/tirreno) to launch Heroku deployment.

### Web installer

Access the installer at `https://your-domain.com/install/` (for [Docker](#docker-installation): `http://localhost:8585/install/`) and provide PostgreSQL database credentials.

**Input options:**

You can enter credentials in two ways:

1. **Database URL** (recommended for Docker/Heroku):
   ```
   postgresql://tirreno:secret@tirreno-db:5432/tirreno
   ```
   The URL will be automatically parsed into individual fields.

2. **Individual fields:**
   - Database username
   - Database password
   - Database host
   - Database port
   - Database name
   - Admin email (optional)

**Testing the database connection:**

Before running the full installation, click the **Test** button to verify your database connection:
- **Green** button = connection successful
- **Red** button = connection failed (check credentials)

This allows you to validate credentials without applying the schema.

**Installation steps:**

When you click **Connect**, the installer runs these steps:

| Step | Checks |
|------|--------|
| Version check | Verifies you have the latest tirreno version |
| Compatibility | PHP version (8.0–8.3), mod_rewrite, PDO PostgreSQL, config folder permissions, .htaccess, cURL, memory limit (128MB) |
| Database params | Validates all required fields are provided |
| Database setup | Tests connection, checks for existing installation, applies schema |
| Config build | Writes `config/local/config.local.ini` with your settings |

**After successful installation:**
1. Delete the `/install` directory
2. Visit `/signup` to create your admin account

### Post-installation steps

1. Delete the `/install` directory
2. Create your admin account at `/signup`
3. Configure the cronjob (see [Cronjob setup](#cronjob-setup))
4. Apply security hardening (see [Security](#security))

---

## Configuration

### Database connection

After installation, configuration is stored in `config/local/config.local.ini`.

### Environment variables

tirreno can be configured via environment variables or config file settings. Environment variables take precedence over config file settings and are useful for Docker/Heroku deployments.

**Required settings:**

| Setting | Env Variable | Config Key | Description |
|---------|--------------|------------|-------------|
| Database | `DATABASE_URL` | `DATABASE_URL` | PostgreSQL connection string (e.g., `postgres://user:pass@host:5432/dbname`) |
| Site URL | `SITE` | `SITE` | Base URL for the application (e.g., `https://tirreno.example.com`) |
| Pepper | `PEPPER` | `PEPPER` | Password pepper for secure hashing (random string, keep secret) |

**Optional settings:**

| Setting | Env Variable | Config Key | Default | Description |
|---------|--------------|------------|---------|-------------|
| Admin email | `ADMIN_EMAIL` | `ADMIN_EMAIL` | — | Administrator email address |
| SMTP login | `MAIL_LOGIN` | `MAIL_LOGIN` | — | SMTP username for sending emails |
| SMTP password | `MAIL_PASS` | `MAIL_PASS` | — | SMTP password |
| Enrichment API | `ENRICHMENT_API` | `ENRICHMENT_API` | `https://api.tirreno.com` | Enrichment API endpoint |
| Force HTTPS | `FORCE_HTTPS` | `FORCE_HTTPS` | `false` | Force HTTPS redirects |
| Forgot password | `ALLOW_FORGOT_PASSWORD` | `ALLOW_FORGOT_PASSWORD` | `false` | Enable forgot password feature |
| Show email/phone | `ALLOW_EMAIL_PHONE` | `ALLOW_EMAIL_PHONE` | `false` | Enable email/phone display |
| Logbook limit | `LOGBOOK_LIMIT` | `LOGBOOK_LIMIT` | `3000` | Maximum logbook entries to retain |

**Config file only settings (`config/config.ini`):**

| Config Key | Default | Description |
|------------|---------|-------------|
| `DEBUG` | `0` | Debug mode (0=off, 1-3=verbosity levels) |
| `SEND_EMAIL` | `1` | Enable email sending |
| `SMTP_DEBUG` | `0` | SMTP debug output |
| `MIN_PASSWORD_LENGTH` | `8` | Minimum password length |

### Cronjob setup

tirreno uses a built-in cron system. Jobs are configured in `config/crons.ini` and invoked through a single cron entry:

**System crontab entry (run every 10 minutes):**

```bash
*/10 * * * * /usr/bin/php /absolute/path/to/tirreno/index.php /cron
```

**Built-in cron jobs (config/crons.ini):**

| Job | Schedule | Description |
|-----|----------|-------------|
| enrichmentQueueHandler | Every minute | Process IP/email/phone enrichment |
| riskScoreQueueHandler | Every minute | Calculate user risk scores |
| batchedNewEvents | Every minute | Process new incoming events |
| blacklistQueueHandler | Every minute | Process blacklist updates |
| deletionQueueHandler | Every minute | Handle data deletion requests |
| notificationsHandler | Every minute | Send alert notifications |
| totals | Every minute | Update dashboard statistics |
| logbookRotation | 0-10 * * * * | Rotate logbook entries |
| retentionPolicyViolations | 0-10 0 * * * | Check retention policy daily |
| queuesClearer | 0-10 0 * * 2 | Clear stale queues weekly |

**Verify cron is running:**
```bash
# Check cron service
systemctl status cron

# View tirreno cron logs
tail -f /var/log/tirreno-cron.log

# Run cron manually to test
/usr/bin/php /absolute/path/to/tirreno/index.php /cron
```

---

## Security

### Post-installation security

**1. Remove the install directory:**
```bash
rm -rf /path/to/tirreno/install/
```
The install directory contains setup scripts that could be exploited if left accessible.

**2. Set proper file permissions:**
```bash
# Restrict config directory
chmod 750 config/
chmod 640 config/*.ini

# Restrict sensitive files
chmod 640 composer.json composer.lock
chmod 640 .htaccess

# Ensure logs are not world-readable
chmod 750 assets/logs/

# Make rules directories writable only by web server
chown -R www-data:www-data assets/rules/
chmod 755 assets/rules/core/ assets/rules/custom/
```

**3. Verify .htaccess protection:**
Ensure your Apache configuration allows `.htaccess` overrides:
```apache
<Directory /path/to/tirreno>
    AllowOverride All
    Require all granted
</Directory>
```

**4. Verify settings file is inaccessible:**
```bash
# Should return 403 Forbidden or 404 Not Found
curl -I https://your-tirreno.com/config/local/config.local.ini
```

### Network security

- Use HTTPS with valid SSL/TLS certificates (Let's Encrypt or commercial CA)
- Redirect all HTTP traffic to HTTPS and use HSTS headers
- Place tirreno in a private subnet; database should not be directly accessible from the internet
- For internal deployments, restrict sensor access to known IPs:

```apache
<Location /sensor/>
    Require ip 10.0.0.0/8
    Require ip 192.168.0.0/16
</Location>
```

### Access control

**1. Admin account security:**
- Use strong, unique passwords
- Limit the number of admin accounts
- Review access logs regularly

**2. Session management:**
- Sessions expire after inactivity (default: 30 minutes)
- Sessions invalidated on password change
- Secure session cookies (HttpOnly, Secure, SameSite)

### Monitoring and logging

**1. Application log files:**

tirreno writes logs to the `assets/logs/` directory:

| Log file | Description |
|----------|-------------|
| `error.log` | Application errors and exceptions |
| `blacklist.log` | Blacklist events — records when users are automatically blacklisted by rules |
| `sql.log` | SQL queries (disabled by default, enable with `PRINT_SQL_LOG_AFTER_EACH_SCRIPT_CALL = 1`) |

Monitor blacklist.log to track automatic fraud detection:
```bash
tail -f assets/logs/blacklist.log
```

**2. Logbook (UI):**
- Monitor the Logbook page for API request patterns
- Check failed events, verify your security settings
- Review error rates and unusual activity

**3. Monitor for suspicious activity:**
- Failed login attempts (brute force detection)
- Unusual API request patterns
- Error rate spikes
- Database query anomalies

**4. Log retention:**
- Retain logs for compliance requirements (typically 90 days to 1 year)
- Secure log storage (separate from application)
- Regular log review and alerting

### Security checklist

Use this checklist for production deployments:

- [ ] Install directory removed
- [ ] File permissions restricted
- [ ] Settings file inaccessible from web
- [ ] Database user has minimal privileges
- [ ] HTTPS enforced with valid certificate
- [ ] Admin passwords strong and unique
- [ ] Logging enabled and monitored
- [ ] Error messages don't expose sensitive information

---

## Migration

### Changing the site URL

When moving tirreno to a new domain or URL, update the `SITE` configuration:

**Option 1: Configuration file**

Edit `config/local/config.local.ini`:
```ini
[globals]
SITE = https://new-domain.com
```

**Option 2: Environment variable**

Set the `SITE` environment variable:
```bash
export SITE=https://new-domain.com
```

For Docker deployments, update the environment in `docker-compose.yml`:
```yaml
environment:
  - SITE=https://new-domain.com
```

**Multiple domains:**

tirreno supports multiple domains (comma-separated):
```ini
SITE = https://primary.com,https://secondary.com
```

After changing the URL:
1. Clear any cached sessions
2. Update your application's tracker endpoint to point to the new URL
3. Verify the Logbook receives events at the new location

### Database migration

**Exporting the database:**

```bash
# Full database backup
pg_dump -U tirreno_app -d tirreno -F c -f tirreno_backup.dump

# Schema only
pg_dump -U tirreno_app -d tirreno --schema-only -f tirreno_schema.sql

# Data only
pg_dump -U tirreno_app -d tirreno --data-only -f tirreno_data.sql
```

**Importing to new server:**

```bash
# Create database on new server
createdb -U postgres tirreno

# Restore from backup
pg_restore -U tirreno_app -d tirreno tirreno_backup.dump

# Or restore from SQL
psql -U tirreno_app -d tirreno < tirreno_schema.sql
psql -U tirreno_app -d tirreno < tirreno_data.sql
```

**Verify migration:**

```bash
# Check table counts
psql -U tirreno_app -d tirreno -c "SELECT COUNT(*) FROM event_account;"
psql -U tirreno_app -d tirreno -c "SELECT COUNT(*) FROM event;"
```

---

## Updating

### Check for updates

To check if a new version is available:

1. Go to **Settings** in the left menu
2. Find the **Check for updates** section showing your current version
3. Click **Check** to see if updates are available

tirreno periodically releases updates with new features and security patches.

### Before updating

1. **Backup your database** before any update
2. **Check the changelog** for breaking changes at [github.com/tirrenotechnologies/tirreno/releases](https://github.com/tirrenotechnologies/tirreno/releases)
3. **Test in staging** if possible before updating production

### Git

```bash
cd /path/to/tirreno
git fetch origin
git pull origin main
```

If you have local changes, stash them first:

```bash
git stash
git pull origin main
git stash pop
```

### Composer

```bash
composer update tirreno/tirreno
```

### Docker

```bash
docker pull tirreno/tirreno:latest
docker-compose down
docker-compose up -d
```

### After updating

1. Remove the installation folder: `rm -rf /path/to/tirreno/install`
2. Clear any application cache if applicable
3. Run database migrations if required (check release notes)
4. Verify the application is working correctly
5. Check the Logbook for any errors

---

## Troubleshooting

### Installation issues

**Common installation errors:**

| Error | Solution |
|-------|----------|
| "Connection refused" | PostgreSQL not running or wrong port |
| "Authentication failed" | Wrong database credentials |
| "Database does not exist" | Create database first: `createdb tirreno` |
| "Permission denied" | Grant permissions: `GRANT ALL ON DATABASE tirreno TO tirreno_app` |
| "PDO PostgreSQL driver" | Install: `apt install php-pgsql` |
| "Config folder permission" | `chmod 755 config && chown www-data:www-data config` |
| "Memory limit" | Set `memory_limit = 128M` in php.ini |

**PHP version check:**

tirreno requires PHP 8.0–8.3. Verify your PHP version:

```bash
# Check PHP CLI version
php -v

# Check PHP version used by web server
php -r "echo PHP_VERSION;"

# Check all required extensions
php -m | grep -E "pdo_pgsql|curl|json|mbstring"
```

If using multiple PHP versions, ensure Apache/Nginx uses the correct one:
```bash
# Check PHP module loaded by Apache
apachectl -M | grep php

# For PHP-FPM, check the socket path in your nginx/apache config
ls -la /run/php/
```

### Sending data issues

**Note:** The API uses form-urlencoded format, not JSON.

**Events not appearing in tirreno:**

1. **Check API key:** Ensure your API key matches the one in tirreno settings
2. **Verify endpoint:** Confirm you're posting to `/sensor/` (with trailing slash)
3. **Check Logbook:** Look for failed requests in the Logbook page
4. **Test with curl:**
```bash
curl -v -X POST https://your-tirreno.com/sensor/ \
  -H "Api-Key: your-api-key" \
  -d "userName=test" \
  -d "ipAddress=1.2.3.4" \
  -d "url=/test" \
  -d "eventTime=2024-12-08 01:01:00.000" \
  -d "eventType=page_view"
```

**Check response codes:**

| Code | Meaning |
|------|---------|
| 200 | Success |
| 401 | Invalid or missing API key |
| 400 | Missing required parameters |
| 429 | Rate limited |
| 500 | Server error |

### Logbook review

The Logbook shows all incoming API requests:

1. **Access:** Navigate to Logbook in the left menu
2. **Columns:**
   - Source IP: Where the request came from
   - Timestamp: When the request was received
   - Endpoint: Which API endpoint was called
   - Status: HTTP response code
3. **Filtering:** Use the search box to filter by any column
4. **Chart:** Shows request volume over time

**What to look for:**
- 401 errors: API key issues
- 400 errors: Missing or invalid parameters
- Gaps in traffic: Network or integration issues
- Unexpected IPs: Verify your application servers

### Cronjob issues

**Common cron issues:**

| Issue | Solution |
|-------|----------|
| Jobs not running | Check system cron is enabled: `systemctl enable cron` |
| "No jobs to run" | Normal if no jobs scheduled for current minute |
| Permission denied | Ensure www-data can read config files |
| Database errors | Check config/local/config.local.ini is readable |

---

## Resources

| Resource | URL |
|----------|-----|
| Live Demo | [play.tirreno.com](https://play.tirreno.com) (admin/tirreno) |
| Documentation | [docs.tirreno.com](https://docs.tirreno.com) |
| Developers Guide | [github.com/tirrenotechnologies/DEVELOPMENT.md](https://github.com/tirrenotechnologies/DEVELOPMENT.md) |
| GitHub | [github.com/tirrenotechnologies/tirreno](https://github.com/tirrenotechnologies/tirreno) |
| GitLab Mirror | [gitlab.com/tirreno/tirreno](https://gitlab.com/tirreno/tirreno) |
| Docker Hub | [hub.docker.com/r/tirreno/tirreno](https://hub.docker.com/r/tirreno/tirreno) |
| Docker Repo | [github.com/tirrenotechnologies/docker](https://github.com/tirrenotechnologies/docker) |
| PHP Tracker | [github.com/tirrenotechnologies/tirreno-php-tracker](https://github.com/tirrenotechnologies/tirreno-php-tracker) |
| Python Tracker | [github.com/tirrenotechnologies/tirreno-python-tracker](https://github.com/tirrenotechnologies/tirreno-python-tracker) |
| Node.js Tracker | [github.com/tirrenotechnologies/tirreno-nodejs-tracker](https://github.com/tirrenotechnologies/tirreno-nodejs-tracker) |
| Community Chat | [chat.tirreno.com](https://chat.tirreno.com) |
| Support Email | ping@tirreno.com |
| Security Email | security@tirreno.com |

---

## Found a mistake?

If you have found a mistake in the documentation, no matter how large or small, please let us know by [creating a new issue](https://github.com/tirrenotechnologies/tirreno/issues) in the tirreno repository.

---

## License

tirreno and this documentation are licensed under the **GNU Affero General Public License v3 (AGPL-3.0)**.

The name "tirreno" is a registered trademark of tirreno technologies sàrl.

---

*tirreno Copyright (C) 2026 tirreno technologies sàrl, Vaud, Switzerland.*

't'
