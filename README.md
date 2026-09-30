# tirreno for administrators

System administration guide for tirreno

### About this guide

Welcome to the tirreno administration guide. This document covers installation, configuration, security hardening, updating, and troubleshooting for tirreno deployments.

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

- **PHP** 8.0–8.3 with PDO_PGSQL, pgsql, cURL, mbstring
- **PHP** `memory_limit` of at least 128 MB
- **PostgreSQL** 12+
- **Apache** with mod_rewrite and `.htaccess` support (`AllowOverride All`)
- Read/write permission for the `/config` directory (the installer writes `config/local/config.local.ini`)

Hardware: 512 MB RAM for PostgreSQL (4 GB recommended), ~3 GB storage per 1M events.

### Download

1. Download the latest version of tirreno: [tirreno-master.zip](https://www.tirreno.com/download/)
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

**Note:** `composer.json` pins the platform to PHP 8.1.32, and the development tools (PHPUnit, PHPStan) require PHP 8.1 or later. On PHP 8.0, or on any production server, install without development dependencies:

```bash
composer create-project --no-dev tirreno/tirreno
```

After installation, configure your web server to point to the tirreno directory and run the [web installer](#web-installer).

### Docker installation

**One line Docker:**

`curl -sL tirreno.com/t.yml | docker compose -f - up -d`

**Manual Docker:**

```bash
# Create network
docker network create tirreno-network

# Start PostgreSQL
docker run -d \
  --name tirreno-db \
  --network tirreno-network \
  -e POSTGRES_DB=tirreno \
  -e POSTGRES_USER=tirreno \
  -e POSTGRES_PASSWORD=secret \
  -v ./db:/var/lib/postgresql/data \
  postgres:15

# Start tirreno
docker run -d \
  --name tirreno-app \
  --network tirreno-network \
  -p 8585:80 \
  -v tirreno:/var/www/html \
  tirreno/tirreno:latest
```

**Compose:**

```yaml
services:
  tirreno-app:
    image: tirreno/tirreno:latest
    ports:
      - "8585:80"
    # Optional: overrides SITE from config.local.ini (see Changing the site URL)
    # environment:
    #   SITE: localhost:8585
    volumes:
      - tirreno:/var/www/html
    networks:
      - tirreno-network
    depends_on:
      - tirreno-db

  tirreno-db:
    image: postgres:15
    environment:
      POSTGRES_DB: tirreno
      POSTGRES_USER: tirreno
      POSTGRES_PASSWORD: secret
    volumes:
      - ./db:/var/lib/postgresql/data
    networks:
      - tirreno-network

networks:
  tirreno-network:

volumes:
  tirreno:
```

Run: `docker compose up -d`

Access `http://localhost:8585/install/` and use database URL `postgresql://tirreno:secret@tirreno-db:5432/tirreno`.

### Heroku deployment

Click [here](https://heroku.com/deploy?template=https://github.com/tirrenotechnologies/tirreno) to launch Heroku deployment.

### Web installer

**Prepare the database first** (not needed for [Docker](#docker-installation), where the database user is a superuser):

The schema needs the `citext`, `pgcrypto` and `pg_stat_statements` extensions. `pg_stat_statements` can only be created by a PostgreSQL superuser, so create the extensions as `postgres` before running the installer:

```bash
sudo -u postgres psql -c "CREATE USER tirreno WITH PASSWORD 'secret';"
sudo -u postgres createdb -O tirreno tirreno
sudo -u postgres psql -d tirreno -c "CREATE EXTENSION IF NOT EXISTS citext; CREATE EXTENSION IF NOT EXISTS pgcrypto; CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"
```

Access the installer at `https://your-domain.com/install/` (for [Docker](#docker-installation): `http://localhost:8585/install/`) and provide PostgreSQL database credentials. Open it using the host name you will use for tirreno: the installer saves the current host as `SITE`.

**Input options:**

You can enter credentials in two ways:

1. **Database URL** (recommended for Docker):
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

This allows you to validate credentials without applying the schema. The test only checks that tirreno can connect; it does not check whether the user can create the required extensions.

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
2. Visit `/signup` to create your admin account. `/signup` is available only until the first account exists.

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
| Site host | `SITE` | `SITE` | Host name(s) of the instance, comma-separated, without protocol (e.g., `tirreno.example.com`; include the port if it is not 80/443, e.g., `localhost:8585`) |
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
| Logbook limit | `LOGBOOK_LIMIT` | `LOGBOOK_LIMIT` | `3000` | Number of logbook records kept per API key during logbook rotation |
| Rate limit RPS | `LEAKY_BUCKET_RPS` | `LEAKY_BUCKET_RPS` | `10` | Sensor rate limit: average allowed requests per second, per API key (`0` disables the limit) |
| Rate limit window | `LEAKY_BUCKET_WINDOW` | `LEAKY_BUCKET_WINDOW` | `20` | Sensor rate limit window in seconds: at most `LEAKY_BUCKET_RPS × LEAKY_BUCKET_WINDOW` requests are accepted within the window (`0` disables the limit) |
| Log to stdout | `LOG_TO_STDOUT` | `LOG_TO_STDOUT` | `false` | Also write log messages to stdout (useful for Docker/Heroku) |
| Debug level | `DEBUG` | `DEBUG` | `0` | Error detail level, see [Debug levels](#debug-levels) |
| Config file | `CONFIG_FILE` | — | `local/config.local.ini` | Alternative local config file for the dashboard, relative to `config/`. The sensor always reads `config/local/config.local.ini` |

**Config file only settings (`config/config.ini`):**

| Config Key | Default | Description |
|------------|---------|-------------|
| `SEND_EMAIL` | `1` | Enable email sending |
| `SMTP_DEBUG` | `0` | SMTP debug output |
| `MIN_PASSWORD_LENGTH` | `8` | Minimum password length |
| `PRINT_SQL_LOG_AFTER_EACH_SCRIPT_CALL` | `0` | Write SQL queries to `assets/logs/sql.log` |

#### Debug levels

| Level | Behaviour |
|-------|-----------|
| `0` | Errors with file and line; warnings and info messages are logged |
| `1` | Same as 0, plus debug messages |
| `2` | Errors with stack trace; warnings, info and debug messages |
| `3` | Errors with stack trace including function arguments; warnings, info and debug messages |

The stack trace is shown on the error page only to logged-in operators. Use `0` in production.

### Cronjob setup

tirreno uses a built-in cron system. Jobs are configured in `config/crons.ini` and invoked through a single cron entry:

**System crontab entry (run every 10 minutes):**

```bash
*/10 * * * * /usr/bin/php /absolute/path/to/tirreno/index.php /cron 
```

Add the entry to the crontab of the web server user (e.g., `crontab -u www-data -e`) so file permissions match. The cron endpoint works only from the command line; over HTTP it returns 404.

Each run executes the jobs whose schedule matches the current time. Jobs with `* * * * *` match every minute, so they run on every invocation (every 10 minutes with the entry above). The `0-10` ranges below are chosen so that the jobs match a `*/10` run at minute 0 or 10. Queue handlers keep processing until their queue is empty or a time limit is reached.

**Built-in cron jobs (config/crons.ini):**

| Job | Schedule (cron expression) | Description |
|-----|----------------------------|-------------|
| enrichmentQueueHandler | `* * * * *` | Process IP/email/phone enrichment |
| riskScoreQueueHandler | `* * * * *` | Calculate entity risk scores |
| batchedNewEvents | `* * * * *` | Process new incoming events |
| blacklistQueueHandler | `* * * * *` | Process blacklist updates |
| deletionQueueHandler | `* * * * *` | Handle data deletion requests |
| notificationsHandler | `* * * * *` | Send alert notifications |
| totals | `* * * * *` | Update dashboard statistics |
| logbookRotation | `0-10 * * * *` | Rotate logbook entries |
| retentionPolicyViolations | `0-10 0 * * *` | Check retention policy daily |
| queuesClearer | `0-10 0 * * 2` | Clear stale queues weekly |

**Verify cron is running:**
```bash
# Check cron service
systemctl status cron

# View cron progress (cron writes to the application log)
tail -f assets/logs/error.log

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
chmod 640 config/*.ini config/local/config.local.ini

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

**1. Roles and permissions:**

Since v0.10.0 tirreno uses role-based access control (RBAC). Operators are assigned roles, and roles grant permissions to pages.

| Role | Default permissions |
|------|---------------------|
| `superuser` | All permissions, including `user_admin` (operator administration) |
| `operator` | All page permissions (`page_view`, `page_edit`, `page_delete`, `page_publish`) except `user_admin`, on all pages except `/cron` |
| `guest` | Visitors who are not logged in: `page_view` and `page_edit` on the signup, login, password recovery and error pages only |

Permissions are assigned per role and page (`dshb_roles_permissions`, `dshb_pages_permissions`). The account created at `/signup` (only one account can be created this way) gets the `operator` role.

**2. Operator account security:**
- Use strong, unique passwords
- Give the `superuser` role to as few people as possible; use `operator` for everyone else
- Remove operators who no longer need access
- Review access logs regularly

**3. Session management:**
- Sessions are stored in the database (`KEEP_SESSION_IN_DB = 1` in `config/config.ini`)
- Serve the dashboard over HTTPS only so session cookies are never sent in plain text

### Monitoring and logging

**1. Application log files:**

tirreno writes logs to the `assets/logs/` directory:

| Log file | Description |
|----------|-------------|
| `error.log` | Application errors and exceptions |
| `blacklist.log` | Blacklist events — records when entities are automatically blacklisted by rules |
| `sql.log` | SQL queries (disabled by default, enable with `PRINT_SQL_LOG_AFTER_EACH_SCRIPT_CALL = 1`) |

For Docker/Heroku deployments, set `LOG_TO_STDOUT = true` to also send log messages to the container output.

Monitor blacklist.log to track automatic fraud detection (the file is created with the first automatic blacklisting):
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

When moving tirreno to a new domain or URL, update the `SITE` configuration. `SITE` holds host names only, without `https://`:

**Option 1: Configuration file**

Edit `config/local/config.local.ini`:
```ini
[globals]
SITE = new-domain.com
```

**Option 2: Environment variable**

Set the `SITE` environment variable. For Apache, set it in the virtual host; a variable exported in your shell does not reach the running web server:
```apache
SetEnv SITE new-domain.com
```

The cron job runs from the command line and does not see `SetEnv`, so keep `SITE` in `config/local/config.local.ini` up to date as well.

For Docker deployments, uncomment and update the `environment` block of `tirreno-app` in `docker-compose.yml`:
```yaml
environment:
  SITE: new-domain.com
```

**Multiple domains:**

tirreno supports multiple domains (comma-separated). Redirects, such as the one to the login page, always go to the first domain:
```ini
SITE = primary.com,secondary.com
```

After changing the URL:
1. Clear any cached sessions
2. Update your application's tracker endpoint to point to the new URL
3. Verify the Logbook receives events at the new location

### Database migration

**Exporting the database:**

```bash
# Connect over TCP (-h 127.0.0.1) so the tirreno password is used;
# local socket connections use peer authentication and fail for other OS users

# Full database backup
pg_dump -h 127.0.0.1 -U tirreno -d tirreno -F c -f tirreno_backup.dump

# Schema only
pg_dump -h 127.0.0.1 -U tirreno -d tirreno --schema-only -f tirreno_schema.sql

# Data only
pg_dump -h 127.0.0.1 -U tirreno -d tirreno --data-only -f tirreno_data.sql
```

**Importing to new server:**

```bash
# Create the user, database and extensions on the new server (as the PostgreSQL superuser)
sudo -u postgres psql -c "CREATE USER tirreno WITH PASSWORD 'secret';"
sudo -u postgres createdb -O tirreno tirreno
sudo -u postgres psql -d tirreno -c "CREATE EXTENSION IF NOT EXISTS citext; CREATE EXTENSION IF NOT EXISTS pgcrypto; CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"

# Restore from backup (--no-comments skips extension comments the tirreno user may not change)
pg_restore --no-comments -h 127.0.0.1 -U tirreno -d tirreno tirreno_backup.dump
```

Use the full backup to move tirreno. Loading `tirreno_data.sql` into a database that already has the schema fails for the `queue_new_events_cursor` row, because the table's trigger runs during the restore.

**Verify migration:**

```bash
# Check table counts
psql -h 127.0.0.1 -U tirreno -d tirreno -c "SELECT COUNT(*) FROM event_account;"
psql -h 127.0.0.1 -U tirreno -d tirreno -c "SELECT COUNT(*) FROM event;"
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
git pull origin master
```

Run Git as the user that owns the files, for example `sudo -u www-data git pull origin master`. Run as root, Git stops with "detected dubious ownership".

If you have local changes, stash them first:

```bash
git stash
git pull origin master
git stash pop
```

### Composer

```bash
composer update tirreno/tirreno
```

This updates tirreno when it was added to a project with `composer require`. In a `composer create-project` installation tirreno is the project itself, so Composer reports "Package tirreno/tirreno listed for update is not locked" and changes nothing; update such an installation with Git or a new `create-project`.

### Docker

```bash
docker pull tirreno/tirreno:latest
docker compose down
docker compose up -d
```

### After updating

1. Remove the installation folder: `rm -rf /path/to/tirreno/install`
2. Clear the application cache in `tmp/` if applicable
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
| "Permission denied" | Grant permissions: `GRANT ALL ON DATABASE tirreno TO tirreno` |
| "PDO PostgreSQL driver" | Install: `apt install php-pgsql` (provides both `pdo_pgsql` and `pgsql`) |
| "Config folder permission" | `chmod 755 config && chown www-data:www-data config` |
| "Memory limit" | Set `memory_limit = 128M` in php.ini |
| "Apply database schema (… permission denied to create extension "pg_stat_statements")" | The database user is not a superuser. Create the extensions as `postgres` (see [Web installer](#web-installer)), then remove the installer lock (next row) and click **Connect** again |
| "Database already locked by another installation process" | A previous installation attempt failed and left its lock. If no other installation is running, remove it: `sudo -u postgres psql -d tirreno -c "DROP TABLE IF EXISTS dshb_install_flag;"` |
| Invalid hostname (TN8001) | The application was accessed using a hostname that doesn't match the configured allowed host(s). This is a security measure to prevent host header attacks. The user must access the application through the correct URL defined in the configuration. |
| Failed DB connect (TN8002) | The application cannot establish a connection to the PostgreSQL database. This could be caused by incorrect database credentials, the database server being down, network issues, or misconfigured connection parameters in the config file. |
| Incomplete config (TN8003) | The application's configuration file (config/local/config.local.ini) is missing required settings or environment variable overrides are not properly set. The application cannot start without complete configuration. |


**PHP version check:**

tirreno requires PHP 8.0–8.3. Verify your PHP version:

```bash
# Check PHP CLI version
php -v

# The web server may use a different PHP version than the CLI;
# the installer's Compatibility step checks the web server's PHP

# Check all required extensions
php -m | grep -E "pdo_pgsql|pgsql|curl|mbstring"
```

If using multiple PHP versions, ensure Apache uses the correct one:
```bash
# Check PHP module loaded by Apache
apachectl -M | grep php
```

### Sending data issues

**Note:** The API uses form-urlencoded format, not JSON.

For required and optional parameters, event types and response codes, see the [Sensor API reference](https://github.com/tirrenotechnologies/DEVELOPMENT.md#sensor-api-reference).

**Events not appearing in tirreno:**

1. **Check Tracking ID:** Ensure the value you send in the `Api-Key` header matches the **Tracking ID** on the **API** page of tirreno
2. **Verify endpoint:** Confirm you're posting to `/sensor/` (with trailing slash)
3. **Check Logbook:** Look for failed requests in the Logbook page. Requests with a missing or unknown Tracking ID do not appear in the Logbook; check the web server error log instead (see below)
4. **Test with curl:**
```bash
curl -v -X POST https://your-tirreno.com/sensor/ \
  -H "Api-Key: your-tracking-id" \
  -d "userName=test" \
  -d "ipAddress=1.2.3.4" \
  -d "url=/test" \
  -d "eventTime=2024-12-08 01:01:00.000" \
  -d "eventType=page_view"
```

**Trace a request:** add `-H "X-Request-Id: test-0001"` to the curl command above. The ID (up to 36 characters) is stored in the database with the event (`event.traceid`), or with the rejected request if validation fails (`event_incorrect.traceid`), so you can match it with your application logs. It is not shown in the dashboard or the Logbook.

**Test from the server command line** (bypasses the web server, useful to rule out Apache or network issues):

```bash
php sensor/index.php --apiKey=your-tracking-id --userName=test --ipAddress=1.2.3.4 \
    --url=/test --eventTime="2024-12-08 01:01:00.000" --eventType=page_view
```
No output means success, errors are printed to stderr (exit code is always 0).

**Check response codes:** see [Response codes](https://github.com/tirrenotechnologies/DEVELOPMENT.md#response-codes). A `200` with an empty body means the request was rejected; the reason is in the web server error log.

**Rejection reasons** are written to the web server error log (e.g., `/var/log/apache2/error.log`):
```
Error 401: Api-Key header is not set
Error 401: API key from the "Api-Key" header is not found
Error 400: Validation error: "Required field is missing or empty" for key "ipAddress"
```

Fields with invalid values are corrected instead of rejected and logged as **Success with warnings** (see [Required parameters](https://github.com/tirrenotechnologies/DEVELOPMENT.md#required-parameters)).

**Rate limit exceeded (429):** raise `LEAKY_BUCKET_RPS` / `LEAKY_BUCKET_WINDOW` (see [Environment variables](#environment-variables)) or reduce traffic. How the limiter works is described in [Rate limiting](https://github.com/tirrenotechnologies/DEVELOPMENT.md#rate-limiting).

### Logbook review

The Logbook columns and status types are described in [Logbook](https://github.com/tirrenotechnologies/OPERATOR.md#logbook) in the user guide.

**What to look for:**
- Request failed: missing required parameters, or a server error
- Success with warnings: invalid values that were corrected (e.g., invalid `eventTime`)
- Rate limit exceeded: raise `LEAKY_BUCKET_RPS` / `LEAKY_BUCKET_WINDOW` or reduce traffic
- No entries at all: Tracking ID issues (these requests are not logged in the Logbook; check the web server error log)
- Gaps in traffic: Network or integration issues
- Unexpected IPs: Verify your application servers

### Cronjob issues

**Manual cron:**

`php /home/user/tirreno/index.php /cron`

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
| Resource center | [tirreno.com/bat](https://www.tirreno.com/bat/) |
| Developers Guide | [github.com/tirrenotechnologies/DEVELOPMENT.md](https://github.com/tirrenotechnologies/DEVELOPMENT.md) |
| Administrator guide | [github.com/tirrenotechnologies/ADMIN.md](https://github.com/tirrenotechnologies/ADMIN.md) |
| Operator guide | [github.com/tirrenotechnologies/OPERATOR.md](https://github.com/tirrenotechnologies/OPERATOR.md) |
| API reference | [github.com/tirrenotechnologies/API.md](https://github.com/tirrenotechnologies/API.md) |
| GitHub | [github.com/tirrenotechnologies/tirreno](https://github.com/tirrenotechnologies/tirreno) |
| GitLab Mirror | [gitlab.com/tirreno/tirreno](https://gitlab.com/tirreno/tirreno) |
| Docker Hub | [hub.docker.com/r/tirreno/tirreno](https://hub.docker.com/r/tirreno/tirreno) |
| Docker Repo | [github.com/tirrenotechnologies/docker](https://github.com/tirrenotechnologies/docker) |
| Packagist | [packagist.org/packages/tirreno/tirreno](https://packagist.org/packages/tirreno/tirreno) |
| PHP Tracker | [github.com/tirrenotechnologies/tirreno-php-tracker](https://github.com/tirrenotechnologies/tirreno-php-tracker) |
| Python Tracker | [github.com/tirrenotechnologies/tirreno-python-tracker](https://github.com/tirrenotechnologies/tirreno-python-tracker) |
| Node.js Tracker | [github.com/tirrenotechnologies/tirreno-nodejs-tracker](https://github.com/tirrenotechnologies/tirreno-nodejs-tracker) |
| WordPress Tracker | [github.com/tirrenotechnologies/tirreno-wordpress-tracker](https://github.com/tirrenotechnologies/tirreno-wordpress-tracker) |
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
