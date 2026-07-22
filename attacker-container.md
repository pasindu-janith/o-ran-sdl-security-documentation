# Create Custom Attacker Container
How to Run the Test Cleanly
To bypass this terminal attachment issue, we just need to run the pod in the background (headless) and then check its specific container logs.

Step 1: Spawn the pod in the background
(Notice we removed the -i --tty --rm flags)

```bash
sudo kubectl run debug-attacker --image=alpine -n ricxapp --labels="app=ricxapp-sdl-xapp" --restart=Never -- nc -v -z -w 3 service-ricplt-dbaas-tcp.ricplt.svc.cluster.local 6379
```
Step 2: Read the logs of the specific attacker container
Wait about 4 seconds for the network timeout to occur, then explicitly ask for the logs of the debug-attacker container:

```Bash
sudo kubectl logs debug-attacker -c debug-attacker -n ricxapp
```
What you will see:
You will see the exact same output you saw with the Caldera agent: nc: connect to ... timed out.

Step 3: Clean up your test
Because we removed the --rm (auto-remove) flag, delete the pod manually:

```Bash
sudo kubectl delete pod debug-attacker -n ricxapp
```
You have now thoroughly proven both the Macro-segmentation (namespace blocking) and Micro-segmentation (pod blocking) of your Zero Trust architecture!