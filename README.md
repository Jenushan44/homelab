# Homelab

## Overview

This homelab is built on a Dell Precision 5820 and is used to gain hands-on experience with virtualization, Linux, Docker, networking, cybersecurity, and self-hosting.

## Infrastructure

- **Server:** Dell Precision 5820
- **Hypervisor:** Proxmox VE
- **Application Server:** Ubuntu Server
- **Containerization:** Docker and Docker Compose

## Deployments

### Soul Eater API

Full-stack application deployed using Docker Compose with a Next.js frontend, FastAPI backend, PostgreSQL database, and Nginx reverse proxy.

[View deployment documentation](docs/deployments/soul-eater-api/README.md)

## Documentation

- [Infrastructure Architecture](docs/infrastructure/architecture.md)
- [Networking](docs/infrastructure/networking.md)
- [Services](docs/infrastructure/services.md)
- [Proxmox Setup](docs/setup/proxmox.md)
- [Ubuntu Server Setup](docs/setup/ubuntu-server.md)

## Planned

- Personal cloud storage
- Windows Server
- Active Directory
- Kali Linux
- Wazuh security monitoring
- Additional self-hosted services
