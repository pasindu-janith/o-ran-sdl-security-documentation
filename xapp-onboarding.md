# xApp Onboarding in Near-RT RIC

## 1. Introduction

This document describes the onboarding and deployment process of xApps in an O-RAN Near-RT RIC environment using DMS CLI and Helm charts.  

The onboarding process includes:
- Preparing Helm repository
- Configuring xApp descriptor and schema
- Onboarding using DMS CLI
- Installing and managing xApps
- Performing health checks

This documentation is created for a Kubernetes-based Near-RT RIC testbed integrated with srsRAN.

---

# 2. System Overview

The xApp onboarding workflow involves the following components:

- Near-RT RIC
- App Manager (AppMgr)
- DMS CLI
- Helm Repository
- Kubernetes RIC Cluster
- xApp Container Image

---

# 3. Local Helm Repository Setup

Start a local Helm repository using ChartMuseum:

```bash
#Create a local helm repository with a port other than 8080 on host
docker run --rm -u 0 -it -d -p 8090:8080 \
  -e DEBUG=1 \
  -e STORAGE=local \
  -e STORAGE_LOCAL_ROOTDIR=/charts \
  -v $(pwd)/charts:/charts \
  chartmuseum/chartmuseum:latest
```

Set Helm repository URL:

```bash
#Set CHART_REPO_URL env variable
export CHART_REPO_URL=http://0.0.0.0:8090
```

---
# DMS_CLI Setup
Install dms_cli tool
```bash
#Git clone appmgr
git clone "https://gerrit.o-ran-sc.org/r/ric-plt/appmgr"

#Change dir to xapp_onboarder
cd appmgr/xapp_orchestrater/dev/xapp_onboarder

#If pip3 is not installed, install using the following command
sudo apt install python3-pip

#In case dms_cli binary is already installed, it can be uninstalled using following command
sudo pip3 uninstall xapp_onboarder

#Install xapp_onboarder using following command
sudo pip3 install ./

```
 If the host user is non-root user, after installing the packages, please assign the permissions to the below filesystems

```bash
#Assign relevant permission for non-root user
sudo chmod 755 /usr/local/bin/dms_cli
sudo chmod -R 755 /usr/local/lib/python3.8
sudo chmod -R 755 /usr/local/lib/python3.8
```
```
Develop xApp -> Build xApp Image -> Push docker image into local registry -> Update xApp Descriptor files with local registry data -> Onboard xApp using dms_cli -> Download Helm charts -> Install xApps 
```
# 4. xApp Descriptor Files

Each xApp requires:

- `config.json`
- `schema.json`
- Helm chart package
- Docker container image

---

## 4.1 Example config.json

```json
{
  "xapp_name": "kpimon",
  "version": "1.0.0",
  "namespace": "ricxapp",
  "container_name": "kpimon",
  "image": "local/kpimon:1.0.0"
}
```

---

## 4.2 Example schema.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "xApp Controls Schema",
  "type": "object",
  "properties": {}
}
```

---

# 5. xApp Onboarding Process

## 5.1 Onboard xApp

```bash
dms_cli onboard \
  --config_file_path=/path/to/config.json \
  --schema_file_path=/path/to/schema.json
```

This step registers the xApp with AppMgr.

---

## 5.2 Download Helm Chart

```bash
dms_cli download_helm_chart \
  --xapp_chart_name=kpimon \
  --version=1.0.0 \
  --output_path=./charts
```

---

## 5.3 Install (Deploy) xApp

```bash
dms_cli install \
  --xapp_chart_name=kpimon \
  --version=1.0.0 \
  --namespace=ricxapp
```

This deploys the xApp into Kubernetes.

---

## 5.4 Verify Deployment

Check running pods:

```bash
kubectl get pods -n ricxapp
```

---

## 5.5 Health Check

```bash
dms_cli health_check \
  --xapp_chart_name=kpimon \
  --namespace=ricxapp
```

---

## 5.6 Upgrade xApp

```bash
dms_cli upgrade \
  --xapp_chart_name=kpimon \
  --old_version=1.0.0 \
  --new_version=1.1.0 \
  --namespace=ricxapp
```

---

## 5.7 Rollback xApp

```bash
dms_cli rollback \
  --xapp_chart_name=kpimon \
  --new_version=1.1.0 \
  --old_version=1.0.0 \
  --namespace=ricxapp
```

---

## 5.8 Uninstall xApp

```bash
dms_cli uninstall --xapp_chart_name=kpimon --namespace=ricxapp
```

---

# 6. Complete Onboarding Workflow

1. Build xApp Docker image
2. Push image to local registry
3. Package Helm chart
4. Start local Helm repository
5. Set `CHART_REPO_URL`
6. Run `dms_cli onboard`
7. Run `dms_cli install`
8. Verify using `kubectl get pods`
9. Perform health check

---

# 7. Expected Deployment Flow with Secure Onboarding

```
SMO (optional)
      ↓
DMS CLI
      ↓
AppMgr
      ↓
Helm
      ↓
Kubernetes
      ↓
xApp Pod
```

---

# 8. Troubleshooting

### Pods Not Starting
```bash
kubectl describe pod <pod-name> -n ricxapp
```

### Check Logs
```bash
kubectl logs <pod-name> -n ricxapp
```

### Verify Helm Repo
```bash
curl http://0.0.0.0:8090/index.yaml
```

---

# 9. Security Considerations (Optional Enhancement)

For secure onboarding:
- Validate xApp signatures before onboarding
- Use SMO for authorization control
- Restrict Helm repository access
- Enforce namespace isolation
- Enable TLS for RIC APIs

---

# 10. Conclusion

This document describes the structured onboarding and lifecycle management of xApps in a Near-RT RIC environment.  

The process ensures:
- Standardized deployment
- Version control
- Health monitoring
- Scalable xApp lifecycle management

---
© 2026 O-RAN SDL Security Documentation (UoR-FYP Group 11, 23rd Batch). All rights reserved.