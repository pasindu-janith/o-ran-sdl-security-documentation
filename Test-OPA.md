# Deploy OPA (Open Policy Agent)
OPA is used as the Policy Decision Point(PDP).

opa-deployment.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opa-pdp
  namespace: ricplt
spec:
  replicas: 1
  selector:
    matchLabels:
      app: opa-pdp
  template:
    metadata:
      labels:
        app: opa-pdp
    spec:
      containers:
      - name: opa
        image: openpolicyagent/opa:latest-envoy
        args:
        - "run"
        - "--server"
        - "--addr=0.0.0.0:8181"
        - "--diagnostic-addr=0.0.0.0:8282"
        - "--set=plugins.envoy_ext_authz_grpc.addr=:9191"
        - "--set=plugins.envoy_ext_authz_grpc.path=envoy/authz/allow"
        - "--set=decision_logs.console=true"
        - "/config/policy.rego"
        ports:
        - containerPort: 9191
          name: grpc
        volumeMounts:
        - name: opa-policy-vol
          mountPath: /config
      volumes:
      - name: opa-policy-vol
        configMap:
          name: opa-policy
---
apiVersion: v1
kind: Service
metadata:
  name: opa-service
  namespace: ricplt
spec:
  selector:
    app: opa-pdp
  ports:
  - name: grpc
    port: 9191
    targetPort: 9191

```

opa-policy.yaml includes the PDP policies
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

Run this command to apply this in RIC cluster.
```bash
kubectl apply -f opa-policy.yaml -f opa-deployment.yaml
```

# Test OPA in RIC Cluster

Port forwarding for test OPA using REST API HTTP Request:

```bash
sudo kubectl port-forward deployment/opa-pdp 8181:8181 -n ricplt
```

Once perform any change in opa-policy.yaml logics, apply it to RIC cluster for configMap

```bash
kubectl apply -f opa-policy.yaml
kubectl rollout restart deployment opa-pdp -n ricplt
```

Create a file called test-input.json
```bash
{
  "input": {
    "attributes": {
      "request": {
        "http": {
          "headers": {
            "x-app-id": "qp",
            "x-sdl-action": "SET"
          }
        }
      }
    }
  }
}
```

Send HTTP Request with test-input.json
```bash
curl -X POST http://localhost:8181/v1/data/envoy/authz -H "Content-Type: application/json" -d @test-input.json
```

Output will look like this:

```bash
{
  "decision_id": "aad290b1-3861-4b55-8285-a6e2dd8618dc",
  "result": {
    "allow": true
  }
}
```