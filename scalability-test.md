# Download the Kubernetes Metrics Server
The Kubernetes Metrics Server was deployed to enable resource utilization monitoring within the Kubernetes cluster. It periodically collects CPU and memory usage statistics from kubelets and exposes these metrics through the Kubernetes Metrics API. These metrics are required for monitoring cluster health and evaluating the resource overhead introduced by the proposed Zero Trust framework.

The latest Metrics Server deployment manifest was downloaded from the official Kubernetes repository:
```bash
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml -O metrics-server.yaml
```

The deployment manifest was then modified before deployment. Specifically, the `--kubelet-insecure-tls` argument was added to the Metrics Server container configuration to start container without error:

```bash
nano metrics-server.yaml
```

```bash
containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls    # <--- ADD THIS LINE
        image: registry.k8s.io/metrics-server/metrics-server:v0.7.2
        ...
```
The --kubelet-insecure-tls option disables kubelet certificate verification when the Metrics Server communicates with worker nodes. This modification was necessary because the Kubernetes cluster used self-signed kubelet certificates that did not contain valid Subject Alternative Names (SANs), preventing successful TLS verification. Enabling this option allows the Metrics Server to securely retrieve resource metrics in the experimental environment.

After updating the configuration, the Metrics Server was deployed using:

```bash
sudo kubectl apply -f metrics-server.yaml

```

The deployment status was verified by monitoring the Metrics Server pod:
```bash
sudo kubectl get pods -n kube-system -w | grep metrics-server
```

Once the pod reached the `RUNNING 1/1` state, cluster resource utilization metrics were retrieved using:
```bash
sudo kubectl top nodes
```

# Scale Up the Number of xApp Pods

Scale up the pod by replication:
```bash
sudo kubectl scale deployment ricxapp-sdl-xapp --replicas=1 -n ricxapp
```


Create `record_node_metrics.sh` to record CPU, Memory utilization with respect to time while replicating xApps:

```bash
#!/bin/bash

OUTPUT_FILE="node_scalability_metrics.csv"
# Setting the exact node name based on your previous logs
NODE_NAME="ubuntu" 

echo "🚀 Starting Node Scalability Logger..."
echo "Timestamp,Total_Running_Pods,Node_CPU(m),Node_CPU(%),Node_Memory(Mi),Node_Memory(%)" > $OUTPUT_FILE

# Loop continuously until you press Ctrl+C
while true; do
  TIMESTAMP=$(date +%H:%M:%S)
  
  # 1. Count all running pods across the entire cluster
  POD_COUNT=$(sudo kubectl get pods -A --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)
  
  # 2. Fetch node metrics and append to CSV
  # The awk command strips out 'm', 'Mi', and '%' so Excel/Python can read raw integers
  sudo kubectl top node $NODE_NAME --no-headers 2>/dev/null | awk -v ts="$TIMESTAMP" -v pods="$POD_COUNT" '{
    gsub(/m/, "", $2);
    gsub(/%/, "", $3);
    gsub(/Mi/, "", $4);
    gsub(/%/, "", $5);
    print ts "," pods "," $2 "," $3 "," $4 "," $5
  }' >> $OUTPUT_FILE
  
  # Poll every 2 seconds
  sleep 2
done
```
Make it executable: 
```bash
chmod +x record_node_metrics.sh
```
Run it in seperate terminal:

```bash
sudo ./record_node_metrics.sh
```
Replicate xApp pods periodically. This script will save the metrices in csv file.

# How to Increase the Pod Limit in Kubernetes Cluster
Assuming you built this cluster using kubeadm (standard for Ubuntu deployments), here is how you override the default limit to allow 250 pods.

Step 1: Edit the Kubelet Configuration
Open the Kubelet configuration file on your Ubuntu node:

```bash
sudo nano /var/lib/kubelet/config.yaml
```
Step 2: Add the maxPods Parameter
Scroll to the bottom of the file and add this line:

YAML
```bash
maxPods: 250
```
(If maxPods already exists in the file, simply change its value to 250). Save and exit (Ctrl+O, Enter, Ctrl+X).

Step 3: Restart the Kubelet Service
Restart the Kubelet daemon so it picks up the new configuration:

```Bash

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```
Step 4: Verify the New Capacity
Give the node a few seconds to report its new status to the API server, then run this command to check its capacity:

```bash
sudo kubectl describe node | grep -i pods
```
You should see pods: 250 listed under the Capacity and Allocatable sections.

What Happens Next?
You do not need to re-run your scale command. Now that the scheduler sees room on the node, it will automatically wake up and begin deploying all of your Pending xApps. You can watch them spin up in real-time:

```Bash
sudo kubectl get pods -n ricxapp -w

```