# Redis Databases commands

Login to the DBAAS container
```bash
kubectl exec -it <your-dbaas-pod-name> -n ricplt -- /bin/sh
```

Run the Redis Monitor Command:
This streams every single command hitting the database to your screen in real-time.
```bash
redis-cli MONITOR
```

View Specific Data:
```bash
redis-cli GET "{ue_namespace},your_specific_key"
```