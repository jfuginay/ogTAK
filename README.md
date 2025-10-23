# TAK Server 5.5 Installation and Federation Guides

Complete guides for installing and configuring TAK Server 5.5 on Ubuntu 24.04 LTS, including federation setup.

## Overview

This repository contains step-by-step installation and configuration guides for:
- TAK Server 5.5 installation on Ubuntu 24.04
- Federation Hub setup and configuration
- Federation with OpenTAKServer or other TAK servers

**Tested On:**
- Ubuntu 24.04.3 LTS (Noble Numbat)
- TAK Server 5.5-RELEASE-58
- TAK Server Federation Hub 5.5-RELEASE-58

## Prerequisites

Before starting, you need to:
1. Download TAK Server packages from https://tak.gov (requires registration)
   - `takserver_5.5-RELEASE58_all.deb`
   - `takserver-fed-hub_5.5-RELEASE58_all.deb` (for federation)
2. Ubuntu 24.04 LTS system with:
   - Minimum 8GB RAM
   - 40GB+ disk space
   - Internet connection
   - sudo access

**Important:** This repository does not include any TAK Server SDK files, packages, or proprietary software. All TAK Server components must be downloaded directly from tak.gov.

## Guides

### 1. TAK Server Installation

**[TAK_SERVER_5.5_COMPLETE_TUTORIAL.md](TAK_SERVER_5.5_COMPLETE_TUTORIAL.md)**
- Complete step-by-step installation guide
- Covers Java 17, PostgreSQL 15, TAK Server 5.5
- Certificate generation and configuration
- Firewall setup
- Troubleshooting common issues

**[TAK_SERVER_5.5_INSTALLATION_GUIDE.md](TAK_SERVER_5.5_INSTALLATION_GUIDE.md)**
- Alternative installation reference
- Quick reference for experienced users

### 2. Federation Setup

**[FEDERATION_SETUP.md](FEDERATION_SETUP.md)**
- Complete Federation Hub installation guide
- Certificate configuration for federation
- Connecting to OpenTAKServer or other TAK servers
- Troubleshooting federation issues
- Key gotchas and common pitfalls

## Quick Start

1. **Install TAK Server** (follow complete tutorial first):
   ```bash
   # See TAK_SERVER_5.5_COMPLETE_TUTORIAL.md for full steps
   sudo dpkg -i takserver_5.5-RELEASE58_all.deb
   ```

2. **Install Federation Hub** (if federating with other servers):
   ```bash
   # See FEDERATION_SETUP.md for complete instructions
   sudo dpkg -i takserver-fed-hub_5.5-RELEASE58_all.deb
   ```

3. **Access TAK Server**:
   - Web UI: `https://YOUR_SERVER_IP:8443`
   - Federation UI: `https://YOUR_SERVER_IP:9100`

## Key Components

### TAK Server Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 8089 | TCP/TLS | TAK Server client connections |
| 8443 | HTTPS | Web administration interface |
| 8444 | HTTPS | Federation HTTPS (with cert) |
| 8446 | HTTPS | Certificate-based HTTPS |
| 6969 | UDP | Multicast (SA awareness) |

### Federation Hub Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 9000 | TCP/TLS | Federation V1 server port |
| 9001 | TCP/TLS | Federation V2 server port |
| 9100 | HTTPS | Federation Hub web UI |
| 9101 | TCP/TLS | Federation Hub V1 broker |
| 9102 | TCP/TLS | Federation Hub V2 broker |

## Important Notes

### Security

- **Never commit certificates or private keys to version control**
- Default passwords should be changed in production
- Certificates in this repo's `.gitignore` are excluded for security
- Review firewall rules before exposing to internet

### Package Downloads

All TAK Server packages must be downloaded from:
- https://tak.gov

Registration is required. We cannot distribute TAK Server software.

### Federation

- Federation Hub is a **separate package** from TAK Server
- Both servers must trust each other's certificates
- Firewall must allow inbound connections on federation ports
- See FEDERATION_SETUP.md for complete details and gotchas

## Troubleshooting

### Common Issues

1. **Java version conflicts**: TAK Server 5.5 requires Java 17
2. **PostgreSQL connection**: Verify PostgreSQL 15 is running
3. **Certificate issues**: Regenerate certificates if expired
4. **Federation not connecting**: Check firewall, certificates, and port configuration

See individual guides for detailed troubleshooting steps.

## Directory Structure

```
.
├── README.md                                    # This file
├── TAK_SERVER_5.5_COMPLETE_TUTORIAL.md         # Full installation guide
├── TAK_SERVER_5.5_INSTALLATION_GUIDE.md        # Alternative reference
├── FEDERATION_SETUP.md                          # Federation guide
└── .gitignore                                   # Security exclusions
```

## Contributing

Issues, corrections, and improvements are welcome. Please ensure:
- No sensitive data (certificates, keys, passwords) is included
- Commands are tested on Ubuntu 24.04
- Documentation is clear and step-by-step

## Disclaimer

This is an unofficial community guide. For official TAK Server documentation and support:
- Official documentation: https://tak.gov
- TAK.gov community forums: https://tak.gov/community

TAK Server is government software. All TAK Server components and packages must be obtained from official sources.

## License

These guides are provided as-is for educational purposes. TAK Server software is subject to its own licensing terms from tak.gov.

## Version History

- **v1.1** - Added Federation Hub setup guide with detailed gotchas
- **v1.0** - Initial TAK Server 5.5 installation guide for Ubuntu 24.04
