# TAK Server 5.5 Installation for Ubuntu 24.04

This repository contains comprehensive guides for installing and configuring TAK Server 5.5 on Ubuntu 24.04.

## Important Gotchas / Known Issues

### Critical: PostgreSQL Version MUST be 15

**Ubuntu 24.04 may try to install PostgreSQL 16 by default, which will cause TAK Server to fail!**

- TAK Server 5.5 **requires PostgreSQL 15** specifically
- Ubuntu 24.04's default apt repository often provides PostgreSQL 16
- Using PostgreSQL 16 will cause database connection errors and server failures
- **Solution**: You MUST explicitly specify PostgreSQL 15 during installation using the PostgreSQL APT repository

See the [PostgreSQL Installation section](TAK_SERVER_5.5_COMPLETE_TUTORIAL.md#postgresql-15-installation) in the tutorial for the correct installation commands.

### Critical: OpenJDK 17 is REQUIRED

**Do NOT remove OpenJDK 17 completely!**

- The TAK Server .deb package has dependencies on OpenJDK 17 packages
- While we use Temurin JDK 17 as the runtime, the OpenJDK 17 packages must be installed to satisfy dependencies
- Removing OpenJDK 17 will cause the .deb installation to fail
- **Solution**: Install both Temurin JDK 17 (for runtime) AND OpenJDK 17 packages (for dependencies)

The tutorial correctly shows installing both - follow it as written.

## Quick Start

Choose the guide that fits your needs:

1. **[Complete Tutorial](TAK_SERVER_5.5_COMPLETE_TUTORIAL.md)** - Step-by-step installation with all commands
   - Best for: First-time installers
   - Includes: Exact commands, verification steps, troubleshooting
   - Time: 1-2 hours

2. **[Installation Guide](TAK_SERVER_5.5_INSTALLATION_GUIDE.md)** - Comprehensive guide with detailed explanations
   - Best for: Understanding the architecture and configuration
   - Includes: Detailed explanations, certificate setup, advanced configuration
   - Time: 2-3 hours

## What's Included

- Full Java 17 setup (Temurin + OpenJDK)
- PostgreSQL 15 database configuration
- Certificate Authority (CA) generation
- TLS/SSL security setup
- Multicast support for local network discovery
- Video streaming capabilities
- File sharing and data packages
- Mission API for planning and collaboration
- Federation framework for connecting multiple TAK servers

## System Requirements

- Ubuntu 24.04 LTS
- Minimum 8GB RAM (16GB recommended)
- 40GB+ disk space
- Static IP address recommended
- TAK Server 5.5 .deb package from [tak.gov](https://tak.gov)

## Before You Start

1. **Read the gotchas above** - they will save you hours of troubleshooting!
2. Download TAK Server 5.5 from tak.gov (requires account)
3. Ensure you have sudo access
4. Have your server's IP address ready

## Getting Help

- Check the [Troubleshooting section](TAK_SERVER_5.5_COMPLETE_TUTORIAL.md#troubleshooting) in the tutorial
- Review TAK Server logs: `/opt/tak/logs/`
- Official TAK documentation: [tak.gov](https://tak.gov)

## Contributing

Found an issue or have improvements? Please submit a pull request or open an issue.

## License

These guides are provided as-is for educational and operational use.

---

**Last Updated**: 2025-10-22
**Tested On**: Ubuntu 24.04.3 LTS
**TAK Server Version**: 5.5-RELEASE-58
