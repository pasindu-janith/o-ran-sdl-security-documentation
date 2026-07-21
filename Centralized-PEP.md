`secure-dbaas-cert.yaml`
```bash
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: secure-dbaas-cert
  namespace: ricplt
spec:
  secretName: secure-dbaas-tls
  dnsNames:
  - secure-dbaas-service.ricplt.svc.cluster.local
  - secure-dbaas-service.ricplt.svc
  - secure-dbaas-service
  issuerRef:
    name: smo-root-ca
    kind: ClusterIssuer
```
```bash
sudo kubectl apply -f secure-dbaas-cert.yaml
```

`fortress-configs.yaml`
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: fortress-envoy-config
  namespace: ricplt
data:
  envoy.yaml: |
    static_resources:
      listeners:
      - name: gateway_listener
        address:
          socket_address: { address: 0.0.0.0, port_value: 8080 }
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: ingress_https
              access_log:
              - name: envoy.access_loggers.stdout
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
              route_config:
                name: local_route
                virtual_hosts:
                - name: backend
                  domains: ["*"]
                  routes:
                  - match: { prefix: "/redis" }
                    route: { cluster: local_translator }
              http_filters:
              # 1. OPA EXTERNAL AUTHORIZATION
              - name: envoy.filters.http.ext_authz
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
                  transport_api_version: V3
                  grpc_service:
                    envoy_grpc:
                      cluster_name: opa_grpc_cluster
                  with_request_body:
                    max_request_bytes: 16384
                    allow_partial_message: false
                    pack_as_bytes: true
              # 2. ROUTER
              - name: envoy.filters.http.router
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

          # ENFORCING TLS TERMINATION AT THE GATEWAY
          transport_socket:
            name: envoy.transport_sockets.tls
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
              common_tls_context:
                tls_certificates:
                - certificate_chain: { filename: "/etc/envoy/certs/tls.crt" }
                  private_key: { filename: "/etc/envoy/certs/tls.key" }

      clusters:
      - name: opa_grpc_cluster
        connect_timeout: 0.25s
        type: STRICT_DNS
        http2_protocol_options: {}
        load_assignment:
          cluster_name: opa_grpc_cluster
          endpoints:
          - lb_endpoints:
            - endpoint: { address: { socket_address: { address: 127.0.0.1, port_value: 9191 } } }
      - name: local_translator
        connect_timeout: 0.25s
        type: STRICT_DNS
        load_assignment:
          cluster_name: local_translator
          endpoints:
          - lb_endpoints:
            - endpoint: { address: { socket_address: { address: 127.0.0.1, port_value: 9090 } } }
```
```bash
sudo kubectl apply -f fortress-configs.yaml
```

`secure-dbaas-service.yaml`
```bash
apiVersion: v1
kind: Service
metadata:
  name: secure-dbaas-service
  namespace: ricplt
spec:
  # This matches the standard OSC RIC DBaaS StatefulSet labels
  selector:
    app: ricplt-dbaas
  ports:
  - name: secure-envoy
    protocol: TCP
    port: 8080
    targetPort: 8080
```

`statefulset-patch.yaml`
```bash
spec:
  template:
    spec:
      containers:
      # 1. Envoy Proxy
      - name: envoy-gateway
        image: envoyproxy/envoy:v1.28.0
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: envoy-config
          mountPath: /etc/envoy/envoy.yaml
          subPath: envoy.yaml
        - name: dbaas-certs
          mountPath: /etc/envoy/certs
          readOnly: true

      # 2. Open Policy Agent
      - name: opa
        image: openpolicyagent/opa:0.61.0-envoy
        args: ["run", "--server", "--config-file=/config/config.yaml", "/policy/policy.rego"]
        ports:
        - containerPort: 9191
        volumeMounts:
        - name: opa-policy
          mountPath: /policy
        - name: opa-config
          mountPath: /config

      # 3. Protocol Translator
      - name: redis-translator
        image: pasindujanith/redis-translator:v1
        env:
        - name: OSC_DBAAS_HOST
          value: "127.0.0.1"
        - name: OSC_DBAAS_PORT
          value: "6379"

      volumes:
      - name: envoy-config
        configMap:
          name: fortress-envoy-config
      - name: opa-policy
        configMap:
          name: fortress-opa-policy
      - name: opa-config
        configMap:
          name: fortress-opa-config
      - name: dbaas-certs
        secret:
          secretName: secure-dbaas-tls
```

```bash
sudo kubectl patch statefulset statefulset-ricplt-dbaas-server -n ricplt --patch-file statefulset-patch.yaml
```

`touchless-fortress-policy.yaml`
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

  - name: inject-ambassador-sidecar
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
            - name: egress-ambassador
              image: pasindujanith/egress-ambassador:v4
              env:
                - name: XAPP_NAME
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.labels['app']
                # TARGET THE NEW SECURE SERVICE
                - name: TARGET_GATEWAY_URL
                  value: "https://secure-dbaas-service.ricplt.svc.cluster.local:8080/redis"
              volumeMounts:
                - name: xapp-tls-volume
                  mountPath: /etc/xapp-certs
                  readOnly: true
          volumes:
            - name: xapp-tls-volume
              secret:
                secretName: "{{request.object.metadata.labels.app}}-certs"
```
```bash
sudo kubectl apply -f touchless-fortress-policy.yaml
```

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: fortress-opa-policy
  namespace: ricplt
data:
  policy.rego: |
    package envoy.authz

    import rego.v1

    # 1. DEFAULT DENY
    default allow := false

    # 2. DYNAMIC JWKS FETCHING
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

    # 3. JWT EXTRACTION & SIGNATURE VERIFICATION (Production Mode)
    decoded_token := payload if {
        auth_header := input.attributes.request.http.headers.authorization
        startswith(auth_header, "Bearer ")
        jwt := substring(auth_header, 7, -1)

        # This will ONLY pass if the token signature matches Keycloak's keys
        [valid, _, payload] := io.jwt.decode_verify(jwt, {"cert": keycloak_jwks})
        valid == true
    }

    # 4. ACTION EXTRACTION
    request_command := cmd if {
        raw := input.attributes.request.http.rawBody
        cmd := upper(base64.decode(raw))
    } else := ""

    # 5. FINAL ALLOW LOGIC
    allow if {
        tok := decoded_token

        xapp_permissions := {
            "ricxapp-sdl-xapp": ["GET", "MGET", "DEL", "PING", "INFO", "KEYS", "SET"],
            "ricxapp-readonly-xapp": ["GET", "MGET", "PING", "INFO", "KEYS"],
            "ricxapp-admin-xapp": ["FLUSHALL", "SET", "GET", "MSET", "MGET", "DEL", "PING", "INFO", "KEYS"]
        }

        client_id := tok.azp
        allowed_cmds := xapp_permissions[client_id]
        cmd_is_allowed(allowed_cmds, request_command)
    }

    cmd_is_allowed(allowed_cmds, body) if {
        cmd := allowed_cmds[_]
        contains(body, cmd)
    }
```