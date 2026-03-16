# Near-Real-Time RIC Installation Guide

This document describes the installation procedure for the **Near Real-Time RAN Intelligent Controller (Near-RT RIC)** in a Kubernetes cluster based on the O-RAN Software Community deployment guides.

The Near-RT RIC provides programmable control for the RAN using **xApps** running on a cloud-native platform.

---

# 1. Architecture Overview

The O-RAN Near-RT RIC deployment is typically split into two Kubernetes clusters:

| Cluster | Purpose |
|-------|-------|
| RIC Cluster | Hosts Near-RT RIC platform and xApps |
| AUX Cluster | Hosts auxiliary services |

Within the **RIC cluster**, the following namespaces are created:

- `ricinfra` – Infrastructure services  
- `ricplt` – RIC platform services  
- `ricxapp` – xApp deployments  

Ingress communication is handled using **Kong Ingress Controller**, allowing services to be accessed through the cluster node IP and port using URL paths.

---

# 2. Prerequisites

Ensure the following software and system requirements are met before deployment.

## Software Requirements

- Kubernetes ≥ 1.16
- Helm ≥ v2.12.3 or v3.5
- Docker
- Linux OS (Ubuntu 20.04 recommended)

## System Requirements

Typical development environment:

| Resource | Requirement |
|--------|--------|
| CPU | ≥ 4 vCPU |
| RAM | ≥ 16 GB |
| Storage | ≥ 40 GB |



---

# 3. Preparing the Kubernetes Environment

Clone the deployment repository containing scripts and Helm charts.

```bash
git clone https://gerrit.o-ran-sc.org/r/ric-plt/ric-dep
cd ric-dep
```

Install Kubernetes, Docker, and Helm using the provided script.

```bash
cd bin
./install_k8s_and_helm.sh
```

If the script stuck at the middle of create any error, replace install_k8s_and_helm.sh script by this:

```bash
#!/bin/bash -x
################################################################################
# Optimized for Ubuntu 20.04 - Kubernetes v1.28.11
################################################################################

usage() {
    echo "Usage: $0 [ -k <k8s version> -d <docker version> -e <helm version> -c <cni-version>]" 1>&2;
    exit 1;
}

wait_for_pods_running () {
  NS="$2"
  # Ensure kubectl is available before running check
  if ! command -v kubectl &> /dev/null; then
    echo "kubectl not found yet. Skipping pod check..."
    return
  fi
  
  CMD="kubectl get pods -n $NS "
  [ "$NS" = "all-namespaces" ] && CMD="kubectl get pods --all-namespaces "
  
  KEYWORD="Running"
  [ "$#" = "3" ] && KEYWORD="${3}.*Running"

  CMD2="$CMD | grep \"$KEYWORD\" | wc -l"
  NUMPODS=$(eval "$CMD2")
  echo "waiting for $NUMPODS/$1 pods running in namespace [$NS] with keyword [$KEYWORD]"
  while [ "$NUMPODS" -lt "$1" ]; do
    sleep 5
    NUMPODS=$(eval "$CMD2")
    echo "> waiting for $NUMPODS/$1 pods running in namespace [$NS] with keyword [$KEYWORD]"
  done 
}

# --- Default Versions ---
KUBEV="1.28.11"
HELMV="3.14.4"
DOCKERV="20.10.21"
KUBECNIV="1.1.1" 

while getopts ":k:d:e:c:" o; do
    case "${o}" in
        e) HELMV=${OPTARG} ;;
        d) DOCKERV=${OPTARG} ;;
        k) KUBEV=${OPTARG} ;;
        c) KUBECNIV=${OPTARG} ;;
        *) usage ;;
    esac
done

set -x
export DEBIAN_FRONTEND=noninteractive
echo "$(hostname -I) $(hostname)" >> /etc/hosts

# Version formatting for Ubuntu 20.04
DOCKERVERSION="${DOCKERV}-0ubuntu1~20.04.2"
KUBEVERSION="${KUBEV}-1.1" 
APTOPTS="--allow-downgrades --allow-change-held-packages --allow-unauthenticated --ignore-hold"

# 1. Prepare Repositories & Keys
# Using --batch --yes to avoid the "Overwrite?" prompt seen in your logs
mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --batch --yes --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" > /etc/apt/sources.list.d/kubernetes.list

# 2. Unhold and Cleanup
# This prevents the "Held packages were changed" error
apt-mark unhold kubeadm kubelet kubectl docker.io kubernetes-cni || true

apt-get update
for PKG in kubeadm docker.io; do
    if dpkg -l | grep -q "${PKG}"; then
        [ "${PKG}" = "kubeadm" ] && kubeadm reset -f --silent && rm -rf ~/.kube
        apt-get -y $APTOPTS purge kubeadm kubelet kubectl kubernetes-cni docker.io
    fi
done
apt-get -y autoremove

# 3. Installation
# Re-installing with specific versions
apt-get install -y $APTOPTS docker.io=${DOCKERVERSION} || apt-get install -y $APTOPTS docker.io
apt-get install -y $APTOPTS kubelet=${KUBEVERSION} kubeadm=${KUBEVERSION} kubectl=${KUBEVERSION} kubernetes-cni

# Lock versions to prevent accidental updates
apt-mark hold docker.io kubelet kubeadm kubectl

# 4. Docker Configuration
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "100m" },
  "storage-driver": "overlay2"
}
EOF
systemctl daemon-reload
systemctl restart docker

# 5. Kubernetes Cluster Init
swapoff -a
# Ensure we use the freshly installed kubeadm
/usr/bin/kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=v${KUBEV}

# Setup Kubeconfig for root
mkdir -p /root/.kube
cp -i /etc/kubernetes/admin.conf /root/.kube/config
export KUBECONFIG=/root/.kube/config
echo "export KUBECONFIG=/root/.kube/config" >> /root/.bashrc

# 6. Pod Network (Flannel)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# 7. Wait for System Pods
# For 1.28, we wait for core components to stabilize
wait_for_pods_running 6 kube-system

# Untaint master to allow pod scheduling (Supports both old and new labels)
kubectl taint nodes --all node-role.kubernetes.io/control-plane- || true
kubectl taint nodes --all node-role.kubernetes.io/master- || true

# 8. Helm Installation
if [ ! -f "helm-v${HELMV}-linux-amd64.tar.gz" ]; then
    wget https://get.helm.sh/helm-v${HELMV}-linux-amd64.tar.gz
fi
tar -zxvf helm-v${HELMV}-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
rm -rf linux-amd64

echo "Deployment Complete. Use 'kubectl get nodes' to verify."

```

Install Helm templates used by RIC components.

```bash
./install_common_templates_to_helm.sh
```

These scripts automatically configure:

- Kubernetes
- Helm
- Container networking (CNI)
- Required system dependencies.
---

# 4. Configure Deployment Recipe

Deployment of RIC components is controlled by a **recipe file**.

Open the example recipe:

```bash
nano RECIPE_EXAMPLE/PLATFORM/example_recipe_latest_stable.yaml
```

Important parameters to configure:

## External Service IPs
Use current IP address of your VM.
```yaml
extsvcplt:
  ricip: "192.168.1.134"
  auxip: "192.168.1.134"
```

- `ricip` – IP address of the RIC cluster ingress
- `auxip` – IP address of the AUX cluster ingress

## Container Image Configuration

Specify the container image registry and version tags for RIC components.

Example:

```yaml
repository: nexus3.o-ran-sc.org:10002/o-ran-sc
tag: latest
```


---

# 5. Deploy the Near-RT RIC Platform

After editing the recipe file, start the deployment.

```bash
cd ric-dep/bin
./install -f ../RECIPE_EXAMPLE/PLATFORM/example_recipe_latest_stable.yaml
```

This command deploys:

- Infrastructure services
- Platform components
- Supporting databases
- Messaging services

Deployment is performed using **Helm charts**.

---

# 6. Verify Deployment

Check if Kubernetes pods are running successfully.

```bash
kubectl get pods -A
```

Important namespaces:

```
ricinfra
ricplt
ricxapp
```

Example output:

```bash
kubectl get pods -n ricplt
```

Verify services:

```bash
kubectl get svc -A
```

---

# 7. Check Container Health

Monitor logs of platform components:

```bash
kubectl logs <pod-name> -n ricplt
```

Check platform services:

```
AppMgr
A1 Mediator
E2 Manager
RMR Router
DB Services
```

Healthy pods indicate successful deployment.

---

# 8. Undeploy RIC (Optional)

To remove the platform deployment:

```bash
cd ric-dep/bin
./undeploy
```

This removes all Helm deployments and Kubernetes resources.

---

# 9. srsRAN Configuration to RIC connection



cu.yml

```bash
# Example config for a locally deployed CU listening on the localhost interface for a DU connection

cu_cp:
  amf:
    addr: 10.53.1.2                     # The address or hostname of the AMF.
    port: 38412
    bind_addr: 10.53.1.1                 # A local IP that the gNB binds to for traffic from the AMF.
    supported_tracking_areas:             # Configure the TA associated with the CU-CP
      - tac: 7
        plmn_list:
          - plmn: "00101"
            tai_slice_support_list:
              - sst: 1
  f1ap:
    bind_addr: 127.0.10.1                 # Configure the F1AP bind address, this will enable the CU-cp to connect to the DU

cu_up:
  f1u:
    socket:                               # Define UDP/IP socket(s) for F1-U interface.
      -                                     # Socket 1
        bind_addr: 127.0.20.1                  # Sets the address that the F1-U socket will bind to.

e2:
  enable_cu_cp_e2: true    # Enable agent for Control Plane
  enable_cu_up_e2: true    # Enable agent for User Plane
  addr: 192.168.1.134      # Connect to the RIC on this machine
  bind_addr: 10.53.1.1
  port: 32222             # Port shown in your docker ps output

log:
  filename: /tmp/cu.log
  all_level: warning

pcap:
  ngap_enable: false
  ngap_filename: /tmp/cu_ngap.pcap

```

du.yml

```bash
f1ap:
  cu_cp_addr: 127.0.10.1        # CU-CP F1-C address
  bind_addr: 127.0.10.2         # DU local F1-C bind address


f1u:
  socket:
    - bind_addr: 127.0.10.2     # DU F1-U bind address


ru_sdr:
  device_driver: zmq
  device_args: tx_port=tcp://127.0.0.1:2000,rx_port=tcp://127.0.0.1:2001,base_srate=23.04e6
  srate: 23.04
  tx_gain: 80
  rx_gain: 40

e2:
  enable_du_e2: true       # Enable agent for the DU
  addr: 192.168.1.134
  bind_addr: 10.53.1.1
  port: 32222


cell_cfg:
  dl_arfcn: 368500                  # ARFCN of the downlink carrier (center frequency).
  band: 3                           # The NR band.
  channel_bandwidth_MHz: 20         # Bandwith in MHz. Number of PRBs will be automatically derived.
  common_scs: 15                    # Subcarrier spacing in kHz used for data.
  plmn: "00101"                     # PLMN broadcasted by the gNB.
  tac: 7                            # Tracking area code (needs to match the core configuration).
  pdcch:
    common:
      ss0_index: 0                  # Set search space zero index to match srsUE capabilities
      coreset0_index: 12            # Set search CORESET Zero index to match srsUE capabilities
    dedicated:
      ss2_type: common              # Search Space type, has to be set to common
      dci_format_0_1_and_1_1: false # Set correct DCI format (fallback)
  prach:
    prach_config_index: 1           # Sets PRACH config to match what is expected by srsUE
  pdsch:
    mcs_table: qam64                # Sets PDSCH MCS to 64 QAM
  pusch:
    mcs_table: qam64                # Sets PUSCH MCS to 64 QAM

pcap:
  e2ap_enable: true                              # Set to true to enable E2AP PCAPs.
  e2ap_du_filename: /tmp/gnb_du_e2ap.pcap        # Path where the DU E2AP PCAP is stored.

metrics:
  layers:
    enable_rlc: true                # Enable RLC metrics
    enable_sched: true              # Enable DU scheduler metrics
  periodicity:
    du_report_period: 1000          # Set DU statistics report period to 1s

```

Refer this link to all configuration reference for CU and DU:

https://docs.srsran.com/projects/project/en/latest/user_manuals/source/config_ref.html

---
© 2026 O-RAN SDL Security Documentation (UoR-FYP Group 11, 23rd Batch). All rights reserved.