# Redis Databases commands

Login to the DBAAS container
```bash
sudo kubectl exec -it statefulset-ricplt-dbaas-server-0 -n ricplt -- redis-cli MONITOR
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