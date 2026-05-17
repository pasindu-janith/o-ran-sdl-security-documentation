# Secure onboarding of xApps

Create a smo-server directory on Desktop.
Create smo_core.py
```bash
import os
import subprocess
import json
from datetime import datetime
from fastapi import FastAPI, Form, Request, HTTPException

app = FastAPI(title="External SMO Core (Zero Trust PKI & Inventory)")

# --- Configurations & Paths ---
CA_KEY = "/etc/smo/certs/smo-root-ca.key"
CA_CERT = "/etc/smo/certs/smo-root-ca.crt"
DB_DIR = "/etc/smo/db"
DEV_DB_PATH = f"{DB_DIR}/developers.json"
INV_DB_PATH = f"{DB_DIR}/inventory.json"

# --- Database Helper Functions ---
def init_system():
    """Bootstraps the SMO PKI and JSON databases."""
    # 1. Init PKI
    os.makedirs(os.path.dirname(CA_KEY), exist_ok=True)
    if not os.path.exists(CA_KEY):
        print("[*] Generating SMO Root CA...")
        subprocess.run(["openssl", "genrsa", "-out", CA_KEY, "4096"], check=True)
        subprocess.run(["openssl", "req", "-x509", "-new", "-nodes", "-key", CA_KEY,
                        "-sha256", "-days", "3650", "-out", CA_CERT,
                        "-subj", "/CN=O-RAN-SMO-Root/O=Telecommunications"], check=True)

    # 2. Init Databases
    os.makedirs(DB_DIR, exist_ok=True)
    if not os.path.exists(DEV_DB_PATH):
        with open(DEV_DB_PATH, "w") as f: json.dump({}, f)
    if not os.path.exists(INV_DB_PATH):
        with open(INV_DB_PATH, "w") as f: json.dump([], f)

def read_db(path):
    with open(path, "r") as f: return json.load(f)

def write_db(path, data):
    with open(path, "w") as f: json.dump(data, f, indent=4)

# Initialize on startup
init_system()


# --- API Endpoints ---

@app.post("/api/v1/admin/developers")
async def register_developer(dev_id: str, name: str, organization: str):
    """Admin Endpoint: Registers a new developer dynamically."""
    db = read_db(DEV_DB_PATH)
    if dev_id in db:
        raise HTTPException(status_code=400, detail="Developer ID already exists.")

    db[dev_id] = {
        "name": name,
        "organization": organization,
        "status": "active",
        "registered_at": datetime.utcnow().isoformat()
    }
    write_db(DEV_DB_PATH, db)
    return {"message": f"Developer '{name}' registered successfully.", "dev_id": dev_id}


@app.post("/api/v1/pki/sign-csr")
async def sign_csr(developer_id: str = Form(...), csr_pem: str = Form(...)):
    """Flow Step 1 & 2: Authenticates developer, receives CSR, returns Signed Cert."""

    # --- ZERO TRUST GATE 1: Verify Identity ---
    dev_db = read_db(DEV_DB_PATH)
    if developer_id not in dev_db:
        raise HTTPException(status_code=403, detail="Access Denied: Unregistered Developer ID.")

    if dev_db[developer_id]["status"] != "active":
        raise HTTPException(status_code=403, detail="Access Denied: Developer account is disabled.")

    print(f"[*] Identity Verified for {dev_db[developer_id]['name']}. Processing CSR...")

    # --- Process CSR ---
    csr_path, cert_path = f"/tmp/{developer_id}.csr", f"/tmp/{developer_id}.crt"
    with open(csr_path, "w") as f: f.write(csr_pem)

    try:
        # Note: In a strict ZT environment, you would also inspect the CSR here to ensure
        # the CN matches the registered developer's name before signing it.
        subprocess.run(["openssl", "x509", "-req", "-in", csr_path, "-CA", CA_CERT,
                        "-CAkey", CA_KEY, "-CAcreateserial", "-out", cert_path,
                        "-days", "365", "-sha256"], check=True)

        with open(cert_path, "r") as f:
            return {"certificate": f.read(), "issued_to": dev_db[developer_id]["name"]}
    except subprocess.CalledProcessError:
        raise HTTPException(status_code=500, detail="Failed to sign CSR")


@app.post("/o1/v1/notify")
async def o1_notification_listener(request: Request):
    """Flow Step 6: Receives O1 Notification and updates Active Inventory."""
    payload = await request.json()
    xapp_name = payload.get('xappName')

    print(f"[*] O1 Notification Received: xApp '{xapp_name}' deployed successfully.")

    # Update Active Inventory
    inventory = read_db(INV_DB_PATH)

    # Create the inventory record
    record = {
        "xappName": xapp_name,
        "status": payload.get("status", "deployed"),
        "last_updated": datetime.utcnow().isoformat(),
        "raw_data": payload
    }

    # Check if updating an existing xApp or adding a new one
    existing_idx = next((i for i, item in enumerate(inventory) if item["xappName"] == xapp_name), None)
    if existing_idx is not None:
        inventory[existing_idx] = record
    else:
        inventory.append(record)

    write_db(INV_DB_PATH, inventory)
    return {"status": "Acknowledged and Inventory Updated"}


@app.get("/api/v1/inventory/xapps")
async def get_xapp_inventory():
    """Admin Endpoint: Returns the list of currently deployed xApps."""
    return {"active_inventory": read_db(INV_DB_PATH)}

```
Create Dockerfile

```bash
FROM python:3.10-slim
RUN apt-get update && apt-get install -y openssl && rm -rf /var/lib/apt/lists/*
WORKDIR /app
RUN pip install fastapi uvicorn requests python-multipart
COPY smo_core.py .
EXPOSE 8000
CMD ["uvicorn", "smo_core:app", "--host", "0.0.0.0", "--port", "8000"]

```
Build image, run container and check logs
```bash
sudo docker build --no-cache -t o-ran-smo:v1 .

sudo docker run -d --name o-ran-smo -p 8000:8000 -v /etc/smo:/etc/smo o-ran-smo:v1

sudo docker logs o-ran-smo -f

```
Register new developers on SMO. Only registered developers' csr will be signed by SMO. This should be done by MNO.

```bash
curl -X POST "http://localhost:8000/api/v1/admin/developers?dev_id=pasindu-dev-4545&name=Pasindu&organization=Ruhuna-Engineering"
```

create xapp-dev directory on Desktop and go to it

```bash
openssl genrsa -out developer.key 2048
openssl req -new -key developer.key -out developer.csr -subj "/CN=xAppDeveloper-01/O=Authorized-Vendors"

CSR_STRING=$(cat developer.csr)
curl -X POST http://localhost:8000/api/v1/pki/sign-csr -F "developer_id=pasindu-dev-4545" -F "csr_pem=$CSR_STRING" | jq -r '.certificate' > developer.crt
```

-------------------------------------------------------------
Create another directory at Desktop for onboarder.
Extract the smo-root-ca.crt from smo-server container.
```bash
sudo docker cp o-ran-smo:/etc/smo/certs/smo-root-ca.crt ./smo-root-ca.crt
```

Create local_ric_gateway.py file. Locate the smo-root-ca.crt in same directory.
```bash
import os
import json
import subprocess
import tarfile
import requests
from fastapi import FastAPI, UploadFile, File, HTTPException

app = FastAPI(title="RIC Zero Trust Onboarder (Local VM)")

# --- Local VM Configurations ---
# Put your smo-root-ca.crt in the exact same folder as this python script
SMO_CA_CERT = "./smo-root-ca.crt"
EXTERNAL_SMO_O1_URL = os.getenv("SMO_O1_URL", "http://localhost:8000/o1/v1/notify")
CHART_REPO_URL = "http://0.0.0.0:8090" # Your local ChartMuseum port

@app.post("/ric/v1/secure-onboard")
async def secure_onboard(
    config_bundle: UploadFile = File(...),
    signature: UploadFile = File(...),
    developer_cert: UploadFile = File(...)
):
    print("\n[*] Received Onboarding Payload. Starting Zero Trust Validation...")
    bundle_path = f"/tmp/{config_bundle.filename}"
    sig_path, cert_path, pubkey_path = "/tmp/sig.bin", "/tmp/dev.crt", "/tmp/pub.pem"
    extract_dir = "/tmp/xapp-config"

    # Save payload to local /tmp
    with open(bundle_path, "wb") as f: f.write(await config_bundle.read())
    with open(sig_path, "wb") as f: f.write(await signature.read())
    with open(cert_path, "wb") as f: f.write(await developer_cert.read())

    try:
        # --- Flow Step 1: Security Cryptographic Validator ---
        print("[*] Verifying Developer Certificate against SMO Root CA...")
        subprocess.run(["openssl", "verify", "-CAfile", SMO_CA_CERT, cert_path], check=True)

        with open(pubkey_path, "w") as pub_f:
            subprocess.run(["openssl", "x509", "-in", cert_path, "-pubkey", "-noout"], stdout=pub_f, check=True)

        print("[*] Verifying Payload SHA-256 Signature...")
        subprocess.run(["openssl", "dgst", "-sha256", "-verify", pubkey_path, "-signature", sig_path, bundle_path], check=True)
        print("[+] Signature Verified! Cryptographic boundary passed.")

        # --- Flow Step 2: Extract Configuration ---
        os.makedirs(extract_dir, exist_ok=True)
        with tarfile.open(bundle_path, "r:gz") as tar:
            tar.extractall(path=extract_dir)

        config_path = os.path.join(extract_dir, "config-file.json")
        schema_path = os.path.join(extract_dir, "schema.json")

        with open(config_path, "r") as f:
            config_data = json.load(f)

        xapp_name = config_data.get("name")
        xapp_version = config_data.get("version")

        if not xapp_name or not xapp_version:
            raise ValueError("Missing 'name' or 'version' in config-file.json")

        print(f"[*] Target xApp: {xapp_name} | Version: {xapp_version}")

        # --- Flow Step 3: dms_cli Onboard ---
        env_vars = os.environ.copy()
        env_vars["CHART_REPO_URL"] = CHART_REPO_URL
        env_vars["KUBECONFIG"] = "/home/pasindu/.kube/config"
        subprocess.run(["rm", "-rf", f"/tmp/helm_template/{xapp_name}"], capture_output=True)
        print(f"[*] Executing dms_cli onboard to {CHART_REPO_URL}...")
        subprocess.run([
            "dms_cli", "onboard",
            f"--config_file_path={config_path}",
            f"--shcema_file_path={schema_path}" # Using exact spelling from O-RAN docs
        ], env=env_vars, check=True)

        # --- Flow Step 4: dms_cli Install ---
        print(f"[*] Executing dms_cli install for {xapp_name}...")
        subprocess.run([
            "dms_cli", "install",
            xapp_name,
            xapp_version,
            "ricxapp"
        ], env=env_vars, check=True)

        # --- Flow Step 5: O1 Notification to SMO ---
        print("[*] Sending O1 Notification to SMO...")
        o1_payload = {
            "notificationType": "notifyMOICreation",
            "xappName": xapp_name,
            "status": "deployed"
        }
        try:
            requests.post(EXTERNAL_SMO_O1_URL, json=o1_payload, timeout=5)
        except requests.exceptions.RequestException as e:
            print(f"[!] Warning: O1 Notification failed (is SMO running?): {e}")

        print("[+] SUCCESS: xApp securely onboarded and installed!")
        return {"status": "Onboarded & Deployed", "xapp": xapp_name, "version": xapp_version}

    except subprocess.CalledProcessError as e:
        print(f"[!] Zero Trust or CLI Error: {e}")
        raise HTTPException(status_code=403, detail="Validation or CLI execution failed.")
    except Exception as e:
        print(f"[!] Internal Error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

```

Install necessary python libraries
```bash
pip install fastapi uvicorn python-multipart requests
```

If there is a websocket error
```bash
python3 -m pip install --upgrade websockets
```

Give sudo access to .kube/config to execute kubectl commands and access kubernetes cluster without using sudo
```bash
# Create a .kube directory for the pasindu user
mkdir -p ~/.kube

# Copy the config file from root
sudo cp /etc/kubernetes/admin.conf ~/.kube/config 2>/dev/null || sudo cp /root/.kube/config ~/.kube/config

# Change the ownership of the file to pasindu
sudo chown $(id -u):$(id -g) ~/.kube/config

export KUBECONFIG=/home/pasindu/.kube/config

```
Run the script

```bash
python3 -m uvicorn local_ric_gateway:app --host 0.0.0.0 --port 8080
```
---------------------------------------------------------------------------------------------------------
We bundle the xApp descriptor files and pass them to the ric-gateway server with signed hashed files, SMO issued certificate for CSR.
```bash
tar -czvf hw-go-config.tar.gz config-file.json schema.json

# 2. Sign the configuration using the developer's private key
openssl dgst -sha256 -sign my-xapp-dev.key -out hw-go-config.sig hw-go-config.tar.gz

# 3. Submit directly to the RIC Secure Onboarder
curl -X POST http://localhost:8080/ric/v1/secure-onboard \
     -F "config_bundle=@hw-go-config.tar.gz" \
     -F "signature=@hw-go-config.sig" \
     -F "developer_cert=@my-xapp-dev.crt"
```