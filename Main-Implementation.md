# Main Implementation Steps

Deploy the Keycloak server as described in keycloak.md

xApp pod contains 3 containers. 
1.xApp container
2.Envoy sidecar
3.Auth Agent Container


# Phase 1: Cert-Manager Setup & PKI Configuration

In our Zero Trust O-RAN architecture, **cert-manager** acts as the automated Public Key Infrastructure (PKI) engine inside the Kubernetes cluster. It is responsible for dynamically minting, delivering, and managing X.509 Mutual TLS (mTLS) certificates for every xApp deployed in the RIC. 

By integrating `cert-manager` with our Kyverno automation pipeline, we achieve a "Zero-Touch" identity provisioning system, completely decoupling security management from xApp development.

## 2. Installation
Cert-manager was deployed into the cluster using its standard Kubernetes manifests. This creates the necessary Custom Resource Definitions (CRDs) and the background controllers.

```bash
# Create the cert-manager namespace and install the stack
sudo kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.yaml
```
Wait until all pods are running. check pod status using this command.
```bash
sudo kubectl get pods -n cert-manager
```

create SMO root certificate as a kube secret within `cert-manager` namespace.
```bash
sudo kubectl create secret tls smo-root-ca-secret \
  --cert=ca.crt \
  --key=ca.key \
  --namespace=cert-manager
```

Apply cluster issuer for issue xApp certificates via cert-manager. create smo-issuer.yaml
```bash
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: smo-root-ca
spec:
  ca:
    secretName: smo-root-ca-secret

```

```bash
sudo kubectl apply -f smo-cluster-issuer.yaml
```

```bash
kubectl get clusterissuer smo-root-ca -o wide
# Expected Output: READY = True
```
# Phase 2: Envoy and auth agent sidecar config

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: zerotrust-sidecar-configs
  namespace: ricxapp
data:
  envoy.yaml: |
    static_resources:
      listeners:
      - name: xapp_tcp_interceptor
        address:
          socket_address: { address: 127.0.0.1, port_value: 6379 }
        filter_chains:
        - filters:
          - name: envoy.filters.network.ext_authz
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.ext_authz.v3.ExtAuthz
              stat_prefix: ext_authz
              grpc_service:
                envoy_grpc:
                  cluster_name: auth_agent_cluster
              transport_api_version: V3
          - name: envoy.filters.network.redis_proxy
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.redis_proxy.v3.RedisProxy
              stat_prefix: egress_redis
              settings: { op_timeout: 5s }
              prefix_routes:
                catch_all_route: { cluster: real_sdl_cluster }
      clusters:
      - name: auth_agent_cluster
        connect_timeout: 0.25s
        type: STRICT_DNS
        http2_protocol_options: {}
        load_assignment:
          cluster_name: auth_agent_cluster
          endpoints:
          - lb_endpoints:
            - endpoint: { address: { socket_address: { address: 127.0.0.1, port_value: 50051 } } }
      - name: real_sdl_cluster
        connect_timeout: 0.5s
        type: STRICT_DNS
        load_assignment:
          cluster_name: real_sdl_cluster
          endpoints:
          - lb_endpoints:
            - endpoint: { address: { socket_address: { address: service-ricplt-dbaas-tcp.ricplt.svc.cluster.local, port_value: 6379 } } }

  agent.py: |
    import grpc, requests, time, os
    from concurrent import futures
    import envoy.service.auth.v3.external_auth_pb2 as auth_pb2
    import envoy.service.auth.v3.external_auth_pb2_grpc as auth_pb2_grpc
    import envoy.service.auth.v3.attribute_context_pb2 as attribute_context_pb2
    from envoy.type.v3 import http_status_pb2 as status_pb2
    import urllib3

    urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

    KEYCLOAK_URL = "https://keycloak.keycloak.svc.cluster.local:8443/realms/ric-realm/protocol/openid-connect/token"
    # Note: Pointing to your deployed OPA gRPC port 9191
    OPA_GRPC_URL = "opa-service.ricplt.svc.cluster.local:9191" 
    XAPP_NAME = os.environ.get("XAPP_NAME", "unknown-xapp")
    CERT, KEY = "/etc/xapp-certs/tls.crt", "/etc/xapp-certs/tls.key"

    class AuthzServicer(auth_pb2_grpc.AuthorizationServicer):
        def __init__(self):
            self.token = None
            self.expiry = 0

        def fetch_token(self):
            print(f"[AGENT] Fetching mTLS token for {XAPP_NAME}...")
            try:
                res = requests.post(KEYCLOAK_URL, data={'grant_type': 'client_credentials', 'client_id': XAPP_NAME}, cert=(CERT, KEY), verify=False)
                if res.status_code == 200:
                    self.token = res.json().get("access_token")
                    self.expiry = time.time() + int(res.json().get("expires_in", 300))
                    return True
            except Exception as e:
                print(f"[AGENT] Keycloak Error: {e}")
            return False

        def Check(self, request, context):
            # 1. Enforce Token
            if not self.token or time.time() > (self.expiry - 10):
                if not self.fetch_token():
                    return auth_pb2.CheckResponse(status=status_pb2.Status(code=grpc.StatusCode.PERMISSION_DENIED.value[0]))

            # 2. Forward to OPA via gRPC using a Synthetic HTTP Context
            # We translate the TCP connection into HTTP headers so your Rego policy works.
            print(f"[AGENT] Asking OPA PDP for permission for {XAPP_NAME}...")
            try:
                with grpc.insecure_channel(OPA_GRPC_URL) as channel:
                    stub = auth_pb2_grpc.AuthorizationStub(channel)
                    
                    # Constructing the Payload to match your Rego 'input.attributes.request.http.headers'
                    opa_request = auth_pb2.CheckRequest(
                        attributes=attribute_context_pb2.AttributeContext(
                            request=attribute_context_pb2.AttributeContext.Request(
                                http=attribute_context_pb2.AttributeContext.HttpRequest(
                                    headers={
                                        "x-app-id": XAPP_NAME,
                                        # Connection-level auth defaults to GET to pass your policy
                                        # True per-command auth requires Redis Proxy RBAC
                                        "x-sdl-action": "SET" 
                                    }
                                )
                            )
                        )
                    )
                    # Forward the request to your OPA PDP
                    opa_response = stub.Check(opa_request)
                    return opa_response
            except Exception as e:
                print(f"[AGENT] OPA Connection Error: {e}")
                return auth_pb2.CheckResponse(status=status_pb2.Status(code=grpc.StatusCode.PERMISSION_DENIED.value[0]))

    if __name__ == '__main__':
        server = grpc.server(futures.ThreadPoolExecutor(max_workers=5))
        auth_pb2_grpc.add_AuthorizationServicer_to_server(AuthzServicer(), server)
        server.add_insecure_port('[::]:50051')
        print("[AGENT] Started on port 50051")
        server.start()
        server.wait_for_termination()

```

# Phase 3: Kyverno Automation apply

Install Kyverno in the cluster.
```bash
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.13.0/install.yaml
```

verify the pods running:
```bash
kubectl get pods -n kyverno -w
```

If not working use another version.(Optional and use only if previous version is not worked)
```bash
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.16.2/install.yaml
```

Provide cert-manager access permissions to Kyverno. create kyverno-rbac.yaml file.

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kyverno:generate-security-resources
  labels:
    app.kubernetes.io/part-of: kyverno
rules:
  # Allow Kyverno to manage cert-manager Certificates
  - apiGroups: ["cert-manager.io"]
    resources: ["certificates"]
    verbs: ["get", "list", "watch", "create", "update", "delete", "patch"]
  
  # Allow Kyverno to manage Keycloak Clients
  - apiGroups: ["keycloak.org"]
    resources: ["keycloakclients"]
    verbs: ["get", "list", "watch", "create", "update", "delete", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kyverno:generate-security-resources-binding
subjects:
  - kind: ServiceAccount
    name: kyverno-background-controller
    namespace: kyverno
  - kind: ServiceAccount
    name: kyverno-admission-controller
    namespace: kyverno
roleRef:
  kind: ClusterRole
  name: kyverno:generate-security-resources
  apiGroup: rbac.authorization.k8s.io

```
Apply kyverno_rbac.yaml to Kubernetes cluster.
```bash
sudo kubectl apply -f kyverno-rbac.yaml
```

Script Kyverno automation policies and apply it to the cluster. Create kyverno-automation.yaml

```bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: touchless-xapp-security
spec:
  rules:
  # 1. GENERATE mTLS CERTIFICATE
  - name: generate-tls-cert
    match:
      any:
      - resources:
          kinds: [Deployment]
          namespaces: [ricxapp]
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

  # 2. INJECT SIDECARS
  - name: inject-sidecars
    match:
      any:
      - resources:
          kinds: [Pod]
          namespaces: [ricxapp]
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
              image: pasindujanith/auth-agent:v1              
              env:
                - name: XAPP_NAME
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.labels['app'] 
              volumeMounts:
                - name: sidecar-configs
                  mountPath: /app/agent.py
                  subPath: agent.py
                - name: xapp-tls-volume
                  mountPath: /etc/xapp-certs
                  readOnly: true
          volumes:
            - name: sidecar-configs
              configMap:
                name: zerotrust-sidecar-configs
            - name: xapp-tls-volume
              secret:
                secretName: "{{request.object.metadata.labels.app}}-certs"

```

```bash
sudo kubectl apply -f kyverno-automation.yaml
```

After Kyverno deployment, onboard a xApp and check certificate generation and management via cert-manager.
```bash
sudo kubectl get certificate -n ricxapp
```

# Common commands

Edit config files

```bash
KUBE_EDITOR="nano" kubectl edit configmap zerotrust-sidecar-configs -n ricxapp
```

Check logs in specific container within a specific pod
```bash
sudo kubectl logs ricxapp-sdl-xapp-686946b7-765hs -c auth-agent -n ricxapp
```


```bash
sudo kubectl describe pod pod_name -n ricxapp
```