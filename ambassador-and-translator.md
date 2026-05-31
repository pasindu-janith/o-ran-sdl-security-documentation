# Egress-Ambassador
Egress-Ambassador is a python based sidecar proxy that sits next to the xApp container at centralized PEP approach. This is the python script to it. 

Create `ambassador.py`

```bash
import socket
import threading
import time
import os
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

KEYCLOAK_URL = os.environ.get("KEYCLOAK_URL", "https://keycloak.keycloak.svc.cluster.local:8443/realms/ric-realm/protocol/openid-connect/token")
# Target the new Security Gateway, NOT the original DBaaS
TARGET_GATEWAY_URL = os.environ.get("TARGET_GATEWAY_URL", "http://secure-dbaas-service.ricplt.svc.cluster.local:8080/redis")

XAPP_NAME = os.environ.get("XAPP_NAME", "unknown-xapp")

CERT_PATH = "/etc/xapp-certs/tls.crt"
KEY_PATH = "/etc/xapp-certs/tls.key"

cached_token = None
token_expiry = 0

def refresh_token_loop():
    global cached_token, token_expiry
    print(f"[AMBASSADOR] Booting up. Target Gateway: {TARGET_GATEWAY_URL}", flush=True)
    while True:
        try:
            if time.time() > (token_expiry - 10) or cached_token is None:
                print(f"[AMBASSADOR] Requesting mTLS token for {XAPP_NAME}...", flush=True)
                res = requests.post(
                    KEYCLOAK_URL,
                    data={'grant_type': 'client_credentials', 'client_id': XAPP_NAME},
                    cert=(CERT_PATH, KEY_PATH),
                    verify=False,
                    timeout=5
                )
                if res.status_code == 200:
                    data = res.json()
                    cached_token = data.get("access_token")
                    expires = int(data.get("expires_in", 120))
                    token_expiry = time.time() + expires
                    print(f"[AMBASSADOR] SUCCESS: Token secured. Sleeping for {expires} seconds.", flush=True)
                else:
                    print(f"[AMBASSADOR] WARNING: Keycloak rejected auth. Status: {res.status_code}", flush=True)
        except Exception as e:
            print(f"[AMBASSADOR] ERROR: Keycloak unreachable: {e}", flush=True)
        time.sleep(5)

def handle_xapp_client(client_socket):
    try:
        raw_resp = client_socket.recv(8192)
        if not raw_resp or not cached_token:
            client_socket.close()
            return

        print("[AMBASSADOR] Intercepted raw TCP command from xApp. Wrapping in HTTP...", flush=True)
        
        headers = {
            "Authorization": f"Bearer {cached_token}",
            "Content-Type": "application/octet-stream"
        }
        
        response = requests.post(TARGET_GATEWAY_URL, headers=headers, data=raw_resp, timeout=5)
        print(f"[AMBASSADOR] Received response from Gateway (Status: {response.status_code}). Unwrapping to TCP...", flush=True)
        
        client_socket.sendall(response.content)
    except Exception as e:
        print(f"[AMBASSADOR] ROUTING ERROR: {e}", flush=True)
    finally:
        client_socket.close()

if __name__ == "__main__":
    threading.Thread(target=refresh_token_loop, daemon=True).start()
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(("127.0.0.1", 6379))
    server.listen(128)
    
    while True:
        client_sock, _ = server.accept()
        threading.Thread(target=handle_xapp_client, args=(client_sock,), daemon=True).start()

```
Create `Dockerfile`:

```bash
FROM python:3.9-slim
WORKDIR /app
RUN pip install --no-cache-dir requests urllib3
COPY ambassador.py .
CMD ["python", "-u", "ambassador.py"]

```
Build the image:
```bash
sudo docker build -t pasindujanith/egress-ambassador:v1 .
```

Pushed it to the DockerHub.
```bash
sudo docker push pasindujanith/egress-ambassador:v1
```

# Redis Translator
This container translates the incoming HTTP requests into raw TCP that compliance with Redis Serialization Protocol.

Create `translator.py`:
```bash
import socket
from flask import Flask, request, Response
import os

app = Flask(__name__)

# Point to the original OSC RIC DBaaS Service
OSC_DBAAS_HOST = os.environ.get("OSC_DBAAS_HOST", "service-ricplt-dbaas-tcp.ricplt.svc.cluster.local")
OSC_DBAAS_PORT = int(os.environ.get("OSC_DBAAS_PORT", 6379))

@app.route('/redis', methods=['POST'])
def translate_http_to_tcp():
    raw_resp_payload = request.get_data()
    if not raw_resp_payload:
        return Response("Empty command context", status=400)
    
    try:
        # Forward to the actual StatefulSet DBaaS
        redis_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        redis_socket.connect((OSC_DBAAS_HOST, OSC_DBAAS_PORT))
        redis_socket.sendall(raw_resp_payload)
        
        redis_response = redis_socket.recv(16384)
        redis_socket.close()
        
        return Response(redis_response, mimetype='application/octet-stream')
    except Exception as e:
        return Response(f"DB Bridging Error: {str(e)}", status=500)

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=9090, threaded=True)
```

Create `Dockerfile`:
```bash
FROM python:3.9-slim
WORKDIR /app
RUN pip install --no-cache-dir flask
COPY translator.py .
CMD ["python", "-u", "translator.py"]
```
Build the docker image:
```bash
sudo docker build -t pasindujanith/redis-translator:v1 .
```

Pushed it to the DockerHub.
```bash
sudo docker push pasindujanith/redis-translator:v1
```