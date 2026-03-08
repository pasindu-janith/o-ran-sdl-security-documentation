# srsRAN + Open5GS Setup Documentation

This document describes the setup procedure for:
- Docker installation
- Open5GS 5G Core (5GC)
- srsRAN gNB
- srsRAN UE

---

# Install Docker

Remove old Docker versions:

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

Update and install required packages:

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
```

Add Docker’s official GPG key:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker repository:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker:

```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verify Docker:

```bash
sudo docker run hello-world
```

---

# Start Open5GS Core (5GC)

Clone the repository:

```bash
git clone https://github.com/srsran/srsRAN_Project.git
cd srsRAN_Project
git checkout d5fa4f0ecb
```

Navigate to Docker directory:

```bash
cd docker/
```

Start 5GC: (Build it only if first time run)

```bash
docker compose up --build 5gc
```

---

#  Build and Start gNB

Install dependencies:

```bash
apt-get install cmake make gcc g++ pkg-config libfftw3-dev libmbedtls-dev libsctp-dev libyaml-cpp-dev libgtest-dev libzmq3-dev git curl jq -y
```

Clone repository (Recommend to clone at Desktop):

```bash
git clone https://github.com/srsran/srsRAN_Project.git
cd srsRAN_Project/
```

Build:

```bash
mkdir build
cd build/
cmake ../ -DENABLE_EXPORT=ON -DENABLE_ZEROMQ=ON
make -j`nproc`
```

Copy binary:

```bash
cp /root/srsRAN_Project/build/apps/gnb /usr/bin/gnb
```

Config file **gnb_zmq.yaml** for gNB at config directory:
```
# This configuration file example shows how to configure the srsRAN Project gNB to allow srsUE to connect to it. 
# This specific example uses ZMQ in place of a USRP for the RF-frontend, and creates an FDD cell with 10 MHz bandwidth. 
# To run the srsRAN Project gNB with this config, use the following command: 
#   sudo ./gnb -c gnb_zmq.yaml

cu_cp:
  amf:
    addr: 10.53.1.2                 # The address or hostname of the AMF.
    port: 38412
    bind_addr: 10.53.1.1            # A local IP that the gNB binds to for traffic from the AMF.
    supported_tracking_areas:
      - tac: 7
        plmn_list:
          - plmn: "00101"
            tai_slice_support_list:
              - sst: 1
  inactivity_timer: 7200            # Sets the UE/PDU Session/DRB inactivity timer to 7200 seconds. Supported: [1 - 7200].

ru_sdr:
  device_driver: zmq                # The RF driver name.
  device_args: tx_port=tcp://127.0.0.1:2000,rx_port=tcp://127.0.0.1:2001,base_srate=23.04e6 # Optionally pass arguments to the selected RF driver.
  srate: 23.04                      # RF sample rate might need to be adjusted according to selected bandwidth.
  tx_gain: 75                       # Transmit gain of the RF might need to adjusted to the given situation.
  rx_gain: 75                       # Receive gain of the RF might need to adjusted to the given situation.

cell_cfg:
  dl_arfcn: 368500                  # ARFCN of the downlink carrier (center frequency).
  band: 3                           # The NR band.
  channel_bandwidth_MHz: 20         # Bandwith in MHz. Number of PRBs will be automatically derived.
  common_scs: 15                    # Subcarrier spacing in kHz used for data.
  plmn: "00101"                     # PLMN broadcasted by the gNB.
  tac: 7                            # Tracking area code (needs to match the core configuration).
  pdcch:
    common:
      ss0_index: 0                  # Set search space zero index to match srsUE capabilities
      coreset0_index: 12            # Set search CORESET Zero index to match srsUE capabilities
    dedicated:
      ss2_type: common              # Search Space type, has to be set to common
      dci_format_0_1_and_1_1: false # Set correct DCI format (fallback)
  prach:
    prach_config_index: 1           # Sets PRACH config to match what is expected by srsUE
  pdsch:
    mcs_table: qam64                # Sets PDSCH MCS to 64 QAM
  pusch:
    mcs_table: qam64                # Sets PUSCH MCS to 64 QAM

log:
  filename: /tmp/gnb.log            # Path of the log file.
  all_level: info                   # Logging level applied to all layers.
  hex_max_size: 0

pcap:
  mac_enable: false                 # Set to true to enable MAC-layer PCAPs.
  mac_filename: /tmp/gnb_mac.pcap   # Path where the MAC PCAP is stored.
  ngap_enable: false                # Set to true to enable NGAP PCAPs.
  ngap_filename: /tmp/gnb_ngap.pcap # Path where the NGAP PCAP is stored.

```

Run gNB:

```bash
sudo gnb -c ~/Desktop/srsRAN_Project/configs/gnb_zmq.yaml
```

---

# Build and Start UE

Install dependencies:

```bash
apt-get install build-essential cmake libfftw3-dev libmbedtls-dev libboost-program-options-dev libconfig++-dev libsctp-dev git curl jq -y
```

Clone repository:

```bash
git clone https://github.com/srsRAN/srsRAN_4G.git
cd srsRAN_4G/
```

Build:

```bash
mkdir build
cd build/
cmake ../ -DENABLE_EXPORT=ON -DENABLE_ZEROMQ=ON
make -j`nproc`
```

Copy binary:

```bash
cp ~/Desktop/srsRAN_4G/build/srsue/src/srsue /usr/bin/srsue
```

Create network namespace:

```bash
ip netns add ue1
ip netns list
```

Config file for UE **ue_zmq.conf** at config directory:

```
[rf]
freq_offset = 0
tx_gain = 50
rx_gain = 40
srate = 23.04e6
nof_antennas = 1

device_name = zmq
device_args = tx_port=tcp://127.0.0.1:2001,rx_port=tcp://127.0.0.1:2000,base_srate=23.04e6

[rat.eutra]
dl_earfcn = 2850
nof_carriers = 0

[rat.nr]
bands = 3
nof_carriers = 1
max_nof_prb = 106
nof_prb = 106


[pcap]
enable = none
mac_filename = /tmp/ue_mac.pcap
mac_nr_filename = /tmp/ue_mac_nr.pcap
nas_filename = /tmp/ue_nas.pcap

[log]
all_level = info
phy_lib_level = none
all_hex_limit = 32
filename = /tmp/ue.log
file_max_size = -1

[usim]
mode = soft
algo = milenage
opc  = 63BFA50EE6523365FF14C1F45F88737D
k    = 00112233445566778899aabbccddeeff
imsi = 001010123456780
imei = 353490069873319

[rrc]
release = 15
ue_category = 4

[nas]
apn = srsapn
apn_protocol = ipv4

[gw]
netns = ue1
ip_devname = tun_srsue
ip_netmask = 255.255.255.0

[gui]
enable = false

```

Run UE:

```bash
sudo srsue ~/Desktop/srsRAN_Project/configs/ue_zmq.conf
```

---

# ✅ Expected Flow

1. Start Docker
2. Start 5GC
3. Start gNB
4. Start UE
5. Verify UE attaches to core network

---


# 📌 Notes

- Ensure ZMQ ports match between gNB and UE configurations.
- Confirm Docker containers are running using:

```bash
docker ps
```
- Check logs if UE fails to attach.
---
---
# Disaggregated RAN (CU+DU+5G Core)

Use same srsRAN_Project directory. 

```bash
cd build/apps/cu/
```

Config file of CU is given below. Locate it at srsRAN_Project/configs/cu.yaml directory.

```bash
cu_cp:
  amf:
    addr: 10.53.1.2                     # The address or hostname of the AMF.
    bind_addr: 10.53.1.1                 # A local IP that the gNB binds to for traffic from the AMF.
    supported_tracking_areas:             # Configure the TA associated with the CU-CP
      - tac: 7
        plmn_list:
          - plmn: "00101"
            tai_slice_support_list:
              - sst: 1
  f1ap:
    bind_addr: 127.0.10.1                 # Configure the F1AP bind address, this will enable the CU-cp to connect to the DU

cu_up:
  f1u:
    socket:                               # Define UDP/IP socket(s) for F1-U interface.
      -                                     # Socket 1
        bind_addr: 127.0.10.1                  # Sets the address that the F1-U socket will bind to.
        
```


Config file of DU is given below. Locate it at srsRAN_Project/configs/du.yaml directory.

```bash
f1ap:
  cu_cp_addr: 127.0.10.1        # CU-CP F1-C address
  bind_addr: 127.0.10.2         # DU local F1-C bind address


f1u:
  socket:
    - bind_addr: 127.0.10.2     # DU F1-U bind address


ru_sdr:
  device_driver: zmq
  device_args: tx_port=tcp://127.0.0.1:2000,rx_port=tcp://127.0.0.1:2001,base_srate=23.04e6
  srate: 23.04
  tx_gain: 80
  rx_gain: 40

cell_cfg:
  dl_arfcn: 368500                  # ARFCN of the downlink carrier (center frequency).
  band: 3                           # The NR band.
  channel_bandwidth_MHz: 20         # Bandwith in MHz. Number of PRBs will be automatically derived.
  common_scs: 15                    # Subcarrier spacing in kHz used for data.
  plmn: "00101"                     # PLMN broadcasted by the gNB.
  tac: 7                            # Tracking area code (needs to match the core configuration).
  pdcch:
    common:
      ss0_index: 0                  # Set search space zero index to match srsUE capabilities
      coreset0_index: 12            # Set search CORESET Zero index to match srsUE capabilities
    dedicated:
      ss2_type: common              # Search Space type, has to be set to common
      dci_format_0_1_and_1_1: false # Set correct DCI format (fallback)
  prach:
    prach_config_index: 1           # Sets PRACH config to match what is expected by srsUE
  pdsch:
    mcs_table: qam64                # Sets PDSCH MCS to 64 QAM
  pusch:
    mcs_table: qam64                # Sets PUSCH MCS to 64 QAM
```

Start CU
```bash

```

Start DU
```bash

```
---
© 2026 O-RAN SDL Security Documentation (UoR-FYP Group 11, 23rd Batch). All rights reserved.