# Project Assignment 1 — Report
## Containerized Web Application with PostgreSQL using Docker Compose and IPvlan

**Name:** Shagun Saini  
**Stack:** FastAPI (Python) + PostgreSQL + IPvlan Networking  
**Host OS:** Windows 11 with WSL2 (Ubuntu)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture & Network Design](#2-architecture--network-design)
3. [Build Optimization Explanation](#3-build-optimization-explanation)
4. [Image Size Comparison](#4-image-size-comparison)
5. [Macvlan vs IPvlan Comparison](#5-macvlan-vs-ipvlan-comparison)
6. [Networking — WSL2 Host Isolation Issue](#6-networking--wsl2-host-isolation-issue)
7. [Proof of Working System](#7-proof-of-working-system)

---

## 1. Project Overview

This project containerizes a full-stack web application consisting of:

- **Backend:** FastAPI (Python) serving a REST API with three endpoints
- **Database:** PostgreSQL 15 with persistent named volume storage
- **Networking:** IPvlan (L2 mode) for container-level LAN IP assignment
- **Orchestration:** Docker Compose with health checks, restart policies, and dependency ordering

The system demonstrates production-ready Docker practices including multi-stage builds, non-root users, minimal base images, and environment-variable-based configuration.

---

## 2. Architecture & Network Design

### System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    WSL2 Host (eth0)                      │
│                  172.29.88.107/20                        │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Docker Environment                  │    │
│  │                                                  │    │
│  │  ┌──────────────────────┐                        │    │
│  │  │   fastapi_backend    │                        │    │
│  │  │  IPvlan: 172.29.80.10│◄──── Client Requests  │    │
│  │  │  Port: 8000          │      (via bridge)      │    │
│  │  │  Image: ~95MB        │                        │    │
│  │  └──────────┬───────────┘                        │    │
│  │             │ psycopg2 / port 5432                │    │
│  │  ┌──────────▼───────────┐                        │    │
│  │  │    postgres_db        │                        │    │
│  │  │  IPvlan: 172.29.80.11│                        │    │
│  │  │  Named Volume: pgdata │                        │    │
│  │  │  Image: ~75MB         │                        │    │
│  │  └──────────────────────┘                        │    │
│  │                                                  │    │
│  │  Networks:                                       │    │
│  │  ├── ipvlan_net  (external, L2, eth0 parent)     │    │
│  │  └── internal_bridge (bridge, host port binding) │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Network Topology

```
LAN Subnet: 172.29.80.0/20
Gateway:    172.29.80.1

┌──────────────┐     IPvlan L2      ┌─────────────────┐
│ fastapi_back │◄──────────────────►│   postgres_db   │
│ 172.29.80.10 │                    │  172.29.80.11   │
└──────┬───────┘                    └────────┬────────┘
       │                                     │
       │         internal_bridge             │
       └─────────────────────────────────────┘
                        │
               Port 8000 binding
                        │
                  WSL2 Host / Browser
```

### IPvlan Network Creation Command

```bash
docker network create \
  -d ipvlan \
  --subnet=172.29.80.0/20 \
  --gateway=172.29.80.1 \
  -o ipvlan_mode=l2 \
  -o parent=eth0 \
  ipvlan_net
```

---

## 3. Build Optimization Explanation

### Multi-Stage Build — Backend (FastAPI)

The backend Dockerfile uses a **two-stage build** to separate compilation from runtime:

```dockerfile
# Stage 1: builder
FROM python:3.11-alpine AS builder
RUN pip install --prefix=/install --no-cache-dir -r requirements.txt

# Stage 2: runtime
FROM python:3.11-alpine AS runtime
COPY --from=builder /install /usr/local
COPY main.py .
```

**Why this matters:**

| Concern | Without Multi-Stage | With Multi-Stage |
|--------|-------------------|-----------------|
| Build tools in image | gcc, musl-dev present | Excluded from final image |
| Image size | ~350MB+ | ~95MB |
| Attack surface | High | Minimal |
| Layer count | Many | Fewer |

### Optimizations Applied

**1. Alpine base images**
- `python:3.11-alpine` instead of `python:3.11` saves ~600MB
- `postgres:15-alpine` instead of `postgres:15` saves ~150MB

**2. `--no-cache-dir` flag**
```dockerfile
RUN pip install --prefix=/install --no-cache-dir -r requirements.txt
```
Prevents pip from storing download cache inside the image layer.

**3. Non-root user**
```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```
Security best practice — process runs with minimal OS privileges.

**4. `.dockerignore` files**
Prevents unnecessary files (`.git`, `__pycache__`, `.env`) from being sent to the build context, reducing build time and preventing secret leakage.

**5. Layer ordering — dependencies before code**
```dockerfile
COPY requirements.txt .          # Changes rarely → cached
RUN pip install ...              # Heavy step — cached after first build
COPY main.py .                   # Changes often → only this layer rebuilds
```
Docker caches layers from top to bottom. Placing stable files first maximizes cache reuse.

**6. `--prefix=/install` trick**
Installs Python packages to a custom prefix directory, making it trivial to `COPY --from=builder /install /usr/local` in the runtime stage — a clean surgical transfer of only the installed packages.

---

## 4. Image Size Comparison

### Before vs After Optimization

| Image | Base | Final Size | Savings |
|-------|------|-----------|---------|
| Backend (no optimization) | python:3.11 | ~980MB | — |
| Backend (alpine only) | python:3.11-alpine | ~220MB | ~760MB |
| Backend (alpine + multi-stage) | python:3.11-alpine | ~95MB | ~885MB |
| Database (default postgres) | postgres:15 | ~379MB | — |
| Database (alpine) | postgres:15-alpine | ~75MB | ~304MB |

### Check actual sizes after build:

```bash
docker images | grep fastapi-postgres-app
```

Expected output:
```
fastapi-postgres-app-backend   latest   <id>   95MB
fastapi-postgres-app-db        latest   <id>   75MB
```

### Why Image Size Matters in Production

- Smaller images = faster CI/CD pull times
- Less storage cost in container registries (Docker Hub, ECR, GCR)
- Reduced attack surface — fewer packages means fewer CVEs
- Faster container startup and scaling

---

## 5. Macvlan vs IPvlan Comparison

Both drivers allow containers to appear as physical devices on the network with their own MAC/IP addresses, bypassing the Docker bridge. However, they differ fundamentally in how they operate at Layer 2.

### Comparison Table

| Feature | Macvlan | IPvlan (L2) |
|---------|---------|-------------|
| **Layer** | L2 (Data Link) | L2 or L3 (configurable) |
| **MAC Address** | Each container gets unique MAC | Containers share host MAC |
| **IP Assignment** | Unique IP per container | Unique IP per container |
| **Host Isolation** | Host cannot reach containers by default | Same isolation issue on WSL2 |
| **Promiscuous Mode** | Required on NIC | Not required |
| **DHCP Support** | Yes (unique MACs) | Limited (shared MAC) |
| **Switch CAM Table** | Multiple entries (one per container) | Single entry (host MAC) |
| **Cloud Support** | Often blocked (AWS, Azure disable it) | Better support in some clouds |
| **WSL2 Support** | Partial | Partial |
| **Use Case** | Legacy apps needing unique MACs | High-density deployments, cloud |

### When to Choose Each

**Use Macvlan when:**
- Containers need to respond to ARP (unique MAC required)
- Running DHCP clients inside containers
- Network equipment needs per-container MAC visibility
- Running on bare-metal Linux with full NIC access

**Use IPvlan when:**
- NIC or switch does not support promiscuous mode
- Cloud environments where unique MACs are blocked
- High container density (no MAC table exhaustion risk)
- L3 mode needed for routing between subnets

### Host Isolation Issue (Macvlan-specific)

With Macvlan, the **host cannot communicate with its own containers** even though other LAN devices can. This happens because the Linux kernel blocks traffic between the physical interface and its Macvlan sub-interfaces at the driver level.

**The workaround** is to create a Macvlan interface on the host itself:

```bash
# Create a macvlan interface on the host
ip link add macvlan-host link eth0 type macvlan mode bridge
ip addr add 172.29.80.1/20 dev macvlan-host
ip link set macvlan-host up

# Add route so host can reach containers
ip route add 172.29.80.0/20 dev macvlan-host
```

IPvlan has the same WSL2 host isolation behavior — see Section 6 for details.

---

## 6. Networking — WSL2 Host Isolation Issue

### The Problem

On **WSL2**, both Macvlan and IPvlan networks suffer from host-to-container isolation. The WSL2 virtual network adapter (`eth0`) is a virtualized Hyper-V interface. When IPvlan sub-interfaces are created on it, the Hyper-V switch drops traffic from the WSL2 host to those sub-interfaces because:

1. Hyper-V enforces MAC address filtering
2. IPvlan containers share the host MAC but use different IPs
3. The Hyper-V virtual switch does not route between the host vNIC and IPvlan sub-interfaces

### The Solution Used in This Project

A **dual-network architecture** was implemented:

```yaml
networks:
  ipvlan_net:       # For static IP assignment (assignment requirement)
    external: true
  internal_bridge:  # For host access via port binding
    driver: bridge
```

Both containers are attached to **both networks simultaneously**:
- `ipvlan_net` → satisfies the assignment requirement, containers have static LAN IPs
- `internal_bridge` + `ports: 8000:8000` → enables `localhost:8000` access from WSL2 host

This mirrors a real production pattern where containers may have both an internal management network and an externally-accessible network interface.

### Verification

```bash
# Confirm ipvlan IP is assigned
docker inspect fastapi_backend | grep IPAddress
# Output: "IPAddress": "172.29.80.10"

docker inspect postgres_db | grep IPAddress  
# Output: "IPAddress": "172.29.80.11"
```

---

## 7. Proof of Working System

### API Endpoints

| Method | Endpoint | Function |
|--------|----------|----------|
| GET | `/health` | Returns `{"status": "ok"}` |
| POST | `/records` | Inserts a record into PostgreSQL |
| GET | `/records` | Returns all records from PostgreSQL |

### Test Commands & Expected Output

```bash
# Health check
curl http://localhost:8000/health
# → {"status":"ok"}

# Insert record
curl -X POST http://localhost:8000/records \
  -H "Content-Type: application/json" \
  -d '{"name": "test", "value": "hello world"}'
# → {"id":1,"name":"test","value":"hello world"}

# Fetch all records
curl http://localhost:8000/records
# → [{"id":1,"name":"test","value":"hello world"}]
```

### Volume Persistence Test

```bash
# Bring stack down and back up
docker compose down
docker compose up -d
sleep 20

# Records should still be present
curl http://localhost:8000/records
# → [{"id":1,...},{"id":2,...}]  ← data survived restart ✓
```

### Network Inspect Command

```bash
docker network inspect ipvlan_net
```

### Docker Compose Status

```bash
docker compose ps
# NAME              STATUS              PORTS
# fastapi_backend   Up (healthy)        0.0.0.0:8000->8000/tcp
# postgres_db       Up (healthy)        5432/tcp
```

---

## Repository Structure

```
fastapi-postgres-app/
├── backend/
│   ├── Dockerfile          # Multi-stage Alpine build
│   ├── main.py             # FastAPI app with 3 endpoints
│   ├── requirements.txt    # Pinned dependencies
│   └── .dockerignore
├── database/
│   ├── Dockerfile          # Custom postgres:15-alpine image
│   └── .dockerignore
├── docker-compose.yml      # Full stack orchestration
├── .env                    # Environment variables (gitignored)
├── .gitignore
└── REPORT.md               # This report
```

---

*Report prepared for Project Assignment 1 — Docker Containerization & Networking*

