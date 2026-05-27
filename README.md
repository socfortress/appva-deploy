# AppVA — Application Vulnerability Assessment

AppVA is a containerized SBOM-based vulnerability scanning platform built for SOC teams. It generates Software Bills of Materials (SBOM) via [Syft](https://github.com/anchore/syft) and scans them for known CVEs via [Grype](https://github.com/anchore/grype), presenting findings through a secure web dashboard with PDF reporting, a remediation tracker, scheduled scans, branch-change monitoring, external notification support (Teams, Slack, Webhook), and API service accounts for automation.

Built by [SOCFortress](https://www.socfortress.co).

---

## Requirements

- Docker Engine 24+
- Docker Compose v2+
- Outbound internet access (for pulling the image and Grype vulnerability database updates)

---

## Quick Start

**1. Clone this repository**

```bash
git clone https://github.com/socfortress/appva-deploy.git
cd appva-deploy
```

**2. Set your admin password**

```bash
export APPVA_ADMIN_PASSWORD=your_secure_password
```

If this variable is not set, a random password is generated and printed to the container logs on first launch.

**3. Start the container**

```bash
docker compose up -d
```

**4. Access the web UI**

Open `https://<your-host>:8443` in your browser. Accept the self-signed certificate warning on first access (see [TLS](#tls) below for custom certificate setup).

Log in with username `admin` and the password you set above.

**5. Complete first-login setup**

On first login you will be prompted to:
1. Change your password
2. Enrol in TOTP two-factor authentication (use any authenticator app — Google Authenticator, Authy, 1Password, etc.)

---

## Configuration

All runtime configuration is managed from the **Settings** page inside the web UI (admin access required). No config file edits are needed after initial deployment.

| Setting | Description |
|---|---|
| GitHub Default Token | Personal Access Token (PAT) used as fallback for private GitHub repositories |
| Syft Binary Path | Path to the Syft binary inside the container (default: `syft`) |
| Grype Binary Path | Path to the Grype binary inside the container (default: `grype`) |
| Grype DB Auto-Update | Whether Grype should update its vulnerability database before each scan |

---

## Data Persistence

All data is stored in the `appva-data` Docker volume mounted at `/data` inside the container:

| Path | Contents |
|---|---|
| `/data/web.db` | SQLite database (users, repositories, scan history, settings) |
| `/data/results/` | SBOM and Grype JSON output files |
| `/data/repos/` | Cloned GitHub repositories |
| `/data/uploads/` | Extracted ZIP source archives |
| `/data/docker-images/` | Docker tar archives |
| `/data/tls/` | TLS certificate and key |
| `/data/config.yml` | YAML configuration file |

---

## TLS

AppVA generates a self-signed RSA certificate on first launch. To use your own certificate:

1. Go to **Settings → TLS Certificate**
2. Upload your `.crt` (certificate), `.key` (private key), and optionally a CA bundle
3. Restart the container: `docker compose restart`

---

## Licensing

AppVA operates in a **free tier** by default, which supports one repository with full functionality (scanning, PDF reports, remediation tracking, scheduler, notification channels).

To unlock **unlimited repositories**, import a valid license key:

1. Go to **Settings → License**
2. Enter your license key in the format `XXXXX-XXXXX-XXXXX-XXXXX`
3. Click **Activate License**

The license is verified online against the SOCFortress licensing server and re-validated automatically every 24 hours. The container requires outbound HTTPS access to `app.cryptolens.io` for license verification.

To obtain a license key, contact [SOCFortress](https://www.socfortress.co).

---

## Branch Watcher

AppVA can monitor a GitHub repository's branch for new commits and alert you via the in-app notification bell — no webhook setup required on the GitHub side.

**How it works:** AppVA periodically runs `git ls-remote` against the remote URL and compares the returned HEAD commit SHA to the last known value. Only a single network round-trip is needed per check — no clone or fetch is performed.

**To enable for a repository:**

1. Open the repository detail page (**Repositories** → click the repo name)
2. In the **Branch Watcher** section, click **Enable Branch Watcher**
3. Choose a check interval (15 min – daily) and optionally enable **Auto-scan on change**
4. Click **Enable**

| Option | Description |
|---|---|
| Check interval | How often to poll the remote branch (15 min, 30 min, 1 h, 2 h, 4 h, or daily) |
| Auto-scan on change | Automatically trigger a full scan when new commits are detected |

When a branch change is detected:
- A **blue** notification appears in the bell (top-right) with the commit change details
- If auto-scan is disabled, the notification prompts you to trigger a scan manually
- If auto-scan is enabled but another scan is already running, the scan is skipped for that cycle and noted in the notification

The watcher uses the repository's per-repo GitHub token if configured, otherwise falls back to the global default token in Settings.

---

## Service Accounts

Service accounts provide static API keys for external automation tools, CI/CD pipelines, and integrations. Unlike human user accounts, service accounts require no password or 2FA — they authenticate with a single bearer token.

Only **admin** users can create, rotate, or delete service accounts (**Service Accounts** in the sidebar).

### Access levels

| Level | Equivalent to | Capabilities |
|---|---|---|
| `read-only` | `read_only` role | View dashboard, results, scan history, repositories |
| `read-write` | `operator` role | Trigger scans, manage repositories, schedules, branch watchers |

### Creating a service account

1. Navigate to **Service Accounts** (admin only)
2. Click **New Account**, enter a name, choose access level, and optionally set an expiry date
3. Click **Create & Generate Token**
4. **Copy the token immediately** — it is shown exactly once and cannot be retrieved afterwards

### Using the token

Pass the token in the `Authorization` header of every request:

```bash
# Trigger a scan (read-write account)
curl -sk -X POST https://<host>:8443/scan/trigger \
  -H "Authorization: Bearer appva_<your-token>" \
  -d "repo_id=1"

# Fetch scan status (any access level)
curl -sk https://<host>:8443/scan/status \
  -H "Authorization: Bearer appva_<your-token>"

# List repositories (any access level)
curl -sk https://<host>:8443/repositories \
  -H "Authorization: Bearer appva_<your-token>"
```

An invalid or expired token returns `HTTP 401` with a JSON error — never a redirect to the login page.

### Token security

- Tokens are prefixed with `appva_` followed by 64 hex characters (256 bits of entropy)
- Only a SHA-256 hash of the token is stored — the raw secret is never persisted after the creation page is left
- Rotate a token at any time from the Service Accounts page; the old token is invalidated immediately

---

## Updating

Pull the latest image and recreate the container:

```bash
docker compose pull
docker compose up -d
```

Your data volume is preserved across updates.

---

## Useful Commands

```bash
# View logs
docker compose logs -f

# Stop
docker compose down

# Stop and remove all data (destructive)
docker compose down -v

# Restart
docker compose restart
```

---

## Roles

| Role | Access |
|---|---|
| `admin` | Full access including user management, service accounts, settings, and license activation |
| `operator` | Manage repositories, trigger scans, manage schedules and branch watchers, view results; no user or settings access |
| `read_only` | View dashboard, results, and documentation only |

For machine-to-machine access without 2FA, use [Service Accounts](#service-accounts) instead of regular user accounts.

---

## Support

- Documentation: available inside the app under **Documentation** in the sidebar
- Issues & feature requests: [SOCFortress](https://www.socfortress.co)
