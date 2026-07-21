`smo-issuer.yaml`
```bash
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: smo-root-ca
spec:
  ca:
    secretName: smo-root-ca-secret
```

`kyverno-rbac.yaml`
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

This document outlines the steps to install the required toolchains, compile the WebAssembly (Wasm) Policy Enforcement Point (PEP) for Envoy, and deploy it to the O-RAN Shared Data Layer (DBaaS).

### ⚠️ The "Golden Combination" Prerequisite

WebAssembly compilation in Go requires strictly matched dependencies. To prevent memory allocation panics (`uintptr` vs. `int`) and toolchain conflicts, you **must** use these exact versions:

* **Go Compiler:** `1.21.x` (Strictly do not use 1.22+)
* **TinyGo Compiler:** `0.30.0`
* **Proxy-Wasm Go SDK:** `v0.23.0`

---

## Step 1: Install the Standard Go Toolchain (v1.21)

TinyGo relies on the standard Go library. You must install the official Go binary, bypassing default OS package managers to ensure version accuracy.

1. **Remove any existing conflicting Go installations:**
```bash
sudo apt remove --purge golang-go golang-* -y
sudo rm -rf /usr/local/go

```


2. **Download and extract Go 1.21.10:**
```bash
wget https://go.dev/dl/go1.21.10.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.10.linux-amd64.tar.gz

```


3. **Add Go to your system PATH:**
```bash
export PATH=$PATH:/usr/local/go/bin

```


*(To make this permanent, add the export line to your `~/.bashrc` or `~/.profile`).*
4. **Verify installation:**
```bash
go version
# Expected: go version go1.21.10 linux/amd64

```



---

## Step 2: Install TinyGo (v0.30.0)

TinyGo is the specialized compiler required to output the `.wasm` format matching the `wasi` target that Envoy expects.

1. **Download the specific Debian package:**
```bash
wget https://github.com/tinygo-org/tinygo/releases/download/v0.30.0/tinygo_0.30.0_amd64.deb

```


2. **Install the package:**
```bash
sudo dpkg -i tinygo_0.30.0_amd64.deb

```


3. **Verify installation:**
```bash
tinygo version
# Expected: tinygo version 0.30.0 linux/amd64 (using go version go1.21.10...)

```


`main.go` older version. without shared memeory.
```bash
package main

import (
        "bytes"
        "fmt"
        "strings"

        "github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm"
        "github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm/types"
)

func main() {
        proxywasm.SetVMContext(&vmContext{})
}

type vmContext struct {
        types.DefaultVMContext
}

func (*vmContext) NewPluginContext(contextID uint32) types.PluginContext {
        return &pluginContext{}
}

type pluginContext struct {
        types.DefaultPluginContext
        xappName string
}

func (ctx *pluginContext) OnPluginStart(pluginConfigurationSize int) types.OnPluginStartStatus {
        configData, err := proxywasm.GetPluginConfiguration()
        if err == nil && len(configData) > 0 {
                ctx.xappName = string(configData)
                proxywasm.LogInfof("[WASM] Started successfully for xApp: %s", ctx.xappName)
        } else {
                ctx.xappName = "unknown-xapp"
                proxywasm.LogWarnf("[WASM] Started with unknown xApp name")
        }
        proxywasm.SetTickPeriodMilliSeconds(100) // Delayed start for cluster binding
        return types.OnPluginStartStatusOK
}

func (ctx *pluginContext) OnTick() {
        proxywasm.SetTickPeriodMilliSeconds(60000)
        ctx.fetchToken()
}

func (ctx *pluginContext) fetchToken() {
        proxywasm.LogInfof("[WASM] Fetching Keycloak token for %s...", ctx.xappName)
        body := fmt.Sprintf("grant_type=client_credentials&client_id=%s", ctx.xappName)
        headers := [][2]string{
                {":method", "POST"},
                {":path", "/realms/ric-realm/protocol/openid-connect/token"},
                {":authority", "keycloak.keycloak.svc.cluster.local:8443"},
                {"content-type", "application/x-www-form-urlencoded"},
        }
        _, err := proxywasm.DispatchHttpCall("keycloak_cluster", headers, []byte(body), nil, 5000, ctx.keycloakCallback)
        if err != nil {
                proxywasm.LogErrorf("[WASM] Failed to dispatch Keycloak call: %v", err)
        }
}

func (ctx *pluginContext) keycloakCallback(numHeaders, bodySize, numTrailers int) {
        body, _ := proxywasm.GetHttpCallResponseBody(0, bodySize)
        respStr := string(body)
        if strings.Contains(respStr, `"access_token":"`) {
                tokenPart := strings.Split(strings.Split(respStr, `"access_token":"`)[1], `"`)[0]
                proxywasm.SetSharedData("jwt_token", []byte(tokenPart), 0)
                proxywasm.LogInfof("[WASM] Successfully fetched and cached new Keycloak token.")
        } else {
                proxywasm.LogErrorf("[WASM] Keycloak response did not contain access_token")
        }
}

func (ctx *pluginContext) NewTcpContext(contextID uint32) types.TcpContext {
        return &redisAuthContext{xappName: ctx.xappName}
}

type redisAuthContext struct {
        types.DefaultTcpContext
        actionData string
        namespace  string
        keyData    string
        xappName   string
}

func (ctx *redisAuthContext) OnDownstreamData(dataSize int, endOfStream bool) types.Action {
        data, err := proxywasm.GetDownstreamData(0, dataSize)
        if err != nil || len(data) == 0 {
                return types.ActionContinue
        }

        payload := string(data)
        lines := strings.Split(payload, "\r\n")

        // Fallback defaults
        ctx.actionData = "UNKNOWN"
        ctx.namespace = "UNKNOWN"
        ctx.keyData = "UNKNOWN"

        // Parse RESP format: *<args>\r\n$<len>\r\n<cmd>\r\n$<len>\r\n<key>...
        if len(lines) >= 5 && strings.HasPrefix(payload, "*") {
                ctx.actionData = strings.ToUpper(lines[2])
                ctx.keyData = lines[4]

                ctx.namespace = "default"
                if strings.Contains(ctx.keyData, ":") {
                        ctx.namespace = strings.Split(ctx.keyData, ":")[0]
                }
        } else if strings.HasPrefix(payload, "PING") || strings.HasPrefix(payload, "INFO") {
                parts := strings.Split(payload, " ")
                ctx.actionData = strings.ToUpper(strings.TrimSpace(parts[0]))
        }

        proxywasm.LogInfof("[WASM] Intercepted Packet -> Cmd: %s | Namespace: %s | Key: %s", ctx.actionData, ctx.namespace, ctx.keyData)

        tokenBytes, _, _ := proxywasm.GetSharedData("jwt_token")
        token := string(tokenBytes)

        opaPayload := fmt.Sprintf(`{
                "input": {
                        "attributes": {
                                "request": {
                                        "http": {
                                                "headers": {
                                                        "x-app-id": "%s",
                                                        "x-sdl-action": "%s",
                                                        "x-sdl-namespace": "%s",
                                                        "x-sdl-key": "%s",
                                                        "authorization": "Bearer %s"
                                                }
                                        }
                                }
                        }
                }
        }`, ctx.xappName, ctx.actionData, ctx.namespace, ctx.keyData, token)

        headers := [][2]string{
                {":method", "POST"},
                {":path", "/v1/data/oransdl/allow"},
                {":authority", "opa-service"},
                {"content-type", "application/json"},
        }

        _, err = proxywasm.DispatchHttpCall("opa_cluster", headers, []byte(opaPayload), nil, 50, ctx.opaCallback)
        if err != nil {
                proxywasm.LogErrorf("[WASM] Failed to dispatch OPA call: %v", err)
                return types.ActionContinue
        }
        return types.ActionPause
}

func (ctx *redisAuthContext) opaCallback(numHeaders, bodySize, numTrailers int) {
        body, _ := proxywasm.GetHttpCallResponseBody(0, bodySize)
        if bytes.Contains(body, []byte(`"result":true`)) {
                proxywasm.LogInfof("[WASM] OPA Approved Command: %s", ctx.actionData)
                proxywasm.ContinueTcpStream()
        } else {
                proxywasm.LogWarnf("[WASM] OPA DENIED Command: %s", ctx.actionData)
                proxywasm.CloseDownstream()
        }
}

```

This is the updated main.go:
```bash
package main

import (
        "bytes"
        "encoding/binary" // Added for shared memory timestamp conversion
        "fmt"
        "strings"
        "time"            // Added for standard time retrieval

        "github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm"
        "github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm/types"
)

func main() {
        proxywasm.SetVMContext(&vmContext{})
}

type vmContext struct {
        types.DefaultVMContext
}

func (*vmContext) NewPluginContext(contextID uint32) types.PluginContext {
        return &pluginContext{}
}

type pluginContext struct {
        types.DefaultPluginContext
        xappName string
}

func (ctx *pluginContext) OnPluginStart(pluginConfigurationSize int) types.OnPluginStartStatus {
        configData, err := proxywasm.GetPluginConfiguration()
        if err == nil && len(configData) > 0 {
                ctx.xappName = string(configData)
                proxywasm.LogInfof("[WASM] Started successfully for xApp: %s", ctx.xappName)
        } else {
                ctx.xappName = "unknown-xapp"
                proxywasm.LogWarnf("[WASM] Started with unknown xApp name")
        }
        proxywasm.SetTickPeriodMilliSeconds(100) // Delayed start for cluster binding
        return types.OnPluginStartStatusOK
}

func (ctx *pluginContext) OnTick() {
        proxywasm.SetTickPeriodMilliSeconds(60000) // 60-second loop

        // 1. Get the current time in nanoseconds using standard Go time
        now := time.Now().UnixNano()

        // 2. Check the shared memory to see if another thread fetched the token recently
        lastFetchBytes, _, err := proxywasm.GetSharedData("last_token_fetch")
        if err == nil && len(lastFetchBytes) == 8 {
                // Convert bytes back to an integer timestamp
                lastFetch := int64(binary.LittleEndian.Uint64(lastFetchBytes))

                // If the token was fetched less than 50 seconds ago (50 billion nanoseconds),
                // this thread does not need to fetch. Another worker thread already did it.
                if now-lastFetch < 50*1000*1000*1000 {
                        return
                }
        }

        // 3. If we get here, this thread is acting as the "Leader".
        // Immediately update the timestamp in shared memory to block other threads from fetching.
        timeBytes := make([]byte, 8)
        binary.LittleEndian.PutUint64(timeBytes, uint64(now))
        proxywasm.SetSharedData("last_token_fetch", timeBytes, 0)

        // 4. Proceed to fetch the token
        ctx.fetchToken()
}

func (ctx *pluginContext) fetchToken() {
        proxywasm.LogInfof("[WASM] Fetching Keycloak token for %s...", ctx.xappName)
        body := fmt.Sprintf("grant_type=client_credentials&client_id=%s", ctx.xappName)
        headers := [][2]string{
                {":method", "POST"},
                {":path", "/realms/ric-realm/protocol/openid-connect/token"},
                {":authority", "keycloak.keycloak.svc.cluster.local:8443"},
                {"content-type", "application/x-www-form-urlencoded"},
        }
        _, err := proxywasm.DispatchHttpCall("keycloak_cluster", headers, []byte(body), nil, 5000, ctx.keycloakCallback)
        if err != nil {
                proxywasm.LogErrorf("[WASM] Failed to dispatch Keycloak call: %v", err)
        }
}

func (ctx *pluginContext) keycloakCallback(numHeaders, bodySize, numTrailers int) {
        body, _ := proxywasm.GetHttpCallResponseBody(0, bodySize)
        respStr := string(body)
        if strings.Contains(respStr, `"access_token":"`) {
                tokenPart := strings.Split(strings.Split(respStr, `"access_token":"`)[1], `"`)[0]
                proxywasm.SetSharedData("jwt_token", []byte(tokenPart), 0)
                proxywasm.LogInfof("[WASM] Successfully fetched and cached new Keycloak token.")
        } else {
                proxywasm.LogErrorf("[WASM] Keycloak response did not contain access_token")
        }
}

func (ctx *pluginContext) NewTcpContext(contextID uint32) types.TcpContext {
        return &redisAuthContext{xappName: ctx.xappName}
}

type redisAuthContext struct {
        types.DefaultTcpContext
        actionData string
        namespace  string
        keyData    string
        xappName   string
}

func (ctx *redisAuthContext) OnDownstreamData(dataSize int, endOfStream bool) types.Action {
        data, err := proxywasm.GetDownstreamData(0, dataSize)
        if err != nil || len(data) == 0 {
                return types.ActionContinue
        }

        payload := string(data)
        lines := strings.Split(payload, "\r\n")

        // Fallback defaults
        ctx.actionData = "UNKNOWN"
        ctx.namespace = "UNKNOWN"
        ctx.keyData = "UNKNOWN"

        // Parse RESP format: *<args>\r\n$<len>\r\n<cmd>\r\n$<len>\r\n<key>...
        if len(lines) >= 5 && strings.HasPrefix(payload, "*") {
                ctx.actionData = strings.ToUpper(lines[2])
                ctx.keyData = lines[4]

                ctx.namespace = "default"
                if strings.Contains(ctx.keyData, ":") {
                        ctx.namespace = strings.Split(ctx.keyData, ":")[0]
                }
        } else if strings.HasPrefix(payload, "PING") || strings.HasPrefix(payload, "INFO") {
                parts := strings.Split(payload, " ")
                ctx.actionData = strings.ToUpper(strings.TrimSpace(parts[0]))
        }

        proxywasm.LogInfof("[WASM] Intercepted Packet -> Cmd: %s | Namespace: %s | Key: %s", ctx.actionData, ctx.namespace, ctx.keyData)

        tokenBytes, _, _ := proxywasm.GetSharedData("jwt_token")
        token := string(tokenBytes)

        opaPayload := fmt.Sprintf(`{
                "input": {
                        "attributes": {
                                "request": {
                                        "http": {
                                                "headers": {
                                                        "x-app-id": "%s",
                                                        "x-sdl-action": "%s",
                                                        "x-sdl-namespace": "%s",
                                                        "x-sdl-key": "%s",
                                                        "authorization": "Bearer %s"
                                                }
                                        }
                                }
                        }
                }
        }`, ctx.xappName, ctx.actionData, ctx.namespace, ctx.keyData, token)

        headers := [][2]string{
                {":method", "POST"},
                {":path", "/v1/data/oransdl/allow"},
                {":authority", "opa-service"},
                {"content-type", "application/json"},
        }

        _, err = proxywasm.DispatchHttpCall("opa_cluster", headers, []byte(opaPayload), nil, 50, ctx.opaCallback)
        if err != nil {
                proxywasm.LogErrorf("[WASM] Failed to dispatch OPA call: %v", err)
                return types.ActionContinue
        }
        return types.ActionPause
}

func (ctx *redisAuthContext) opaCallback(numHeaders, bodySize, numTrailers int) {
        body, _ := proxywasm.GetHttpCallResponseBody(0, bodySize)
        if bytes.Contains(body, []byte(`"result":true`)) {
                proxywasm.LogInfof("[WASM] OPA Approved Command: %s", ctx.actionData)
                proxywasm.ContinueTcpStream()
        } else {
                proxywasm.LogWarnf("[WASM] OPA DENIED Command: %s", ctx.actionData)
                proxywasm.CloseDownstream()
        }
}

```

```bash
tinygo build -o auth-agent.wasm -scheduler=none -target=wasi main.go
```
```bash
sudo kubectl delete configmap wasm-binary-config -n ricxapp --ignore-not-found
sudo kubectl create configmap wasm-binary-config --from-file=auth-agent.wasm -n ricxapp
```
```bash
sudo kubectl delete pod -l app=ricxapp-sdl-xapp -n ricxapp
```


`zerotrust-sidecar-configs.yaml`
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
      - name: redis_interceptor
        address:
          socket_address: { address: 127.0.0.1, port_value: 6379 }
        filter_chains:
        - filters:
          # 1. WASM Filter (Zero Trust Gatekeeper)
          - name: envoy.filters.network.wasm
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.wasm.v3.Wasm
              config:
                name: "redis_authz_wasm"
                configuration:
                  "@type": type.googleapis.com/google.protobuf.StringValue
                  value: "XAPP_NAME_PLACEHOLDER"
                vm_config:
                  runtime: "envoy.wasm.runtime.v8"
                  code:
                    local:
                      filename: "/etc/envoy/auth-agent.wasm"

          # 2. TCP Proxy (Only reached if WASM allows)
          - name: envoy.filters.network.tcp_proxy
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
              stat_prefix: egress_tcp
              cluster: real_sdl_cluster

      clusters:
      - name: opa_cluster
        connect_timeout: 0.25s
        type: STRICT_DNS
        load_assignment:
          cluster_name: opa_cluster
          endpoints:
          - lb_endpoints:
            - endpoint: { address: { socket_address: { address: opa-service.ricplt.svc.cluster.local, port_value: 8181 } } }

      - name: keycloak_cluster
        connect_timeout: 1.0s
        type: STRICT_DNS
        load_assignment:
          cluster_name: keycloak_cluster
          endpoints:
          - lb_endpoints:
            - endpoint: { address: { socket_address: { address: keycloak.keycloak.svc.cluster.local, port_value: 8443 } } }
        # Keep mTLS for Keycloak auth
        transport_socket:
          name: envoy.transport_sockets.tls
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
            common_tls_context:
              tls_certificates:
                - certificate_chain: { filename: "/etc/xapp-certs/tls.crt" }
                  private_key: { filename: "/etc/xapp-certs/tls.key" }

      - name: real_sdl_cluster
        connect_timeout: 0.5s
        type: STRICT_DNS
        load_assignment:
          cluster_name: real_sdl_cluster
          endpoints:
          - lb_endpoints:
            # 1. Target the secure Envoy sidecar port (6380) instead of native Redis (6379)
            - endpoint: { address: { socket_address: { address: service-ricplt-dbaas-tcp.ricplt.svc.cluster.local, port_value: 6380 } } }
        # 2. Re-enable mTLS so it can unlock the DBaaS Envoy front door
        transport_socket:
          name: envoy.transport_sockets.tls
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
            sni: service-ricplt-dbaas-tcp.ricplt.svc.cluster.local
            common_tls_context:
              validation_context:
                trusted_ca: { filename: "/etc/xapp-certs/ca.crt" }
              tls_certificates:
                - certificate_chain: { filename: "/etc/xapp-certs/tls.crt" }
                  private_key: { filename: "/etc/xapp-certs/tls.key" }
```

`dbaas-cert.yaml`
```bash
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: dbaas-server-cert
  namespace: ricplt
spec:
  # This matches the exact secret name your Pod is waiting for
  secretName: dbaas-server-certs
  duration: 8760h # 1 year
  renewBefore: 360h # 15 days
  commonName: service-ricplt-dbaas-tcp.ricplt.svc.cluster.local
  dnsNames:
    # Essential for Envoy to verify the identity of the DBaaS across the cluster network
    - service-ricplt-dbaas-tcp
    - service-ricplt-dbaas-tcp.ricplt
    - service-ricplt-dbaas-tcp.ricplt.svc
    - service-ricplt-dbaas-tcp.ricplt.svc.cluster.local
    - localhost
  issuerRef:
    name: smo-root-ca
    kind: ClusterIssuer
```

`dbaas-sidecar-config.yaml`
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: dbaas-sidecar-config
  namespace: ricplt
data:
  envoy.yaml: |
    static_resources:
      listeners:
      - name: dbaas_mtls_ingress
        address:
          socket_address: { address: 0.0.0.0, port_value: 6380 }
        filter_chains:
        - filters:
          - name: envoy.filters.network.tcp_proxy
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
              stat_prefix: ingress_tcp
              cluster: local_redis_cluster
          transport_socket:
            name: envoy.transport_sockets.tls
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
              common_tls_context:
                validation_context:
                  trusted_ca: { filename: "/etc/dbaas-certs/ca.crt" }
                tls_certificates:
                  - certificate_chain: { filename: "/etc/dbaas-certs/tls.crt" }
                    private_key: { filename: "/etc/dbaas-certs/tls.key" }
              require_client_certificate: true
      clusters:
      - name: local_redis_cluster
        connect_timeout: 0.25s
        type: STATIC
        load_assignment:
          cluster_name: local_redis_cluster
          endpoints:
          - lb_endpoints:
            - endpoint: { address: { socket_address: { address: 127.0.0.1, port_value: 6379 } } }
```

`dbaas-patches.yaml`
```bash
# 1. StatefulSet Patch
spec:
  template:
    spec:
      containers:
      - name: dbaas-envoy-proxy
        image: envoyproxy/envoy:v1.28.0
        ports:
        - containerPort: 6380
        volumeMounts:
        - name: proxy-config
          mountPath: /etc/envoy
        - name: dbaas-certs
          mountPath: /etc/dbaas-certs
          readOnly: true
      volumes:
      - name: proxy-config
        configMap:
          name: dbaas-sidecar-config
      - name: dbaas-certs
        secret:
          secretName: dbaas-server-certs
---
# 2. Service Patch
spec:
  ports:
  - port: 6379
    targetPort: 6380 # Redirect traffic to Envoy
```

`dbaas-network-policy.yaml`

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: dbaas-secure-ingress
  namespace: ricplt
spec:
  podSelector:
    matchLabels:
      app: ricplt-dbaas
  policyTypes:
  - Ingress
  ingress:
  # --- TIER 1: Untrusted xApp Namespace ---
  # Force all external xApp traffic through the secure mTLS Envoy proxy
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ricxapp
    ports:
    - protocol: TCP
      port: 6380

  # --- TIER 2: Trusted RIC Platform Namespace ---
  # Allow core O-RAN components (E2term, E2mgr) to use the native port.
  # An empty podSelector {} means "all pods within the same namespace (ricplt)"
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 6379
```



DBaaS network policy with untrusted ricplt namespace
```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: dbaas-secure-ingress
  namespace: ricplt
spec:
  podSelector:
    matchLabels:
      app: ricplt-dbaas
  policyTypes:
  - Ingress
  ingress:
  # --- TIER 1: Untrusted xApp Namespace (Forced to Envoy mTLS Gatekeeper) ---
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ricxapp
    ports:
    - protocol: TCP
      port: 6380 

  # --- TIER 2: Trusted RIC Platform Components (Strict Micro-segmentation) ---
  # Only these specific pods are allowed to hit the native plaintext port.
  - from:
    # Allow E2 Manager
    - podSelector:
        matchLabels:
          app: ricplt-e2mgr
    # Allow E2 Termination
    - podSelector:
        matchLabels:
          app: ricplt-e2term-alpha
    # Allow Routing Manager
    - podSelector:
        matchLabels:
          app: ricplt-rtmgr
    # Allow Subscription Manager
    - podSelector:
        matchLabels:
          app: ricplt-submgr
    ports:
    - protocol: TCP
      port: 6379
```

## Install Calico for NetworkPolicy Enforcement
```bash
sudo kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
sudo kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml

wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
sed -i 's/192.168.0.0\/16/10.244.0.0\/16/g' custom-resources.yaml
sudo kubectl apply -f custom-resources.yaml
sudo kubectl get pods -n calico-system -w
```
If error occurred at a pod: `Readiness probe failed: calico/node is not ready: BIRD is not ready: Error querying BIRD: unable to connect to BIRDv4 socket`

```bash
sudo kubectl patch installation default --type=merge -p '{"spec": {"calicoNetwork": {"nodeAddressAutodetectionV4": {"canReach": "8.8.8.8"}}}}'
sudo kubectl get pods -n calico-system -w
```

## Apply Kyverno Automation

`kyverno-automation.yaml`
```bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: touchless-xapp-security
spec:
  rules:
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
              env:
                - name: XAPP_NAME
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.labels['app']
              command: ["/bin/sh", "-c"]
              args:
                - |
                  sed "s/XAPP_NAME_PLACEHOLDER/$XAPP_NAME/g" /etc/envoy-template/envoy.yaml > /etc/envoy/envoy.yaml &&
                  envoy -c /etc/envoy/envoy.yaml
              volumeMounts:
                - name: sidecar-configs
                  mountPath: /etc/envoy-template
                - name: empty-envoy-dir
                  mountPath: /etc/envoy
                - name: wasm-plugin
                  mountPath: /etc/envoy/auth-agent.wasm
                  subPath: auth-agent.wasm
                - name: xapp-tls-volume
                  mountPath: /etc/xapp-certs
                  readOnly: true
          volumes:
            - name: sidecar-configs
              configMap:
                name: zerotrust-sidecar-configs
            - name: empty-envoy-dir
              emptyDir: {}
            - name: wasm-plugin
              configMap:
                name: wasm-binary-config
            - name: xapp-tls-volume
              secret:
                secretName: "{{request.object.metadata.labels.app}}-certs"
```



`opa-policy.yaml`

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: opa-policy
  namespace: ricplt
data:
  policy.rego: |
    package oransdl

    import rego.v1

    default allow := false


    # THE CENTRALIZED POLICY MAP
    xapp_permissions := {
        "ricxapp-sdl-xapp": ["GET", "MGET", "DEL", "PING", "INFO", "KEYS"],
        "ricxapp-readonly-xapp": ["GET", "MGET", "PING", "INFO", "KEYS"],
        "ricxapp-admin-xapp": ["FLUSHALL", "SET", "GET", "MSET", "MGET", "DEL", "PING", "INFO", "KEYS"]
    }

    # AUTOMATED KEYCLOAK JWKS FETCH
    # We MUST use raw_body because OPA expects a string for the JWKS
    keycloak_jwks := jwks_response.raw_body if {
        jwks_request := {
            "method": "GET",
            "url": "https://keycloak.keycloak.svc.cluster.local:8443/realms/ric-realm/protocol/openid-connect/certs",
            "tls_insecure_skip_verify": true,
            "force_cache": true,
            "force_cache_duration_seconds": 3600
        }
        jwks_response := http.send(jwks_request)
        jwks_response.status_code == 200
    }


    # EXTRACTION
    decoded_token := payload if {
        auth_header := input.attributes.request.http.headers.authorization
        startswith(auth_header, "Bearer ")
        jwt := substring(auth_header, 7, -1)
        [_, payload, _] := io.jwt.decode(jwt)
    }


    # DIAGNOSTIC CHECKS
    is_logic_valid if {
        caller_name := decoded_token.azp
        allowed_commands := xapp_permissions[caller_name]
        action := upper(input.attributes.request.http.headers["x-sdl-action"])
        action in allowed_commands
    }

    is_signature_valid if {
        auth_header := input.attributes.request.http.headers.authorization
        jwt := substring(auth_header, 7, -1)

        [valid, _, _] := io.jwt.decode_verify(jwt, {
            "cert": keycloak_jwks,
            "aud": decoded_token.aud,
            "iss": decoded_token.iss
        })
        valid == true
    }

    # FINAL DECISION
    allow if {
        is_logic_valid
        is_signature_valid
    }
```
## Execution:
Execute the following command to open a live stream inside the secured DBaaS environment. When the xApp successfully transmits data, the decrypted RESP payload will populate in standard output, confirming end-to-end authorization and delivery.

```bash
sudo kubectl exec -it statefulset-ricplt-dbaas-server-0 -c container-ricplt-dbaas-redis -n ricplt -- redis-cli MONITOR
```