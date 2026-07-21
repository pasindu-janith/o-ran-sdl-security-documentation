# MITRE Caldera Configurations & Security Testing Guide

**A concise guide for setting up MITRE Caldera and running attack simulations focused on xApp / sidecar testing in Kubernetes.**

---

## Table of Contents
- [Installation and Setup](#installation-and-setup)
- [Agent Deployment in Kubernetes](#agent-deployment-in-kubernetes)
- [Attack Simulations](#attack-simulations)
  - [xApp Internal Discovery](#xapp-internal-discovery)
  - [Envoy Bypass Validation (Layer 4)](#envoy-bypass-validation-layer-4)
  - [Envoy Bypass Validation (Layer 7)](#envoy-bypass-validation-layer-7)
  - [Intra-Pod Security Boundary Validation](#intra-pod-security-boundary-validation)
  - [Deep Sandbox Evasion & Privilege Escalation Sweep](#deep-sandbox-evasion--privilege-escalation-sweep)

---

## Installation and Setup

### 1. Update the system and install prerequisites
```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
  git \
  curl \
  build-essential \
  libffi-dev \
  libssl-dev
```

### 2. Install Miniconda (optional, recommended for Python env)
```bash
cd ~/Downloads || cd ~
curl -O https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh -b -p $HOME/miniconda3
~/miniconda3/bin/conda init bash
source ~/.bashrc
```

Accept Conda terms (if required):
```bash
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
```

### 3. Create and activate Caldera environment
```bash
conda create -n caldera-server python=3.10 -y
conda activate caldera-server
```

### 4. Install Go (used by some Caldera plugins/tools)
```bash
conda install -c conda-forge go=1.21 --override-channels -y
go version
```

### 5. Clone MITRE Caldera and install dependencies
```bash
cd ~/Desktop || cd ~
git clone https://github.com/mitre/caldera.git --recursive
cd caldera
pip install -r requirements.txt
```

### 6. Install Node.js (if needed)
```bash
conda install -c conda-forge nodejs -y
```

### 7. Start Caldera (development / insecure mode)
```bash
python3 server.py --insecure
# Dashboard: http://localhost:8888
```

## Agent Deployment in Kubernetes

1. Deploy or restart your xApp to ensure the agent bootstraps and registers with Caldera.

```bash
sudo kubectl rollout restart deployment/ricxapp-sdl-xapp -n ricxapp
sudo kubectl get pods -n ricxapp -w
```

Wait for the pod to reach a ready state (e.g., 3/3 Running). The Caldera agent should appear in the Caldera UI automatically.

## Attack Simulations

High-level steps to create and run operations in Caldera:

- Go to **Operations** → **Create Operation**
- Choose or create an **Adversary/Profile** containing abilities
- Configure: Operation Name, Adversary, Group (e.g., `red`), State (Run immediately), Autonomous (Enabled)
- Start the operation and monitor results in the Web UI

### xApp Internal Discovery

Create an operation using the `Discovery` adversary with relevant discovery abilities. Example: `xApp_Internal_Discovery`.

### Envoy Bypass Validation (Layer 4 TCP Handshake)

Profile: `Envoy Bypass Validation` — description: Validates bypass of local Envoy proxy to Redis DB.

Ability metadata:
- Tactic: Lateral Movement / Discovery
- Technique ID: T1046
- Platform: Linux
- Executor: sh/bash

Command (ability shell payload):
```bash
printf "=== Initiating Zero-Trust Envoy Bypass Verification ===\n" \
  && nc -w 3 -z service-ricplt-dbaas-tcp.ricplt.svc.cluster.local 6379 \
  && printf "\n[!] TEST RESULT: ENVOY BYPASS SUCCESSFUL!\n--> Status: VULNERABLE.\n\n" \
  || printf "\n[+] TEST RESULT: ENVOY BYPASS BLOCKED!\n--> Status: SECURE.\n\n"
```

Launch the operation using the created profile against the target group (e.g., `red`).

### Envoy Bypass Validation (Layer 7 Application – Payload Injection)

Command (Layer 7 check):
```bash
printf "=== Deep Layer 7 Envoy Bypass Verification ===\n" \
  && if printf "PING\r\n" | nc -w 3 service-ricplt-dbaas-tcp.ricplt.svc.cluster.local 6379 | grep -q "PONG"; then \
    printf "\n[!] CRITICAL VULNERABILITY: ENVOY BYPASS SUCCESSFUL!\n"; \
  else \
    printf "\n[+] TEST RESULT: ENVOY BYPASS BLOCKED!\n"; \
  fi
```

### Intra-Pod Security Boundary Validation

Bundle several abilities to audit filesystem, processes, and local network interfaces inside the pod.

Ability: Audit Shared Pod Volumes
```bash
echo "=== MOUNT POINTS ===" && mount | grep -E "emptyDir|volume" || echo "No shared volumes mapped."
echo "=== ROOT DIRECTORY SCAN ===" && ls -la /var/log/ /tmp/ 2>&1
```

Ability: Audit Cross-Container Processes
```bash
echo "=== VISIBLE PROCESSES ===" && ps aux || ps -ef
echo "=== PROC COUNT ===" && ls -d /proc/[0-9]* | wc -l
```

Ability: Audit Pod-Local Ports
```bash
echo "=== OPEN INTERFACES ===" && ss -tulpn || netstat -an
echo "=== PROBING LOCAL PORTS ==="
for port in 8001 15000 15020 9000; do
  curl -s -m 2 http://127.0.0.1:$port/config_dump || true
done
```

Bundle these abilities into an adversary profile such as `xApp_Pod_Isolation_Assessment` and run an operation (e.g., `Framework_Isolation_Run`).

### Deep Sandbox Evasion & Privilege Escalation Sweep

Replace the above abilities with deeper payloads to check write access, serviceaccount tokens, capabilities, namespaces, and admin endpoints.

Deep Ability examples:
```bash
# 1) SECURITY CONTEXT & WRITE STATUS
id
[ -w /etc/passwd ] && echo "[!] CRITICAL: Container root filesystem is WRITABLE." || echo "[+] Clean: Root filesystem is read-only or restricted."

# 2) SERVICEACCOUNT TOKEN EXPOSURE
[ -d /var/run/secrets/kubernetes.io/serviceaccount ] && \
  (echo "[!] Vulnerability Found: ServiceAccount token directory is accessible!"; ls -la /var/run/secrets/kubernetes.io/serviceaccount; echo "Token Prefix: $(head -c 15 /var/run/secrets/kubernetes.io/serviceaccount/token)...") \
  || echo "[+] Secure: No default Kubernetes token mounted."

# 3) DEEP WRITABLE MOUNT AUDIT
find / -writable -type d 2>/dev/null | grep -E -v "^/(proc|sys|dev|tmp)" || echo "No unexpected writable directories found outside /tmp."
```

Another deep check (capabilities & namespaces):
```bash
command -v capsh >/dev/null && capsh --print || grep -E "CapInh|CapPrm|CapEff|CapBnd" /proc/self/status
ls -l /proc/self/ns/
ps -eo pid,ppid,user,stat,comm || ps aux
```

Check proxy admin endpoints (Envoy/admin):
```bash
ss -tuanp || netstat -an
curl -s --connect-timeout 1 http://127.0.0.1:15000/server_info | head -n 3 || true
curl -s --connect-timeout 1 http://127.0.0.1:8001/server_info | head -n 3 || true
curl -s --connect-timeout 1 http://127.0.0.1:9000/server_info | head -n 3 || true
```

## Next Steps

- Save or update abilities in Caldera UI
- Create or restart operations
- Monitor execution results via the Caldera Operations dashboard
