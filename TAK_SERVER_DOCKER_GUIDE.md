# TAK Server 5.5 Docker Installation and Setup Guide

**Complete guide for running TAK Server 5.5 in Docker containers**

*Tested on: Ubuntu 24.04.3 LTS (Noble Numbat) with Docker 24.0+*

---

## IMPORTANT GOTCHAS - READ FIRST!

### TAK Server Packages Required

**You MUST download TAK Server packages from tak.gov before building Docker images!**

- TAK Server software cannot be redistributed
- You need `takserver_5.5-RELEASE58_all.deb` from https://tak.gov
- For federation, also download `takserver-fed-hub_5.5-RELEASE58_all.deb`
- Place these files in your build directory before running `docker build`

### PostgreSQL Version: MUST Use Version 15

**TAK Server 5.5 only works with PostgreSQL 15!**

- Do NOT use PostgreSQL 16 or newer
- The Dockerfile in this guide uses PostgreSQL 15 specifically
- Using the wrong version will cause database connection failures

### Persistent Data is Critical

**Always use Docker volumes for data persistence!**

- Without volumes, all data (certificates, configs, database) is lost when containers stop
- This guide uses named volumes for PostgreSQL data and TAK Server certificates
- Never skip the volume mounting steps

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Directory Setup](#directory-setup)
3. [Dockerfile Creation](#dockerfile-creation)
4. [Docker Compose Configuration](#docker-compose-configuration)
5. [Building the Docker Image](#building-the-docker-image)
6. [Running TAK Server with Docker Compose](#running-tak-server-with-docker-compose)
7. [Certificate Generation](#certificate-generation)
8. [Accessing TAK Server](#accessing-tak-server)
9. [Federation Setup in Docker](#federation-setup-in-docker)
10. [Managing Containers](#managing-containers)
11. [Troubleshooting](#troubleshooting)
12. [Production Considerations](#production-considerations)

---

## Prerequisites

### System Requirements

- Docker Engine 20.10+ or Docker Desktop
- Docker Compose 2.0+
- Minimum 8GB RAM (12GB+ recommended)
- 40GB+ disk space
- Internet connection
- sudo/root access

### Install Docker

**For Ubuntu 24.04:**

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

### Download TAK Server Package

1. Register and download from https://tak.gov
2. Download `takserver_5.5-RELEASE58_all.deb`
3. (Optional) Download `takserver-fed-hub_5.5-RELEASE58_all.deb` for federation

---

## Directory Setup

Create a working directory for your Docker setup:

```bash
# Create project directory
mkdir -p ~/tak-server-docker
cd ~/tak-server-docker

# Create subdirectories
mkdir -p packages certs config logs

# Copy TAK Server package to packages directory
cp /path/to/takserver_5.5-RELEASE58_all.deb packages/

# If using federation, also copy federation hub package
# cp /path/to/takserver-fed-hub_5.5-RELEASE58_all.deb packages/
```

Your directory structure should look like:
```
tak-server-docker/
├── packages/
│   └── takserver_5.5-RELEASE58_all.deb
├── certs/           (will contain certificates)
├── config/          (will contain configurations)
├── logs/            (will contain log files)
├── Dockerfile
└── docker-compose.yml
```

---

## Dockerfile Creation

Create a `Dockerfile` in the `~/tak-server-docker` directory:

```dockerfile
FROM ubuntu:24.04

# Set environment variables
ENV DEBIAN_FRONTEND=noninteractive
ENV JAVA_HOME=/usr/lib/jvm/temurin-17-jdk-amd64
ENV PATH=$JAVA_HOME/bin:$PATH

# Install system dependencies
RUN apt-get update && apt-get install -y \
    wget \
    curl \
    gnupg \
    apt-transport-https \
    software-properties-common \
    ca-certificates \
    lsb-release \
    netcat-openbsd \
    procps \
    iproute2 \
    net-tools \
    vim \
    && rm -rf /var/lib/apt/lists/*

# Install Temurin Java 17
RUN wget -qO - https://packages.adoptium.net/artifactory/api/gpg/key/public | gpg --dearmor -o /usr/share/keyrings/adoptium-archive-keyring.gpg && \
    echo "deb [signed-by=/usr/share/keyrings/adoptium-archive-keyring.gpg] https://packages.adoptium.net/artifactory/deb noble main" > /etc/apt/sources.list.d/adoptium.list && \
    apt-get update && \
    apt-get install -y temurin-17-jdk && \
    rm -rf /var/lib/apt/lists/*

# Install OpenJDK 17 (required for TAK Server dependencies)
RUN apt-get update && \
    apt-get install -y openjdk-17-jdk-headless && \
    rm -rf /var/lib/apt/lists/*

# Set alternatives to use Temurin
RUN update-alternatives --set java /usr/lib/jvm/temurin-17-jdk-amd64/bin/java

# Install PostgreSQL 15 client tools
RUN wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor -o /usr/share/keyrings/postgresql-archive-keyring.gpg && \
    echo "deb [signed-by=/usr/share/keyrings/postgresql-archive-keyring.gpg] http://apt.postgresql.org/pub/repos/apt noble-pgdg main" > /etc/apt/sources.list.d/pgdg.list && \
    apt-get update && \
    apt-get install -y postgresql-client-15 && \
    rm -rf /var/lib/apt/lists/*

# Copy TAK Server package
COPY packages/takserver_5.5-RELEASE58_all.deb /tmp/

# Install TAK Server
RUN dpkg -i /tmp/takserver_5.5-RELEASE58_all.deb || apt-get install -f -y && \
    rm /tmp/takserver_5.5-RELEASE58_all.deb

# Fix TAK Server startup scripts to use full paths
RUN sed -i 's|awk |/usr/bin/awk |g' /opt/tak/db-utils/takserver-setup-db.sh && \
    sed -i 's|java |/usr/bin/java |g' /opt/tak/db-utils/takserver-setup-db.sh

# Create directories for volumes
RUN mkdir -p /opt/tak/certs /opt/tak/logs /opt/tak/certs/files

# Expose ports
# TAK Server ports
EXPOSE 8089 8443 8444 8446 6969/udp
# Federation ports (if using federation)
EXPOSE 9000 9001 9100 9101 9102

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD nc -z localhost 8443 || exit 1

# Default command
CMD ["/opt/tak/takserver.sh", "start"]
```

**Save this as `Dockerfile` in your `~/tak-server-docker` directory.**

---

## Docker Compose Configuration

Create a `docker-compose.yml` file in the `~/tak-server-docker` directory:

```yaml
version: '3.8'

services:
  # PostgreSQL 15 Database
  postgres:
    image: postgres:15-alpine
    container_name: tak-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: cot
      POSTGRES_USER: martiuser
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-changeme}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - tak-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U martiuser -d cot"]
      interval: 10s
      timeout: 5s
      retries: 5

  # TAK Server
  takserver:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: tak-server
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      - JAVA_HOME=/usr/lib/jvm/temurin-17-jdk-amd64
      - PATH=/usr/lib/jvm/temurin-17-jdk-amd64/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=cot
      - DB_USERNAME=martiuser
      - DB_PASSWORD=${POSTGRES_PASSWORD:-changeme}
    volumes:
      - tak-certs:/opt/tak/certs
      - tak-logs:/opt/tak/logs
      - ./config:/opt/tak/config-overlay:ro
    ports:
      - "8089:8089"   # TAK Server client connections
      - "8443:8443"   # Web admin interface
      - "8444:8444"   # Federation HTTPS
      - "8446:8446"   # Certificate-based HTTPS
      - "6969:6969/udp" # Multicast
    networks:
      - tak-network
    healthcheck:
      test: ["CMD", "nc", "-z", "localhost", "8443"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

volumes:
  postgres-data:
    name: tak-postgres-data
  tak-certs:
    name: tak-certs
  tak-logs:
    name: tak-logs

networks:
  tak-network:
    name: tak-network
    driver: bridge
```

**Save this as `docker-compose.yml` in your `~/tak-server-docker` directory.**

### Environment Variables

Create a `.env` file to store sensitive configuration:

```bash
# Create .env file
cat > .env << 'EOF'
# PostgreSQL password
POSTGRES_PASSWORD=your_secure_password_here

# Change this to a strong password!
EOF

# Secure the .env file
chmod 600 .env
```

**IMPORTANT**: Change `your_secure_password_here` to a strong password!

---

## Building the Docker Image

Build the TAK Server Docker image:

```bash
cd ~/tak-server-docker

# Build the image (this may take 5-10 minutes)
docker compose build

# Verify the image was built
docker images | grep tak
```

You should see an image named something like `tak-server-docker-takserver`.

---

## Running TAK Server with Docker Compose

### First-Time Setup

When running TAK Server for the first time, you need to initialize the database:

```bash
# Start PostgreSQL only first
docker compose up -d postgres

# Wait for PostgreSQL to be ready (about 10-15 seconds)
docker compose ps

# Initialize TAK Server database
docker compose run --rm takserver /opt/tak/db-utils/takserver-setup-db.sh

# The script will prompt you for database connection details:
# - Hostname: postgres
# - Port: 5432
# - Database name: cot
# - Username: martiuser
# - Password: (use the password from your .env file)

# After database setup completes, start all services
docker compose up -d
```

### Starting TAK Server

After the first-time setup:

```bash
cd ~/tak-server-docker

# Start all services
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs -f takserver
```

### Stopping TAK Server

```bash
# Stop all services
docker compose down

# Stop and remove volumes (WARNING: this deletes all data!)
docker compose down -v
```

---

## Certificate Generation

Certificates are required for TAK Server to function. Generate them inside the running container:

### Generate Certificates

```bash
# Enter the TAK Server container
docker exec -it tak-server bash

# Navigate to certificate directory
cd /opt/tak/certs

# Run certificate generation script
./makeRootCa.sh

# Generate server certificate (use your server's IP or hostname)
./makeCert.sh server takserver

# Generate admin certificate
./makeCert.sh client admin

# Generate user certificates as needed
./makeCert.sh client user1
./makeCert.sh client user2

# Exit container
exit
```

### Export Certificates

To use certificates outside the container:

```bash
# Copy certificates from container to host
docker cp tak-server:/opt/tak/certs/files ./certs/

# List exported certificates
ls -lh certs/
```

The `certs/` directory will contain:
- `admin.p12` - Admin certificate for web UI
- `user1.p12` - User certificate for TAK clients
- `takserver.jks` - Server keystore
- Various `.pem` files

### Update CoreConfig.xml

After generating certificates, you need to configure TAK Server:

```bash
# Copy CoreConfig.xml from container
docker cp tak-server:/opt/tak/CoreConfig.xml ./config/

# Edit CoreConfig.xml with your settings
vim ./config/CoreConfig.xml
```

**Key changes to make in CoreConfig.xml:**

1. Update database connection (should already be correct):
```xml
<repository enable="true"
            numAutoRetries="2"
            connectionTimeoutMS="5000">
    <connection
        url="jdbc:postgresql://postgres:5432/cot"
        username="martiuser"
        password="your_password_here"/>
</repository>
```

2. Update certificate paths (should already be correct):
```xml
<certificateSigning CA="TAKServer">
    <certificateLocation>/opt/tak/certs/files</certificateLocation>
    <TAKServerCAPassword>atakatak</TAKServerCAPassword>
</certificateSigning>
```

3. Save and copy back to container:
```bash
docker cp ./config/CoreConfig.xml tak-server:/opt/tak/CoreConfig.xml

# Restart TAK Server to apply changes
docker compose restart takserver
```

---

## Accessing TAK Server

### Web Administration Interface

After TAK Server starts successfully:

1. **Access Web UI**: `https://YOUR_SERVER_IP:8443`

2. **Install Admin Certificate**:
   - Copy `certs/admin.p12` to your computer
   - Double-click to install (password: `atakatak`)
   - In browser, import the certificate
   - Access the web UI again

3. **Create Admin User**:
   ```bash
   # Enter container
   docker exec -it tak-server bash

   # Create admin user
   java -jar /opt/tak/utils/UserManager.jar usermod -A -p <password> admin

   exit
   ```

4. **Login**: Use username `admin` and the password you set

### Verify TAK Server Status

```bash
# Check if services are running
docker compose ps

# View TAK Server logs
docker compose logs -f takserver

# Check specific log files
docker exec tak-server tail -f /opt/tak/logs/takserver.log
docker exec tak-server tail -f /opt/tak/logs/takserver-messaging.log

# Verify ports are listening
docker exec tak-server netstat -tlnp
```

### Test Client Connections

From a TAK client (ATAK, WinTAK, iTAK):

1. Install client certificate (e.g., `user1.p12`)
2. Configure connection:
   - Server: `YOUR_SERVER_IP`
   - Port: `8089`
   - Protocol: `TLS`
3. Connect and verify

---

## Federation Setup in Docker

To enable federation between TAK servers, you need to set up the Federation Hub.

### Update Dockerfile for Federation

Add federation package installation to your Dockerfile:

```dockerfile
# After the TAK Server installation section, add:

# Copy Federation Hub package (if using federation)
COPY packages/takserver-fed-hub_5.5-RELEASE58_all.deb /tmp/

# Install Federation Hub
RUN dpkg -i /tmp/takserver-fed-hub_5.5-RELEASE58_all.deb || apt-get install -f -y && \
    rm /tmp/takserver-fed-hub_5.5-RELEASE58_all.deb

# Fix Federation Hub startup scripts
RUN sed -i 's|awk |/usr/bin/awk |g' /opt/tak/federation-hub/scripts/*.sh && \
    sed -i 's|java |/usr/bin/java |g' /opt/tak/federation-hub/scripts/*.sh

# Expose federation ports
EXPOSE 9000 9001 9100 9101 9102
```

### Federation Docker Compose

Create a separate service for Federation Hub in `docker-compose.yml`:

```yaml
  # Federation Hub (optional)
  federation-hub:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: tak-federation-hub
    restart: unless-stopped
    depends_on:
      - takserver
    environment:
      - JAVA_HOME=/usr/lib/jvm/temurin-17-jdk-amd64
    volumes:
      - tak-certs:/opt/tak/certs:ro
      - federation-config:/opt/tak/federation-hub/config
      - federation-logs:/opt/tak/federation-hub/logs
    ports:
      - "9000:9000"   # Federation V1
      - "9001:9001"   # Federation V2
      - "9100:9100"   # Federation Hub UI
      - "9101:9101"   # Federation Hub V1 broker
      - "9102:9102"   # Federation Hub V2 broker
    networks:
      - tak-network
    command: ["/opt/tak/federation-hub/scripts/federation-hub.sh", "start"]

volumes:
  # Add these volumes
  federation-config:
    name: tak-federation-config
  federation-logs:
    name: tak-federation-logs
```

### Federation Certificate Setup

```bash
# Generate federation certificates inside container
docker exec -it tak-server bash

cd /opt/tak/certs

# Generate federation server certificate
./makeCert.sh server federation-server

# Generate federation client certificates for remote servers
./makeCert.sh client remote-server-1

exit

# Copy federation certificates
docker cp tak-server:/opt/tak/certs/files/federation-server.jks ./certs/
docker cp tak-server:/opt/tak/certs/files/remote-server-1.p12 ./certs/
```

### Configure Federation

See [FEDERATION_SETUP.md](FEDERATION_SETUP.md) for detailed federation configuration instructions.

---

## Managing Containers

### Common Commands

```bash
# Start services
docker compose up -d

# Stop services
docker compose down

# Restart specific service
docker compose restart takserver

# View logs
docker compose logs -f takserver
docker compose logs -f postgres

# Execute commands in container
docker exec -it tak-server bash

# View resource usage
docker stats

# Remove everything (including volumes - WARNING: deletes all data!)
docker compose down -v
```

### Backup and Restore

**Backup:**

```bash
# Backup PostgreSQL database
docker exec tak-postgres pg_dump -U martiuser cot > tak-backup-$(date +%Y%m%d).sql

# Backup certificates
docker cp tak-server:/opt/tak/certs/files ./backup-certs-$(date +%Y%m%d)

# Backup configuration
docker cp tak-server:/opt/tak/CoreConfig.xml ./backup-config-$(date +%Y%m%d).xml
```

**Restore:**

```bash
# Restore PostgreSQL database
cat tak-backup-YYYYMMDD.sql | docker exec -i tak-postgres psql -U martiuser -d cot

# Restore certificates
docker cp ./backup-certs-YYYYMMDD tak-server:/opt/tak/certs/files

# Restore configuration
docker cp ./backup-config-YYYYMMDD.xml tak-server:/opt/tak/CoreConfig.xml

# Restart services
docker compose restart
```

---

## Troubleshooting

### Container Won't Start

**Check logs:**
```bash
docker compose logs takserver
docker compose logs postgres
```

**Common issues:**

1. **PostgreSQL not ready**:
   ```bash
   # Wait longer for PostgreSQL to initialize
   docker compose up -d postgres
   sleep 30
   docker compose up -d takserver
   ```

2. **Database connection failed**:
   - Verify `.env` file has correct password
   - Check CoreConfig.xml database settings
   - Ensure database was initialized: `docker compose run --rm takserver /opt/tak/db-utils/takserver-setup-db.sh`

3. **Port conflicts**:
   ```bash
   # Check if ports are already in use
   sudo netstat -tlnp | grep -E "8089|8443|5432"

   # Change ports in docker-compose.yml if needed
   ```

### Cannot Access Web UI

**Verify certificate installation:**
- Ensure `admin.p12` is imported in your browser
- Certificate password is `atakatak` by default
- Try different browsers (Chrome, Firefox, Edge)

**Check firewall:**
```bash
# On host machine
sudo ufw allow 8443/tcp
sudo ufw status
```

**Verify TAK Server is listening:**
```bash
docker exec tak-server netstat -tlnp | grep 8443
```

### Database Errors

**Reset database:**
```bash
# WARNING: This deletes all data!
docker compose down
docker volume rm tak-postgres-data
docker compose up -d postgres
sleep 30
docker compose run --rm takserver /opt/tak/db-utils/takserver-setup-db.sh
docker compose up -d
```

**Check PostgreSQL logs:**
```bash
docker compose logs postgres
```

### Certificate Issues

**Regenerate certificates:**
```bash
# Enter container
docker exec -it tak-server bash

# Remove old certificates
rm -rf /opt/tak/certs/files/*

# Regenerate
cd /opt/tak/certs
./makeRootCa.sh
./makeCert.sh server takserver
./makeCert.sh client admin

exit

# Restart
docker compose restart takserver
```

### High Memory Usage

**Limit Java heap size:**

Add to `docker-compose.yml` under `takserver` service:

```yaml
    environment:
      - JAVA_OPTS=-Xmx2048m -Xms1024m
```

**Monitor resources:**
```bash
docker stats
```

### Container Logs

**View all logs:**
```bash
# Follow logs
docker compose logs -f

# Last 100 lines
docker compose logs --tail=100

# Specific service
docker compose logs -f takserver

# Save logs to file
docker compose logs > tak-logs-$(date +%Y%m%d).log
```

**Inside container logs:**
```bash
# TAK Server logs
docker exec tak-server tail -f /opt/tak/logs/takserver.log
docker exec tak-server tail -f /opt/tak/logs/takserver-messaging.log
docker exec tak-server tail -f /opt/tak/logs/takserver-api.log
```

---

## Production Considerations

### Security

1. **Change default passwords**:
   - PostgreSQL password in `.env`
   - Certificate password (edit scripts before running)
   - Admin user password

2. **Use secrets management**:
   ```yaml
   # In docker-compose.yml
   secrets:
     postgres_password:
       file: ./secrets/postgres_password.txt
   ```

3. **Enable firewall**:
   ```bash
   sudo ufw allow 8089/tcp
   sudo ufw allow 8443/tcp
   sudo ufw enable
   ```

4. **Use TLS for PostgreSQL**:
   - Configure PostgreSQL to require SSL
   - Update CoreConfig.xml connection string

### Performance

1. **Increase database resources**:
   ```yaml
   postgres:
     environment:
       POSTGRES_SHARED_BUFFERS: 256MB
       POSTGRES_WORK_MEM: 16MB
     deploy:
       resources:
         limits:
           memory: 2G
   ```

2. **Optimize Java heap**:
   ```yaml
   takserver:
     environment:
       - JAVA_OPTS=-Xmx4096m -Xms2048m
   ```

3. **Use host networking** (Linux only, better performance):
   ```yaml
   takserver:
     network_mode: host
   ```

### High Availability

1. **Use Docker Swarm or Kubernetes** for orchestration
2. **External PostgreSQL** database (AWS RDS, managed PostgreSQL)
3. **Load balancer** for multiple TAK Server instances
4. **Shared storage** for certificates (NFS, S3, etc.)

### Monitoring

1. **Health checks** (already configured in docker-compose.yml)

2. **Prometheus metrics**:
   ```bash
   # Add Prometheus exporter to docker-compose.yml
   ```

3. **Log aggregation**:
   - Use ELK stack (Elasticsearch, Logstash, Kibana)
   - Or Loki + Grafana

### Backups

**Automated backup script** (`backup.sh`):

```bash
#!/bin/bash
BACKUP_DIR="/backups/tak-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Backup database
docker exec tak-postgres pg_dump -U martiuser cot > "$BACKUP_DIR/database.sql"

# Backup certificates
docker cp tak-server:/opt/tak/certs/files "$BACKUP_DIR/certs"

# Backup config
docker cp tak-server:/opt/tak/CoreConfig.xml "$BACKUP_DIR/"

# Compress
tar -czf "$BACKUP_DIR.tar.gz" "$BACKUP_DIR"
rm -rf "$BACKUP_DIR"

echo "Backup completed: $BACKUP_DIR.tar.gz"
```

**Schedule with cron**:
```bash
# Daily backup at 2 AM
0 2 * * * /home/user/tak-server-docker/backup.sh
```

---

## Summary

You now have TAK Server 5.5 running in Docker containers with:

- ✅ PostgreSQL 15 database in a container
- ✅ TAK Server in a container
- ✅ Persistent data using Docker volumes
- ✅ Certificate generation and management
- ✅ Web UI access on port 8443
- ✅ Client connections on port 8089
- ✅ (Optional) Federation Hub setup

### Next Steps

1. **Generate certificates** for your users
2. **Configure clients** (ATAK, WinTAK, iTAK)
3. **Test connections** from clients
4. **Set up federation** (if needed) - see [FEDERATION_SETUP.md](FEDERATION_SETUP.md)
5. **Configure backups** for production use
6. **Harden security** (change passwords, enable firewall)

### Useful Commands

```bash
# Start
docker compose up -d

# Stop
docker compose down

# Logs
docker compose logs -f takserver

# Shell access
docker exec -it tak-server bash

# Restart
docker compose restart takserver

# Backup
docker exec tak-postgres pg_dump -U martiuser cot > backup.sql

# Status
docker compose ps
```

---

## Additional Resources

- **Official TAK Documentation**: https://tak.gov
- **TAK Community Forums**: https://tak.gov/community
- **Docker Documentation**: https://docs.docker.com
- **Docker Compose Documentation**: https://docs.docker.com/compose/

For TAK Server installation on bare metal, see:
- [TAK_SERVER_5.5_COMPLETE_TUTORIAL.md](TAK_SERVER_5.5_COMPLETE_TUTORIAL.md)
- [TAK_SERVER_5.5_INSTALLATION_GUIDE.md](TAK_SERVER_5.5_INSTALLATION_GUIDE.md)

For federation setup, see:
- [FEDERATION_SETUP.md](FEDERATION_SETUP.md)

---

## License

This guide is provided as-is for educational purposes. TAK Server software is subject to its own licensing terms from tak.gov.

## Disclaimer

This is an unofficial community guide. For official support:
- Official documentation: https://tak.gov
- TAK.gov community forums: https://tak.gov/community

TAK Server is government software. All TAK Server components must be obtained from official sources.
