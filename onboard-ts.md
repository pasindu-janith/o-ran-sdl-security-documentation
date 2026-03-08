---
title: "Traffic Steering (TS) xApp Onboarding Guide"
author: "Pasindu Janith"
project: "O-RAN Near-RT RIC Testbed"
version: "1.0"
date: "2026-03-08"
---

# Traffic Steering (TS) xApp Onboarding

## 1. Introduction

Traffic Steering (TS) xApp is an application deployed on the Near-Real-Time RAN Intelligent Controller (Near-RT RIC).  
Its primary function is to control and optimize user traffic across different radio access networks.

The TS xApp receives policies from the A1 interface and applies them through the RIC platform to control traffic steering decisions in the RAN environment.

The onboarding and deployment process involves:

- Building the xApp container image
- Publishing the image to a Docker registry
- Preparing xApp descriptors
- Onboarding the xApp using the DMS CLI
- Deploying the xApp through the RIC platform

---

# 2. Prerequisites

Before onboarding the TS xApp, ensure the following components are installed:

- OSC Near-RT RIC Platform
- Kubernetes cluster
- Helm
- Docker
- DMS CLI tool
- Local Helm repository (ChartMuseum)

A local Helm repository must be running to store the xApp Helm charts.

---

# 3. Setup Local Helm Repository

Start a ChartMuseum container to host Helm charts.

```bash
docker run --rm -u 0 -it -d \
-p 8090:8080 \
-e DEBUG=1 \
-e STORAGE=local \
-e STORAGE_LOCAL_ROOTDIR=/charts \
-v $(pwd)/charts:/charts \
chartmuseum/chartmuseum:latest
```

Set the Helm repository environment variable:

```bash
export CHART_REPO_URL=http://0.0.0.0:8090
```

This environment variable allows the DMS CLI to communicate with the Helm chart repository.

---
# Create Local registry to Push xApp Images  

xApp images are needed to be stored locally. So, local docker registry will be created by using docker's inbuilt container.

```bash
sudo docker run -d -p 5000:5000 --restart=always --name local-registry registry:2
```

---


# 4. Clone TS xApp Repository

Clone the Traffic Steering xApp sources under following git repos.

```bash
cd ~/Desktop
git clone "https://gerrit.o-ran-sc.org/r/ric-app/hw-go"
git clone "https://gerrit.o-ran-sc.org/r/ric-app/ts"
git clone "https://gerrit.o-ran-sc.org/r/ric-app/ad"
git clone "https://gerrit.o-ran-sc.org/r/ric-app/qp"

```

---

# 5. Build TS xApp Docker Image

Build the TS xApp container image.

```bash
cd ts
docker build . -t 127.0.0.1:5000/o-ran-sc/ric-app-ts:latest
```

Example output:

```
Successfully built 76bfe8867640
```

Push the image to the local Docker registry.

```bash
docker push 127.0.0.1:5000/o-ran-sc/ric-app-ts:latest
```

Example push result:

```
latest: digest: sha256:dacda165621b198ef62bf0f3412c03893bc359da0157eaa731ea8af005385874
```

---

# 6. Configure xApp Descriptor

Navigate to the xApp descriptor directory.

```bash
cd xapp_descriptor
```

Edit the configuration file.

```bash
nano config-file.json
```

Modify the image registry and tag:

```json
{
  "xapp_name": "trafficxapp",
  "version": "1.2.5",
  "containers": [
    {
      "name": "trafficxapp",
      "image": {
        "registry": "127.0.0.1:5000",
        "name": "o-ran-sc/ric-app-ts",
        "tag": "latest"
      }
    }
  ]
}
```

---

# 7. Onboard TS xApp

Run the DMS CLI onboarding command.

```bash
dms_cli onboard --config_file_path=config-file.json --shcema_file_path=schema.json
```

Expected output:

```
{
 "status": "Created"
}
```

This registers the xApp descriptor with the RIC platform. 

---

# 8. Download Helm Chart

Download the Helm chart generated during onboarding.

```bash
sudo CHART_REPO_URL=http://0.0.0.0:8090 dms_cli download_helm_chart trafficxapp 1.2.5 --output_path=/files/helm_xapp
```

Expected output:

```
status: OK
```

---

# 9. Install TS xApp

Deploy the TS xApp to the RIC cluster.

```bash
sudo CHART_REPO_URL=http://0.0.0.0:8090 dms_cli install --xapp_chart_name=trafficxapp --version=1.2.5 --namespace=ricxapp
```

Expected output:

```
status: OK
```

The TS xApp will now run inside the `ricxapp` namespace.

---

# 10. Verify Deployment

Check running pods:

```bash
kubectl get pods -n ricxapp
```

Example:

```
trafficxapp-7d9f7b5c4d-abc12   Running
```

Check logs:

```bash
kubectl logs <pod-name> -n ricxapp
```

---

# 11. Create A1 Policy for TS xApp

Create a policy type definition.

```bash
nano ts-policy-type-20008.json
```

Example policy definition:

```json
{
  "name": "tspolicy",
  "description": "TS policy parameters",
  "policy_type_id": 20008,
  "create_schema": {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "title": "TS Policy",
    "type": "object",
    "properties": {
      "threshold": {
        "type": "integer",
        "default": 0
      }
    }
  }
}
```

Upload the policy type to the A1 mediator.

```bash
curl -v -X PUT \
"http://${KONG_IP}:32080/a1mediator/A1-P/v2/policytypes/20008" \
-H "accept: application/json" \
-H "Content-Type: application/json" \
-d @ts-policy-type-20008.json
```

Create a policy instance.

```bash
curl -X PUT \
--header "Content-Type: application/json" \
--data '{"threshold":5}' \
http://<KONG_IP>:32080/a1mediator/A1-P/v2/policytypes/20008/policies/tsapolicy145
```

---

# 12. TS xApp Deployment Workflow

```
Build TS xApp Docker Image
        ↓
Push Image to Docker Registry
        ↓
Prepare xApp Descriptor
        ↓
Onboard xApp using DMS CLI
        ↓
Download Helm Chart
        ↓
Install xApp in ricxapp Namespace
        ↓
Create A1 Policy for Traffic Steering
```

---
© 2026 O-RAN SDL Security Documentation (UoR-FYP Group 11, 23rd Batch). All rights reserved.