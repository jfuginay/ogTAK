# OpenLDAP Setup Guide for TAK Server

Complete guide for setting up OpenLDAP authentication with TAK Server using Docker.

## Overview

This guide shows how to set up OpenLDAP in Docker for TAK Server authentication, allowing centralized user management instead of managing individual certificates.

## Prerequisites

- TAK Server 5.5 installed and running
- Docker and Docker Compose installed
- Ubuntu 24.04 LTS (or similar Linux distribution)

## Quick Start

### 1. Create OpenLDAP Docker Setup

```bash
# Create directory
mkdir -p ~/openldap-docker
cd ~/openldap-docker

# Create docker-compose.yml (see below)
```

### 2. Docker Compose Configuration

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  openldap:
    image: osixia/openldap:1.5.0
    container_name: openldap
    hostname: openldap
    restart: unless-stopped
    environment:
      LDAP_ORGANISATION: "TAK Organization"
      LDAP_DOMAIN: "tak.local"
      LDAP_ADMIN_PASSWORD: "admin123"
      LDAP_CONFIG_PASSWORD: "config123"
      LDAP_READONLY_USER: "false"
      LDAP_RFC2307BIS_SCHEMA: "false"
      LDAP_BACKEND: "mdb"
      LDAP_TLS: "true"
      LDAP_TLS_ENFORCE: "false"
    ports:
      - "389:389"
      - "636:636"
    volumes:
      - openldap-data:/var/lib/ldap
      - openldap-config:/etc/ldap/slapd.d
      - openldap-certs:/container/service/slapd/assets/certs/
    networks:
      - ldap-network

  phpldapadmin:
    image: osixia/phpldapadmin:0.9.0
    container_name: phpldapadmin
    hostname: phpldapadmin
    restart: unless-stopped
    environment:
      PHPLDAPADMIN_LDAP_HOSTS: "openldap"
      PHPLDAPADMIN_HTTPS: "false"
    ports:
      - "8080:80"
    depends_on:
      - openldap
    networks:
      - ldap-network

volumes:
  openldap-data:
    name: openldap-data
  openldap-config:
    name: openldap-config
  openldap-certs:
    name: openldap-certs

networks:
  ldap-network:
    name: ldap-network
    driver: bridge
```

### 3. Create Bootstrap Users

Create `bootstrap.ldif`:

```ldif
# Create Organizational Units
dn: ou=people,dc=tak,dc=local
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=tak,dc=local
objectClass: organizationalUnit
ou: groups

# Create Groups
dn: cn=tak-users,ou=groups,dc=tak,dc=local
objectClass: groupOfNames
cn: tak-users
description: TAK Users Group
member: cn=admin,dc=tak,dc=local

dn: cn=tak-admins,ou=groups,dc=tak,dc=local
objectClass: groupOfNames
cn: tak-admins
description: TAK Administrators Group
member: cn=admin,dc=tak,dc=local

# Create Users
dn: cn=john.doe,ou=people,dc=tak,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: john.doe
sn: Doe
givenName: John
uid: john.doe
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/john.doe
loginShell: /bin/bash
mail: john.doe@tak.local
userPassword: password123
description: TAK User - John Doe

dn: cn=jane.smith,ou=people,dc=tak,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: jane.smith
sn: Smith
givenName: Jane
uid: jane.smith
uidNumber: 10002
gidNumber: 10002
homeDirectory: /home/jane.smith
loginShell: /bin/bash
mail: jane.smith@tak.local
userPassword: password123
description: TAK Administrator - Jane Smith
```

### 4. Start OpenLDAP

```bash
# Start containers
docker compose up -d

# Wait for startup (about 10 seconds)
sleep 10

# Import bootstrap data
docker cp bootstrap.ldif openldap:/tmp/bootstrap.ldif
docker exec openldap ldapadd -x -D "cn=admin,dc=tak,dc=local" -w admin123 -f /tmp/bootstrap.ldif

# Verify import
docker exec openldap ldapsearch -x -b "dc=tak,dc=local" -D "cn=admin,dc=tak,dc=local" -w admin123 | grep "dn:"
```

### 5. Configure TAK Server

#### Via Web UI (Recommended)

1. Open TAK Server web UI: https://YOUR_SERVER_IP:8443
2. Navigate to **Settings → Security**
3. Scroll to **Authentication Configuration (LDAP)**
4. Fill in:
   - **URL**: `ldap://YOUR_SERVER_IP:389`
   - **User String**: `ou=people,dc=tak,dc=local`
   - **Service Account DN**: `cn=admin,dc=tak,dc=local`
   - **Service Account Password**: `admin123`
   - **Group Base RDN**: `ou=groups,dc=tak,dc=local`
   - **Group Prefix**: `cn=`
   - **Update Interval**: `600000`
5. Click **"Test Service Account"** - should show success
6. Click **"Save"**

TAK Server will automatically reload the configuration.

## Access Points

### phpLDAPadmin Web Interface
- **URL**: http://localhost:8080
- **Login DN**: `cn=admin,dc=tak,dc=local`
- **Password**: `admin123`

### TAK Server Web UI
- **URL**: https://YOUR_SERVER_IP:8443
- Use admin certificate (admin.p12) for access

## Test Users

| Username | Password | Email | DN |
|----------|----------|-------|-----|
| john.doe | password123 | john.doe@tak.local | cn=john.doe,ou=people,dc=tak,dc=local |
| jane.smith | password123 | jane.smith@tak.local | cn=jane.smith,ou=people,dc=tak,dc=local |

## Managing Users

### Add User via phpLDAPadmin (Easy)

1. Open http://localhost:8080
2. Login with admin credentials
3. Navigate to `ou=people,dc=tak,dc=local`
4. Click "Create new entry here"
5. Select template: "Generic: User Account"
6. Fill in user details:
   - Common Name (cn): username
   - Surname (sn): Last name
   - Given Name: First name
   - User ID (uid): username
   - Email: user@tak.local
   - Password: (set password)
7. Click "Create Object"

### Add User via Command Line

Create `newuser.ldif`:

```ldif
dn: cn=new.user,ou=people,dc=tak,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: new.user
sn: User
givenName: New
uid: new.user
uidNumber: 10003
gidNumber: 10003
homeDirectory: /home/new.user
loginShell: /bin/bash
mail: new.user@tak.local
userPassword: password123
description: New TAK User
```

Import:
```bash
docker exec -i openldap ldapadd -x -D "cn=admin,dc=tak,dc=local" -w admin123 < newuser.ldif
```

### List All Users

```bash
docker exec openldap ldapsearch -x -b "ou=people,dc=tak,dc=local" -D "cn=admin,dc=tak,dc=local" -w admin123
```

### Test User Authentication

```bash
docker exec openldap ldapwhoami -x -D "cn=john.doe,ou=people,dc=tak,dc=local" -w password123
```

## TAK Client Configuration

### ATAK/WinTAK/iTAK Settings

When connecting a TAK client:

1. **Server Address**: YOUR_SERVER_IP
2. **Port**: 8089
3. **Protocol**: TLS
4. **Authentication**: Username/Password
5. **Username**: `john.doe`
6. **Password**: `password123`

No certificate needed - LDAP handles authentication.

## Docker Commands

```bash
# Start OpenLDAP
cd ~/openldap-docker
docker compose up -d

# Stop OpenLDAP
docker compose down

# View logs
docker compose logs -f openldap

# Restart after changes
docker compose restart openldap

# Check status
docker compose ps
```

## Troubleshooting

### LDAP Connection Test

```bash
# Test basic connection
ldapsearch -x -H ldap://localhost:389 -b "dc=tak,dc=local" -D "cn=admin,dc=tak,dc=local" -w admin123

# Test user authentication
docker exec openldap ldapwhoami -x -D "cn=john.doe,ou=people,dc=tak,dc=local" -w password123
```

### TAK Server LDAP Issues

1. **Check TAK Server logs:**
   ```bash
   sudo tail -f /opt/tak/logs/takserver-messaging.log | grep -i ldap
   sudo tail -f /opt/tak/logs/takserver-api.log | grep -i ldap
   ```

2. **Verify LDAP is running:**
   ```bash
   docker compose ps
   ```

3. **Test from TAK Server host:**
   ```bash
   ldapsearch -x -H ldap://localhost:389 -b "dc=tak,dc=local" -D "cn=admin,dc=tak,dc=local" -w admin123
   ```

### Reset LDAP Data

```bash
# WARNING: This deletes all LDAP data!
docker compose down -v
docker compose up -d
# Re-import bootstrap.ldif
```

## Backup and Restore

### Backup LDAP Data

```bash
# Backup all entries
docker exec openldap slapcat -v -l ~/ldap-backup-$(date +%Y%m%d).ldif

# Backup specific OU
docker exec openldap ldapsearch -x -b "ou=people,dc=tak,dc=local" -D "cn=admin,dc=tak,dc=local" -w admin123 > ~/ldap-users-backup-$(date +%Y%m%d).ldif
```

### Restore LDAP Data

```bash
# Stop OpenLDAP
docker compose down

# Clear old data
docker volume rm openldap-data

# Start fresh
docker compose up -d
sleep 10

# Restore from backup
docker exec -i openldap slapadd < ~/ldap-backup-YYYYMMDD.ldif

# Restart
docker compose restart openldap
```

## Security Considerations

**This is a TEST/DEVELOPMENT setup!**

For production:

1. **Change default passwords:**
   - Admin password in docker-compose.yml
   - User passwords from `password123`

2. **Use LDAPS (port 636):**
   - Configure SSL certificates
   - Update TAK Server URL to `ldaps://`

3. **Restrict network access:**
   ```bash
   # Firewall rules
   sudo ufw allow from 192.168.0.0/24 to any port 389
   sudo ufw deny 389
   ```

4. **Strong passwords:**
   - Enforce password complexity
   - Set password expiration policies

5. **Regular backups:**
   - Automate LDAP backups
   - Test restore procedures

## Advanced Configuration

### Add User to Group

```ldif
dn: cn=tak-admins,ou=groups,dc=tak,dc=local
changetype: modify
add: member
member: cn=john.doe,ou=people,dc=tak,dc=local
```

Apply:
```bash
docker exec -i openldap ldapmodify -x -D "cn=admin,dc=tak,dc=local" -w admin123 < add-to-group.ldif
```

### Change User Password

Via phpLDAPadmin:
1. Navigate to user
2. Click "Password" link
3. Enter new password
4. Click "Update Password"

Via command line:
```bash
docker exec openldap ldappasswd -x -D "cn=admin,dc=tak,dc=local" -w admin123 -s newpassword123 "cn=john.doe,ou=people,dc=tak,dc=local"
```

## Integration with TAK Server Groups

In TAK Server web UI:

1. Go to **Settings → Groups**
2. Map LDAP groups to TAK permissions:
   - `tak-admins` → Full admin access
   - `tak-users` → Standard user access
3. Users in LDAP groups automatically get corresponding TAK permissions

## Reference

### LDAP Structure

```
dc=tak,dc=local
├── ou=people (users)
│   ├── cn=john.doe
│   ├── cn=jane.smith
│   └── (other users)
└── ou=groups
    ├── cn=tak-users
    └── cn=tak-admins
```

### Default Credentials

| Service | Username/DN | Password |
|---------|-------------|----------|
| LDAP Admin | cn=admin,dc=tak,dc=local | admin123 |
| LDAP Config | cn=admin,cn=config | config123 |
| Test User 1 | john.doe | password123 |
| Test User 2 | jane.smith | password123 |

### Ports

| Port | Service | Protocol |
|------|---------|----------|
| 389 | LDAP | TCP |
| 636 | LDAPS | TCP/SSL |
| 8080 | phpLDAPadmin | HTTP |
| 8089 | TAK Server | TLS |
| 8443 | TAK Server Web UI | HTTPS |

## Next Steps

1. ✅ Change default passwords
2. ✅ Add real users via phpLDAPadmin
3. ✅ Test authentication from TAK client
4. ✅ Configure group-based permissions
5. ✅ Set up regular backups
6. ✅ Consider LDAPS for production

## Additional Resources

- OpenLDAP Documentation: https://www.openldap.org/doc/
- phpLDAPadmin: http://phpldapadmin.sourceforge.net/
- TAK Server Documentation: https://tak.gov

## Support

For issues:
- Check logs: `docker compose logs openldap`
- Verify connectivity: Test LDAP connection from TAK Server host
- Review TAK Server logs for LDAP errors

---

**Status**: Development/Testing Setup
**Security Level**: Low (change passwords for production!)
**Tested On**: Ubuntu 24.04 LTS with TAK Server 5.5-RELEASE-58
