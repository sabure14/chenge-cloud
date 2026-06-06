# Enterprise Home Data Center Infrastructure & Virtual Network

**Project Status:** Active / Production
**Core Stack:** OPNsense | Proxmox VE | Ubuntu | Docker | Tailscale

---

## 1. Problem Statement & Objectives

### The Challenge
The reliance on public cloud infrastructure for personal data management introduces privacy risks, ongoing subscription costs, and a lack of granular control over data sovereignty. Furthermore, studying cloud and network engineering requires a functional, hands-on environment to test routing, virtualization, and security models. Standard consumer networking hardware cannot provide the required routing capabilities, VLAN segregation, or edge firewalling needed to simulate an enterprise environment.

### The Objective
To engineer and deploy a self-hosted, enterprise-grade data center that provides highly available, secure access to critical microservices (credential management, media archiving, documentation, and media streaming). The architecture must enforce a zero-trust security model, ensuring that no internal IP addresses or services are exposed to the public internet while remaining accessible to authorized remote endpoints.

---

## 2. Architecture Design

The infrastructure is built on a tiered model, physically separating the edge routing boundary from the compute and storage layers.

### Physical Layer (Hardware)
*   **Edge Router / Firewall:** Topton N100 Micro-Appliance (10.42.10.1)
*   **Switching / Wireless Access:** GL.iNet Flint 2 configured in "Dumb AP" mode (10.42.10.2)
*   **Compute Hypervisor:** Beelink S12 Pro (10.42.10.11)
*   **WAN Gateway:** Arris S33 Cable Modem

### Virtualization & Compute Layer
*   **Hypervisor:** Proxmox VE
*   **Primary Workload Node:** Ubuntu Server VM (10.42.10.50)
*   **Containerization Engine:** Docker with Docker Compose for stack orchestration.

### Application Layer (Microservices)
The environment hosts several critical stateful services deployed as Docker containers:
*   **Vaultwarden:** Cryptographic credential and password management.
*   **Immich:** High-performance photo and video archiving.
*   **Joplin:** End-to-end encrypted technical documentation and note synchronization.
*   **Audiobookshelf:** Media ingestion and streaming pipeline.

---

## 3. Security & Isolation Strategy

The environment adheres to a strict zero-trust model, utilizing defense-in-depth methodologies. 

### Edge Security
*   **WAN Ingress:** OPNsense acts as the perimeter firewall. All inbound WAN ports are strictly blocked by default. No services are port-forwarded to the public internet.

### The Zero-Trust Overlay (Tailscale)
Instead of traditional perimeter VPNs, remote access is handled via a Tailscale mesh overlay network (`100.x.x.x`). 
*   **Encrypted Tunnels:** Endpoints (laptops, mobile devices) negotiate point-to-point WireGuard tunnels directly to the Ubuntu VM.
*   **Proxy Routing (`tailscale serve`):** Services are bound to internal ports. Tailscale's built-in proxy daemon listens on the tailnet interface and routes requests (e.g., HTTPS on Port 8081 for Vaultwarden) exclusively to authenticated tailnet endpoints, preventing lateral movement across the physical local area network (LAN).
*   **DNS Resolution:** MagicDNS is utilized for internal hostname resolution, completely bypassing external DNS leaks for local service discovery.

---

## 4. Lifecycle Management & Operations

To ensure high availability and prevent data loss, the environment utilizes strict lifecycle and disaster recovery protocols.

### Version Control & Deployment
*   **Image Pinning:** Volatile and highly active services (e.g., Immich) are pinned to specific version tags in their respective `docker-compose.yml` files to prevent breaking changes during automated pulls.
*   **Security Patching:** Critical security infrastructure (e.g., Vaultwarden, Tailscale) utilizes `:latest` tags or rapid manual update schedules to ensure immediate vulnerability patching.

### Automated Disaster Recovery
*   **Configuration Backups:** All Docker Compose files, environment variables, and routing configurations are tracked and backed up.
*   **Stateful Data:** Scheduled execution of `backup-stack.sh` automates the archiving of persistent volumes (e.g., the `vw-data` directory and SQLite databases). 
*   **Maintenance Windows:** Routine execution of `docker system prune -a` ensures host storage is optimized and prevents out-of-space write errors on the virtual disks.
