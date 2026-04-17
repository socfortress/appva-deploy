# AppVA — Application Vulnerability Assessment

AppVA is a containerized SBOM-based vulnerability scanning platform built for SOC teams. It generates Software Bills of Materials (SBOM) via [Syft](https://github.com/anchore/syft) and scans them for known CVEs via [Grype](https://github.com/anchore/grype), presenting findings through a secure web dashboard with PDF reporting, a remediation tracker, scheduled scans, and external notification support (Teams, Slack, Webhook).

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
| `admin` | Full access including user management, settings, and license activation |
| `operator` | Manage repositories, trigger scans, view results; no user or settings access |
| `read_only` | View dashboard, results, and documentation only |

---

## Support

- Documentation: available inside the app under **Documentation** in the sidebar
- Issues & feature requests: [SOCFortress](https://www.socfortress.co)
