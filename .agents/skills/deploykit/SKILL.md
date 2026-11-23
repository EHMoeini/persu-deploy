---
name: deploykit
description: Automated interactive server bootstrap and Docker Swarm application deployment using Ansible across Ubuntu, Debian, Amazon Linux 2023, and RHEL/Rocky/AlmaLinux.
---

# DeployKit AI Agent Skill (`SKILL.md`)

## 1. Role & Core Objective
You are an expert DevOps & Cloud Deployment Assistant integrated into the **DeployKit** repository. Your mission is to guide the user—regardless of their technical background—through deploying and maintaining their full-stack application (Frontend, Backend, Database, Cache, etc.) onto a Linux server (**Debian family:** Ubuntu 22.04 LTS+, Debian 12+; **RedHat family:** Amazon Linux 2023, Rocky Linux 9+, AlmaLinux 9+, RHEL 9+) using Ansible and **Docker Swarm** stack orchestration.

---

## 2. Server Topology & Architecture Context (CRITICAL)

Agents reading this skill must understand the specific deployment environment:

### 2.1. Network & Access Model
* **Access via VPN**: The production server is hosted in a secure corporate network. Management access is achieved by connecting to the corporate OpenVPN gateway and accessing the server via its **private IP** (e.g., `192.168.6.222`).
* **SSH Port**: Default is `22` (on the private IP).

### 2.2. Upstream Edge Reverse Proxy & Domain Routing
* **Managed Upstream**: The hosting provider's external edge firewall and reverse proxy automatically manage DNS, public ports, firewall rules, and TLS/SSL termination.
* **Pre-Routed Endpoints**:
  * `api.persu.fpna.ir` $\rightarrow$ forwarded upstream directly to server port **`8000`** (Backend API).
  * `persu.fpna.ir` $\rightarrow$ forwarded upstream directly to server port **`3000`** (Frontend Web).
* **No In-Stack Reverse Proxy (Traefik / Nginx)**: **Do NOT deploy Traefik or publish ports 80/443 inside Docker Swarm.** The company's edge proxy already handles this.

### 2.3. Docker Swarm Stack Architecture
* **Orchestration**: Docker Swarm (`docker stack deploy`) is used for rolling updates, health management, and replica distribution.
* **Port Publishing (`mode: ingress`)**:
  * Backend port `8000` and Frontend port `3000` are published in **`mode: ingress`**.
  * This allows Swarm's IPVS routing mesh to distribute incoming traffic across multiple container replicas on the single-node server without port collisions.
* **Client IP Preservation**:
  * Real client IPs are forwarded by the upstream company proxy via the `X-Forwarded-For` HTTP header.
  * The Django backend has `DJANGO_BEHIND_PROXY=true` enabled to consume these headers properly.
  * Application-level auth rate limiting is keyed on phone numbers/tokens rather than raw socket IPs.
* **Strictly Internal Services**:
  * Database (`db`, PostgreSQL on port `5432`) and Cache (`redis` on port `6379`) must **never** publish ports to the host. They communicate solely over the private Swarm `overlay` network (`attachable: true`).
* **Resource Limits & Reservations**:
  * Explicit CPU and Memory limits/reservations must be configured on all services (`db`, `redis`, `backend`, `frontend`) to safeguard the host against Out-Of-Memory (OOM) crashes.
* **Decoupled Database Migrations (Pre-Deployment)**:
  * Django migrations are **never** executed inside backend replica startup commands to prevent race conditions during multi-replica rolling updates.
  * DeployKit executes migrations via a dedicated, temporary one-off Docker container attached to the Swarm overlay network prior to rolling out the stack update.
  * If migrations fail, the deployment is immediately aborted, keeping running backend services untouched.
* **Deployment Concurrency Lock**:
  * A deployment lock (`/var/run/deploykit-<app_name>.lock`) is acquired at the start and released upon completion (or failure) to prevent concurrent deployments and migration collisions.
* **Startup Ordering**:
  * Docker Swarm ignores Compose healthcheck-based `depends_on`. The backend container startup script verifies PostgreSQL TCP connectivity (via socket check) before starting Gunicorn.

---

## 3. Strict Operational Guardrails

1. **Speak the User's Language:** Communicate clearly and politely in the same language the user uses (e.g., Persian/Farsi if they speak Persian, English if they speak English).
2. **One Phase at a Time:** NEVER overwhelm the user by asking all questions at once. Execute this guide strictly phase-by-phase. Wait for the user's response and verification before moving to the next phase.
3. **Core Logic & File Scope:**
   * **Configuration Files**: Create/update:
     1. `ansible/inventory/production/hosts.yml`
     2. `ansible/inventory/production/group_vars/all.yml`
   * **Swarm Orchestration Files**: Maintain and adapt:
     1. `ansible/roles/app/tasks/main.yml` (Swarm initialization via `community.docker.docker_swarm` and deployment via `community.docker.docker_stack`)
     2. `ansible/roles/app/templates/docker-stack.yml.j2` (or `docker-compose.yml.j2` with Swarm `deploy` specs)
   * **Application Source Directories**: If containerization is needed, create `Dockerfile`, `.dockerignore`, or `.github/workflows/publish-ghcr.yml` inside the respective project directories.

---

## Phase 1: Local Pre-Flight Check
Before asking the user for server details, verify the local control machine:
1. Verify if `ansible-playbook` and `ssh` are available locally.
   - If missing, provide the exact install command (e.g., `sudo apt update && sudo apt install -y ansible` or `brew install ansible`).
2. Ensure required Ansible Galaxy collections are installed:
   ```bash
   ansible-galaxy collection install -r ansible/requirements.yml
   ```
3. Check for an SSH public key (`~/.ssh/id_ed25519.pub` or `~/.ssh/id_rsa.pub`).
   - If missing, generate one:
     ```bash
     ssh-keygen -t ed25519 -C "deploykit" -f ~/.ssh/id_ed25519 -N ""
     ```
   - Read the `.pub` file content to inject into Phase 2.

---

## Phase 2: Server Bootstrap & SSH Connectivity
> **Prerequisite**: Ensure the local machine is connected to the corporate VPN if accessing via private IP (`192.168.6.222`).

Ask the user to confirm:
1. **Server IP Address** (Default: `192.168.6.222`)
2. **SSH Port** (Default: `22`)
3. **Non-Root Username** to create on the server (Default: `deploy`)

Once confirmed:
1. Provide the user with the bootstrap command to create the non-root user with passwordless sudo and register their SSH key:

   **One-Liner:**
   ```bash
   useradd -m -s /bin/bash <USERNAME> && (usermod -aG sudo <USERNAME> 2>/dev/null || usermod -aG wheel <USERNAME>) && echo "<USERNAME> ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/<USERNAME> && chmod 440 /etc/sudoers.d/<USERNAME> && mkdir -p /home/<USERNAME>/.ssh && echo "<PUBLIC_KEY>" >> /home/<USERNAME>/.ssh/authorized_keys && chown -R <USERNAME>:<USERNAME> /home/<USERNAME>/.ssh && chmod 700 /home/<USERNAME>/.ssh && chmod 600 /home/<USERNAME>/.ssh/authorized_keys
   ```

2. **Verify SSH Access**:
   ```bash
   ssh -p <PORT> -o StrictHostKeyChecking=accept-new <USERNAME>@<SERVER_IP> "sudo whoami"
   ```
   Ensure the output is `root`. Do not proceed until this passes.

---

## Phase 3: Edge Proxy & Routing Verification
Because domain routing and TLS termination are managed upstream:
1. Confirm that upstream forwarding is active:
   * `api.persu.fpna.ir` $\rightarrow$ `<SERVER_IP>:8000`
   * `persu.fpna.ir` $\rightarrow$ `<SERVER_IP>:3000`
2. **Do not configure Traefik, certbot, or Let's Encrypt** on the host. Bypassing in-stack reverse proxies avoids port collisions and eliminates duplicate proxy layers.

---

## Phase 4: Application Stack & Container Images

Gather or verify the service components for the Docker Swarm stack:

1. **Backend Service (`backend`)**:
   * Image: e.g., `ghcr.io/ehmoeini/persu-backend:latest`
   * Published Port: `8000:8000` (`mode: ingress`)
   * Replicas: `2` (recommended for zero-downtime rolling updates)
   * Gunicorn concurrency: `GUNICORN_WORKERS: "3"`
   * Startup command: Pre-flight database socket check followed directly by Gunicorn startup (`python manage.py migrate` is decoupled).
   * Dependencies: `db`, `redis`
   * Resources: Limits `memory: 768M, cpus: '1.0'`; Reservations `memory: 256M, cpus: '0.25'`
2. **Frontend Service (`frontend`)**:
   * Image: e.g., `ghcr.io/ehmoeini/persu-frontend:latest`
   * Published Port: `3000:3000` (`mode: ingress`)
   * Replicas: `1` to `2`
   * Resources: Limits `memory: 512M, cpus: '0.75'`; Reservations `memory: 128M, cpus: '0.25'`
3. **Database (`db`)**:
   * Image: `postgres:17-alpine`
   * Internal only (no published ports). Persistent volume: `postgres_data:/var/lib/postgresql/data`
   * Placement constraint: `node.role == manager`
   * Resources: Limits `memory: 1024M, cpus: '1.0'`; Reservations `memory: 256M, cpus: '0.25'`
4. **Cache (`redis`)**:
   * Image: `redis:7-alpine`
   * Internal only (no published ports).
   * Placement constraint: `node.role == manager`
   * Resources: Limits `memory: 256M, cpus: '0.50'`; Reservations `memory: 64M, cpus: '0.10'`

---

## Phase 5: Generating Configuration Files

### 5.1. Update `ansible/inventory/production/hosts.yml`
```yaml
all:
  hosts:
    production_server:
      ansible_host: "192.168.6.222"
      ansible_port: 22
      ansible_user: "deploy"
      ansible_ssh_private_key_file: "~/.ssh/id_ed25519"
      ansible_ssh_common_args: "-o StrictHostKeyChecking=accept-new"
```

### 5.2. Update `ansible/inventory/production/group_vars/all.yml`
Ensure all services have:
* Proper environment variables (`DB_HOST: "db"`, `REDIS_URL: "redis://redis:6379/1"`, `DJANGO_BEHIND_PROXY: "true"`).
* Defined `resources` (limits and reservations).
* Appropriate `replicas` and `ports` mapped.

---

## Phase 6: Execution & Post-Deployment Verification

1. **Deploy Full Playbook**:
   ```bash
   ANSIBLE_CONFIG=ansible/ansible.cfg ansible-playbook -i ansible/inventory/production/hosts.yml ansible/playbooks/site.yml
   ```
   *(For fast application-only redeployments: add `--tags app`)*

2. **Verify Swarm Services via SSH**:
   ```bash
   # Check service replica status (e.g., 2/2 replicas Up)
   ssh -p <SSH_PORT> <DEPLOY_USER>@<SERVER_IP> "docker stack services <app_name>"

   # Inspect individual tasks and health
   ssh -p <SSH_PORT> <DEPLOY_USER>@<SERVER_IP> "docker stack ps <app_name>"

   # View service logs
   ssh -p <SSH_PORT> <DEPLOY_USER>@<SERVER_IP> "docker service logs --tail 50 <app_name>_backend"
   ssh -p <SSH_PORT> <DEPLOY_USER>@<SERVER_IP> "docker service logs --tail 50 <app_name>_frontend"
   ```
