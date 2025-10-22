# TAK Server 5.5 Installation Guide for Ubuntu 24.04

**Complete step-by-step tutorial for installing TAK Server 5.5 on Ubuntu 24.04**

*Tested and working on: Ubuntu 24.04.3 LTS (Noble Numbat)*

---

## IMPORTANT GOTCHAS - READ FIRST!

### PostgreSQL Version: MUST Use Version 15

**Ubuntu 24.04 may install PostgreSQL 16 by default, which BREAKS TAK Server!**

- TAK Server 5.5 **only works with PostgreSQL 15**
- Ubuntu's default `apt install postgresql` often installs version 16
- PostgreSQL 16 will cause database connection failures and server crashes
- **You MUST explicitly install PostgreSQL 15 using the PostgreSQL APT repository**
- Follow the [PostgreSQL 15 Installation](#postgresql-15-installation) section exactly as written

### Java: Keep OpenJDK 17 Installed

**Do NOT remove OpenJDK 17 after installing Temurin!**

- TAK Server .deb package has dependencies on OpenJDK 17 packages
- Both Temurin JDK 17 (runtime) AND OpenJDK 17 (dependencies) are required
- This tutorial installs both - follow it exactly as written
- Removing OpenJDK 17 will break the installation

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Java Installation (Temurin 17 + OpenJDK 17)](#java-installation)
3. [PostgreSQL 15 Installation](#postgresql-15-installation)
4. [TAK Server Installation](#tak-server-installation)
5. [Database Configuration](#database-configuration)
6. [Certificate Generation](#certificate-generation)
7. [CoreConfig.xml Configuration](#coreconfig-configuration)
8. [Fix Startup Scripts](#fix-startup-scripts)
9. [Firewall Configuration](#firewall-configuration)
10. [Starting and Testing TAK Server](#starting-and-testing)
11. [Accessing TAK Server](#accessing-tak-server)
12. [Troubleshooting](#troubleshooting)

---

## Prerequisites

- Ubuntu 24.04 LTS with sudo access
- Minimum 8GB RAM recommended
- 40GB+ disk space
- Internet connection
- TAK Server 5.5 .deb package downloaded from tak.gov

---

## Java Installation

TAK Server 5.5 requires Java 17. We'll install both Temurin and OpenJDK to satisfy all dependencies.

**IMPORTANT**: This section installs BOTH Temurin JDK 17 AND OpenJDK 17. Both are required!
- Temurin 17: Used as the runtime Java environment
- OpenJDK 17: Required to satisfy .deb package dependencies

### 1. Remove any existing Java 21 or other versions (NOT Java 17!)

```bash
# Remove Java 21 and other versions, but we'll reinstall OpenJDK 17 later
sudo apt-get remove -y openjdk-21-jdk openjdk-21-jre openjdk-21-jdk-headless openjdk-21-jre-headless
sudo apt-get autoremove -y
```

**Note**: We're removing Java 21 if present, but we'll install OpenJDK 17 in step 5 below.

### 2. Install Temurin Java 17

```bash
# Install prerequisites
sudo apt-get update
sudo apt-get install -y wget apt-transport-https gpg

# Add Adoptium repository
wget -qO - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo gpg --dearmor -o /usr/share/keyrings/adoptium-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/adoptium-archive-keyring.gpg] https://packages.adoptium.net/artifactory/deb noble main" | sudo tee /etc/apt/sources.list.d/adoptium.list

# Install Temurin JDK 17
sudo apt-get update
sudo apt-get install -y temurin-17-jdk
```

### 3. Set JAVA_HOME environment variable

```bash
echo 'export JAVA_HOME=/usr/lib/jvm/temurin-17-jdk-amd64' | sudo tee -a /etc/environment
echo 'export PATH=$JAVA_HOME/bin:$PATH' | sudo tee -a /etc/environment
source /etc/environment
```

### 4. Verify Java installation

```bash
java -version
echo $JAVA_HOME
```

Expected output: `openjdk version "17.0.16" ... Temurin-17.0.16+8`

### 5. Install OpenJDK 17 (REQUIRED for .deb package dependencies)

**CRITICAL**: Do not skip this step! The TAK Server .deb package will fail to install without OpenJDK 17.

```bash
sudo apt-get install -y openjdk-17-jdk openjdk-17-jre
```

**Why both Temurin and OpenJDK?**
- The TAK Server .deb package explicitly requires `openjdk-17-jdk` in its dependencies
- We use Temurin as the actual runtime because it's more stable for production
- Both must coexist on the system

---

## PostgreSQL 15 Installation

**WARNING: PostgreSQL Version is Critical!**

TAK Server 5.5 **ONLY works with PostgreSQL 15**. Do NOT use version 16!

**Why this matters:**
- Ubuntu 24.04 may install PostgreSQL 16 by default if you use `apt install postgresql`
- PostgreSQL 16 is incompatible with TAK Server 5.5 and will cause severe issues:
  - Database connection failures
  - Server crashes on startup
  - Schema migration errors
- **You MUST use the PostgreSQL APT repository to explicitly install version 15**

Follow these steps exactly to install PostgreSQL 15:

### 1. Add PostgreSQL APT Repository

```bash
sudo apt-get install -y curl ca-certificates

sudo install -d /usr/share/postgresql-common/pgdg

sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```

### 2. Install PostgreSQL 15 and PostGIS

```bash
sudo apt-get update
sudo apt-get install -y postgresql-15 postgresql-client-15 postgresql-contrib-15 postgresql-15-postgis-3
```

### 3. Configure PostgreSQL to use Port 5432

```bash
sudo sed -i 's/port = 5433/port = 5432/' /etc/postgresql/15/main/postgresql.conf
sudo systemctl restart postgresql@15-main
```

### 4. Verify PostgreSQL Installation

**CRITICAL VERIFICATION**: Make sure you have PostgreSQL 15, not 16!

```bash
# Check version - MUST show "15.x"
psql --version

# Should output: psql (PostgreSQL) 15.x
# If you see 16.x, you installed the wrong version!

# Check service status
sudo systemctl status postgresql@15-main

# Test database access
sudo -u postgres psql -c "\l"
```

**If you see PostgreSQL 16 instead of 15:**
1. Remove PostgreSQL 16: `sudo apt-get purge postgresql-16 postgresql-client-16`
2. Go back to step 1 and follow the instructions exactly
3. Make sure you specify `-15` in the package names

---

## TAK Server Installation

### 1. Install the TAK Server .deb Package

```bash
cd ~/Downloads
sudo dpkg --force-depends -i takserver_5.5-RELEASE58_all.deb
```

**Note:** The `--force-depends` flag is used because we have Temurin Java instead of the exact OpenJDK package name in the dependency list.

### 2. Verify Installation

```bash
ls -la /opt/tak/
```

---

## Database Configuration

### 1. Create CoreConfig.xml from Example

```bash
cd /opt/tak
sudo cp CoreConfig.example.xml CoreConfig.xml
```

### 2. Set Database Password

```bash
sudo sed -i 's/password=""/password="takserver123"/' /opt/tak/CoreConfig.xml
```

**Important:** Change `takserver123` to a secure password for production!

### 3. Update Database Connection to Use Localhost

```bash
sudo sed -i 's/tak-database/localhost/' /opt/tak/CoreConfig.xml
```

### 4. Verify Database Configuration

```bash
sudo grep "connection url" /opt/tak/CoreConfig.xml
```

Expected: `<connection url="jdbc:postgresql://localhost:5432/cot" username="martiuser" password="takserver123" />`

### 5. Create Database User and Schema

```bash
sudo -u postgres psql -c "CREATE ROLE martiuser LOGIN ENCRYPTED PASSWORD 'takserver123' SUPERUSER INHERIT CREATEDB NOCREATEROLE;"
sudo -u postgres psql -c "CREATE DATABASE cot OWNER martiuser;"
```

### 6. Apply TAK Server Database Schema

```bash
cd /opt/tak/db-utils
sudo java -jar SchemaManager.jar upgrade
```

Expected output: `TAK server database schema is up to date.`

---

## Certificate Generation

### 1. Configure Certificate Metadata

```bash
cd /opt/tak/certs
sudo sed -i 's/STATE=${STATE}/STATE=YourState/' /opt/tak/certs/cert-metadata.sh
sudo sed -i 's/CITY=${CITY}/CITY=YourCity/' /opt/tak/certs/cert-metadata.sh
sudo sed -i 's/ORGANIZATIONAL_UNIT=${ORGANIZATIONAL_UNIT}/ORGANIZATIONAL_UNIT=TAKServer/' /opt/tak/certs/cert-metadata.sh
```

**Replace `YourState` and `YourCity` with your actual location.**

### 2. Generate Root Certificate Authority

```bash
cd /opt/tak/certs
sudo ./makeRootCa.sh
```

When prompted for CA name, enter something unique (e.g., `MyTAKServer`).

### 3. Generate Server Certificate

```bash
sudo ./makeCert.sh server takserver
```

### 4. Generate Client Certificates

Generate an admin certificate:
```bash
sudo ./makeCert.sh client admin
```

Generate user certificates:
```bash
sudo ./makeCert.sh client user1
sudo ./makeCert.sh client user2
```

### 5. Verify Certificates

```bash
ls -lh /opt/tak/certs/files/
```

You should see `.pem`, `.p12`, `.jks` files for each certificate.

---

## CoreConfig Configuration

### 1. Enable Multicast for SA Discovery

```bash
cd /opt/tak
sudo sed -i 's/<!-- <input _name="SAproxy" protocol="mcast" group="239.2.3.1" port="6969" proxy="true" auth="anonymous" \/> -->/<input _name="SAproxy" protocol="mcast" group="239.2.3.1" port="6969" proxy="true" auth="anonymous" \/>/' /opt/tak/CoreConfig.xml
```

### 2. Enable Server Announcement (Optional)

**First, get your server's IP address:**
```bash
hostname -I | awk '{print $1}'
```

**Then enable announcement with your IP:**
```bash
SERVER_IP=$(hostname -I | awk '{print $1}')
sudo sed -i "s/ip=\"192.168.1.1\"/ip=\"$SERVER_IP\"/" /opt/tak/CoreConfig.xml
sudo sed -i 's/<!--<announce enable="true" uid="Marti1"/<announce enable="true" uid="TakServer1"/' /opt/tak/CoreConfig.xml
sudo sed -i 's/<\/announce>-->/<\/announce>/' /opt/tak/CoreConfig.xml
```

### 3. Verify Configuration

```bash
grep -E "(SAproxy|announce)" /opt/tak/CoreConfig.xml
```

**Features enabled by default in TAK Server 5.5:**
- ✅ Video streaming (`streamingbroker enable="true"`)
- ✅ File sharing (enabled by default)
- ✅ Mission API (enabled by default)
- ✅ HTTPS on ports 8443, 8444, 8446
- ✅ TLS on port 8089

---

## Fix Startup Scripts

**CRITICAL STEP:** The TAK Server startup scripts need absolute paths for `awk`, `rm`, and `java` to work properly under systemd.

### 1. Fix setenv.sh

```bash
sudo sed -i 's|TOTALRAMBYTES=`awk|TOTALRAMBYTES=`/usr/bin/awk|' /opt/tak/setenv.sh
```

### 2. Fix All Startup Scripts

```bash
sudo sed -i 's|^rm -rf|/usr/bin/rm -rf|' /opt/tak/takserver-messaging.sh
sudo sed -i 's|^java |/usr/bin/java |' /opt/tak/takserver-messaging.sh
sudo sed -i 's|^java |/usr/bin/java |' /opt/tak/takserver-api.sh
sudo sed -i 's|^java |/usr/bin/java |' /opt/tak/takserver-config.sh
```

### 3. Create TAKIgniteConfig.xml

```bash
sudo cp /opt/tak/TAKIgniteConfig.example.xml /opt/tak/TAKIgniteConfig.xml
```

---

## Firewall Configuration

### 1. Open TAK Server Ports

```bash
sudo ufw allow 8089/tcp comment 'TAK Server TLS'
sudo ufw allow 8443/tcp comment 'TAK Server HTTPS'
sudo ufw allow 8444/tcp comment 'TAK Server Federation HTTPS'
sudo ufw allow 8446/tcp comment 'TAK Server Cert HTTPS'
sudo ufw allow 6969/udp comment 'TAK Server Multicast'
```

### 2. Verify Firewall Rules

```bash
sudo ufw status
```

---

## Starting and Testing

### 1. Enable TAK Server on Boot

```bash
sudo systemctl enable takserver
```

### 2. Start TAK Server

```bash
sudo systemctl start takserver
```

### 3. Check Service Status

```bash
sudo systemctl status takserver
sudo systemctl status takserver-messaging
sudo systemctl status takserver-api
sudo systemctl status takserver-config
```

### 4. Verify Java Processes are Running

```bash
ps aux | grep "tak.server" | grep -v grep
```

Expected: You should see 2-3 Java processes for messaging, API, and config.

### 5. Verify Ports are Listening

```bash
sudo netstat -plnt | grep -E "(8089|8443|8444|8446)"
```

Expected output:
```
tcp  0  0  0.0.0.0:8089  0.0.0.0:*  LISTEN  <pid>/java
tcp  0  0  0.0.0.0:8443  0.0.0.0:*  LISTEN  <pid>/java
tcp  0  0  0.0.0.0:8444  0.0.0.0:*  LISTEN  <pid>/java
tcp  0  0  0.0.0.0:8446  0.0.0.0:*  LISTEN  <pid>/java
```

### 6. Test HTTPS Connection

```bash
curl -k https://localhost:8446/
```

You should receive HTML response (login page).

---

## Accessing TAK Server

### Web Interface

- **Main HTTPS (requires client cert):** https://YOUR_SERVER_IP:8443
- **Certificate Enrollment:** https://YOUR_SERVER_IP:8446
- **Federation (requires cert):** https://YOUR_SERVER_IP:8444

### Client Configuration

**For ATAK/WinTAK clients:**
1. Copy client certificate (`.p12` file) from `/opt/tak/certs/files/` to your device
2. Import the certificate into your TAK client
3. Configure server connection:
   - **Protocol:** SSL/TLS
   - **Address:** YOUR_SERVER_IP
   - **Port:** 8089
   - **Use certificate authentication**

**Default certificate password:** `atakatak`

### Multicast Discovery

If on the same local network, ATAK clients can discover the server automatically via multicast on `239.2.3.1:6969`.

---

## Troubleshooting

### Services Not Starting

Check console logs:
```bash
cat /opt/tak/logs/takserver-messaging-console.log
cat /opt/tak/logs/takserver-api-console.log
cat /opt/tak/logs/takserver-config-console.log
```

### "java: not found" or "awk: not found" Errors

Make sure you completed the [Fix Startup Scripts](#fix-startup-scripts) section.

### PostgreSQL Connection Errors

```bash
# Verify PostgreSQL is running on port 5432
sudo netstat -plnt | grep 5432

# Check database exists
sudo -u postgres psql -c "\l"

# Verify martiuser can connect
sudo -u postgres psql -U martiuser -d cot -c "SELECT version();"
```

### Ports Not Listening

```bash
# Restart services
sudo systemctl restart takserver

# Wait 30 seconds for startup
sleep 30

# Check processes
ps aux | grep java
```

### Certificate Issues

```bash
# Regenerate certificates if needed
cd /opt/tak/certs
sudo rm -rf files/
sudo ./makeRootCa.sh
sudo ./makeCert.sh server takserver
sudo ./makeCert.sh client admin
```

### View Real-Time Logs

```bash
# Messaging service logs
tail -f /opt/tak/logs/takserver-messaging-console.log

# API service logs
tail -f /opt/tak/logs/takserver-api-console.log
```

---

## Summary of Working Ports

| Port | Protocol | Service | Purpose |
|------|----------|---------|---------|
| 5432 | TCP | PostgreSQL | Database |
| 8089 | TCP | TLS | Client connections (ATAK/WinTAK) |
| 8443 | TCP | HTTPS | Web interface (requires client cert) |
| 8444 | TCP | HTTPS | Federation connections |
| 8446 | TCP | HTTPS | Certificate enrollment |
| 6969 | UDP | Multicast | SA autodiscovery |

---

## Next Steps

1. **Import admin certificate into browser** to access web interface
2. **Create user accounts** via UserManager tool
3. **Configure groups and permissions** in CoreConfig.xml
4. **Set up federation** (if connecting to other TAK Servers)
5. **Configure data packages and plugins** as needed

---

## Additional Resources

- TAK Server Configuration Guide: `/opt/tak/docs/TAK_Server_Configuration_Guide.pdf`
- Official TAK.gov: https://tak.gov
- TAK Server installed at: `/opt/tak/`
- Certificates location: `/opt/tak/certs/files/`
- Configuration file: `/opt/tak/CoreConfig.xml`

---

**Installation tested on:** Ubuntu 24.04.3 LTS
**TAK Server version:** 5.5-RELEASE-58
**Java version:** Temurin 17.0.16 + OpenJDK 17
**PostgreSQL version:** 15.14

---

*This tutorial was created through hands-on installation and testing. All commands have been verified to work on Ubuntu 24.04.*
