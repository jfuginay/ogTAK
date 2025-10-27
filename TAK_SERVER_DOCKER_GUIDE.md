# TAK Server 5.5 Docker Installation Guide

**Complete guide for running TAK Server 5.5 using official Docker distribution**

*Tested on: Ubuntu 24.04.3 LTS (Noble Numbat) with Docker 24.0+*

---

## IMPORTANT - READ FIRST!

### Use Official TAK Server Docker Distribution

**Download the official Docker distribution from tak.gov!**

- TAK Server provides pre-configured Docker images and docker-compose setup
- Do NOT build custom Docker images
- Download `takserver-docker-5.5-RELEASE58.tar.gz` from https://tak.gov
- The official Docker distribution includes everything pre-configured

### What You Get from tak.gov

The official Docker distribution includes:
- Pre-built TAK Server Docker image
- Pre-configured `docker-compose.yml`
- PostgreSQL 15 configuration
- Nginx reverse proxy configuration
- Certificate generation scripts
- Ready-to-run setup

### Prerequisites

- Docker Engine 20.10+ or Docker Desktop
- Docker Compose 2.0+
- Minimum 8GB RAM (12GB+ recommended)
- 40GB+ disk space
- Downloaded TAK Server Docker package from tak.gov

---

## Table of Contents

1. [Install Docker](#install-docker)
2. [Download TAK Server Docker Package](#download-tak-server-docker-package)
3. [Extract and Setup](#extract-and-setup)
4. [Configure Environment](#configure-environment)
5. [Start TAK Server](#start-tak-server)
6. [Generate Certificates](#generate-certificates)
7. [Access Web Interface](#access-web-interface)
8. [Verify Server Status](#verify-server-status)
9. [Managing Containers](#managing-containers)
10. [Troubleshooting](#troubleshooting)
11. [Backup and Restore](#backup-and-restore)

---

## Install Docker

### For Ubuntu 24.04

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group (optional, to run docker without sudo)
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker compose version
```

Expected output:
```
Docker version 24.0.x
Docker Compose version 2.x.x
```

---

## Download TAK Server Docker Package

### Register and Download from tak.gov

1. Go to https://tak.gov
2. Create an account or login
3. Navigate to Downloads section
4. Download **TAK Server Docker Distribution**:
   - File: `takserver-docker-5.5-RELEASE58.tar.gz`
   - Size: ~1-2GB

### Verify Download

```bash
# Check file exists
ls -lh ~/Downloads/takserver-docker-*.tar.gz

# Optional: Verify checksum (if provided by tak.gov)
sha256sum ~/Downloads/takserver-docker-5.5-RELEASE58.tar.gz
```

---

## Extract and Setup

### Create Working Directory

```bash
# Create directory for TAK Server
mkdir -p ~/tak-server-docker
cd ~/tak-server-docker

# Extract the package
tar -xzf ~/Downloads/takserver-docker-5.5-RELEASE58.tar.gz

# The extraction creates a directory structure
ls -la
```

Expected directory structure:
```
tak-server-docker/
├── docker-compose.yml
├── tak/
│   ├── Dockerfile
│   └── ...
├── certs/
├── db-data/
└── README.md
```

### Review Official Documentation

```bash
# Read the official README
cat README.md

# Review the docker-compose configuration
cat docker-compose.yml
```

---

## Configure Environment

### Create Environment File

The official distribution typically includes a `.env.example` file:

```bash
# Copy example environment file
cp .env.example .env

# Edit environment variables
nano .env
```

**Key environment variables to set:**

```bash
# PostgreSQL configuration
POSTGRES_PASSWORD=your_secure_password_here

# TAK Server hostname/IP
TAK_SERVER_HOST=your_server_ip_or_hostname

# Certificate organization details
CERT_COUNTRY=US
CERT_STATE=YourState
CERT_CITY=YourCity
CERT_ORG=YourOrganization
CERT_OU=TAKServer
```

**IMPORTANT**: Change `your_secure_password_here` to a strong password!

### Secure Environment File

```bash
# Restrict permissions on .env file
chmod 600 .env
```

---

## Start TAK Server

### First-Time Initialization

When running TAK Server for the first time:

```bash
cd ~/tak-server-docker

# Pull required images (if not included in tar)
docker compose pull

# Start PostgreSQL first
docker compose up -d db

# Wait for database to be ready (30 seconds)
sleep 30

# Check database is healthy
docker compose ps

# Initialize TAK Server database
docker compose run --rm tak-server /opt/tak/db-utils/takserver-setup-db.sh

# Start all services
docker compose up -d
```

### Check Startup

```bash
# View all containers
docker compose ps

# Follow TAK Server logs
docker compose logs -f tak-server

# Wait for startup to complete (look for "Server startup complete" message)
```

Expected output:
```
NAME                STATUS              PORTS
tak-postgres        Up (healthy)        5432/tcp
tak-server          Up (healthy)        8089/tcp, 8443/tcp, 8444/tcp, 8446/tcp
```

---

## Generate Certificates

Certificates are required for TAK Server authentication.

### Generate Root CA

```bash
# Access the TAK Server container
docker exec -it tak-server bash

# Navigate to certificate directory
cd /opt/tak/certs

# Set certificate details (if not already in environment)
export STATE="California"
export CITY="SanFrancisco"
export ORGANIZATIONAL_UNIT="TAKServer"

# Generate root CA
./makeRootCa.sh

# Exit on success
```

### Generate Server Certificate

```bash
# Still inside the container
cd /opt/tak/certs

# Generate server certificate (use your server's hostname or IP)
./makeCert.sh server takserver

# Verify certificate was created
ls -la files/ | grep takserver
```

### Generate Admin Certificate

```bash
# Generate admin certificate for web UI access
./makeCert.sh client admin

# Generate additional user certificates as needed
./makeCert.sh client user1
./makeCert.sh client user2

# Exit container
exit
```

### Export Certificates from Container

```bash
# Create local certs directory
mkdir -p ~/tak-certs

# Copy certificates from container to host
docker cp tak-server:/opt/tak/certs/files/admin.p12 ~/tak-certs/
docker cp tak-server:/opt/tak/certs/files/user1.p12 ~/tak-certs/
docker cp tak-server:/opt/tak/certs/files/ca.pem ~/tak-certs/

# Set proper permissions
chmod 644 ~/tak-certs/*.p12

# List certificates
ls -lh ~/tak-certs/
```

The default certificate password is: **atakatak**

### Restart TAK Server

After generating certificates, restart the server:

```bash
docker compose restart tak-server

# Wait for restart and check logs
docker compose logs -f tak-server
```

---

## Access Web Interface

### Copy Admin Certificate

```bash
# Copy admin.p12 to your local machine
# The certificate is at: ~/tak-certs/admin.p12
# Password: atakatak
```

### Import Certificate to Browser

**Firefox:**

1. Open Firefox Settings
2. Privacy & Security → Certificates → View Certificates
3. Click "Your Certificates" tab
4. Click "Import"
5. Select `admin.p12`
6. Enter password: **atakatak**
7. Click OK
8. **Restart Firefox**

**Chrome/Chromium:**

1. Open Chrome Settings
2. Privacy and Security → Security → Manage certificates
3. Click "Your certificates" tab
4. Click "Import"
5. Select `admin.p12`
6. Enter password: **atakatak**
7. Click OK
8. **Restart Chrome**

### Access Web UI

1. Open your browser
2. Navigate to: **https://localhost:8443** (or **https://YOUR_SERVER_IP:8443**)
3. Accept the security warning (self-signed certificate)
4. When prompted, select the **admin** certificate
5. You should now see the TAK Server web interface

**Note:** The browser will ask you to select a certificate - choose the admin certificate you just imported.

---

## Verify Server Status

### Check Docker Containers

```bash
# View running containers
docker compose ps

# Expected output shows all containers healthy
# NAME                STATUS              PORTS
# tak-postgres        Up (healthy)        5432/tcp
# tak-server          Up (healthy)        8089/tcp, 8443/tcp, ...
```

### Check TAK Server Services

```bash
# Check services inside container
docker exec tak-server bash -c "ps aux | grep -i tak | grep -v grep"

# Check listening ports
docker exec tak-server netstat -tlnp

# View logs
docker compose logs --tail=50 tak-server
```

### Verify Ports are Accessible

```bash
# From host machine, check ports are listening
sudo netstat -tlnp | grep -E ':(8089|8443|8444|8446)'

# Or use ss command
ss -tlnp | grep -E ':(8089|8443|8444|8446)'
```

Expected ports:
- **8089** - TAK Server client connections (TLS)
- **8443** - Web administration interface (HTTPS)
- **8444** - Federation HTTPS
- **8446** - Certificate enrollment port

### Test Web Interface Access

```bash
# Test HTTPS connection (will show certificate error, but confirms server is listening)
curl -k https://localhost:8443

# Should return HTML content or certificate required message
```

---

## Managing Containers

### Common Commands

**Start all services:**
```bash
docker compose up -d
```

**Stop all services:**
```bash
docker compose down
```

**Restart specific service:**
```bash
docker compose restart tak-server
```

**View logs:**
```bash
# Follow logs for all services
docker compose logs -f

# Follow logs for specific service
docker compose logs -f tak-server

# View last 100 lines
docker compose logs --tail=100 tak-server
```

**Access container shell:**
```bash
# Access TAK Server container
docker exec -it tak-server bash

# Access PostgreSQL container
docker exec -it tak-postgres bash
```

**View resource usage:**
```bash
docker stats
```

**Stop and remove all data (WARNING - deletes everything!):**
```bash
docker compose down -v
```

---

## Troubleshooting

### Container Won't Start

**Check logs:**
```bash
docker compose logs tak-server
docker compose logs db
```

**Common issues:**

1. **Port already in use:**
   ```bash
   # Check what's using the port
   sudo netstat -tlnp | grep 8443

   # Stop conflicting service or change port in docker-compose.yml
   ```

2. **Database not ready:**
   ```bash
   # Ensure PostgreSQL is healthy before starting TAK Server
   docker compose up -d db
   sleep 30
   docker compose logs db
   docker compose up -d tak-server
   ```

3. **Permission errors:**
   ```bash
   # Fix permissions on volumes
   sudo chown -R 1000:1000 ./db-data
   sudo chown -R 1000:1000 ./certs
   ```

### Cannot Access Web UI

**Certificate not accepted:**

1. Verify certificate is imported in browser
2. Restart browser after importing certificate
3. Clear browser cache and cookies
4. Try a different browser (Firefox, Chrome)
5. Ensure certificate password was entered correctly (atakatak)

**Connection refused:**

```bash
# Verify TAK Server is running
docker compose ps

# Check if port 8443 is listening
docker exec tak-server netstat -tlnp | grep 8443

# Check firewall
sudo ufw status
sudo ufw allow 8443/tcp
```

**Certificate error `ERR_BAD_SSL_CLIENT_AUTH_CERT`:**

This means the admin certificate isn't installed. Follow the certificate import steps again and restart your browser.

### Database Connection Errors

**Reset database:**
```bash
# WARNING: This deletes all data!
docker compose down
docker volume rm tak-server-docker_db-data
docker compose up -d db
sleep 30
docker compose run --rm tak-server /opt/tak/db-utils/takserver-setup-db.sh
docker compose up -d
```

**Check PostgreSQL:**
```bash
# Connect to database
docker exec -it tak-postgres psql -U martiuser -d cot

# List tables
\dt

# Exit
\q
```

### Certificate Issues

**Regenerate all certificates:**
```bash
# Access container
docker exec -it tak-server bash

# Backup existing certificates
cd /opt/tak/certs
mv files files.backup
mkdir files

# Regenerate root CA and certificates
./makeRootCa.sh
./makeCert.sh server takserver
./makeCert.sh client admin

exit

# Restart server
docker compose restart tak-server
```

### View Detailed Logs

**TAK Server logs inside container:**
```bash
# View main server log
docker exec tak-server tail -f /opt/tak/logs/takserver.log

# View messaging log
docker exec tak-server tail -f /opt/tak/logs/takserver-messaging.log

# View API log
docker exec tak-server tail -f /opt/tak/logs/takserver-api.log

# View all logs
docker exec tak-server tail -f /opt/tak/logs/*.log
```

---

## Backup and Restore

### Backup

Create a backup script (`backup.sh`):

```bash
#!/bin/bash
BACKUP_DIR="$HOME/tak-backups/backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "Creating TAK Server backup in $BACKUP_DIR"

# Backup PostgreSQL database
echo "Backing up database..."
docker exec tak-postgres pg_dump -U martiuser cot > "$BACKUP_DIR/database.sql"

# Backup certificates
echo "Backing up certificates..."
docker cp tak-server:/opt/tak/certs/files "$BACKUP_DIR/certs"

# Backup configuration
echo "Backing up configuration..."
docker cp tak-server:/opt/tak/CoreConfig.xml "$BACKUP_DIR/CoreConfig.xml"

# Backup docker-compose and .env
cp docker-compose.yml "$BACKUP_DIR/"
cp .env "$BACKUP_DIR/.env.backup"

# Create compressed archive
echo "Compressing backup..."
tar -czf "$BACKUP_DIR.tar.gz" -C "$(dirname $BACKUP_DIR)" "$(basename $BACKUP_DIR)"
rm -rf "$BACKUP_DIR"

echo "Backup completed: $BACKUP_DIR.tar.gz"
```

**Run backup:**
```bash
chmod +x backup.sh
./backup.sh
```

### Restore

```bash
# Extract backup
tar -xzf backup-YYYYMMDD-HHMMSS.tar.gz

# Stop TAK Server
docker compose down

# Restore database
cat backup-YYYYMMDD-HHMMSS/database.sql | docker exec -i tak-postgres psql -U martiuser -d cot

# Restore certificates
docker cp backup-YYYYMMDD-HHMMSS/certs/. tak-server:/opt/tak/certs/files/

# Restore configuration
docker cp backup-YYYYMMDD-HHMMSS/CoreConfig.xml tak-server:/opt/tak/CoreConfig.xml

# Restart services
docker compose up -d

# Verify
docker compose logs -f tak-server
```

### Automated Backups with Cron

```bash
# Edit crontab
crontab -e

# Add daily backup at 2 AM
0 2 * * * /home/yourusername/tak-server-docker/backup.sh >> /home/yourusername/tak-backups/backup.log 2>&1
```

---

## Production Considerations

### Security Best Practices

1. **Change default passwords:**
   ```bash
   # Change PostgreSQL password in .env file
   # Change certificate password by editing cert scripts before generation
   ```

2. **Configure firewall:**
   ```bash
   # Allow only necessary ports
   sudo ufw allow 8089/tcp
   sudo ufw allow 8443/tcp
   sudo ufw deny 5432/tcp  # Don't expose PostgreSQL to internet
   sudo ufw enable
   ```

3. **Use strong certificates:**
   - Generate unique certificates for each user
   - Set certificate expiration appropriately
   - Revoke compromised certificates immediately

4. **Enable HTTPS only:**
   - Ensure all connections use TLS/SSL
   - Disable any HTTP endpoints

### Performance Tuning

**Increase Java heap size:**

Edit `docker-compose.yml`:
```yaml
services:
  tak-server:
    environment:
      - JAVA_OPTS=-Xmx4096m -Xms2048m
```

**Increase PostgreSQL resources:**

Edit `docker-compose.yml`:
```yaml
services:
  db:
    environment:
      POSTGRES_SHARED_BUFFERS: "256MB"
      POSTGRES_WORK_MEM: "16MB"
    deploy:
      resources:
        limits:
          memory: 2G
```

### Monitoring

**Health checks:**
```bash
# Check container health
docker compose ps

# Monitor resources
docker stats

# Set up alerts for container failures
```

**Log monitoring:**
```bash
# Use log aggregation tools like ELK Stack or Grafana Loki
# Configure docker logging driver for centralized logging
```

---

## Additional Resources

### Official Documentation
- TAK Server Documentation: https://tak.gov
- TAK Community Forums: https://tak.gov/community
- Docker Documentation: https://docs.docker.com

### Related Guides in This Repository
- [Complete Installation Tutorial](TAK_SERVER_5.5_COMPLETE_TUTORIAL.md) - Bare metal installation
- [Installation Guide](TAK_SERVER_5.5_INSTALLATION_GUIDE.md) - Alternative installation reference
- [Federation Setup](FEDERATION_SETUP.md) - Connect multiple TAK Servers

---

## Summary

You now have TAK Server 5.5 running in Docker with:

- ✅ Official TAK Server Docker distribution from tak.gov
- ✅ PostgreSQL 15 database
- ✅ Persistent data using Docker volumes
- ✅ Certificate-based authentication
- ✅ Web UI accessible on port 8443
- ✅ Client connections on port 8089

### Quick Reference

```bash
# Start services
docker compose up -d

# Stop services
docker compose down

# View logs
docker compose logs -f tak-server

# Access container
docker exec -it tak-server bash

# Restart
docker compose restart tak-server

# Backup
./backup.sh

# Check status
docker compose ps
```

---

## License

This guide is provided as-is for educational purposes. TAK Server software is subject to its own licensing terms from tak.gov.

## Disclaimer

This is an unofficial community guide. For official support:
- Official documentation: https://tak.gov
- TAK.gov community forums: https://tak.gov/community

TAK Server is government software. All TAK Server components must be obtained from official sources (tak.gov).
