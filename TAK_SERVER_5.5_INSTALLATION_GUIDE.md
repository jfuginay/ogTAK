# TAK Server 5.5 Installation Guide for Ubuntu 24.04

**Complete guide for installing an official TAK Server 5.5 from tak.gov**

---

## CRITICAL GOTCHAS - READ BEFORE STARTING!

### PostgreSQL Version: MUST Use Version 15 (NOT 16!)

**Ubuntu 24.04 will install PostgreSQL 16 by default, which BREAKS TAK Server!**

- TAK Server 5.5 **only works with PostgreSQL 15**
- Running `apt install postgresql` on Ubuntu 24.04 installs version 16
- PostgreSQL 16 causes database connection failures and server crashes
- **You MUST explicitly install PostgreSQL 15 from the PostgreSQL APT repository**
- See [System Prerequisites](#system-prerequisites) section for correct installation

### Java: OpenJDK 17 is REQUIRED (Don't Remove It!)

**The guide mentions removing OpenJDK, but you actually need to keep OpenJDK 17!**

- TAK Server .deb package requires OpenJDK 17 packages as dependencies
- You need BOTH Temurin JDK 17 (runtime) AND OpenJDK 17 (dependencies)
- Do not completely remove OpenJDK 17 after installing Temurin
- The installation will fail without OpenJDK 17 packages

---

This comprehensive guide will walk you through setting up a production-ready TAK Server with:
- Temurin Java 17 (Eclipse Adoptium)
- PostgreSQL database
- Full Certificate Authority (CA) setup
- Multicast support for local network discovery
- Video streaming capabilities
- File sharing and data packages
- Mission API for planning and data sync
- Federation framework (for connecting to other TAK servers)

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Java Environment Setup](#java-environment-setup)
3. [System Prerequisites](#system-prerequisites)
4. [PostgreSQL 15 Installation](#postgresql-15-installation)
5. [TAK Server Installation](#tak-server-installation)
6. [Database Configuration](#database-configuration)
6. [Certificate Authority Setup](#certificate-authority-setup)
7. [Server Configuration](#server-configuration)
8. [Starting the Server](#starting-the-server)
9. [Testing and Validation](#testing-and-validation)
10. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### System Requirements
- **OS**: Ubuntu 24.04 LTS (this guide)
- **RAM**: Minimum 8GB, recommended 16GB+
- **Disk**: Minimum 20GB free space
- **Network**: Static IP recommended
- **User**: sudo privileges required

### What You'll Need
- TAK Server 5.5 release files from [tak.gov](https://tak.gov)
  - Download the latest TAK Server 5.5 release
  - You'll need an account on tak.gov to download

### System Information
Check your current setup:
```bash
# Check Ubuntu version
cat /etc/os-release

# Check available memory
free -h

# Check disk space
df -h

# Get your server IP address
hostname -I
```

---

## Java Environment Setup

TAK Server 5.5 requires Java 17. We'll use Eclipse Temurin as the primary runtime, but we also need OpenJDK 17 packages for dependencies.

**IMPORTANT**: You need BOTH Temurin JDK 17 AND OpenJDK 17 installed!
- Temurin 17: Better for production runtime
- OpenJDK 17: Required by TAK Server .deb package dependencies

### Step 1: Remove Existing Java 21 (Keep Java 17!)

First, check what Java versions are installed:
```bash
java -version
dpkg -l | grep -i openjdk
```

Remove only Java 21 and other non-17 versions:
```bash
# Remove Java 21 (if present)
sudo apt-get remove -y openjdk-21-jdk openjdk-21-jre openjdk-21-jdk-headless openjdk-21-jre-headless
sudo apt-get autoremove -y
```

**Note**: We do NOT remove OpenJDK 17 here - if it's installed, leave it. We'll install it in Step 4 if not present.

### Step 2: Install Temurin Java 17

Add the Eclipse Adoptium repository:
```bash
# Install prerequisites
sudo apt-get update
sudo apt-get install -y wget apt-transport-https gpg

# Add Adoptium GPG key
wget -qO - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo gpg --dearmor -o /usr/share/keyrings/adoptium-archive-keyring.gpg

# Add Adoptium repository
echo "deb [signed-by=/usr/share/keyrings/adoptium-archive-keyring.gpg] https://packages.adoptium.net/artifactory/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
```

**What this does:** Adds the official Eclipse Adoptium repository to your system so you can install Temurin JDK packages.

Install Temurin JDK 17:
```bash
sudo apt-get update
sudo apt-get install -y temurin-17-jdk
```

Verify the installation:
```bash
java -version
# Should show: OpenJDK Runtime Environment Temurin-17...

javac -version
# Should show: javac 17.x.x

# Check Java home
echo $JAVA_HOME
# If empty, set it (see below)
```

### Step 3: Configure Java Environment Variables

Set JAVA_HOME permanently:
```bash
# Find Java installation path
JAVA_PATH=$(dirname $(dirname $(readlink -f $(which java))))
echo $JAVA_PATH

# Add to system environment
echo "export JAVA_HOME=$JAVA_PATH" | sudo tee -a /etc/environment
echo "export PATH=\$PATH:\$JAVA_HOME/bin" | sudo tee -a /etc/environment

# Load the new environment (or logout/login)
source /etc/environment

# Verify
echo $JAVA_HOME
java -version
```

**Important:** JAVA_HOME must be set for TAK Server to function properly.

### Step 4: Install OpenJDK 17 (REQUIRED for Dependencies)

**CRITICAL**: The TAK Server .deb package requires OpenJDK 17 packages!

```bash
# Install OpenJDK 17 packages
sudo apt-get install -y openjdk-17-jdk openjdk-17-jre
```

**Why do we need both Temurin AND OpenJDK?**
- The TAK Server .deb has explicit package dependencies on `openjdk-17-jdk`
- Installation will fail with "dependency problems" without OpenJDK 17
- Temurin provides the actual runtime (better for production)
- Both can coexist without conflicts

Verify both are installed:
```bash
# Check Java version (should show Temurin)
java -version

# Check OpenJDK packages are installed
dpkg -l | grep openjdk-17
```

---

## System Prerequisites

### Step 1: Update System Packages

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

### Step 2: Install Required Utilities

**IMPORTANT**: Do NOT install PostgreSQL from default repositories! We need version 15 specifically.

```bash
sudo apt-get install -y \
    unzip \
    curl \
    wget \
    net-tools \
    openssl \
    zip
```

**Note**: We deliberately left out `postgresql` here. We'll install PostgreSQL 15 specifically in the next steps.

**What these do:**
- `unzip`: Extract TAK Server archives
- `curl/wget`: Download files
- `net-tools`: Network diagnostics
- `openssl`: Certificate generation
- `zip`: Package certificates

**PostgreSQL 15 will be installed separately** using the official PostgreSQL repository to ensure we get the correct version.

### Step 3: Configure System Limits

TAK Server needs increased file descriptor limits:

```bash
# Edit limits configuration
sudo nano /etc/security/limits.conf
```

Add these lines at the end:
```
*                soft    nofile          32768
*                hard    nofile          32768
tak              soft    nofile          32768
tak              hard    nofile          32768
```

**Why?** TAK Server maintains many simultaneous connections and needs more file descriptors than the default.

Apply the changes:
```bash
# Logout and login, or reboot
sudo reboot
# Or check current limits:
ulimit -n
```

### Step 4: Configure Network Settings (Optional for Production)

Enable IPv4 forwarding for multicast:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

---

## PostgreSQL 15 Installation

**CRITICAL**: Install PostgreSQL 15 BEFORE installing TAK Server!

### Step 1: Add PostgreSQL APT Repository

```bash
# Install prerequisites
sudo apt-get install -y curl ca-certificates

# Create directory for PostgreSQL keys
sudo install -d /usr/share/postgresql-common/pgdg

# Download PostgreSQL GPG key
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

# Add PostgreSQL repository
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```

### Step 2: Install PostgreSQL 15

```bash
sudo apt-get update
sudo apt-get install -y postgresql-15 postgresql-client-15 postgresql-contrib-15 postgresql-15-postgis-3
```

**IMPORTANT**: Notice we explicitly specify `-15` in all package names!

### Step 3: Verify PostgreSQL 15 is Installed

```bash
# Check version - MUST show 15.x, NOT 16.x
psql --version

# Should output: psql (PostgreSQL) 15.x
```

**If you see version 16 instead of 15:**
```bash
# Remove PostgreSQL 16
sudo apt-get purge postgresql-16 postgresql-client-16 postgresql-contrib-16
sudo apt-get autoremove -y

# Then repeat steps 1-2 above
```

### Step 4: Configure and Start PostgreSQL 15

```bash
# Ensure PostgreSQL 15 uses port 5432
sudo sed -i 's/port = 5433/port = 5432/' /etc/postgresql/15/main/postgresql.conf

# Start and enable PostgreSQL
sudo systemctl start postgresql@15-main
sudo systemctl enable postgresql@15-main

# Verify it's running
sudo systemctl status postgresql@15-main
```

---

## TAK Server Installation

### Step 1: Prepare Installation Directory

```bash
# Create TAK directory
mkdir -p /home/j/Documents/ogTAK
cd /home/j/Documents/ogTAK

# Create subdirectories
mkdir -p {downloads,certs,backups}
```

### Step 2: Download TAK Server 5.5

1. Go to [https://tak.gov](https://tak.gov)
2. Log in with your account
3. Navigate to Downloads → TAK Server
4. Download **TAK Server 5.5** (release file, usually named like `takserver-5.5-RELEASE-XX.zip`)
5. Place the downloaded file in `/home/j/Documents/ogTAK/downloads/`

### Step 3: Extract TAK Server

```bash
cd /home/j/Documents/ogTAK

# Extract the release file (adjust filename as needed)
unzip downloads/takserver-*.zip

# This creates a directory like: takserver-5.5-RELEASE-XX
# Rename for convenience
mv takserver-5.5-RELEASE-* takserver
cd takserver
```

**Directory structure should now be:**
```
/home/j/Documents/ogTAK/
├── downloads/
├── certs/
├── backups/
└── takserver/
    ├── db-utils/
    ├── certs/
    ├── tak/
    └── ...
```

### Step 4: Run TAK Server Installer

```bash
cd /home/j/Documents/ogTAK/takserver

# Run the installer script
sudo ./setup.sh
```

The installer will:
- Create `tak` user and group
- Set up directory permissions
- Install systemd service files
- Configure initial settings

**Follow the prompts:**
- Accept default installation path (usually `/opt/tak`)
- Choose to install PostgreSQL (yes)
- Note the database password shown (SAVE THIS!)

---

## Database Configuration

### Step 1: Initialize PostgreSQL

Check PostgreSQL status:
```bash
sudo systemctl status postgresql
```

If not running:
```bash
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### Step 2: Configure TAK Database

The installer should have created the database, but verify:

```bash
# Switch to postgres user
sudo -u postgres psql

# Inside psql:
\l                          # List databases (should see 'cot')
\c cot                      # Connect to TAK database
\dt                         # List tables
\q                          # Quit
```

**Manual database setup (if needed):**
```bash
cd /home/j/Documents/ogTAK/takserver/db-utils

# Run database initialization
sudo -u postgres ./configure.sh
```

This creates:
- Database: `cot`
- User: `martiuser`
- Schema and tables for TAK Server

**Save the database credentials shown!**

---

## Certificate Authority Setup

TAK Server requires TLS certificates for secure communication. We'll create a complete Certificate Authority from scratch.

### Step 1: Navigate to Certificate Tools

```bash
cd /home/j/Documents/ogTAK/takserver/certs
ls
```

You should see:
- `cert-metadata.sh` - Configuration file
- `makeRootCa.sh` - Creates CA
- `makeCert.sh` - Creates certificates

### Step 2: Configure Certificate Metadata

Edit the certificate configuration:
```bash
nano cert-metadata.sh
```

**Important settings to customize:**
```bash
COUNTRY="US"
STATE="Virginia"
CITY="YourCity"
ORGANIZATION="YourOrganization"
ORGANIZATIONAL_UNIT="TAK"

# Server settings
CAPASS="ata2024SecureCAPass"        # Change this! CA password
PASS="ata2024SecurePass"            # Change this! Certificate password

# Your server details
IPADDRESS="192.168.0.50"           # Your server IP
HOSTADDRESS="takserver.local"      # Your hostname/domain
```

**CRITICAL:** Save these passwords securely! You'll need them for:
- Generating new certificates
- Configuring TAK Server
- Client connections

### Step 3: Generate Certificate Authority

```bash
# Create the CA (one time only)
./makeRootCa.sh

# Verify CA was created
ls -la files/
```

You should see:
- `ca.pem` - CA certificate
- `ca-do-not-share.jks` - CA keystore (NEVER share this!)
- `ca-do-not-share.key` - CA private key (NEVER share this!)
- `truststore-root.jks` - Trust store for clients

**What is a CA?** It's like being your own certificate authority (like VeriSign or Let's Encrypt). All certificates you create will be signed by this CA.

### Step 4: Generate Server Certificate

```bash
# Create server certificate
./makeCert.sh server takserver

# Verify
ls -la files/
```

New files created:
- `takserver.jks` - Server keystore (used by TAK Server)
- `takserver.pem` - Server certificate
- `takserver.key` - Server private key

### Step 5: Generate Admin Certificate

```bash
# Create admin certificate for web UI access
./makeCert.sh client admin

# Verify
ls -la files/
```

New files created:
- `admin.jks` - Admin keystore
- `admin.pem` - Admin certificate
- `admin.key` - Admin private key
- `admin.p12` - Admin PKCS12 file (for browser import)

**The admin.p12 file is what you'll import into your browser to access the web UI.**

### Step 6: Generate Initial Client Certificates

Create certificates for testing:

```bash
# Create test client certificate
./makeCert.sh client testclient1

# Create another for multi-user testing
./makeCert.sh client testclient2
```

Each client gets:
- `.p12` file for ATAK/WinTAK/iTAK
- `.jks` keystore
- `.pem` certificate

### Step 7: Copy Certificates to TAK Server

```bash
# Copy server keystores to TAK configuration
sudo cp files/takserver.jks /opt/tak/certs/
sudo cp files/truststore-root.jks /opt/tak/certs/

# Set permissions
sudo chown tak:tak /opt/tak/certs/*.jks
sudo chmod 600 /opt/tak/certs/*.jks

# Also backup all certificates
cp -r files/ /home/j/Documents/ogTAK/certs/backup-$(date +%Y%m%d)
```

### Step 8: Document Your Certificates

Create a certificate inventory:
```bash
cat > /home/j/Documents/ogTAK/certs/CERTIFICATE_INVENTORY.txt << 'EOF'
TAK Server Certificate Inventory
=================================

CA Password: [YOUR_CA_PASSWORD]
Certificate Password: [YOUR_CERT_PASSWORD]

Generated: $(date)
Server IP: 192.168.0.50

Certificates Created:
--------------------
1. Certificate Authority (CA)
   - ca.pem (share with clients)
   - ca-do-not-share.jks (NEVER share)
   - truststore-root.jks (clients need this)

2. Server Certificate
   - takserver.jks (server uses this)
   - takserver.pem

3. Admin Certificate
   - admin.p12 (import to browser)
   - Used for: https://192.168.0.50:8443

4. Client Certificates
   - testclient1.p12 (for ATAK/WinTAK)
   - testclient2.p12

To create new client certificates:
-----------------------------------
cd /home/j/Documents/ogTAK/takserver/certs
./makeCert.sh client <username>

Files will be in: files/<username>.p12
EOF
```

**Save this file securely with your passwords!**

---

## Server Configuration

Now we'll configure TAK Server with all requested features.

### Step 1: Backup Original Configuration

```bash
sudo cp /opt/tak/CoreConfig.xml /opt/tak/CoreConfig.xml.original
```

### Step 2: Edit CoreConfig.xml

```bash
sudo nano /opt/tak/CoreConfig.xml
```

This is the main configuration file. We'll configure:

#### A. Network Listeners

Find the `<network>` section and configure inputs:

```xml
<network multicastTTL="5">
    <input _name="stdssl" protocol="tls" port="8089"/>
    <input _name="stdtcp" protocol="tcp" port="8088"/>
    <input _name="streamtcp" protocol="stcp" port="8089"/>

    <!-- Multicast for local network discovery -->
    <input _name="multicast" protocol="udp" port="6969" group="239.2.3.1" iface="eth0"/>
</network>
```

**Explanation:**
- `8089/tls`: Secure connection for clients (ATAK/WinTAK)
- `8088/tcp`: Non-TLS connection (optional, can disable)
- `6969/udp multicast`: Local network auto-discovery
- Change `iface="eth0"` to your network interface (check with `ip link`)

#### B. Certificate Configuration

Find `<tls>` section:

```xml
<tls context="TLSv1.2" keystore="JKS" keystoreFile="/opt/tak/certs/takserver.jks" keystorePass="ata2024SecurePass" truststore="JKS" truststoreFile="/opt/tak/certs/truststore-root.jks" truststorePass="ata2024SecurePass"/>
```

**Replace passwords with your actual certificate passwords!**

#### C. Enable Federation

Find `<federation>` section:

```xml
<federation-server>
    <federation v2Enabled="true" enableStcp="true">
        <!-- Federation connections will be added here later -->
    </federation>
</federation-server>
```

#### D. Enable Video Streaming

Find or add `<plugins>` section:

```xml
<plugins>
    <plugin enabled="true" className="tak.server.plugins.VideoStreamingPlugin"/>
</plugins>
```

#### E. Enable Mission API

Find `<repository>` section:

```xml
<repository enable="true" numDbConnections="100">
    <connection url="jdbc:postgresql://localhost:5432/cot" username="martiuser" password="[DB_PASSWORD]"/>
</repository>
```

**Use the database password from installation!**

#### F. Enable File Sharing

Find `<content>` section:

```xml
<content>
    <dataPackages maxSizeBytes="104857600" directory="/opt/tak/data/packages"/>
</content>
```

**This enables 100MB max file size for data packages.**

### Step 3: Configure Groups and Permissions

```bash
sudo nano /opt/tak/UserAuthenticationFile.xml
```

Add user groups:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<UserAuthenticationFile>
    <User identifier="admin" password="" groupList="__ANON__"/>

    <!-- Add your users here -->
    <User identifier="CN=testclient1" groupList="__ANON__"/>
    <User identifier="CN=testclient2" groupList="__ANON__"/>
</UserAuthenticationFile>
```

**User identifiers match certificate Common Names (CN).**

### Step 4: Verify Configuration

Check configuration syntax:
```bash
sudo java -jar /opt/tak/utils/UserManager.jar configcheck /opt/tak/CoreConfig.xml
```

---

## Starting the Server

### Step 1: Start TAK Server Service

```bash
# Start TAK Server
sudo systemctl start takserver

# Check status
sudo systemctl status takserver

# Enable auto-start on boot
sudo systemctl enable takserver
```

### Step 2: Monitor Logs

```bash
# Watch main log
sudo tail -f /opt/tak/logs/takserver.log

# Watch messaging log
sudo tail -f /opt/tak/logs/takserver-messaging.log
```

**Look for:**
- "TAK Server started successfully"
- No error messages
- Port bindings (8089, 8443, etc.)

### Step 3: Check Listening Ports

```bash
sudo netstat -tlnp | grep java
```

Should show:
- `8089` - TLS client connections
- `8443` - HTTPS web UI
- `8444` - Certificate enrollment
- `8446` - Federation
- `6969` - Multicast (UDP)
- `5432` - PostgreSQL

---

## Testing and Validation

### Step 1: Access Web UI

1. **Import Admin Certificate to Browser:**
   - Copy `admin.p12` from server to your computer
   - Import to browser:
     - **Firefox**: Settings → Privacy & Security → Certificates → View Certificates → Your Certificates → Import
     - **Chrome**: Settings → Privacy and security → Security → Manage certificates → Import
   - Password is your certificate password

2. **Access Web Interface:**
   ```
   https://192.168.0.50:8443
   ```
   - Browser will ask which certificate to use → Select "admin"
   - Accept any security warnings (self-signed cert)

3. **Create Admin Account:**
   - First login creates admin user
   - Set username and password
   - Save credentials!

### Step 2: Test Client Connection (ATAK)

1. **Copy client certificate to device:**
   - Transfer `testclient1.p12` to Android/iOS/Windows device

2. **Import to ATAK/WinTAK/iTAK:**
   - Settings → Network Preferences → Manage Server Connections
   - Add Connection:
     - Address: `192.168.0.50`
     - Port: `8089`
     - Protocol: `SSL`
   - Import certificate when prompted

3. **Connect:**
   - Should show "Connected" status
   - Check web UI for connected clients

### Step 3: Test Multicast

On another device on the same network:
- Open ATAK
- Enable multicast discovery
- Should auto-discover server at `239.2.3.1:6969`

### Step 4: Test Mission API

In web UI:
- Go to Missions
- Create New Mission
- Add data layers
- Share with connected clients

### Step 5: Test File Sharing

- Upload a data package (image, KML, etc.) via web UI
- Should appear in `/opt/tak/data/packages`
- Verify clients can download

---

## Troubleshooting

### Server Won't Start

Check logs:
```bash
sudo journalctl -u takserver -f
sudo tail -f /opt/tak/logs/takserver.log
```

Common issues:
- **Java not found**: Verify JAVA_HOME is set
- **Port already in use**: Check for conflicting services
- **Database connection failed**: Verify PostgreSQL is running and credentials are correct

### Can't Access Web UI

```bash
# Verify port 8443 is listening
sudo netstat -tlnp | grep 8443

# Check firewall
sudo ufw status
sudo ufw allow 8443/tcp
```

### Certificate Issues

**Browser won't accept admin.p12:**
- Verify password is correct
- Re-export with: `openssl pkcs12 -in admin.p12 -out admin.pem -nodes`
- Check certificate validity: `openssl pkcs12 -info -in admin.p12`

**Client can't connect:**
- Verify client certificate CN matches UserAuthenticationFile.xml
- Check certificate not expired
- Verify truststore includes CA

### Multicast Not Working

```bash
# Verify interface supports multicast
ip maddr show

# Check multicast route
netstat -g

# Test multicast send
# (requires netcat or specialized tools)
```

Enable multicast on interface:
```bash
sudo ip link set eth0 multicast on
```

### Database Connection Issues

```bash
# Check PostgreSQL status
sudo systemctl status postgresql

# Verify database exists
sudo -u postgres psql -l | grep cot

# Test connection
sudo -u postgres psql -d cot -c "SELECT version();"

# Reset database password if needed
sudo -u postgres psql
ALTER USER martiuser WITH PASSWORD 'newpassword';
```

Update CoreConfig.xml with new password.

### Performance Issues

Monitor resources:
```bash
# Check memory
free -h

# Check CPU
top -u tak

# Check disk I/O
iostat -x 5
```

Optimize PostgreSQL:
```bash
sudo nano /etc/postgresql/15/main/postgresql.conf

# Increase these values:
shared_buffers = 2GB
effective_cache_size = 6GB
work_mem = 50MB
```

**Note**: Path is `/etc/postgresql/15/main/` because we're using PostgreSQL 15, not 16!

Restart PostgreSQL:
```bash
sudo systemctl restart postgresql
```

---

## Additional Resources

### Useful Commands

```bash
# Restart TAK Server
sudo systemctl restart takserver

# View all logs
sudo journalctl -u takserver --since "1 hour ago"

# Check server version
cat /opt/tak/version.txt

# Generate new client certificate
cd /home/j/Documents/ogTAK/takserver/certs
./makeCert.sh client newuser

# Backup database
sudo -u postgres pg_dump cot > /home/j/Documents/ogTAK/backups/cot-$(date +%Y%m%d).sql

# Restore database
sudo -u postgres psql cot < /home/j/Documents/ogTAK/backups/cot-YYYYMMDD.sql
```

### Important Files and Directories

```
/opt/tak/                           - TAK Server installation
/opt/tak/CoreConfig.xml            - Main configuration
/opt/tak/UserAuthenticationFile.xml - User permissions
/opt/tak/logs/                     - Server logs
/opt/tak/certs/                    - Server certificates
/opt/tak/data/packages/            - Data packages
/var/lib/postgresql/              - PostgreSQL data
```

### Security Best Practices

1. **Change default passwords immediately**
2. **Use strong CA and certificate passwords**
3. **Never share ca-do-not-share.jks or .key files**
4. **Regularly backup certificates and database**
5. **Keep TAK Server updated**
6. **Use firewall to restrict access**
7. **Monitor logs for suspicious activity**
8. **Rotate certificates annually**

### Getting Help

- **Official TAK Documentation**: [https://tak.gov/documentation](https://tak.gov/documentation)
- **TAK.gov Support Forum**: [https://tak.gov/forum](https://tak.gov/forum)
- **TAK Server GitHub**: [https://github.com/TAK-Product-Center](https://github.com/TAK-Product-Center)

---

## Summary Checklist

- [ ] Java Temurin 17 installed and verified
- [ ] PostgreSQL installed and configured
- [ ] TAK Server 5.5 extracted and installed
- [ ] Certificate Authority created
- [ ] Server, admin, and client certificates generated
- [ ] CoreConfig.xml configured for multicast, video, files, missions
- [ ] TAK Server started successfully
- [ ] Web UI accessible with admin certificate
- [ ] Client connection tested
- [ ] All passwords and certificates documented and backed up

---

**Installation completed!** Your TAK Server 5.5 is now ready for production use.

**Next Steps:**
1. Configure federation connections to other TAK servers
2. Create user accounts and groups
3. Set up SSL certificates from a real CA (optional, for public servers)
4. Configure automated backups
5. Set up monitoring and alerting

---

*Last updated: 2025-10-22*
*Guide version: 1.0*
