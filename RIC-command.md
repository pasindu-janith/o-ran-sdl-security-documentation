# RIC Commands

View All pods in Kuberrnetes cluster
```bash
sudo kubectl get pod -A
```

View pods in specific namespace
```bash
sudo kubectl get pod -n ricplt
```


Delete all pods and restart them within a specific namespace:
```bash
sudo kubectl delete pod --all -n ricplt
```

Check Connected E2 Nodes to the RIC:

```bash
curl -X GET http://10.244.0.136:3800/v1/nodeb/states | jq .
```

