# Test OPA in RIC Cluster

Port forwarding for test OPA using REST API HTTP Request:

```bash
sudo kubectl port-forward deployment/opa-pdp 8181:8181 -n ricplt
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

    import rego.v1

    # Only 'allow' is defined at the top level, so it is the only thing returned
    default allow = false

    allow if {
        # Move all variables inside the rule to hide them from the API output
        xapp_roles := {
            "qp": ["writer"],
            "ts": ["reader"],
            "rx": ["admin"]
        }

        role_permissions := {
            "reader": ["GET", "EXISTS"],
            "writer": ["GET", "SET", "DEL"],
            "admin":  ["GET", "SET", "DEL", "FLUSHALL"]
        }

        xapp_id := input.attributes.request.http.headers["x-app-id"]
        requested_action := input.attributes.request.http.headers["x-sdl-action"]

        # The RBAC Evaluation
        some role in xapp_roles[xapp_id]
        requested_action in role_permissions[role]
    }
    
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