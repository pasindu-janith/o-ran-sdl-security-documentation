# DID/VC based approach as Zero Trust Architecture for O-RAN Shared Data Layer (SDL)

## Overview
This approach replaces the centralized Identity Provider (Keycloak/JWT) trust model with a **decentralized identity** model based on W3C Decentralized Identifiers (DIDs) and Verifiable Credentials (VCs), anchored on a Hyperledger Indy ledger. Instead of an xApp fetching a bearer token from a central IdP on every cycle, the RIC issues each xApp a cryptographically signed Verifiable Credential once, at onboarding time. At runtime the xApp's sidecar proves possession of that credential by constructing a signed Verifiable Presentation (VP), without any network round-trip to an identity server.

The RIC plays the role of **VC Issuer**, the xApp is the **VC Holder**, and the Auth Agent sidecar acts as the **VC Verifier**. Trust is rooted in the Indy ledger (which anchors the RIC's DID) and in Ed25519 signatures (which anchor the credential and presentation), not in a live session with an identity server.

xApp pod contains 3 containers.
1. xApp container
   - Hosts the core xApp application logic.
2. Envoy sidecar
   - Acts as a secure proxy for inbound and outbound traffic.
   - Intercepts the raw SDL (Redis) TCP connection and issues an `ext_authz` gRPC check before allowing traffic through.
3. Auth Agent Container (DID/VC, v2)
   - Loads the xApp's DID/VC wallet from a mounted Kubernetes Secret.
   - Verifies the VC's signature and issuer at startup, and proves DID ownership via a VP on every `Check` request.
   - Extracts plain verified claims from the credential and forwards them to OPA for the final policy decision.

```mermaid
flowchart TB
    classDef hostFill fill:#f5f5f5,stroke:#000,stroke-width:1px,stroke-dasharray: 4 4;
    classDef clusterFill fill:#e1f5fe,stroke:#000,stroke-width:1px;
    classDef podFill fill:#fff9c4,stroke:#000,stroke-width:1px;
    classDef greenBox fill:#bdf4cc,stroke:#000,stroke-width:1px;
    classDef purpleBox fill:#d9c2ff,stroke:#000,stroke-width:1px;
    classDef pinkBox fill:#ffc7ce,stroke:#000,stroke-width:1px;
    classDef yellowBox fill:#ffeeb5,stroke:#000,stroke-width:1px;

    subgraph Host [Ubuntu Host - Docker Compose]
        direction TB
        VonWeb["Von Network Webserver\n(:9000)"]:::pinkBox
        VonNodes["4x Indy Validator Nodes\n(:9701-9708)"]:::pinkBox
        VonWeb --- VonNodes
    end

    subgraph RIC_Cluster [RIC Cluster]

        subgraph ricplt_ns [ricplt namespace]
            direction TB
            ACAPy["ACA-Py\nRIC Identity Agent\n(:3000 / :3001)"]:::yellowBox
            PG[("PostgreSQL\nWallet Store")]:::pinkBox
            OPA["Open Policy\nAgent"]:::yellowBox
            ACAPy --- PG
        end

        subgraph xApp_Pod [xApp Pod - ricxapp namespace]
            direction LR
            subgraph InnerFlow [ ]
                direction TB
                xApp["xApp container\n\nSDL API"]:::greenBox
                Envoy["Envoy\nSidecar"]:::purpleBox
                xApp -- "(1) GET/SET req." --> Envoy
                Envoy --> xApp
            end
            AuthAgent["Auth Agent v2\n(DIDKit)\n\n(3)   (4)   (5)"]:::greenBox
            Wallet[("Wallet Secret\n/wallet")]:::pinkBox
            Envoy -- "(2) gRPC CheckRequest" --> AuthAgent
            AuthAgent -- "(6) ALLOW/DENY" --> Envoy
            AuthAgent --- Wallet
        end

        Kyverno["Kyverno\nClusterPolicy"]:::yellowBox
        Redis[("Redis\nDatabase")]:::pinkBox
    end

    Provisioner["provision_xapp_wallet.py\n(Provisioner)"]:::purpleBox

    ACAPy -. "register RIC Endorser DID" .-> VonWeb
    Provisioner -- "generate sov DID" --> ACAPy
    Provisioner -. "register xApp DID" .-> VonWeb
    Provisioner -- "create wallet Secret" --> Wallet
    Kyverno -. "mount wallet + inject sidecars" .-> xApp_Pod

    AuthAgent -- "(4) plain verified claims" --> OPA
    OPA -- "(5) allow/deny" --> AuthAgent
    Envoy -- "(7)" --> Redis
    Redis --> Envoy

    class RIC_Cluster clusterFill;
    class xApp_Pod podFill;
    class Host hostFill;
    style InnerFlow fill:none,stroke:none;
```

# Phase 1: Von Network (Hyperledger Indy Ledger) Setup

The Indy ledger is the root of trust for every DID in this architecture. We attempted to run Von Network as native Kubernetes Deployments (`von-network-k8s.yaml`) first, but each node's genesis file is generated dynamically by the container's startup script and the 4 validator pods need to discover each other's genesis transactions before the pool can reach consensus. Kubernetes' pod scheduling and DNS propagation timing caused the genesis generation to race between nodes, so the pool never stabilized. Von Network was moved outside the cluster and run as a **Docker Compose stack on the Ubuntu host** instead — this is the version actually used for the working testbed.

```bash
# Clone Von Network on the Ubuntu host (outside the K8s cluster)
git clone https://github.com/bcgov/von-network.git
cd von-network

# Build the images
./manage build

# Start a 4-node local Indy pool + web UI, bound to the host's LAN IP
./manage start --logs
```

Verify the ledger is up:
```bash
curl http://<HOST_IP>:9000/status
```

The Von Network webserver exposes a genesis file and a self-serve DID registration endpoint that both ACA-Py and the provisioner script use:
```bash
# Fetch the genesis transactions (used by ACA-Py and copied into every xApp wallet)
curl http://<HOST_IP>:9000/genesis -o ric-genesis.txn

# Self-serve DID registration endpoint used to anchor new DIDs on the ledger
curl -X POST http://<HOST_IP>:9000/register \
  -H "Content-Type: application/json" \
  -d '{"did": "<did>", "verkey": "<verkey>"}'
```

Copy `ric-genesis.txn` to a path reachable by the RIC cluster tooling (referenced later as `GENESIS_PATH` and mounted into ACA-Py as a ConfigMap):
```bash
cp ric-genesis.txn ~/did-vc-setup/ric-genesis.txn
```

The abandoned `von-network-k8s.yaml` manifest is kept for reference — it deploys `von-node1..4` and `von-webserver` as Deployments in a `ricidentity` namespace, but is **not** part of the working deployment:
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: von-node1
  namespace: ricidentity
spec:
  replicas: 1
  selector:
    matchLabels:
      app: von-node1
  template:
    metadata:
      labels:
        app: von-node1
    spec:
      containers:
      - name: indy-node
        image: ghcr.io/bcgov/von-network-base:latest
        env:
        - name: NODE_NUM
          value: "1"
        - name: NODE_COUNT
          value: "4"
        - name: LEDGER_SEED
          value: "000000000000000000000000Steward1"
        - name: IPS
          value: "von-node1,von-node2,von-node3,von-node4"
        - name: REGISTER_NEW_DIDS
          value: "true"
        ports:
        - containerPort: 9701
        - containerPort: 9702
        volumeMounts:
        - name: node-data
          mountPath: /home/indy/.indy_client
      volumes:
      - name: node-data
        emptyDir: {}
# ... von-node2, von-node3, von-node4 and von-webserver follow the same pattern,
# each with a distinct NODE_NUM / LEDGER_SEED / port pair.
```

# Phase 2: ACA-Py Identity Agent Deployment (ricplt namespace)

[Aries Cloud Agent Python (ACA-Py)](https://github.com/hyperledger/aries-cloudagent-python) is deployed inside the `ricplt` namespace as the RIC's identity agent. It talks to the Von Network ledger over the genesis file, and persists its wallet in PostgreSQL so DIDs survive pod restarts.

Create the genesis ConfigMap from the file fetched in Phase 1:
```bash
kubectl create configmap indy-genesis \
  --from-file=genesis.txn=ric-genesis.txn \
  -n ricplt
```

Create `acapy-postgres.yaml`:
```bash
apiVersion: v1
kind: Secret
metadata:
  name: acapy-postgres-secret
  namespace: ricplt
type: Opaque
stringData:
  POSTGRES_DB: acapy
  POSTGRES_USER: acapy
  POSTGRES_PASSWORD: acapy-ric-password-2024
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: acapy-postgres
  namespace: ricplt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: acapy-postgres
  template:
    metadata:
      labels:
        app: acapy-postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        envFrom:
        - secretRef:
            name: acapy-postgres-secret
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: acapy-postgres
  namespace: ricplt
spec:
  selector:
    app: acapy-postgres
  ports:
  - port: 5432
    targetPort: 5432
```

Create `acapy-agent.yaml`:
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: acapy-ric-agent
  namespace: ricplt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: acapy-ric-agent
  template:
    metadata:
      labels:
        app: acapy-ric-agent
    spec:
      containers:
      - name: acapy
        image: ghcr.io/hyperledger/aries-cloudagent-python:py3.9-0.10.4
        command:
        - aca-py
        - start
        - --inbound-transport
        - http
        - "0.0.0.0"
        - "3000"
        - --outbound-transport
        - http
        - --endpoint
        - "http://acapy-ric-agent.ricplt:3000"
        - --admin
        - "0.0.0.0"
        - "3001"
        - --admin-insecure-mode
        - --genesis-file
        - /genesis/genesis.txn
        - --seed
        - "RIC00000000000000000Endorser001!"
        - --wallet-type
        - askar
        - --wallet-name
        - ric-wallet
        - --wallet-key
        - ric-wallet-key-secure-2024
        - --wallet-storage-type
        - postgres_storage
        - --wallet-storage-config
        - '{"url":"acapy-postgres:5432","db_name":"acapy"}'
        - --wallet-storage-creds
        - '{"account":"acapy","password":"acapy-ric-password-2024"}'
        - --auto-provision
        - --log-level
        - info
        - --label
        - "RIC-Identity-Agent"
        ports:
        - containerPort: 3000
          name: agent
        - containerPort: 3001
          name: admin
        volumeMounts:
        - name: genesis
          mountPath: /genesis
        readinessProbe:
          httpGet:
            path: /status/ready
            port: 3001
          initialDelaySeconds: 20
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /status/live
            port: 3001
          initialDelaySeconds: 30
          periodSeconds: 30
      volumes:
      - name: genesis
        configMap:
          name: indy-genesis
---
apiVersion: v1
kind: Service
metadata:
  name: acapy-ric-agent
  namespace: ricplt
spec:
  selector:
    app: acapy-ric-agent
  ports:
  - name: agent
    port: 3000
    targetPort: 3000
  - name: admin
    port: 3001
    targetPort: 3001
```

Apply both manifests and wait for ACA-Py to report ready:
```bash
kubectl apply -f acapy-postgres.yaml -f acapy-agent.yaml
kubectl rollout status deployment/acapy-ric-agent -n ricplt
```

Port-forward the Admin API for provisioning work done from the host:
```bash
sudo kubectl port-forward -n ricplt svc/acapy-ric-agent 3001:3001
curl -s http://localhost:3001/status | python3 -m json.tool
```

Note: `--seed "RIC00000000000000000Endorser001!"` deterministically derives ACA-Py's own Indy DID and verkey. That DID is **not** present on a fresh ledger by default, so it must be self-registered the same way xApp DIDs are (Phase 7) before it can write schema/cred-def transactions as an Endorser.

# Phase 3: RIC Endorser DID, Schema, and Credential Definition

Once ACA-Py is running, it needs its own DID registered on the ledger with Endorser rights before it can publish a credential schema.

Fetch ACA-Py's public DID and register it via Von Network's self-serve endpoint:
```bash
# Get the DID/verkey ACA-Py derived from its seed
curl -s http://localhost:3001/wallet/did/public | python3 -m json.tool

# Register it on the ledger (grants NYM/Endorser role)
curl -X POST http://<HOST_IP>:9000/register \
  -H "Content-Type: application/json" \
  -d '{"did": "KewdxLKBU9Fgu5aac8PH4R", "verkey": "<verkey_from_above>", "role": "ENDORSER"}'
```

This is the **RIC Endorser DID**: `KewdxLKBU9Fgu5aac8PH4R`, anchored on the Indy ledger and used as the trust root the Auth Agent checks against (`TRUSTED_RIC_DID` / `RIC_DID`).

Publish the SDL access credential schema through the ACA-Py Admin API:
```bash
curl -X POST http://localhost:3001/schemas \
  -H "Content-Type: application/json" \
  -d '{
    "schema_name": "sdl-access-credential",
    "schema_version": "1.0",
    "attributes": [
      "xapp_name", "xapp_version", "allowed_namespaces", "permissions",
      "ric_realm", "ric_issuer_sov_did", "sov_did", "issued_at", "valid_until"
    ]
  }'
```
Expected `schema_id`: `KewdxLKBU9Fgu5aac8PH4R:2:sdl-access-credential:1.0`

Publish the credential definition (binds the schema to the RIC's signing key):
```bash
curl -X POST http://localhost:3001/credential-definitions \
  -H "Content-Type: application/json" \
  -d '{
    "schema_id": "KewdxLKBU9Fgu5aac8PH4R:2:sdl-access-credential:1.0",
    "tag": "sdl-access-v1",
    "support_revocation": false
  }'
```
Expected `credential_definition_id`: `KewdxLKBU9Fgu5aac8PH4R:3:CL:8:sdl-access-v1`

Store these ledger-anchored identifiers as a ConfigMap that the provisioner and Auth Agent both read from:
```bash
kubectl create configmap ric-vc-config -n ricplt \
  --from-literal=RIC_DID=KewdxLKBU9Fgu5aac8PH4R \
  --from-literal=LEDGER_URL=http://<HOST_IP>:9000 \
  --from-literal=SCHEMA_ID=KewdxLKBU9Fgu5aac8PH4R:2:sdl-access-credential:1.0 \
  --from-literal=CRED_DEF_ID=KewdxLKBU9Fgu5aac8PH4R:3:CL:8:sdl-access-v1
```

# Phase 4: RIC Issuer Identity (DIDKit Signing Key)

ACA-Py 0.10.4 with an `askar` wallet does not expose the `/vc/credentials/issue` W3C endpoint, so W3C VC signing is done separately with **DIDKit** using a `did:key` issuer identity that is bound to the RIC's Indy sov DID. This identity is generated once and persisted as a Kubernetes Secret so every xApp credential is signed by the same RIC issuer key across provisioning runs.

Create `create_ric_issuer.py`:
```python
import json, os, asyncio, inspect
from pathlib import Path
import didkit

ISSUER_PATH = Path("/home/pasindu/did-vc-setup/ric-issuer.json")
RIC_SOV_DID = os.environ.get("RIC_SOV_DID")

if not RIC_SOV_DID:
    raise Exception("RIC_SOV_DID environment variable is missing")

def didkit_call(fn, *args):
    async def runner():
        result = fn(*args)
        if inspect.isawaitable(result):
            result = await result
        return result
    return asyncio.run(runner())

if ISSUER_PATH.exists():
    print(f"[!] Issuer file already exists: {ISSUER_PATH}")
    print("[!] Not overwriting it.")
    with open(ISSUER_PATH, "r") as f:
        existing = json.load(f)
    safe = dict(existing)
    safe["issuer_jwk"] = "[PRIVATE KEY HIDDEN]"
    print(json.dumps(safe, indent=2))
    exit(0)

issuer_jwk = didkit_call(didkit.generate_ed25519_key)
issuer_did = didkit_call(didkit.key_to_did, "key", issuer_jwk)

fragment = issuer_did.split(":")[-1]
issuer_vm = f"{issuer_did}#{fragment}"

issuer = {
    "label": "RIC Identity Authority",
    "ric_sov_did": RIC_SOV_DID,
    "issuer_did": issuer_did,
    "issuer_vm": issuer_vm,
    "issuer_jwk": issuer_jwk
}

ISSUER_PATH.write_text(json.dumps(issuer, indent=2))
os.chmod(ISSUER_PATH, 0o600)

safe = dict(issuer)
safe["issuer_jwk"] = "[PRIVATE KEY HIDDEN]"

print("[+] Persistent RIC issuer identity created")
print(json.dumps(safe, indent=2))
```

Run it once, binding the DIDKit issuer key to the RIC Endorser sov DID from Phase 3:
```bash
RIC_SOV_DID=KewdxLKBU9Fgu5aac8PH4R python3 create_ric_issuer.py
```

This produces `ric-issuer.json` (kept `0o600`, private key redacted from any log output):
```json
{
  "label": "RIC Identity Authority",
  "ric_sov_did": "KewdxLKBU9Fgu5aac8PH4R",
  "issuer_did": "did:key:z6Mkm8GwKYg4W7zFzos9eK8RQNvgufA8ymvHfjdKW97irKZt",
  "issuer_vm": "did:key:z6Mkm8GwKYg4W7zFzos9eK8RQNvgufA8ymvHfjdKW97irKZt#z6Mkm8GwKYg4W7zFzos9eK8RQNvgufA8ymvHfjdKW97irKZt",
  "issuer_jwk": "{\"kty\":\"OKP\",\"crv\":\"Ed25519\",\"x\":\"YyTIFO5hrC7dsBpLnPByNfwe0Vo5xnak5W6aV742w3M\",\"d\":\"EHodX8cKMlzYZHRaf_DzzVOg-z4X3t90llwa9sZwYNE\"}"
}
```

In this testbed the issuer keypair is regenerated fresh whenever `ric-issuer.json` is deleted — in production this key would be treated as the RIC's permanent signing identity and rotated deliberately, not regenerated per provisioning run.

Load it into the cluster as a Secret so the provisioner can read it without touching the filesystem of a running pod:
```bash
kubectl create secret generic ric-issuer-secret -n ricplt \
  --from-file=ric-issuer.json=ric-issuer.json
```

# Phase 5: Auth Agent v2 (DIDKit-enabled Envoy ext_authz Sidecar)

Auth Agent v2 extends the JWT-based Auth Agent from the Localized PEP approach (see `Main-Implementation.md`) — it keeps the same `ext_authz` gRPC contract with Envoy, but replaces Keycloak token verification with DID/VC verification.

Create `Dockerfile` (built on top of the existing `pasindujanith/auth-agent:v1` base image, adding DIDKit):
```bash
FROM pasindujanith/auth-agent:v1

RUN pip install --no-cache-dir didkit

COPY agent.py /app/agent.py

RUN python3 -m py_compile /app/agent.py
RUN python3 -c "import didkit; print('DIDKit installed OK')"

CMD ["python3", "/app/agent.py"]
```

Create `agent.py`:
```python
import os
import json
import time
import grpc
import asyncio
import inspect
from datetime import datetime
from concurrent import futures

import envoy.service.auth.v3.external_auth_pb2 as auth_pb2
import envoy.service.auth.v3.external_auth_pb2_grpc as auth_pb2_grpc
import envoy.service.auth.v3.attribute_context_pb2 as attribute_context_pb2
from envoy.type.v3 import http_status_pb2 as http_status_pb2

try:
    from google.rpc import status_pb2 as google_status_pb2
    from google.rpc import code_pb2 as google_code_pb2
except Exception:
    google_status_pb2 = None
    google_code_pb2 = None


# ── Config ────────────────────────────────────────────────────────────────────
WALLET_PATH = os.environ.get("WALLET_PATH", "/wallet")
OPA_GRPC_URL = os.environ.get("OPA_GRPC_URL", "opa-service.ricplt.svc.cluster.local:9191")
XAPP_NAME = os.environ.get("XAPP_NAME", "unknown-xapp")

# Should come from ric-vc-config ConfigMap
TRUSTED_RIC_DID = os.environ.get("RIC_DID", "KewdxLKBU9Fgu5aac8PH4R")


# ── DIDKit wrapper for DIDKit 0.3.3 ───────────────────────────────────────────
def didkit_call(fn, *args):
    async def runner():
        result = fn(*args)
        if inspect.isawaitable(result):
            result = await result
        return result

    return asyncio.run(runner())


# ── Envoy deny response ───────────────────────────────────────────────────────
def deny_response(reason="Access denied"):
    print(f"[AGENT] DENY: {reason}")

    if google_status_pb2 is not None:
        return auth_pb2.CheckResponse(
            status=google_status_pb2.Status(
                code=google_code_pb2.PERMISSION_DENIED,
                message=reason,
            ),
            denied_response=auth_pb2.DeniedHttpResponse(
                status=http_status_pb2.HttpStatus(
                    code=http_status_pb2.StatusCode.Value("Forbidden")
                ),
                body=reason,
            ),
        )

    # Fallback for older generated protobuf packages
    return auth_pb2.CheckResponse(
        status=http_status_pb2.HttpStatus(
            code=http_status_pb2.StatusCode.Value("Forbidden")
        )
    )


# ── Helper functions ──────────────────────────────────────────────────────────
def read_json_file(path):
    with open(path, "r") as f:
        return json.load(f)


def did_key_vm(did):
    """
    For did:key:z6Mkabc..., verification method is:
    did:key:z6Mkabc...#z6Mkabc...
    """
    fragment = did.split(":")[-1]
    return f"{did}#{fragment}"


def action_to_permission(action):
    """
    Maps SDL/Redis style actions to VC permissions.
    Adjust this later if your OPA policy uses different action names.
    """
    if not action:
        return None

    action = action.upper()

    if action in ["GET", "READ", "MGET", "EXISTS"]:
        return "read"

    if action in ["SET", "WRITE", "POST", "PUT", "DELETE", "DEL"]:
        return "write"

    return action.lower()


def parse_json_list(value):
    if isinstance(value, list):
        return value
    if isinstance(value, str):
        try:
            return json.loads(value)
        except Exception:
            return [v.strip() for v in value.split(",") if v.strip()]
    return []


# ── Load wallet at startup ────────────────────────────────────────────────────
def load_wallet():
    did_path = os.path.join(WALLET_PATH, "did.json")
    vc_path = os.path.join(WALLET_PATH, "vc.json")
    issuer_path = os.path.join(WALLET_PATH, "issuer.json")

    if not os.path.exists(did_path):
        print(f"[AGENT] ERROR: Missing {did_path}")
        return None, None, None, None

    if not os.path.exists(vc_path):
        print(f"[AGENT] ERROR: Missing {vc_path}")
        return None, None, None, None

    if not os.path.exists(issuer_path):
        print(f"[AGENT] ERROR: Missing {issuer_path}")
        return None, None, None, None

    did_data = read_json_file(did_path)
    vc_data = read_json_file(vc_path)
    issuer_data = read_json_file(issuer_path)

    # xApp private key must be in did.json
    xapp_jwk = did_data.get("jwk")
    if not xapp_jwk:
        print("[AGENT] ERROR: xApp private JWK missing in did.json")
        return None, None, None, None

    # issuer.json must contain only public issuer data
    if "jwk" in issuer_data or "issuer_jwk" in issuer_data:
        print("[AGENT] ERROR: issuer.json contains issuer private key. Unsafe wallet.")
        return None, None, None, None

    print("[AGENT] Wallet loaded")
    print(f"[AGENT] xApp DID       : {did_data.get('did')}")
    print(f"[AGENT] xApp Sov DID   : {did_data.get('sov_did')}")
    print(f"[AGENT] RIC Issuer DID : {issuer_data.get('issuer_did')}")
    print(f"[AGENT] RIC Sov DID    : {issuer_data.get('ric_sov_did')}")
    print(f"[AGENT] Trusted RIC DID: {TRUSTED_RIC_DID}")

    return did_data, vc_data, issuer_data, xapp_jwk


# ── Verify VC at startup ──────────────────────────────────────────────────────
def verify_vc_at_startup():
    if DID_DATA is None or VC_DATA is None or ISSUER_DATA is None:
        return False

    try:
        import didkit

        vc_issuer = VC_DATA.get("issuer")
        trusted_issuer = ISSUER_DATA.get("issuer_did")

        if vc_issuer != trusted_issuer:
            print("[AGENT] ERROR: VC issuer mismatch")
            print(f"[AGENT] VC issuer      : {vc_issuer}")
            print(f"[AGENT] Trusted issuer : {trusted_issuer}")
            return False

        issuer_ric_did = ISSUER_DATA.get("ric_sov_did")
        if issuer_ric_did != TRUSTED_RIC_DID:
            print("[AGENT] ERROR: issuer.json RIC DID mismatch")
            print(f"[AGENT] issuer.json RIC DID : {issuer_ric_did}")
            print(f"[AGENT] trusted RIC DID     : {TRUSTED_RIC_DID}")
            return False

        subj = VC_DATA.get("credentialSubject", {})
        vc_ric_did = subj.get("ric_issuer_sov_did")

        if vc_ric_did != TRUSTED_RIC_DID:
            print("[AGENT] ERROR: VC subject RIC DID mismatch")
            print(f"[AGENT] VC RIC DID      : {vc_ric_did}")
            print(f"[AGENT] Trusted RIC DID : {TRUSTED_RIC_DID}")
            return False

        vc_xapp_name = subj.get("xapp_name")
        if XAPP_NAME != "unknown-xapp" and vc_xapp_name != XAPP_NAME:
            print("[AGENT] ERROR: VC xApp name mismatch")
            print(f"[AGENT] VC xApp name  : {vc_xapp_name}")
            print(f"[AGENT] Env XAPP_NAME : {XAPP_NAME}")
            return False

        verify_raw = didkit_call(
            didkit.verify_credential,
            json.dumps(VC_DATA),
            "{}",
        )
        verify_result = json.loads(verify_raw)

        if verify_result.get("errors"):
            print(f"[AGENT] ERROR: VC cryptographic verification failed: {verify_result['errors']}")
            return False

        print("[AGENT] VC issuer, RIC DID, xApp name, and signature verified")
        return True

    except Exception as e:
        print(f"[AGENT] VC startup verification error: {e}")
        return False


# ── Extract claims from VC ────────────────────────────────────────────────────
def extract_claims():
    try:
        subj = VC_DATA.get("credentialSubject", {})

        valid_until = subj.get("valid_until")
        if valid_until:
            now = datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%SZ")
            if now > valid_until:
                print(f"[AGENT] VC expired at {valid_until}")
                return None

        allowed_namespaces = parse_json_list(subj.get("allowed_namespaces", []))
        permissions = parse_json_list(subj.get("permissions", []))

        return {
            "xapp_name": subj.get("xapp_name", XAPP_NAME),
            "xapp_did": subj.get("id"),
            "xapp_sov_did": subj.get("sov_did"),
            "allowed_namespaces": allowed_namespaces,
            "permissions": permissions,
            "ric_issuer_sov_did": subj.get("ric_issuer_sov_did"),
            "schema_id": subj.get("schema_id"),
            "cred_def_id": subj.get("cred_def_id"),
            "valid_until": valid_until,
        }

    except Exception as e:
        print(f"[AGENT] Claim extraction error: {e}")
        return None


# ── Verify xApp DID ownership using VP ────────────────────────────────────────
def verify_did_ownership(nonce):
    try:
        import didkit

        xapp_did = DID_DATA.get("did")
        xapp_jwk = XAPP_JWK

        if not xapp_did or not xapp_jwk:
            print("[AGENT] Missing xApp DID or private JWK")
            return False

        xapp_vm = did_key_vm(xapp_did)

        vp = {
            "@context": [
                "https://www.w3.org/2018/credentials/v1",
                "https://w3id.org/security/suites/ed25519-2020/v1",
            ],
            "type": ["VerifiablePresentation"],
            "verifiableCredential": [VC_DATA],
        }

        proof_options = {
            "type": "Ed25519Signature2020",
            "verificationMethod": xapp_vm,
            "proofPurpose": "authentication",
            "challenge": nonce,
            "domain": "ric.internal",
        }

        signed_vp = didkit_call(
            didkit.issue_presentation,
            json.dumps(vp),
            json.dumps(proof_options),
            xapp_jwk,
        )

        verify_raw = didkit_call(
            didkit.verify_presentation,
            signed_vp,
            json.dumps({
                "challenge": nonce,
                "domain": "ric.internal",
            }),
        )

        verify_result = json.loads(verify_raw)

        if verify_result.get("errors"):
            print(f"[AGENT] VP verification failed: {verify_result['errors']}")
            return False

        print(f"[AGENT] DID ownership verified: {xapp_did}")
        return True

    except Exception as e:
        print(f"[AGENT] VP verification error: {e}")
        return False


# ── Optional local VC permission check ────────────────────────────────────────
def local_vc_permission_check(claims, request_headers):
    """
    This does a simple local pre-check before OPA.
    OPA remains the final policy decision point.
    """
    action = request_headers.get("x-sdl-action", "SET")
    required_permission = action_to_permission(action)

    if required_permission and required_permission not in claims["permissions"]:
        print(f"[AGENT] VC permission check failed. Required={required_permission}, VC={claims['permissions']}")
        return False

    req_namespace = request_headers.get("x-sdl-namespace")

    if req_namespace:
        if req_namespace not in claims["allowed_namespaces"]:
            print(f"[AGENT] VC namespace check failed. Requested={req_namespace}, VC={claims['allowed_namespaces']}")
            return False

    return True


# ── Query OPA via gRPC ────────────────────────────────────────────────────────
def query_opa_grpc(claims, request_headers):
    try:
        headers = {
            "x-app-id": claims["xapp_name"],
            "x-sdl-action": request_headers.get("x-sdl-action", "SET"),
            "x-vc-verified": "true",
            "x-ric-issuer-did": ISSUER_DATA.get("issuer_did", ""),
            "x-ric-sov-did": claims.get("ric_issuer_sov_did", ""),
            "x-xapp-did": claims.get("xapp_did", ""),
            "x-xapp-sov-did": claims.get("xapp_sov_did", ""),
            "x-allowed-namespaces": ",".join(claims.get("allowed_namespaces", [])),
            "x-permissions": ",".join(claims.get("permissions", [])),
        }

        # Preserve useful original request headers if present
        for key in [":method", ":path", "x-sdl-namespace", "x-sdl-action"]:
            if key in request_headers:
                headers[key] = request_headers[key]

        with grpc.insecure_channel(OPA_GRPC_URL) as channel:
            stub = auth_pb2_grpc.AuthorizationStub(channel)
            opa_request = auth_pb2.CheckRequest(
                attributes=attribute_context_pb2.AttributeContext(
                    request=attribute_context_pb2.AttributeContext.Request(
                        http=attribute_context_pb2.AttributeContext.HttpRequest(
                            headers=headers
                        )
                    )
                )
            )

            opa_response = stub.Check(opa_request)
            print("[AGENT] OPA gRPC response received")
            return opa_response

    except Exception as e:
        print(f"[AGENT] OPA gRPC error: {e}")
        return deny_response("OPA error")


# ── Startup ───────────────────────────────────────────────────────────────────
print(f"[AGENT] Starting Auth Agent for {XAPP_NAME}")

DID_DATA, VC_DATA, ISSUER_DATA, XAPP_JWK = load_wallet()

VC_STARTUP_OK = False
if DID_DATA is not None and VC_DATA is not None and ISSUER_DATA is not None:
    VC_STARTUP_OK = verify_vc_at_startup()

if VC_STARTUP_OK:
    print("[AGENT] DID/VC startup verification passed")
else:
    print("[AGENT] DID/VC startup verification failed. All requests will be denied.")


# ── gRPC Check handler ────────────────────────────────────────────────────────
class AuthzServicer(auth_pb2_grpc.AuthorizationServicer):

    def Check(self, request, context):
        timestamp = datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S")
        print(f"\n[AGENT] [{timestamp}] CheckRequest for {XAPP_NAME}")

        if not VC_STARTUP_OK:
            return deny_response("VC startup verification failed")

        request_headers = dict(request.attributes.request.http.headers)

        # Step 1: Extract and validate VC claims
        claims = extract_claims()
        if claims is None:
            return deny_response("Invalid or expired VC")

        print(f"[AGENT] VC claims: namespaces={claims['allowed_namespaces']} permissions={claims['permissions']}")

        # Step 2: Prove xApp owns DID key using VP
        nonce = f"req-{XAPP_NAME}-{int(time.time())}"
        if not verify_did_ownership(nonce):
            return deny_response("DID ownership verification failed")

        # Step 3: Simple local VC permission check
        if not local_vc_permission_check(claims, request_headers):
            return deny_response("VC permission check failed")

        # Step 4: OPA final decision
        print(f"[AGENT] DID/VC verified. Querying OPA for {claims['xapp_name']}...")
        return query_opa_grpc(claims, request_headers)


if __name__ == "__main__":
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=5))
    auth_pb2_grpc.add_AuthorizationServicer_to_server(AuthzServicer(), server)
    server.add_insecure_port("[::]:50051")

    print("[AGENT] Auth Agent DID/VC mode started on port 50051")
    server.start()

    try:
        server.wait_for_termination()
    except KeyboardInterrupt:
        print("\n[AGENT] Shutdown requested by user")
        server.stop(0)
        print("[AGENT] Auth Agent stopped cleanly")
```

Build and push the image:
```bash
cd auth-agent-v2
sudo docker build -t ashank2001/auth-agent:v2 .
sudo docker push ashank2001/auth-agent:v2
```

# Phase 6: OPA Policy (Reused, Unchanged from the Localized PEP Approach)

No DID/VC-specific Rego policy was written for this testbed. OPA still runs the exact same `opa-policy.yaml` deployed for the JWT-based approach (see `Test-OPA.md`) — the `opa-pdp` Deployment and `opa-policy` ConfigMap were not touched:
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: opa-policy
  namespace: ricplt
data:
  policy.rego: |
    package envoy.authz

    # Required for modern OPA engines
    import rego.v1

    # 1. Define Data Globally (Outside the rule to prevent unification bugs)
    xapp_roles = {
        "ricxapp-sdl-xapp": ["writer"],
        "ts": ["reader"],
        "rx": ["admin"]
    }

    role_permissions = {
        "reader": ["GET", "EXISTS"],
        "writer": ["GET", "SET", "DEL"],
        "admin":  ["GET", "SET", "DEL", "FLUSHALL"]
    }

    # 2. Default deny
    default allow := false

    # 3. The Evaluation Rule (Now safely using 'if')
    allow if {
        xapp_id := input.attributes.request.http.headers["x-app-id"]
        action := input.attributes.request.http.headers["x-sdl-action"]

        # Lookup roles for the xApp
        roles := xapp_roles[xapp_id]

        # Iterate through the xApp's roles
        role := roles[_]

        # Lookup permissions for that role
        perms := role_permissions[role]

        # Check if requested action matches one of the permissions
        perms[_] == action
    }
```

This means the ABAC decision is still made against the **static `xapp_roles` / `role_permissions` table**, keyed only on `x-app-id` and `x-sdl-action` — the same two headers OPA consumed under the Localized PEP approach. `query_opa_grpc` in Phase 5 forwards a richer set of headers derived from the verified VC (`x-vc-verified`, `x-ric-sov-did`, `x-xapp-did`, `x-permissions`, `x-allowed-namespaces`), but the current Rego policy does not read any of them — they pass through unused. The only enforcement of VC-derived permissions/namespaces happening today is `local_vc_permission_check()` inside the Auth Agent itself (Phase 5, Step 3), before OPA is ever called. Making OPA evaluate the VC claims directly (as opposed to the static role table) is future work, not something implemented in this testbed.

Verify OPA is reachable and returns the expected decision for the existing policy:
```bash
sudo kubectl port-forward deployment/opa-pdp 8181:8181 -n ricplt

curl -X POST http://localhost:8181/v1/data/envoy/authz \
  -H "Content-Type: application/json" \
  -d '{"input":{"attributes":{"request":{"http":{"headers":{"x-app-id":"ricxapp-sdl-xapp","x-sdl-action":"SET"}}}}}}'
```

# Phase 7: Kyverno Wallet Injection

Kyverno's mutation rule from the Localized PEP approach (`Main-Implementation.md`) is extended to also mount each xApp's `xapp-wallet-<name>` Secret into the Auth Agent container at `/wallet`, and to inject the trusted `RIC_DID` from the `ric-vc-config` ConfigMap so the agent knows what issuer DID to trust without a code change.

Create `kyverno-v2.yaml`:
```bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: touchless-xapp-security
spec:
  admission: true
  background: true
  validationFailureAction: Audit
  rules:
  - name: generate-tls-cert
    match:
      any:
      - resources:
          kinds:
          - Deployment
          namespaces:
          - ricxapp
    generate:
      apiVersion: cert-manager.io/v1
      kind: Certificate
      name: "{{request.object.metadata.name}}-cert"
      namespace: ricxapp
      synchronize: true
      data:
        spec:
          secretName: "{{request.object.metadata.name}}-certs"
          commonName: "{{request.object.metadata.name}}"
          issuerRef:
            name: smo-root-ca
            kind: ClusterIssuer

  - name: inject-sidecars
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - ricxapp
    mutate:
      patchStrategicMerge:
        spec:
          containers:
          - (name): "?*"
            env:
            - name: DBAAS_SERVICE_HOST
              value: "127.0.0.1"
            - name: DBAAS_SERVICE_PORT
              value: "6379"

          - name: envoy-proxy
            image: envoyproxy/envoy:v1.28.0
            volumeMounts:
            - name: sidecar-configs
              mountPath: /etc/envoy/envoy.yaml
              subPath: envoy.yaml

          - name: auth-agent
            image: ashank2001/auth-agent:v2
            imagePullPolicy: Always
            env:
            - name: XAPP_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['app']
            - name: WALLET_PATH
              value: /wallet
            - name: RIC_DID
              valueFrom:
                configMapKeyRef:
                  name: ric-vc-config
                  key: RIC_DID
            volumeMounts:
            - name: sidecar-configs
              mountPath: /app/agent.py
              subPath: agent.py
            - name: xapp-tls-volume
              mountPath: /etc/xapp-certs
              readOnly: true
            - name: xapp-wallet
              mountPath: /wallet
              readOnly: true

          volumes:
          - name: sidecar-configs
            configMap:
              name: zerotrust-sidecar-configs
          - name: xapp-tls-volume
            secret:
              secretName: "{{request.object.metadata.labels.app}}-certs"
          - name: xapp-wallet
            secret:
              secretName: "xapp-wallet-{{request.object.metadata.labels.app}}"
```

Apply it to the cluster:
```bash
sudo kubectl apply -f kyverno-v2.yaml
```

Note: Kyverno's `generate` rule for Certificates runs unconditionally on every `Deployment` in `ricxapp`, but the `xapp-wallet-<name>` Secret referenced by the `inject-sidecars` rule must already exist (it is created by the provisioner in Phase 8) **before** the xApp pod is scheduled — otherwise the pod stays in `ContainerCreating` waiting on a missing Secret volume.

# Phase 8: xApp DID/VC Wallet Provisioning

`provision_xapp_wallet.py` is the operator-run script that onboards a single xApp into the DID/VC trust framework. It performs, in order:

1. Generates an Indy `sov` DID/verkey pair for the xApp via ACA-Py's wallet API.
2. Registers that sov DID on the Indy ledger through Von Network's `/register` endpoint.
3. Generates a **separate** `did:key` signing keypair for the xApp via DIDKit (used only for VP signing, not ledger anchored).
4. Builds and signs a W3C Verifiable Credential (`SDLAccessCredential`) with the RIC issuer's DIDKit key from Phase 4, embedding the xApp's allowed namespaces and permissions as `credentialSubject` claims.
5. Verifies the signed VC immediately after signing (fail fast if signing produced something invalid).
6. Builds a throwaway test VP and verifies it end-to-end, to catch verification-method mismatches before they show up as a runtime outage in the Auth Agent.
7. Reads the ledger genesis file.
8. Creates (replacing any existing) `xapp-wallet-<xapp_name>` Secret in the `ricxapp` namespace containing `did.json`, `vc.json`, `issuer.json`, and `genesis.txn`.

Create `provision_xapp_wallet.py`:
```python
"""
Provisions DID/VC wallet for an xApp.
Usage: python3 provision_xapp_wallet.py <xapp_name> <namespaces_csv> <permissions_csv>
Example: python3 provision_xapp_wallet.py ricxapp-sdl-xapp e2-metrics,kpi-store read,write
"""

import sys, json, requests, os, asyncio, inspect, base64
from datetime import datetime, timedelta
from kubernetes import client as k8s_client, config as k8s_config

ACAPY_ADMIN  = "http://localhost:3001"
CONFIG_NAMESPACE = "ricplt"
CONFIGMAP_NAME = "ric-vc-config"
ISSUER_SECRET_NAME = "ric-issuer-secret"
XAPP_NAMESPACE = "ricxapp"
GENESIS_PATH = "/home/pasindu/did-vc-setup/ric-genesis.txn"


# Run DIDKit calls in a thread pool to avoid event loop conflicts
# DIDKit 0.3.3 uses an internal Rust/Tokio runtime that conflicts
# with Python's asyncio event loop context

def didkit_call(fn, *args):
    """
    DIDKit 0.3.3 sometimes returns asyncio Future objects.
    This wrapper executes the call and awaits the Future before returning.
    """
    async def runner():
        result = fn(*args)
        if inspect.isawaitable(result):
            result = await result
        return result

    return asyncio.run(runner())

def load_kubeconfig():
    kubeconfig_paths = [
        os.path.expanduser("~/.kube/config"),
        "/home/pasindu/.kube/config",
        "/root/.kube/config",
        "/etc/kubernetes/admin.conf",
    ]

    for kpath in kubeconfig_paths:
        if os.path.exists(kpath):
            try:
                k8s_config.load_kube_config(config_file=kpath)
                print(f"[+] Loaded kubeconfig from {kpath}")
                return
            except Exception as e:
                print(f"[!] Could not load {kpath}: {e}")

    raise Exception("Cannot find valid kubeconfig")


def load_ric_vc_config():
    v1 = k8s_client.CoreV1Api()
    cm = v1.read_namespaced_config_map(CONFIGMAP_NAME, CONFIG_NAMESPACE)

    data = cm.data or {}

    required = ["RIC_DID", "LEDGER_URL", "SCHEMA_ID", "CRED_DEF_ID"]
    missing = [k for k in required if k not in data]

    if missing:
        raise Exception(f"Missing values in ConfigMap {CONFIGMAP_NAME}: {missing}")

    return {
        "ric_sov_did": data["RIC_DID"],
        "ledger_url": data["LEDGER_URL"],
        "schema_id": data["SCHEMA_ID"],
        "cred_def_id": data["CRED_DEF_ID"],
    }


def load_ric_issuer_secret():
    v1 = k8s_client.CoreV1Api()
    sec = v1.read_namespaced_secret(ISSUER_SECRET_NAME, CONFIG_NAMESPACE)

    encoded = sec.data.get("ric-issuer.json")
    if not encoded:
        raise Exception("ric-issuer.json not found inside ric-issuer-secret")

    issuer_json = base64.b64decode(encoded).decode("utf-8")
    issuer = json.loads(issuer_json)

    required = ["ric_sov_did", "issuer_did", "issuer_vm", "issuer_jwk"]
    missing = [k for k in required if k not in issuer]

    if missing:
        raise Exception(f"Missing values in ric-issuer.json: {missing}")

    return issuer

def provision(xapp_name, allowed_namespaces, permissions):
    load_kubeconfig()

    ric_config = load_ric_vc_config()
    ric_issuer = load_ric_issuer_secret()

    ric_sov_did = ric_config["ric_sov_did"]
    ledger_url = ric_config["ledger_url"]
    schema_id = ric_config["schema_id"]
    cred_def_id = ric_config["cred_def_id"]

    issuer_jwk = ric_issuer["issuer_jwk"]
    issuer_did = ric_issuer["issuer_did"]
    issuer_vm = ric_issuer["issuer_vm"]
    issuer_bound_sov_did = ric_issuer["ric_sov_did"]

    if issuer_bound_sov_did != ric_sov_did:
        raise Exception(
            f"RIC issuer mismatch: Secret has {issuer_bound_sov_did}, ConfigMap has {ric_sov_did}"
        )

    print(f"[+] RIC Indy DID   : {ric_sov_did}")
    print(f"[+] RIC Issuer DID : {issuer_did}")
    print(f"[+] Schema ID      : {schema_id}")
    print(f"[+] Cred Def ID    : {cred_def_id}")
    import didkit
    print(f"\n[*] Provisioning DID/VC wallet for: {xapp_name}")

    # ── Step A: Generate xApp sov DID via ACA-Py (unchanged) ─────────────────
    print("[*] Generating Ed25519 DID keypair via ACA-Py...")
    resp = requests.post(
        f"{ACAPY_ADMIN}/wallet/did/create",
        json={"method": "sov", "options": {"key_type": "ed25519"}},
        timeout=30
    )
    resp.raise_for_status()
    did_data    = resp.json()["result"]
    xapp_did    = did_data["did"]
    xapp_verkey = did_data["verkey"]
    print(f"[+] xApp sov DID: {xapp_did}")

    # ── Step B: Register xApp DID on Indy ledger ─────────────────────────────
    print("[*] Registering DID on Indy ledger...")
    ledger_resp = requests.post(
        f"{ledger_url}/register",
        json={"did": xapp_did, "verkey": xapp_verkey},
        timeout=30
    )
    if ledger_resp.status_code != 200:
        raise Exception(f"Ledger NYM failed: {ledger_resp.status_code} {ledger_resp.text}")
    print(f"[+] DID registered on ledger: {xapp_did}")

    # ── Step C: Generate keypairs via DIDKit ──────────────────────────────────
    print("[*] Generating xApp signing keypair via DIDKit...")
    xapp_jwk     = didkit_call(didkit.generate_ed25519_key)
    xapp_key_did = didkit_call(didkit.key_to_did, "key", xapp_jwk)
    xapp_fragment = xapp_key_did.split(":")[-1]
    xapp_vm = f"{xapp_key_did}#{xapp_fragment}"
    print(f"[+] xApp key DID: {xapp_key_did}")

    # ── Step D: Build and sign W3C VC ─────────────────────────────────────────
    print("[*] Signing W3C Verifiable Credential...")
    issued_at   = datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%SZ")
    valid_until = (datetime.utcnow() + timedelta(days=30)).strftime("%Y-%m-%dT%H:%M:%SZ")

    vc_unsigned = {
        "@context": [
            "https://www.w3.org/2018/credentials/v1",
            "https://w3id.org/security/suites/ed25519-2020/v1",
            {
                "@vocab": "https://example.org/ric-sdl#"
            }
    	],
        "id": f"urn:uuid:vc-{xapp_name}-{int(datetime.utcnow().timestamp())}",
        "type": ["VerifiableCredential", "SDLAccessCredential"],
        "issuer": issuer_did,
        "issuanceDate": issued_at,
        "expirationDate": valid_until,
        "credentialSubject": {
            "id":                 xapp_key_did,
            "xapp_name":          xapp_name,
            "xapp_version":       "1.0.0",
            "allowed_namespaces": json.dumps(allowed_namespaces),
            "permissions":        json.dumps(permissions),
            "ric_realm":          "ric-realm",
            "ric_issuer_sov_did": ric_sov_did,
            "schema_id": schema_id,
            "cred_def_id": cred_def_id,
            "sov_did":            xapp_did,
            "issued_at":          issued_at,
            "valid_until":        valid_until,
        }
    }

    proof_options = json.dumps({
        "type":               "Ed25519Signature2020",
        "verificationMethod": issuer_vm,
        "proofPurpose":       "assertionMethod"
    })

    vc_signed_str = didkit_call(
        didkit.issue_credential,
        json.dumps(vc_unsigned),
        proof_options,
        issuer_jwk
    )
    vc_signed   = json.loads(vc_signed_str)
    proof_value = vc_signed.get("proof", {}).get("proofValue", "")
    if not proof_value:
        raise Exception("VC signing failed — no proofValue in proof")
    print(f"[+] VC signed. proofValue: {proof_value[:40]}...")

    # ── Step E: Verify the VC ─────────────────────────────────────────────────
    print("[*] Verifying signed VC...")
    verify_raw    = didkit_call(didkit.verify_credential, vc_signed_str, "{}")
    verify_result = json.loads(verify_raw)
    if verify_result.get("errors"):
        raise Exception(f"VC verification failed: {verify_result['errors']}")
    print("[+] VC verification passed")

    # ── Step F: Test VP construction and verification ─────────────────────────
    print("[*] Testing VP construction and verification...")
    nonce  = "provisioner-test-nonce"
    test_vp = {
        "@context": ["https://www.w3.org/2018/credentials/v1"],
        "type":     ["VerifiablePresentation"],
        "verifiableCredential": [vc_signed]
    }
    vp_proof = json.dumps({
        "type":               "Ed25519Signature2020",
        "verificationMethod": xapp_vm,
        "proofPurpose":       "authentication",
        "challenge":          nonce,
        "domain":             "ric.internal"
    })
    vp_signed_str = didkit_call(
        didkit.issue_presentation,
        json.dumps(test_vp),
        vp_proof,
        xapp_jwk
    )
    vp_result = json.loads(didkit_call(
        didkit.verify_presentation,
        vp_signed_str,
        json.dumps({"challenge": nonce, "domain": "ric.internal"})
    ))
    if vp_result.get("errors"):
        raise Exception(f"VP test failed: {vp_result['errors']}")
    print("[+] VP test passed — Auth Agent will verify correctly at runtime")

    # ── Step G: Read genesis file ─────────────────────────────────────────────
    with open(GENESIS_PATH, "r") as f:
        genesis_content = f.read()

    # ── Step H: Load kubeconfig ───────────────────────────────────────────────
    print("[*] Creating K8s wallet Secret...")
    kubeconfig_paths = [
        os.path.expanduser("~/.kube/config"),
        "/home/pasindu/.kube/config",
        "/root/.kube/config",
        "/etc/kubernetes/admin.conf",
    ]
    loaded = False
    for kpath in kubeconfig_paths:
        if os.path.exists(kpath):
            try:
                k8s_config.load_kube_config(config_file=kpath)
                print(f"[+] Loaded kubeconfig from {kpath}")
                loaded = True
                break
            except Exception as e:
                print(f"[!] Could not load {kpath}: {e}")
    if not loaded:
        raise Exception("Cannot find valid kubeconfig")

    # ── Step I: Create K8s Secret ─────────────────────────────────────────────
    v1          = k8s_client.CoreV1Api()
    secret_name = f"xapp-wallet-{xapp_name}"

    try:
        v1.delete_namespaced_secret(secret_name, "ricxapp")
        print(f"[*] Deleted existing wallet for {xapp_name}")
    except k8s_client.exceptions.ApiException as e:
        if e.status != 404:
            raise

    secret = k8s_client.V1Secret(
        metadata=k8s_client.V1ObjectMeta(
            name=secret_name,
            namespace="ricxapp",
            labels={
                "app":        xapp_name,
                "managed-by": "did-vc-provisioner",
                "sov-did":    xapp_did,
            }
        ),
        string_data={
            "did.json": json.dumps({
                "did":     xapp_key_did,
                "sov_did": xapp_did,
                "verkey":  xapp_verkey,
                "jwk":     xapp_jwk
            }),
            "vc.json":     json.dumps(vc_signed),
            "issuer.json": json.dumps({
                "label": "RIC Identity Authority",
                "ric_sov_did": ric_sov_did,
                "issuer_did": issuer_did,
                "issuer_vm": issuer_vm
            }),
            "genesis.txn": genesis_content,
        }
    )

    v1.create_namespaced_secret("ricxapp", secret)
    print(f"[+] Wallet Secret '{secret_name}' created in ricxapp namespace")

    print(f"""
╔══════════════════════════════════════════════════════╗
║  DID/VC Provisioning Complete (Real Signed VC)       ║
╠══════════════════════════════════════════════════════╣
║  xApp Name    : {xapp_name:<37}║
║  Sov DID      : {xapp_did:<37}║
║  Key DID      : {xapp_key_did[:37]:<37}║
║  Issuer DID   : {issuer_did[:37]:<37}║
║  Wallet Secret: {secret_name:<37}║
║  Valid Until  : {valid_until:<37}║
║  Proof        : Ed25519Signature2020 ✓ (real)        ║
╚══════════════════════════════════════════════════════╝
""")
    return xapp_did


if __name__ == "__main__":
    if len(sys.argv) < 4:
        print("Usage: python3 provision_xapp_wallet.py <xapp_name> <namespaces_csv> <permissions_csv>")
        print("Example: python3 provision_xapp_wallet.py ricxapp-sdl-xapp e2-metrics,kpi-store read,write")
        sys.exit(1)

    provision(
        sys.argv[1],
        sys.argv[2].split(","),
        sys.argv[3].split(",")
    )
```

Install its Python dependencies and run it directly for one xApp:
```bash
pip install requests kubernetes didkit

python3 provision_xapp_wallet.py ricxapp-sdl-xapp e2-metrics,kpi-store read,write
```

Inspect the resulting Secret:
```bash
sudo kubectl get secret xapp-wallet-ricxapp-sdl-xapp -n ricxapp -o jsonpath='{.data.vc\.json}' | base64 -d | python3 -m json.tool
```

# Phase 9: Secure xApp Onboarding Script

`secure_xapp_onboard.sh` ties Phase 8 provisioning into the same `dms_cli` onboarding flow described in `xapp-onboarding.md`, so a single command builds the xApp image, onboards it via DMS CLI, provisions its DID/VC wallet **before** the pod is created, then installs the xApp and verifies the wallet actually landed inside the Auth Agent sidecar.

Create `secure_xapp_onboard.sh`:
```bash
#!/bin/bash

set -e

# Usage:
# ./secure_xapp_onboard.sh <repo-path> <descriptor-path> <xapp-name> <version> <namespace> <allowed-namespaces> <permissions>
#
# Example:
# ./secure_xapp_onboard.sh ~/custom-sdl-xapp ~/custom-sdl-xapp/descriptor sdl-xapp 1.0.1 ricxapp ue-metrics read,write

REPO_PATH=$1
DESC_PATH=$2
XAPP_CHART_NAME=$3
VERSION=$4
NAMESPACE=$5
ALLOWED_NAMESPACES=$6
PERMISSIONS=$7

if [ -z "$REPO_PATH" ] || [ -z "$DESC_PATH" ] || [ -z "$XAPP_CHART_NAME" ] || [ -z "$VERSION" ] || [ -z "$NAMESPACE" ] || [ -z "$ALLOWED_NAMESPACES" ] || [ -z "$PERMISSIONS" ]; then
  echo "Usage:"
  echo "./secure_xapp_onboard.sh <repo-path> <descriptor-path> <xapp-name> <version> <namespace> <allowed-namespaces> <permissions>"
  echo
  echo "Example:"
  echo "./secure_xapp_onboard.sh ~/custom-sdl-xapp ~/custom-sdl-xapp/descriptor sdl-xapp 1.0.1 ricxapp ue-metrics read,write"
  exit 1
fi

RUNTIME_XAPP_NAME="${NAMESPACE}-${XAPP_CHART_NAME}"
IMAGE_NAME="127.0.0.1:5000/${XAPP_CHART_NAME}:${VERSION}"

echo "======================================================"
echo "[0] Secure DID/VC xApp onboarding"
echo "Chart xApp name   : $XAPP_CHART_NAME"
echo "Runtime xApp name : $RUNTIME_XAPP_NAME"
echo "Version           : $VERSION"
echo "Namespace         : $NAMESPACE"
echo "Image             : $IMAGE_NAME"
echo "Allowed namespaces: $ALLOWED_NAMESPACES"
echo "Permissions       : $PERMISSIONS"
echo "======================================================"

echo
echo "[1] Building Docker image..."
cd "$REPO_PATH"
sudo docker build -t "$IMAGE_NAME" .

echo
echo "[2] Pushing Docker image..."
sudo docker push "$IMAGE_NAME"

echo
echo "[3] Running dms_cli onboard..."
cd "$DESC_PATH"
CHART_REPO_URL=http://127.0.0.1:8090

sudo CHART_REPO_URL=$CHART_REPO_URL dms_cli onboard \
  --config_file_path=config-file.json \
  --shcema_file_path=schema.json | tee /tmp/dms_onboard.log

if grep -qiE "Cannot connect|Service not ready|error_message|Failed to connect" /tmp/dms_onboard.log; then
  echo "[ERROR] dms_cli onboard failed. Fix local Helm chart repo before continuing."
  exit 1
fi
echo

echo
echo "[4] Checking ACA-Py admin API..."
if ! curl -fsS http://localhost:3001/status >/dev/null; then
  echo "[ERROR] ACA-Py admin API is not reachable at localhost:3001"
  echo
  echo "Open another terminal and run:"
  echo "sudo kubectl port-forward -n ricplt svc/acapy-ric-agent 3001:3001"
  echo
  echo "Then rerun this script."
  exit 1
fi

echo "[5] Provisioning DID/VC wallet before xApp install..."
cd ~/did-vc-setup/provisioner
python3 provision_xapp_wallet.py "$RUNTIME_XAPP_NAME" "$ALLOWED_NAMESPACES" "$PERMISSIONS"

echo
echo "[6] Verifying wallet Secret..."
sudo kubectl get secret "xapp-wallet-${RUNTIME_XAPP_NAME}" -n "$NAMESPACE"

echo
echo "[7] Skipping uninstall. This flow keeps existing xApps running."

echo
echo "[8] Installing xApp using dms_cli..."
sudo CHART_REPO_URL=$CHART_REPO_URL dms_cli install "$XAPP_CHART_NAME" "$VERSION" "$NAMESPACE"

echo
echo "[9] Waiting for xApp pod to appear..."
sleep 15

POD=$(sudo kubectl get pods -n "$NAMESPACE" --sort-by=.metadata.creationTimestamp \
  | grep "$RUNTIME_XAPP_NAME" | tail -n1 | awk '{print $1}')

if [ -z "$POD" ]; then
  echo "[ERROR] No pod found for $RUNTIME_XAPP_NAME"
  sudo kubectl get pods -n "$NAMESPACE"
  exit 1
fi

echo "Pod: $POD"

echo
echo "[10] Waiting for pod to become Ready..."
sudo kubectl wait --for=condition=Ready pod/"$POD" -n "$NAMESPACE" --timeout=180s || true

echo
echo "[11] Showing pod status..."
sudo kubectl get pod "$POD" -n "$NAMESPACE"

echo
echo "[12] Showing injected containers..."
sudo kubectl get pod "$POD" -n "$NAMESPACE" \
  -o jsonpath='{range .spec.containers[*]}{.name}{" -> "}{.image}{"\n"}{end}'

echo
echo "[13] Verifying wallet mount inside Auth-Agent..."
sudo kubectl exec -n "$NAMESPACE" "$POD" -c auth-agent -- ls -l /wallet || {
  echo "[ERROR] Auth-Agent or wallet mount not found. Kyverno injection may have failed."
  exit 1
}

echo
echo "[14] Showing Auth-Agent logs..."
sudo kubectl logs -n "$NAMESPACE" "$POD" -c auth-agent --tail=80

echo
echo "[15] Showing xApp logs..."
sudo kubectl logs -n "$NAMESPACE" "$POD" -c "$XAPP_CHART_NAME" --tail=80 || true

echo
echo "======================================================"
echo "[+] Secure DID/VC onboarding completed"
echo "Runtime xApp name: $RUNTIME_XAPP_NAME"
echo "Wallet Secret    : xapp-wallet-${RUNTIME_XAPP_NAME}"
echo "Pod              : $POD"
echo "======================================================"
```

Run it end to end:
```bash
chmod +x secure_xapp_onboard.sh
./secure_xapp_onboard.sh ~/custom-sdl-xapp ~/custom-sdl-xapp/descriptor sdl-xapp 1.0.1 ricxapp e2-metrics,kpi-store read,write
```

## Request Lifecycle & Traffic Flow

1. **Onboarding (one-time, out-of-band):** the operator runs `secure_xapp_onboard.sh`, which provisions the xApp's sov DID, key DID, and signed VC, and stores them as the `xapp-wallet-<name>` Secret before the xApp pod is ever scheduled.
2. **Injection:** Kyverno mutates the incoming xApp Pod, adding the Envoy sidecar, the Auth Agent v2 sidecar, and mounting the wallet Secret at `/wallet`.
3. **Startup verification:** on boot, the Auth Agent loads the wallet and immediately verifies the VC's signature and issuer/RIC DID chain (`verify_vc_at_startup`). If this fails, every subsequent request is denied without contacting OPA.
4. **Initiation:** the xApp executes an SDL command; Envoy's `ext_authz` filter intercepts the TCP connection and sends a `CheckRequest` to the Auth Agent over gRPC (`:50051`).
5. **Claim extraction:** the Auth Agent extracts and expiry-checks the plain claims (`allowed_namespaces`, `permissions`) from the already-verified VC.
6. **Proof of possession:** the Auth Agent constructs a fresh Verifiable Presentation over the VC, signs it with the xApp's private JWK, and verifies it with DIDKit — proving the sidecar actually holds the private key bound to the credential, not just a copy of the VC JSON.
7. **Local pre-check:** a cheap local permission/namespace check runs before bothering OPA, to short-circuit obviously-denied requests.
8. **ABAC decision:** the Auth Agent sends `x-app-id` and `x-sdl-action` (plus the extra VC-derived headers, currently unused by Rego) to OPA over gRPC; OPA evaluates the same `opa-policy.yaml` role table from Phase 6 and returns allow/deny.
9. **Enforcement:** the Auth Agent relays OPA's decision back to Envoy as the `CheckResponse`; Envoy either forwards the TCP connection to Redis or drops it.

## Known Limitations / Testbed Decisions
- Von Network runs on the Ubuntu host via Docker Compose, not inside Kubernetes — the native `von-network-k8s.yaml` deployment was abandoned after genesis file generation timing prevented the 4-node pool from reaching consensus.
- DIDKit is used for VC signing instead of ACA-Py's built-in W3C credential endpoint, because ACA-Py 0.10.4 with the `askar` wallet type does not expose `/vc/credentials/issue`.
- The RIC issuer keypair (`ric-issuer.json`) is generated fresh per provisioning setup by `create_ric_issuer.py`; in a production deployment this would be persisted permanently as the RIC's signing identity and rotated deliberately rather than regenerated.
- `did:key` is used for the xApp's VP-signing DID, kept separate from its ledger-anchored `did:sov` identity — the sov DID proves ledger registration, the key DID proves possession at runtime.
- Full DIDComm-based credential exchange (issuer-to-holder protocol messages) is not implemented; the signed VC is delivered out-of-band by writing it directly into the xApp's wallet Secret during provisioning.
- OPA's Rego policy was **not** rewritten for DID/VC — it is the same static `xapp_roles`/`role_permissions` table from the Localized PEP approach, keyed on `x-app-id`/`x-sdl-action` only. The VC-derived claims the Auth Agent forwards (`x-vc-verified`, `x-permissions`, `x-allowed-namespaces`, `x-ric-sov-did`) are enforced solely by `local_vc_permission_check()` inside the Auth Agent, not by OPA.

## Common commands

Check ACA-Py, Von Network and OPA reachability:
```bash
curl -s http://localhost:3001/status | python3 -m json.tool
curl -s http://<HOST_IP>:9000/status
sudo kubectl port-forward deployment/opa-pdp 8181:8181 -n ricplt
```

Check logs in a specific container within a specific pod:
```bash
sudo kubectl logs ricxapp-sdl-xapp-686946b7-765hs -c auth-agent -n ricxapp
```

Check which agent.py code is actually running inside the sidecar:
```bash
sudo kubectl exec ricxapp-sdl-xapp-686946b7-5wn25 -c auth-agent -n ricxapp -- cat /app/agent.py
```

Inspect a wallet Secret's contents without decoding manually:
```bash
sudo kubectl get secret xapp-wallet-ricxapp-sdl-xapp -n ricxapp -o json \
  | python3 -c "import sys,json,base64; d=json.load(sys.stdin)['data']; [print(k, '=>', base64.b64decode(v)[:200]) for k,v in d.items()]"
```

Re-provision a wallet after rotating permissions for an already-onboarded xApp:
```bash
python3 provision_xapp_wallet.py ricxapp-sdl-xapp e2-metrics,kpi-store,ue-metrics read,write
sudo kubectl rollout restart deployment ricxapp-sdl-xapp -n ricxapp
```
