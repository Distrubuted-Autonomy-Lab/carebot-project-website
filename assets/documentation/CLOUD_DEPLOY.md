# Deploying Carebot to Google Cloud Platform

## Overview

This guide explains how to deploy the Carebot application to Google Cloud Platform (GCP) using Docker containers and Docker Compose. The deployment uses three services: a Django web application (gunicorn), an Nginx reverse proxy, and a PostGIS database.

## Architecture

The GCP deployment consists of three containerized services:

1. **Web Service**: Django application running with Gunicorn
   - Uses `docker-compose.gcloud.yml` with `gunicorn` command
   - Serves the Python Django application
   - Internally listens on port 8000

2. **Nginx Service**: Reverse proxy and static file server
   - Routes incoming traffic to the web service
   - Serves static files
   - Listens on port 80 (HTTP)
   - Container name: `website.chat.proxy`

3. **PostGIS Service**: PostgreSQL database with GIS extensions
   - Provides persistent data storage
   - Uses named volume `postgis_data` for persistence
   - Internally listens on port 5432

## Prerequisites

- A GCP project with Compute Engine enabled
- A VM instance running Docker and Docker Compose
- Access to your GCP instance via SSH
- The application source code and configuration files

## Deployment Steps

### 1. Set Up the Virtual Machine

Before deploying the containers, you need to provision a Compute Engine instance and prepare it with the necessary Docker runtime.

#### A. Create the Startup Script

Create a file named startup-script-app-server.sh on your local machine:

```bash
#!/bin/bash
set -euo pipefail
LOG=/var/log/startup-carebot.log
exec > >(tee -a "$LOG") 2>&1

echo "=== Startup begin: $(date -Is) ==="

# Install Docker
apt-get update
apt-get install -y ca-certificates curl gnupg lsb-release apt-transport-https cron
curl -fsSL https://get.docker.com -o /tmp/get-docker.sh
sh /tmp/get-docker.sh
systemctl enable --now docker

# Install gcloud CLI & Postgres Client
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | gpg --dearmor -o /etc/apt/keyrings/cloud.google.gpg
echo "deb [signed-by=/etc/apt/keyrings/cloud.google.gpg] http://packages.cloud.google.com/apt cloud-sdk main" > /etc/apt/sources.list.d/google-cloud-sdk.list
apt-get update && apt-get install -y google-cloud-cli postgresql-client-17

# Configure Docker Auth for GCP Artifact Registry
gcloud --quiet auth configure-docker us-east1-docker.pkg.dev

# Add User to Docker Group
usermod -aG docker $USER_NAME

echo "=== Startup done: $(date -Is) ==="
```

#### B. Provision the VM

Run the following command to create an e2-medium instance (recommended for PostGIS) with the startup script attached:

```bash
gcloud compute instances create carebot-vm \
    --scopes=https://www.googleapis.com/auth/cloud-platform \
    --project=YOUR_PROJECT_ID \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud \
    --boot-disk-size=50GB \
    --tags=http-server,https-server \
    --metadata-from-file=startup-script=./startup-script-app-server.sh
```

Recommended Specs:

- Machine Type: e2-medium (2 vCPUs, 4GB RAM)
- OS: Ubuntu 22.04 LTS
- Firewall: Ensure "Allow HTTP traffic" and "Allow HTTPS traffic" are checked.

### 2. Prepare Environment Variables

Create a `.env-gcloud` file based on `.env-gcloud.example` with your specific configuration:

```bash
# Database connection (PostGIS container)
DATABASE_NAME=your_database_name
DATABASE_USER=your_database_user
DATABASE_PASSWORD=your_database_password
DATABASE_HOST=postgis
DATABASE_PORT=5432
DATABASE_URL=postgresql://your_database_user:your_database_password@postgis:5432/your_database_name

# Django settings
SECRET_KEY=your_django_secret_key
ALLOWED_HOSTS=your.domain.com,your_ip_address

# API keys
GOOGLE_API_KEY=your_google_api_key

# Django superuser
DJANGO_SUPERUSER_USERNAME=your_superuser_username
DJANGO_SUPERUSER_EMAIL=your_superuser_email@example.com
DJANGO_SUPERUSER_PASSWORD=your_superuser_password

# Database configuration
POSTGRES_DB=website.chat.application
POSTGRES_USER=your_db_user
POSTGRES_PASSWORD=your_db_password
```

**Key Configuration Values:**
- `DATABASE_HOST=postgis` - Uses Docker service name for inter-container communication
- `ALLOWED_HOSTS` - Should include your domain and/or IP address
- `POSTGRES_DB` - Defaults to `website.chat.application`
- All secret keys and passwords should be strong and securely managed

POSTGRES_* variables configure the database container, while DATABASE_* must match those values for Django to connect. These values must be identical.

### 3. Upload Code to GCP

Transfer your application code to the GCP instance:

```bash
gcloud compute scp --recurse . carebot-vm:~/carebot/
gcloud compute scp .env-gcloud carebot-vm:~/carebot/root/
```

Or use `git clone` if the repository is accessible:

```bash
git clone <your-repo-url> ~/carebot/
cd ~/carebot/root
```

And copy your .env-gcloud file to the VM:

```bash
gcloud compute scp .env-gcloud carebot-vm:~/carebot/root/
```

### 4. Configuration on GCP Instance

SSH into your GCP instance:

```bash
gcloud compute ssh carebot-vm
```

Navigate to the application directory:

```bash
cd ~/carebot/root
```

### 5. Deploy with Docker Compose

Build and start all services using the GCP-specific compose file:

```bash
docker compose -f docker-compose.yml -f docker-compose.gcloud.yml up -d
```

**What this does:**
- Builds the web service Docker image (if not already built)
- Builds the Nginx Docker image (if not already built)
- Starts all three services (web, nginx, postgis)
- Runs in detached mode (`-d`)

### 6. Initialize Database

After services are running, you may need to run migrations:

```bash
docker compose -f docker-compose.yml -f docker-compose.gcloud.yml exec web python manage.py migrate
```

Create a superuser if not already created:

```bash
docker compose -f docker-compose.yml -f docker-compose.gcloud.yml exec web python manage.py createsuperuser
```

### 7. Collect Static Files

Collect static files for Nginx to serve:

```bash
docker compose -f docker-compose.yml -f docker-compose.gcloud.yml exec web python manage.py collectstatic --noinput
```

## Volumes and Persistent Storage

The deployment uses three named Docker volumes:

```yaml
volumes:
  static_volume:      # Nginx-served static files
  media_volume:       # User-uploaded media files
  postgis_data:       # Database data persistence
```

**Important:** These volumes persist between container restarts. Data is preserved even if containers are stopped and restarted.

## Port Mapping

- **Port 80** (HTTP): Nginx reverse proxy - main entry point
- **Port 8000** (Internal): Gunicorn web service - accessed only by Nginx
- **Port 5432**: (Internal): PostGIS database - for external access if needed

**Recommendation:** Configure a Google Cloud Firewall rule to only allow HTTP/HTTPS traffic on ports 80/443.

## SSL/HTTPS Configuration

To add HTTPS support:

1. **Option A: Cloud Load Balancer** - Use GCP's Cloud Load Balancer with SSL/TLS termination
2. **Option B: Nginx SSL** - Configure SSL certificates in the Nginx container
3. **Option C: Let's Encrypt** - Use Certbot with Docker

Update the Nginx configuration in `app/nginx/` to handle HTTPS redirects.

## Monitoring and Maintenance

### View Logs

Check service logs:

```bash
docker compose -f docker-compose.yml -f docker-compose.gcloud.yml logs -f web
docker compose -f docker-compose.yml -f docker-compose.gcloud.yml logs -f nginx
docker compose -f docker-compose.yml -f docker-compose.gcloud.yml logs -f postgis
```

### Database Backup

To backup the PostGIS database:

```bash
docker compose -f docker-compose.yml -f docker-compose.gcloud.yml exec postgis pg_dump -U your_db_user your_database_name > backup.sql
```

### Database Restore

To restore from a backup:

```bash
docker compose -f docker-compose.yml -f docker-compose.gcloud.yml exec -T postgis psql -U your_db_user your_database_name < backup.sql
```

### Stop Services

To stop all services (preserving data):

```bash
docker compose -f docker-compose.yml -f docker-compose.gcloud.yml down
```

To stop and remove all volumes (delete data):

```bash
docker compose -f docker-compose.yml -f docker-compose.gcloud.yml down -v
```

### Restart Services

To restart without rebuilding:

```bash
docker compose -f docker-compose.yml -f docker-compose.gcloud.yml restart
```

## Troubleshooting

### Services won't start
1. Check for port conflicts: `netstat -tulpn | grep -E ':80|:8000|:5432'`
2. Verify `.env-gcloud` file exists and contains all required variables
3. Check Docker daemon is running: `systemctl status docker`

### Database connection issues
1. Verify PostGIS service is running: `docker compose -f docker-compose.yml -f docker-compose.gcloud.yml ps`
2. Check database credentials in `.env-gcloud` match between web and postgis services
3. Test connection from web service: `docker compose -f docker-compose.gcloud.yml exec web psql -h postgis -U your_db_user -d your_database_name`

### Application errors
1. Check application logs: `docker compose -f docker-compose.yml -f docker-compose.gcloud.yml logs web`
2. Verify `SECRET_KEY` and `ALLOWED_HOSTS` are correctly set
3. Ensure all API keys are valid and active

## Security Recommendations

1. **Environment Variables**: Never commit `.env-gcloud` to version control. Use GCP Secret Manager for sensitive values.
3. **SSL/TLS**: Configure HTTPS on the load balancer or Nginx reverse proxy.
4. **Firewall Rules**: Use GCP Cloud Firewall to restrict traffic to necessary ports only.
5. **Secrets Management**: Store API keys and credentials in GCP Secret Manager, not in `.env-gcloud`.
6. **Regular Updates**: Keep Docker images updated with the latest security patches.
7. **Backups**: Implement regular automated backups of the PostGIS database.
