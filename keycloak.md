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

# 2.1 Create the Keycloak Namespace
```bash
kubectl create namespace keycloak
```

# 2.2 Deployment Manifest
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
        args: ["start-dev", "--https-client-auth=request"]
        env:
        - name: KC_BOOTSTRAP_ADMIN_USERNAME
          value: "admin"
        - name: KC_BOOTSTRAP_ADMIN_PASSWORD
          value: "admin"
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
            memory: '512Mi'
            cpu: '500m'
          limits:
            memory: '1Gi'
            cpu: '1000m'
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

# 2.3 Apply and Verify
```bash
kubectl apply -f ~/keycloak-deployment.yaml
kubectl get pods -n keycloak -w
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
16.	In the Subject DN field enter: .*CN=xapp-test.*
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


