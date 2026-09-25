# AirflowSumo — Setup Log

## Project
**Task:** Migrate Apache Airflow (currently Dockerized) to Kubernetes
**Internship:** AMCNSS
**Team:** Awash, Rachita (+ others)
**Repo:** github.com/awashbhandari/airflowsumo (public)

---

## Task 1: Git Repository

- Repo `airflowsumo` created fresh on GitHub, public visibility
- Initialized with a README (so `main` branch and first commit existed)
- Cloned locally:
```bash
  git clone https://github.com/awashbhandari/airflowsumo.git
  cd airflowsumo
```
- Created `namespace.yaml` at repo root:
```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: airflowsumo
```
- Applied it (tested locally on Minikube later): `kubectl create -f namespace.yaml`
- Added team members as collaborators: Settings → Collaborators → Add people → Write access
- Created personal branch for task work:
```bash
  git checkout -b awash/minikube-setup
  git push -u origin awash/minikube-setup
```
- Work happens on personal branches, never directly on `main`; merges into
  `main` happen via Pull Request after review

### Concepts learned
- **Namespace** = logical grouping inside a Kubernetes cluster, isolates one
  project's resources (pods, services, secrets) from others sharing the
  same cluster. Useful for organization, avoiding name clashes, access
  control, resource limits, and easy bulk cleanup (`kubectl delete namespace`)
- `apiVersion: v1` / `kind: Namespace` — tells Kubernetes what kind of object
  to create; `-f <file>` tells kubectl to read the object definition from a file
- `kubectl create` vs `kubectl apply` — `create` errors if the object already
  exists; `apply` creates or updates, safer for iterative editing
- Branch names with `/` (e.g. `awash/minikube-setup`) are just a naming
  convention, not real folders — unlike file paths (`docs/NOTES.md`), which
  are real directory structure
- New branches start as an exact copy of the branch they're created from, so
  files already on `main` (like `namespace.yaml`) automatically carry over
- After a PR merge, files land directly in `main`'s structure — no folder
  named after the branch is ever created
- `.git` is a hidden folder (dot-prefixed) — use `ls -a` to see it

---

## Task 2: Minikube Installation

### AWS account setup
- Set a monthly budget alert (Unblended costs, no action rule — email alert only)
- Enabled MFA on root
- Created IAM user (`awash-dev`) with `AmazonEC2FullAccess` — avoid using
  root for daily work
- Learned: IAM users have **zero** permissions by default, must explicitly
  attach policies (e.g. also needed `ServiceQuotasReadOnlyAccess` later to
  check EC2 quotas)

### EC2 instance attempts

**Attempt 1 — t3.medium, RHEL (latest)**
- Launch failed: "specified instance type is not eligible for Free Tier" (warning, not blocker)
- Actual failure: "Launch initiation failed"
- Checked Service Quotas → EC2 → Running On-Demand Standard instances → quota was 5 vCPUs (not the problem)
- Free-tier eligible types on this account: t3.micro, t3.small, c7i-flex.large, t8i.micro, t8i.small
- Likely cause: new-account temporary restriction on non-free-tier types

**Attempt 2 — t3.small, RHEL 10 (auto-selected)**
- Instance launched successfully
- SSH troubleshooting:
  - `ssh: Connection timed out` — investigated security group inbound rule (port 22)
  - `curl ifconfig.me` returned IPv6, not matching the IPv4-only security group rule
  - `curl -4 ifconfig.me` forced IPv4 check — confirmed IP actually matched security group already
  - Real fix: retried SSH after a short wait (instance likely needed extra time after "Running" to accept SSH)
  - Later, intermittent timeouts returned repeatedly even with correct IP in the rule
  - Root cause identified: ISP appears to use CGNAT / frequently rotating IPs,
    so "My IP" in the security group goes stale within minutes
  - Working fix: temporarily set inbound SSH source to `0.0.0.0/0` (anywhere)
    — acceptable short-term for a learning instance using key-based auth,
    not left on long-term
- Docker install:
```bash
  sudo dnf install -y dnf-plugins-core
  sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
  sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
  sudo systemctl start docker
```
  - `systemctl start docker` failed: "Job for docker.service failed"
  - `sudo dockerd` (run directly) revealed the real error: missing kernel
    module `xt_addrtype`, needed for Docker's iptables-based networking
  - `sudo modprobe xt_addrtype` → "Module not found" — module doesn't exist
    on this kernel at all
  - Root cause: RHEL 10's cloud kernel doesn't ship legacy
    iptables/netfilter modules Docker's default network driver depends on
  - **Decision: relaunch with RHEL 9** instead of patching RHEL 10 (too
    advanced/fragile a fix for this stage)

**Attempt 3 — t3.small, RHEL 9**
- Reused existing key pair (`minikube-key`) and security group
- Docker install commands (same as above) — package install stage hit:
Killed

  during `dnf install -y dnf-plugins-core`
  - `free -h` showed only ~538MiB available (OOM kill — t3.small only has 2GB RAM total)
  - Fix: added 2GB swap space
```bash
    sudo dd if=/dev/zero of=/swapfile bs=1M count=2048
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```
  - Verified: `swapon --show`
  - Reran installs — completed successfully with swap active
- Docker started and enabled cleanly this time:
```bash
  sudo systemctl start docker
  sudo systemctl enable docker
  sudo usermod -aG docker $USER
```
  - `usermod -aG docker $USER` adds current user to the `docker` group so
    Docker commands work without `sudo`
  - Group membership only applies on a new login session — required
    logout/login (`exit` + reconnect) or `newgrp docker` to take effect
  - Got "permission denied ... docker.sock" until reconnecting, then resolved
- Verified Docker:
```bash
  docker ps
  docker run hello-world
```
  → worked, "Hello from Docker!" confirmed

### kubectl install
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

### Minikube install
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

### Minikube start — blocked by memory
```bash
minikube start --driver=docker
```
→ `RSRC_INSUFFICIENT_CONTAINER_MEMORY: docker only has 717MiB available,
less than the required 1800MiB for Kubernetes`
- Root cause: `t3.small` (2GB total RAM) doesn't leave enough free memory
  for Kubernetes once OS + Docker overhead is accounted for, even with swap
- **Decision: upgrade to `t3.medium` (4GB RAM)** for the actual Minikube run

### Concepts learned
- `dnf` installs OS-native RPM packages (e.g. Docker, via its official repo);
  standalone tools like `kubectl`/`minikube` are plain downloaded binaries via
  `curl`, no package manager involved
- `chmod +x` marks a downloaded file as executable; moving a binary to
  `/usr/local/bin/` puts it on the system PATH so it can be run from
  anywhere by name alone
- A **container runtime** (Docker here) is the software that actually runs
  containers; Kubernetes/Minikube orchestrate but rely on a runtime underneath
- Not all RHEL versions are equally Docker-compatible out of the box — check
  kernel module support if `docker.service` fails to start
- Small instances (2GB RAM) need swap space and/or a bigger instance type for
  package installs and Kubernetes to run reliably
- `free -h` and `swapon --show` are useful commands to check memory/swap status

---
