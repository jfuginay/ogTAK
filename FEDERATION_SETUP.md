# TAK Server 5.5 Federation Setup Guide

Complete guide for setting up federation between TAK Server 5.5 and OpenTAKServer (or other TAK servers).

**Prerequisites:**
- TAK Server 5.5 installed and running (see TAK_SERVER_5.5_COMPLETE_TUTORIAL.md)
- TAK Server Federation Hub package downloaded from tak.gov
- Federation certificates from the remote server you want to federate with

---

## Table of Contents

1. [Understanding TAK Server Federation](#understanding-federation)
2. [Installing Federation Hub](#installing-federation-hub)
3. [Configuring Certificates](#configuring-certificates)
4. [Firewall Configuration](#firewall-configuration)
5. [Federation Hub Configuration](#federation-hub-configuration)
6. [Starting Federation Services](#starting-federation-services)
7. [Configuring Federation Connections](#configuring-federation-connections)
8. [Troubleshooting](#troubleshooting)
9. [Key Gotchas](#key-gotchas)

---

## Understanding Federation

TAK Server federation allows multiple TAK servers to share data across different networks and administrative boundaries. TAK Server 5.5 uses the **Federation Hub** component to handle federation connections.

**Important:** The Federation Hub is a separate package from the main TAK Server and must be installed separately.

**Federation Ports:**
- **9000** - Federation V1 (legacy)
- **9001** - Federation V2 server port
- **9100** - Federation Hub UI (web management interface)
- **9101** - Federation Hub V1 broker port
- **9102** - Federation Hub V2 broker port (most commonly used)

---

## Installing Federation Hub

### 1. Install the Federation Hub Package

```bash
sudo dpkg -i ~/Downloads/takserver-fed-hub_5.5-RELEASE58_all.deb
```

### 2. Install Required Dependencies

The federation hub scripts require `awk` which may not be in the default PATH:

```bash
sudo apt-get install -y gawk
```

### 3. Fix Federation Hub Scripts

The federation hub scripts run as the `tak` user and need full paths to system utilities.

**Fix all three federation hub scripts:**

```bash
# Fix awk paths
sudo sed -i 's|awk |/usr/bin/awk |g' /opt/tak/federation-hub/scripts/federation-hub-broker.sh
sudo sed -i 's|awk |/usr/bin/awk |g' /opt/tak/federation-hub/scripts/federation-hub-policy.sh
sudo sed -i 's|awk |/usr/bin/awk |g' /opt/tak/federation-hub/scripts/federation-hub-ui.sh

# Fix java paths
sudo sed -i 's|^java |/usr/bin/java |g' /opt/tak/federation-hub/scripts/federation-hub-broker.sh
sudo sed -i 's|^java |/usr/bin/java |g' /opt/tak/federation-hub/scripts/federation-hub-policy.sh
sudo sed -i 's|^java |/usr/bin/java |g' /opt/tak/federation-hub/scripts/federation-hub-ui.sh
```

---

## Configuring Certificates

### 1. Create Federation Hub Certificate Directory

```bash
sudo mkdir -p /opt/tak/federation-hub/certs/files
```

### 2. Copy TAK Server Certificates to Federation Hub

```bash
sudo cp /opt/tak/certs/files/takserver.jks /opt/tak/federation-hub/certs/files/
sudo cp /opt/tak/certs/files/fed-truststore.jks /opt/tak/federation-hub/certs/files/
sudo cp /opt/tak/certs/files/ca.pem /opt/tak/federation-hub/certs/files/
```

### 3. Import Remote Server Certificates

If federating with OpenTAKServer or another TAK server, you need to import their CA certificate to your federation truststore.

**Example: Importing OpenTAKServer CA certificate**

```bash
# Copy the remote server's CA certificate to your certs directory
sudo cp ~/Downloads/opentakserver-ca.pem /opt/tak/certs/files/

# Import to federation truststore
cd /opt/tak/certs/files
sudo keytool -import -trustcacerts -alias opentakserver \
  -file opentakserver-ca.pem \
  -keystore fed-truststore.jks \
  -storepass atakatak \
  -noprompt

# Copy updated truststore to federation hub
sudo cp /opt/tak/certs/files/fed-truststore.jks /opt/tak/federation-hub/certs/files/
```

**Replace:**
- `opentakserver` with a unique alias for the remote server
- `opentakserver-ca.pem` with your remote server's CA certificate file

### 4. Enable Admin Access to Federation Hub UI

```bash
sudo java -jar /opt/tak/federation-hub/jars/federation-hub-manager.jar \
  /opt/tak/certs/files/admin.pem
```

---

## Firewall Configuration

Open all required federation ports:

```bash
# TAK Server federation ports
sudo ufw allow 9000/tcp comment 'TAK Server Federation V1'
sudo ufw allow 9001/tcp comment 'TAK Server Federation V2 Port'

# Federation Hub ports
sudo ufw allow 9100/tcp comment 'Federation Hub UI'
sudo ufw allow 9101/tcp comment 'Federation Hub V1'
sudo ufw allow 9102/tcp comment 'Federation Hub V2'

# Verify
sudo ufw status numbered | grep -E "9000|9001|9100|9101|9102"
```

---

## Federation Hub Configuration

### 1. Configure Database Password

Edit the federation hub broker configuration:

```bash
sudo nano /opt/tak/federation-hub/configs/federation-hub-broker.yml
```

Find the line `dbPassword:` and set it to match your PostgreSQL password (default: `takserver123`):

```yaml
dbPassword: takserver123
```

**Note:** The Federation Hub uses MongoDB by default, but it can work without it for basic federation. The config file references port 27017 which is MongoDB's default port.

### 2. Verify Configuration

Check that the configuration file has correct settings:

```bash
cat /opt/tak/federation-hub/configs/federation-hub-broker.yml
```

Key settings:
- `v1Enabled: true` and `v1Port: 9101` for V1 federation
- `v2Enabled: true` and `v2Port: 9102` for V2 federation
- Certificate paths point to `/opt/tak/federation-hub/certs/files/`

---

## Starting Federation Services

### 1. Start Federation Hub Components

```bash
sudo /etc/init.d/federation-hub-broker start
sudo /etc/init.d/federation-hub-policy start
sudo /etc/init.d/federation-hub-ui start
```

### 2. Verify Services are Running

```bash
# Check processes
ps aux | grep federation-hub | grep -v grep

# Check ports
sudo netstat -tlnp | grep -E "9100|9101|9102"
```

### 3. Check Logs

```bash
# View broker startup
sudo tail -f /opt/tak/federation-hub/logs/federation-hub-broker-console.log

# View policy manager
sudo tail -f /opt/tak/federation-hub/logs/federation-hub-policy-console.log

# View UI
sudo tail -f /opt/tak/federation-hub/logs/federation-hub-ui-console.log
```

**Note:** The broker may show Ignite connection retries during startup - this is normal and will resolve once it connects to TAK Server.

---

## Configuring Federation Connections

### 1. Edit CoreConfig.xml

```bash
sudo nano /opt/tak/CoreConfig.xml
```

### 2. Configure Federation Server

Find the `<federation>` section and update it:

```xml
<federation missionFederationDisruptionToleranceRecencySeconds="43200">
    <federation-server port="9000" v1enabled="false" v2port="9001" v2enabled="true">
        <tls context="TLSv1.2" keymanager="SunX509"
             keystore="JKS" keystoreFile="/opt/tak/certs/files/takserver.jks" keystorePass="atakatak"
             truststore="JKS" truststoreFile="/opt/tak/certs/files/fed-truststore.jks" truststorePass="atakatak"/>
    </federation-server>

    <!-- Outgoing federation connection example -->
    <federation-outgoing name="OpenTAKServer" enabled="true">
        <v2 remoteServerId="" remoteServerDiscovery="false" fallback="true">
            <protocolVersion version="2"/>
            <reconnectInterval intervalMillis="30000"/>
            <maxRetries count="1"/>
            <federate shareAlerts="true" shareSA="true" autoGroup="OpenTAK"/>
            <contact>REMOTE_SERVER_IP:9102</contact>
        </v2>
    </federation-outgoing>

    <fileFilter>
        <fileExtension>pref</fileExtension>
    </fileFilter>
</federation>
```

**Replace:**
- `OpenTAKServer` with a descriptive name for the remote server
- `REMOTE_SERVER_IP:9102` with the actual IP address and port of the remote server
- `autoGroup="OpenTAK"` with your desired group name for federated data

### 3. Restart TAK Server

```bash
sudo systemctl restart takserver
```

### 4. Verify Federation Connection

Check TAK Server logs for federation activity:

```bash
sudo tail -f /opt/tak/logs/takserver-messaging.log | grep -i federation
```

---

## Troubleshooting

### Federation Hub Won't Start

**Symptom:** Scripts report "java: command not found" or "awk: command not found"

**Solution:** Verify paths are correct in scripts:

```bash
# Test as tak user
sudo -u tak /usr/bin/java -version
sudo -u tak /usr/bin/awk 'BEGIN {print "test"}'

# Verify script has full paths
grep -n "java\|awk" /opt/tak/federation-hub/scripts/federation-hub-broker.sh
```

### Federation Ports Not Listening

**Symptom:** Ports 9101/9102 not showing in `netstat`

**Possible causes:**
1. Federation Hub broker still connecting to Ignite cluster (wait 30-60 seconds)
2. Certificate issues
3. Configuration errors

**Check logs:**

```bash
sudo tail -100 /opt/tak/federation-hub/logs/federation-hub-broker-console.log
```

### Invalid Maximum Heap Size Error

**Symptom:** "Invalid maximum heap size: -Xmxm"

**Cause:** The `awk` command failed to get memory information

**Solution:** Ensure `awk` path is correct in scripts (see step 3 of Installation)

### Cannot Import Certificate to Truststore

**Symptom:** "keytool error: java.io.FileNotFoundException"

**Solution:** Verify certificate file exists and path is correct:

```bash
ls -la /opt/tak/certs/files/opentakserver-ca.pem
```

---

## Key Gotchas

### 1. Federation Hub is Separate from TAK Server
The Federation Hub is **not** included in the main TAK Server package. You must download and install `takserver-fed-hub_5.5-RELEASE58_all.deb` separately from tak.gov.

### 2. Script PATH Issues
Federation Hub scripts run as the `tak` user which has a minimal PATH. You **must** use full paths (`/usr/bin/java`, `/usr/bin/awk`) in the scripts or they will fail with "command not found" errors.

### 3. Two Certificate Locations
Certificates must exist in **two** locations:
- `/opt/tak/certs/files/` (TAK Server)
- `/opt/tak/federation-hub/certs/files/` (Federation Hub)

Always copy certificates to both locations.

### 4. Multiple Configuration Files
Federation configuration exists in multiple places:
- `/opt/tak/CoreConfig.xml` - Federation server and outgoing connections
- `/opt/tak/federation-hub/configs/federation-hub-broker.yml` - Federation Hub ports and settings

Both must be configured correctly.

### 5. Port Confusion
- **9001** = TAK Server's federation V2 server port (inbound from TAK Server to Federation Hub)
- **9102** = Federation Hub's V2 broker port (inbound from other servers to this Federation Hub)

When connecting TO another server, you connect to their port 9102.
When accepting connections FROM other servers, they connect to your port 9102.

### 6. Firewall Rules Must Allow External Access
If federating with servers on different networks, ensure firewall rules allow **inbound** connections on ports 9101/9102, not just outbound.

### 7. Certificate Trust Chain
Both servers must trust each other's certificates. Import the **remote server's CA certificate** to your `fed-truststore.jks`, and they must import yours.

### 8. Ignite Cluster Connection Delay
When Federation Hub starts, it needs to connect to TAK Server's Ignite cluster. This can take 30-60 seconds. Don't panic if ports aren't immediately listening.

### 9. No systemd Service by Default
Federation Hub uses SysV init scripts, not systemd. Use:
- `/etc/init.d/federation-hub-broker start` (NOT `systemctl start`)
- Check status with `ps aux | grep federation-hub`

### 10. Database Password Required
Even if not using MongoDB, the `dbPassword` field in `federation-hub-broker.yml` must be set (can be same as PostgreSQL password).

---

## Testing Federation

### Access Federation Hub UI

1. Open a web browser with the admin certificate (admin.p12) installed
2. Navigate to: `https://YOUR_SERVER_IP:9100/index.html`
3. Select the admin certificate when prompted
4. You should see the Federation Hub management interface

### Verify Outgoing Connection

Check if your server is attempting to connect to the remote server:

```bash
sudo tail -f /opt/tak/federation-hub/logs/federation-hub-broker-console.log | grep -i "connect"
```

### Monitor Federation Traffic

```bash
# Watch for incoming federation connections
sudo netstat -an | grep -E "9101|9102"

# Watch TAK Server messaging logs
sudo tail -f /opt/tak/logs/takserver-messaging.log | grep -i federation
```

---

## Additional Resources

- TAK Server documentation: https://tak.gov
- TAK Server Community: https://tak.gov/community
- For issues specific to TAK Server 5.5 on Ubuntu 24.04, see TAK_SERVER_5.5_COMPLETE_TUTORIAL.md

---

## Quick Reference Commands

```bash
# Start all federation services
sudo /etc/init.d/federation-hub-broker start
sudo /etc/init.d/federation-hub-policy start
sudo /etc/init.d/federation-hub-ui start

# Stop all federation services
sudo /etc/init.d/federation-hub-broker stop
sudo /etc/init.d/federation-hub-policy stop
sudo /etc/init.d/federation-hub-ui stop

# Check federation service status
ps aux | grep federation-hub | grep -v grep
sudo netstat -tlnp | grep -E "9100|9101|9102"

# View logs
sudo tail -f /opt/tak/federation-hub/logs/federation-hub-broker-console.log
sudo tail -f /opt/tak/logs/takserver-messaging.log

# Restart TAK Server
sudo systemctl restart takserver
```
