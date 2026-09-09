# O-RAN SDL Security
## Implementation Steps for Clean Install

Install OSC RIC in a VM having following specs:

| Requirement | Specification |
|---|---|
| **Operating System (OS)** | Ubuntu 20.04 LTS |
| **CPU** | Minimum 8 Cores (16+ recommended for full cluster operations) |
| **RAM** | Minimum 16 GB (32 GB recommended) |
| **Disk Space** | Minimum 60 GB |
| **Network** | Internet access with bridged connection |

Follow the `OSC-RIC setup.md` file for OSC-RIC installation.

### Deploy xApps
Follow the `xapp-onboarding.md` file for setup the cluster to onboard xapps via `dms_cli`.

### Implement Framework in OSC RIC Cluster

First we need to deploy Keycloak and Open Policy Agent within our cluster as centralized security services.

For Keycloak: follow `keycloak.md`.
For OPA: follow `OPA.md`.

### Implement Localized PEP Security Framework

Follow the `Localized-PEP.md` file to implement Zeto-Trust security framework having Localized Policy Enforcement Point within the xApp pod. 

### Implement Centralized PEP Security Framework

Follow the `Centralized-PEP.md` file to implement Zeto-Trust security framework having Centralized Policy Enforcement Point within the DBaaS pod. 

### Disk Expansion

If you want to expand the disk in VM,

1. Turn off the VM and go to edit VM settings
2. Go to Hard Drive and click expand
3. Enter the new space you allocate and update
4. Turn on VM 
5. Follow the steps given in `disk-space-issue.md` to add expanded diskspace into Ubuntu


