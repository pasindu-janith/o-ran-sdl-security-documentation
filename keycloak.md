# Keycloak Integration into OSC Near-RT RIC

# 1. Certificate Authority Setup
A Certificate Authority (CA) is the root of trust for the entire authentication framework. Keycloak trusts the CA, and every xApp certificate must be signed by this CA. In a production deployment this would be a proper PKI CA integrated with the AppMgr onboarding workflow. For lab or development purposes, a self-signed CA created with OpenSSL is sufficient and functionally identical.

# 1.1. Create the CA
Create a working directory for all certificates:
```bash
mkdir -p ~/ric-certs && cd ~/ric-certs
```
Generate the CA private key (4096-bit RSA for strong security):
```bash
openssl genrsa -out ca.key 4096
```
Generate the CA certificate valid for 10 years:
```bash
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=RIC-CA/O=RIC-Lab/C=LK"
```
Verify the CA certificate:
```bash
openssl x509 -in ca.crt -text -noout | grep -E "Subject:|Issuer:|Not After"
```
Expected output — Issuer and Subject are identical for a self-signed CA:
```bash
Issuer: CN=RIC-CA, O=RIC-Lab, C=LK
Subject: CN=RIC-CA, O=RIC-Lab, C=LK
Not After : Apr 10 XX:XX:XX 2036 GMT
```

# 1.2 Create the Keycloak Server Certificate
Keycloak needs its own TLS certificate to serve HTTPS. This certificate must include the VM IP address as a Subject Alternative Name (SAN) so that clients can verify it correctly.

Get your VM IP address:
```bash
VM_IP=$(hostname -I | awk '{print $1}')
echo "VM IP: $VM_IP"
```
Generate the Keycloak server key and CSR:
```bash
openssl genrsa -out keycloak-server.key 2048
openssl req -new -key keycloak-server.key -out keycloak-server.csr \
  -subj "/CN=keycloak/O=RIC-Lab/C=LK"
```
Create the SAN extension file including both the VM IP and localhost:
```bash
cat > san.ext <<EOF
subjectAltName=IP:${VM_IP},IP:127.0.0.1,DNS:localhost
EOF
```
Sign the certificate with the CA:
```bash
openssl x509 -req -days 365 \
  -in keycloak-server.csr \
  -CA ca.crt -CAkey ca.key \
  -CAcreateserial \
  -out keycloak-server.crt \
  -extfile san.ext
```

# 1.3 Create the Java Truststore
Keycloak is a Java application and requires trusted CA certificates in JKS (Java KeyStore) format. The truststore is what enables Keycloak to verify that an xApp's certificate was signed by the trusted CA during mTLS authentication.
```bash
keytool -import \
  -alias ric-ca \
  -file ca.crt \
  -keystore truststore.jks \
  -storepass changeit \
  -noprompt
```
```bash
Verify the CA is in the truststore:
keytool -list -keystore truststore.jks -storepass changeit
```

# 1.4 Store Certificates as Kubernetes Secrets
The certificates must be available inside the Keycloak pod via Kubernetes secrets mounted as volumes.

Create the TLS secret for the Keycloak server certificate:
```bash
cd ~/ric-certs
kubectl create secret tls keycloak-tls \
  --cert=keycloak-server.crt \
  --key=keycloak-server.key \
  -n keycloak
```
Create the secret containing the CA certificate and truststore:
```bash
kubectl create secret generic keycloak-ca \
  --from-file=ca.crt=ca.crt \
  --from-file=truststore.jks=truststore.jks \
  -n keycloak
```
Verify secrets were created:
```bash
kubectl get secrets -n keycloak
```

# 2. Deploying Keycloak in Kubernetes
Keycloak is deployed as a standard Kubernetes Deployment in its own namespace alongside the RIC platform. For development and research purposes, Keycloak runs in dev mode using an embedded H2 database. This means realm and client configuration will be lost if the pod is restarted. For production, replace the H2 database with PostgreSQL backed by a persistent volume.
⚠  WARNING
Keycloak previously ran in start-dev mode with an embedded H2 in-memory database, causing all realm and client configuration to be lost on every pod restart or VM reboot. This section replaces that setup with a PostgreSQL-backed deployment. Realm configuration now survives indefinitely.


# 2.1 Create the Keycloak Namespace
```bash
kubectl create namespace keycloak
```
# 2.2 Create the Database Credentials Secret
Store PostgreSQL credentials as a Kubernetes secret so they are never hardcoded in the deployment manifest.
```bash
kubectl create secret generic keycloak-db-secret \
  --from-literal=POSTGRES_USER=keycloak \
  --from-literal=POSTGRES_PASSWORD=keycloak123 \
  --from-literal=POSTGRES_DB=keycloak \
  -n keycloak

```
# 2.3 Install Local Storage Provisioner
A bare-metal Kubernetes cluster has no StorageClass by default, which prevents PersistentVolumeClaims from binding. Install the local-path-provisioner — the same storage backend used by K3s — to enable persistent volumes on a single-node cluster
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```
Wait for the provisioner to be ready
```bash
kubectl rollout status deployment/local-path-provisioner -n local-path-storage
```
Verify the storage class is now available:
```bash
kubectl get storageclass
```
Expected output:
```bash
NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
local-path   rancher.io/local-path   Delete          WaitForFirstConsumer
```

# 2.4 PostgreSQL Deployment Manifest
Create the deployment file:
```bash
nano ~/postgres-deployment.yaml 
```
following manifest:
```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: keycloak
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: keycloak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        envFrom:
        - secretRef:
            name: keycloak-db-secret
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: postgres-data
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: keycloak
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432

```

Apply and wait for postgres to be ready:
```bash
kubectl apply -f ~/postgres-deployment.yaml
kubectl rollout status deployment/postgres -n keycloak

```
# 2.5 Keycloak Deployment Manifest
Create the deployment file:
```bash
nano ~/keycloak-deployment.yaml
```
Use the following manifest. Note that NodePorts 32090 (HTTP) and 32444 (HTTPS) must be free — check existing allocations with kubectl get svc -A if conflicts occur:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: keycloak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
      - name: keycloak
        image: quay.io/keycloak/keycloak:latest
        args: ["start", "--https-client-auth=request", "--db=postgres", "--optimized=false"]
        env:
        - name: KC_BOOTSTRAP_ADMIN_USERNAME
          value: "admin"
        - name: KC_BOOTSTRAP_ADMIN_PASSWORD
          value: "admin"
        - name: KC_HTTP_ENABLED
          value: "true"
        - name: KC_PROXY_HEADERS
          value: "xforwarded"
        - name: KC_HTTPS_CERTIFICATE_FILE
          value: "/etc/keycloak-tls/tls.crt"
        - name: KC_HTTPS_CERTIFICATE_KEY_FILE
          value: "/etc/keycloak-tls/tls.key"
        - name: KC_HTTPS_TRUST_STORE_FILE
          value: "/etc/keycloak-ca/truststore.jks"
        - name: KC_HTTPS_TRUST_STORE_PASSWORD
          value: "changeit"
        - name: KC_HOSTNAME_STRICT
          value: "false"
        - name: KC_HOSTNAME_STRICT_HTTPS
          value: "false"
        - name: KC_DB
          value: "postgres"
        - name: KC_DB_URL
          value: "jdbc:postgresql://postgres.keycloak.svc.cluster.local:5432/keycloak"
        - name: KC_DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: keycloak-db-secret
              key: POSTGRES_USER
        - name: KC_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: keycloak-db-secret
              key: POSTGRES_PASSWORD
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 8443
          name: https
        volumeMounts:
        - name: keycloak-tls
          mountPath: /etc/keycloak-tls
          readOnly: true
        - name: keycloak-ca
          mountPath: /etc/keycloak-ca
          readOnly: true
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
      volumes:
      - name: keycloak-tls
        secret:
          secretName: keycloak-tls
      - name: keycloak-ca
        secret:
          secretName: keycloak-ca
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: keycloak
spec:
  selector:
    app: keycloak
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 32090
    name: http
  - port: 8443
    targetPort: 8443
    nodePort: 32444
    name: https
  type: NodePort

```

# 2.6 Apply and Verify
```bash
kubectl apply -f ~/keycloak-deployment.yaml
kubectl get pods -n keycloak -w
kubectl rollout status deployment/keycloak -n keycloak
```
Wait until the pod shows STATUS = Running. Then verify Keycloak has started on both HTTP and HTTPS:
```bash
kubectl logs -n keycloak deployment/keycloak | grep -E 'started|ERROR'
```
Expected log line confirming successful startup:
```bash
Keycloak 26.x.x on JVM started in Xs. Listening on: http://0.0.0.0:8080 and https://0.0.0.0:8443
```
Verify HTTPS is reachable using the CA certificate:
```bash
curl --cacert ~/ric-certs/ca.crt \
  https://localhost:32444/realms/master/.well-known/openid-configuration
```
NOTE: Keycloak takes approximately 40-60 seconds to start. If the curl fails immediately after the pod shows Running, wait a moment and retry.

# 3. Configuring Keycloak for xApp Authentication
All configuration in this section is performed through the Keycloak Admin Console. Access it at:
http://<VM-IP>:32090
Login with username admin and password admin.

# 3.1 Create the RIC Realm
A realm in Keycloak is an isolated authentication domain. Creating a dedicated realm for the RIC keeps xApp authentication completely separate from the Keycloak master realm.

1.	Click the Keycloak dropdown at the top-left of the admin console (shows 'master').
2.	Click Create Realm.
3.	Set Realm name to ric-realm.
4.	Ensure Enabled is ON.
5.	Click Create.

All subsequent configuration in this section is performed within ric-realm.

# 3.2 Create an xApp Client
Each xApp that needs to access the SDL requires a corresponding client registration in Keycloak. The client ID should match the CN of the xApp's certificate to keep identity management clear.

Step 1 — Basic Client Setup
6.	In the left sidebar click Clients, then Create client.
7.	Set Client type to OpenID Connect.
8.	Set Client ID to xapp-test (or the CN of the xApp certificate).
9.	Click Next.

Step 2 — Capability Configuration
10.	Turn Client authentication ON.
11.	Check Service accounts roles (enables machine-to-machine client credentials flow).
12.	Uncheck Standard flow (not needed for RIC services).
13.	Click Next, then Save.

Step 3 — Configure X.509 Certificate Authentication
14.	Open the xapp-test client and navigate to the Credentials tab.
15.	Change Client Authenticator from Client Id and Secret to X509 Certificate.
16.	In the Subject DN field enter: .*CN=xapp-test.* //.*CN=xapp-test.*  
17.	Check the Allow regex pattern comparison checkbox.
18.	Click Save.

NOTE: The Subject DN field uses a regex pattern to match the certificate subject. The .* wildcards accommodate spaces that OpenSSL places around the = sign in distinguished names (e.g. CN = xapp-test vs CN=xapp-test).

# 3.3 Verify Client Configuration via API
Use the Keycloak Admin API to confirm the client is correctly configured:

Get an admin access token:
```bash
TOKEN=$(curl -s -k -X POST \
  https://localhost:32444/realms/master/protocol/openid-connect/token \
  -d "grant_type=password" \
  -d "client_id=admin-cli" \
  -d "username=admin" \
  -d "password=admin" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
```
Query the xapp-test client configuration:
```bash
curl -s -k \
  -H "Authorization: Bearer $TOKEN" \
  https://localhost:32444/admin/realms/ric-realm/clients | \
  python3 -c "
import sys,json
clients = json.load(sys.stdin)
for c in clients:
    if c['clientId'] == 'xapp-test':
        print(json.dumps(c, indent=2))
"
```
Confirm these values in the response:
Attribute	Expected Value
clientAuthenticatorType	client-x509
x509.subjectdn	.*CN=xapp-test.*
x509.allow.regex.pattern.comparison	true
serviceAccountsEnabled	true

# 4. xApp Certificate Issuance
Each xApp requires its own unique X.509 certificate signed by the RIC CA. In a production deployment this process is automated during AppMgr onboarding. During development, certificates are created manually with OpenSSL.

# 4.1 Create an xApp Certificate
cd ~/ric-certs

Generate the xApp private key:
```bash
openssl genrsa -out xapp-test.key 2048
```
Generate the Certificate Signing Request. The CN must match the Client ID registered in Keycloak:
```bash
openssl req -new -key xapp-test.key -out xapp-test.csr \
  -subj "/CN=xapp-test/O=RIC-Lab/C=LK"
```
Sign the CSR with the RIC CA:
```bash
openssl x509 -req -days 365 \
  -in xapp-test.csr \
  -CA ca.crt -CAkey ca.key \
  -CAcreateserial \
  -out xapp-test.crt
```
Verify the certificate subject:
```bash
openssl x509 -in xapp-test.crt -text -noout | grep Subject
```
Expected output:
Subject: CN = xapp-test, O = RIC-Lab, C = LK

# 4.2 Store Certificate as Kubernetes Secret
In a real deployment the sidecar proxy inside the xApp pod needs access to the certificate and private key. These are stored as Kubernetes secrets and mounted into the pod as a volume.
```bash
kubectl create secret generic xapp-test-tls \
  --from-file=tls.crt=~/ric-certs/xapp-test.crt \
  --from-file=tls.key=~/ric-certs/xapp-test.key \
  --from-file=ca.crt=~/ric-certs/ca.crt \
  -n ricxapp
```
 
# 5. Testing the mTLS Token Flow
Before deploying the sidecar proxy, the mTLS token request should be tested directly from the command line to confirm the entire authentication chain is functioning.

# 5.1 mTLS Token Request
This simulates exactly what the sidecar proxy will do at runtime:
```bash
curl --cacert ~/ric-certs/ca.crt \
  --cert ~/ric-certs/xapp-test.crt \
  --key ~/ric-certs/xapp-test.key \
  -X POST \
  https://localhost:32444/realms/ric-realm/protocol/openid-connect/token \
  -d "grant_type=client_credentials" \
  -d "client_id=xapp-test"

Successful response:
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 300
}
```

# 5.2 Decode and Inspect the Token
The access token is a JWT. Decode it to inspect the claims:
```bash
TOKEN=$(curl -s --cacert ~/ric-certs/ca.crt \
  --cert ~/ric-certs/xapp-test.crt \
  --key ~/ric-certs/xapp-test.key \
  -X POST \
  https://localhost:32444/realms/ric-realm/protocol/openid-connect/token \
  -d "grant_type=client_credentials" \
  -d "client_id=xapp-test" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
```
The decoded token will contain fields including iss (issuer URL), sub (subject), exp (expiry timestamp), and azp (authorised party matching your client ID).

 
# 6. OIDC Discovery Endpoint
The OIDC discovery endpoint is a well-known URL that advertises all of Keycloak's authentication capabilities and endpoint addresses. SDL and other RIC components use this URL to automatically discover how to validate tokens without hardcoded configuration.

curl --cacert ~/ric-certs/ca.crt \
  https://localhost:32444/realms/ric-realm/.well-known/openid-configuration

# 7. xApp Claims in JWT Tokens
Each xApp that authenticates with Keycloak using its X.509 certificate receives a JWT access token. By default this token only contains identity fields. To enable fine-grained SDL access control, additional claims must be injected into the token — specifically the SDL namespaces the xApp is permitted to access and the operations it is allowed to perform.

The claims are stored in Keycloak's PostgreSQL database against the service account user that belongs to each client. Protocol mappers read these stored attributes at token issuance time and inject them as claims into the signed JWT. No changes to the certificate are needed.

# 7.1 How Claims Flow into the Token
The flow from xApp authentication to a claim-bearing JWT involves four components:

•	Keycloak client: registered with clientId matching the certificate CN and X.509 authenticator type.
•	Service account user: automatically created by Keycloak for each client; holds the claim values as user attributes in PostgreSQL.
•	Protocol mappers: configured on the client; read the user attributes and inject them as named claims in the access token.
•	JWT access token: returned to the xApp after mTLS authentication; contains the injected claims alongside standard OIDC fields.

# 7.2 Step-by-Step: Attaching Claims to a Client
Step 1 — Get an Admin Token
All configuration is performed via the Keycloak Admin REST API. Obtain an admin token from the master realm first. This token is used for all subsequent API calls in this section.

```bash
MASTER_TOKEN=$(curl -s --cacert ~/ric-certs/ca.crt -X POST \
  https://localhost:32444/realms/master/protocol/openid-connect/token \
  -d "grant_type=password" \
  -d "client_id=admin-cli" \
  -d "username=admin" \
  -d "password=admin" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
 
echo "Token: ${MASTER_TOKEN:0:20}..."
```

Step 2 — Get the Client UUID and Service Account User ID
Each Keycloak client has an internal UUID distinct from its clientId. The service account user ID is also needed to set attribute values. Retrieve both:

```bash
# Get client UUID
CLIENT_UUID=$(curl -s --cacert ~/ric-certs/ca.crt \
  -H "Authorization: Bearer $MASTER_TOKEN" \
  "https://localhost:32444/admin/realms/ric-realm/clients?clientId=ricxapp-sdl-xapp" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)[0]['id'])")
echo "Client UUID: $CLIENT_UUID"
 
# Get service account user ID
SA_ID=$(curl -s --cacert ~/ric-certs/ca.crt \
  -H "Authorization: Bearer $MASTER_TOKEN" \
  "https://localhost:32444/admin/realms/ric-realm/clients/$CLIENT_UUID/service-account-user" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "Service account user ID: $SA_ID"

```

Step 3 — Set Claim Values on the Service Account User
The actual claim values (what gets injected into the token) are stored as user attributes on the service account user. Set them with a single PUT request:

```bash
curl -s --cacert ~/ric-certs/ca.crt \
  -H "Authorization: Bearer $MASTER_TOKEN" \
  -H "Content-Type: application/json" \
  -X PUT "https://localhost:32444/admin/realms/ric-realm/users/$SA_ID" \
  -d '{
    "username": "service-account-ricxapp-sdl-xapp",
    "attributes": {
      "x_app_id":    ["ricxapp-sdl-xapp"],
      "x_sdl_action": ["SET"]
    }
  }'

# Verify the attributes were saved:
curl -s --cacert ~/ric-certs/ca.crt \
  -H "Authorization: Bearer $MASTER_TOKEN" \
  "https://localhost:32444/admin/realms/ric-realm/users/$SA_ID" | \
  python3 -c "import sys,json; u=json.load(sys.stdin); print(u.get('attributes','MISSING'))"

#Expected output:
{'x_app_id': ['ricxapp-sdl-xapp'], 'x_sdl_action': ['SET']}

```

Step 4 — Add Protocol Mappers
Protocol mappers are the bridge between stored user attributes and JWT claims. Add one mapper per claim. The user.attribute field must exactly match the attribute key set in Step 3. The claim.name field is what appears in the JWT.
Save the following as a script to avoid terminal corruption of JSON field names:

```bash
cat > ~/add-mappers.sh << 'EOF'
#!/bin/bash
MASTER_TOKEN=$(curl -s --cacert ~/ric-certs/ca.crt -X POST \
  https://localhost:32444/realms/master/protocol/openid-connect/token \
  -d "grant_type=password" -d "client_id=admin-cli" \
  -d "username=admin" -d "password=admin" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
 
CLIENT_UUID=$(curl -s --cacert ~/ric-certs/ca.crt \
  -H "Authorization: Bearer $MASTER_TOKEN" \
  "https://localhost:32444/admin/realms/ric-realm/clients?clientId=ricxapp-sdl-xapp" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)[0]['id'])")
 
echo "Client UUID: $CLIENT_UUID"
 
# Mapper: x_app_id -> x_app_id claim
curl -s --cacert ~/ric-certs/ca.crt \
  -H "Authorization: Bearer $MASTER_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "https://localhost:32444/admin/realms/ric-realm/clients/$CLIENT_UUID/protocol-mappers/models" \

  -d '{"name":"x-app-id","protocol":"openid-connect","protocolMapper":"oidc-usermodel-attribute-mapper","config":{"user.attribute":"x_app_id","claim.name":"x_app_id","jsonType.label":"String","access.token.claim":"true","id.token.claim":"false","userinfo.token.claim":"false","aggregate.attrs":"false"}}'
echo "x-app-id mapper: $?"
 
# Mapper: x_sdl_action -> x_sdl_action claim
curl -s --cacert ~/ric-certs/ca.crt \
  -H "Authorization: Bearer $MASTER_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "https://localhost:32444/admin/realms/ric-realm/clients/$CLIENT_UUID/protocol-mappers/models" \
  -d '{"name":"x-sdl-action","protocol":"openid-connect","protocolMapper":"oidc-usermodel-attribute-mapper","config":{"user.attribute":"x_sdl_action","claim.name":"x_sdl_action","jsonType.label":"String","access.token.claim":"true","id.token.claim":"false","userinfo.token.claim":"false","aggregate.attrs":"false"}}'
echo "x-sdl-action mapper: $?"
EOF
 
bash ~/add-mappers.sh


```

Step 5 — Request a Token and Verify Claims
Extract the certificate and key from the cert-manager secret, then request a token and decode the claims:

```bash
# Extract cert-manager issued certificate
kubectl get secret ricxapp-sdl-xapp-certs -n ricxapp \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/xapp.crt
kubectl get secret ricxapp-sdl-xapp-certs -n ricxapp \
  -o jsonpath='{.data.tls\.key}' | base64 -d > /tmp/xapp.key
 
# Confirm the CN matches the Keycloak client ID
openssl x509 -in /tmp/xapp.crt -noout -subject
 
# Request token via mTLS
curl -s --cacert ~/ric-certs/ca.crt \
  --cert /tmp/xapp.crt \
  --key /tmp/xapp.key \
  -X POST \
  https://localhost:32444/realms/ric-realm/protocol/openid-connect/token \
  -d "grant_type=client_credentials" \
  -d "client_id=ricxapp-sdl-xapp" | \
  python3 -c "
import sys,json,base64
resp=json.load(sys.stdin)
if 'error' in resp:
    print('ERROR:', resp); sys.exit(1)
t=resp['access_token']
p=t.split('.')[1]
p+='='*(-len(p)%4)
c=json.loads(base64.b64decode(p))
print()
print('All claims:')
for k,v in sorted(c.items()):
    print(f'  {k}: {v}')
print()
print('SDL claims:')
print('  x_app_id    :', c.get('x_app_id','MISSING'))
print('  x_sdl_action:', c.get('x_sdl_action','MISSING'))
"
 
# Cleanup
rm -f /tmp/xapp.crt /tmp/xapp.key

# Expected output:
All claims:
  acr: 1
  aud: account
  azp: ricxapp-sdl-xapp
  exp: 1777434464
  iss: https://localhost:32444/realms/ric-realm
  jti: tRrtcc:...
  scope: email profile
  sub: 7497c729-83ec-41fe-b246-d1a54ef1b400
  x_app_id: ricxapp-sdl-xapp
  x_sdl_action: SET
 
SDL claims:
  x_app_id    : ricxapp-sdl-xapp
  x_sdl_action: SET

```
# 7.3 Idempotent Full Setup Script
The following script handles the complete claims setup for any xApp client. It checks whether each resource already exists before creating it, so it is safe to run multiple times without duplicating configuration. Use this as the standard onboarding script for new xApps.

```bash
cat > ~/setup-xapp-claims.sh << 'EOF'
#!/bin/bash
set -e
 
XAPP_CLIENT="ricxapp-sdl-xapp"          # Keycloak client ID = cert CN
SA_USERNAME="service-account-$XAPP_CLIENT"
SDL_NAMESPACES="ric.test"
SDL_OPERATIONS="read,write"
X_APP_ID="$XAPP_CLIENT"
X_SDL_ACTION="SET"
 
echo "===== xApp Claims Setup: $XAPP_CLIENT ====="
 
MASTER_TOKEN=$(curl -s --cacert ~/ric-certs/ca.crt -X POST \
  https://localhost:32444/realms/master/protocol/openid-connect/token \
  -d "grant_type=password" -d "client_id=admin-cli" \
  -d "username=admin" -d "password=admin" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
echo "[1] Admin token acquired"
 
CLIENT_UUID=$(curl -s --cacert ~/ric-certs/ca.crt \
  -H "Authorization: Bearer $MASTER_TOKEN" \
  "https://localhost:32444/admin/realms/ric-realm/clients?clientId=$XAPP_CLIENT" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)[0]['id'])")
echo "[2] Client UUID: $CLIENT_UUID"
 
SA_ID=$(curl -s --cacert ~/ric-certs/ca.crt \
  -H "Authorization: Bearer $MASTER_TOKEN" \
  "https://localhost:32444/admin/realms/ric-realm/clients/$CLIENT_UUID/service-account-user" | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "[3] Service account user ID: $SA_ID"
 
curl -s --cacert ~/ric-certs/ca.crt \
  -H "Authorization: Bearer $MASTER_TOKEN" \
  -H "Content-Type: application/json" \
  -X PUT "https://localhost:32444/admin/realms/ric-realm/users/$SA_ID" \
  -d "{\"username\":\"$SA_USERNAME\",\"attributes\":{\"x_app_id\":[\"$X_APP_ID\"],\"x_sdl_action\":[\"$X_SDL_ACTION\"],\"sdl_namespaces\":[\"$SDL_NAMESPACES\"],\"sdl_operations\":[\"$SDL_OPERATIONS\"]}}"
echo "[4] Attributes set"
 
EXISTING=$(curl -s --cacert ~/ric-certs/ca.crt \
  -H "Authorization: Bearer $MASTER_TOKEN" \
  "https://localhost:32444/admin/realms/ric-realm/clients/$CLIENT_UUID/protocol-mappers/models" | \
  python3 -c "import sys,json; print([m['name'] for m in json.load(sys.stdin)])")
 
for MAPPER_NAME in "x-app-id" "x-sdl-action" "sdl-namespaces" "sdl-operations"; do
  if echo "$EXISTING" | grep -q "$MAPPER_NAME"; then
    echo "[5] Mapper $MAPPER_NAME already exists, skipping."
  else
    case $MAPPER_NAME in
      x-app-id)      UA="x_app_id";      CN="x_app_id";;
      x-sdl-action)  UA="x_sdl_action";  CN="x_sdl_action";;
      sdl-namespaces) UA="sdl_namespaces"; CN="sdl_ns";;
      sdl-operations) UA="sdl_operations"; CN="sdl_ops";;
    esac
    curl -s --cacert ~/ric-certs/ca.crt \
      -H "Authorization: Bearer $MASTER_TOKEN" \
      -H "Content-Type: application/json" \
      -X POST "https://localhost:32444/admin/realms/ric-realm/clients/$CLIENT_UUID/protocol-mappers/models" \
      -d "{\"name\":\"$MAPPER_NAME\",\"protocol\":\"openid-connect\",\"protocolMapper\":\"oidc-usermodel-attribute-mapper\",\"config\":{\"user.attribute\":\"$UA\",\"claim.name\":\"$CN\",\"jsonType.label\":\"String\",\"access.token.claim\":\"true\",\"id.token.claim\":\"false\",\"userinfo.token.claim\":\"false\",\"aggregate.attrs\":\"false\"}}"
    echo "[5] Mapper $MAPPER_NAME added"
  fi
done
 
echo ""
echo "===== Verifying token claims ====="
kubectl get secret ricxapp-sdl-xapp-certs -n ricxapp \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/xapp.crt
kubectl get secret ricxapp-sdl-xapp-certs -n ricxapp \
  -o jsonpath='{.data.tls\.key}' | base64 -d > /tmp/xapp.key
 
curl -s --cacert ~/ric-certs/ca.crt \
  --cert /tmp/xapp.crt --key /tmp/xapp.key \
  -X POST \
  https://localhost:32444/realms/ric-realm/protocol/openid-connect/token \
  -d "grant_type=client_credentials" \
  -d "client_id=$XAPP_CLIENT" | \
  python3 -c "
import sys,json,base64
t=json.load(sys.stdin)['access_token']
p=t.split('.')[1]; p+='='*(-len(p)%4)
c=json.loads(base64.b64decode(p))
print('x_app_id    :', c.get('x_app_id','MISSING'))
print('x_sdl_action:', c.get('x_sdl_action','MISSING'))
print('sdl_ns      :', c.get('sdl_ns','MISSING'))
print('sdl_ops     :', c.get('sdl_ops','MISSING'))
ok=all(c.get(k) for k in ['x_app_id','x_sdl_action','sdl_ns','sdl_ops'])
print('STATUS:', 'SUCCESS' if ok else 'FAILED')
"
rm -f /tmp/xapp.crt /tmp/xapp.key
EOF
 
bash ~/setup-xapp-claims.sh


```