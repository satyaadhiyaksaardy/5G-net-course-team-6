# 5G Healthcare Simulation with OpenAirInterface (OAI) in a Virtualized Environment

## Project Overview

This project demonstrates a full end-to-end 5G standalone (SA) network using the OpenAirInterface (OAI) 5G stack in a virtualized environment, tailored for a healthcare IoT scenario. For example, wearable health sensors (5G UEs) transmit patient data to a cloud-based health monitoring application over a 5G network. The setup includes:

- **OAI 5G Core (CN5G)**
- **OAI gNB (5G base station)**
- **OAI nrUE (5G user equipment)**

By following this guide, you'll create a virtual testbed showcasing how 5G enables new healthcare possibilities and supports smart medical devices. The README is organized into: prerequisites, environment setup, configuration, deployment, network testing, and healthcare data flow simulation.

## System Architecture

```
[ Wearable UE ] <--RF--> [ gNB (RAN) ] <--IP--> [ 5G Core (AMF, SMF, UPF) ] <--IP--> [ Healthcare App Server ]
```
- **UE**: Virtualized nrUE process simulating wearable sensors.
- **gNB**: OAI softmodem providing simulated RF and NG interfaces.
- **Core**: Dockerized CN5G containers (AMF, SMF, UPF, NRF, AUSF).
- **Server**: Health monitoring service receiving sensor data over the 5G data network.

## Prerequisites

- **Hardware Resources**: A machine or cloud instance capable of multiple VMs/containers. Recommended: two or three Ubuntu 24.04 LTS VMs with:
  - Combined Core/gNB host: ≥8-core x86\_64 CPU (3.5 GHz+), 32 GB RAM
  - UE host: \~8 GB RAM
  - (Optional) SDR hardware requires appropriate slots/ports
- **Operating System**: Ubuntu 24.04 LTS 64‑bit
- **Virtualization Platform**: VirtualBox, VMware, KVM, or Docker. Ensure bridged or internal networking between VMs.
- **Networking**: Network interface for gNB→Core. For RF simulation, no public spectrum needed. For SDR, install UHD drivers and attach device to gNB host.
- **Software & Tools**: `git`, `build-essential`, `docker`, `docker-compose` (Compose V2 plugin), `wireshark` (for latency tests), internet access.

> **Note:** This guide assumes a fully virtualized radio environment using OAI’s RF simulator (no physical hardware). For SDR use (e.g., USRP B210/N300), meet additional prerequisites (UHD drivers, device access).

## Environment Setup

### A. Setting up the OAI 5G Core (CN5G)

On the Core VM (Ubuntu 24.04):

1. **Install Docker & dependencies**

   ```bash
   sudo apt update
   sudo apt install -y git net-tools curl ca-certificates
   # Install Docker Engine & Compose plugin
   sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   # Add user to docker group, then reboot
   sudo usermod -aG docker $USER && sudo reboot
   ```

2. **Obtain OAI CN5G configuration**

   ```bash
   wget -O ~/oai-cn5g.zip "https://gitlab.eurecom.fr/oai/openairinterface5g/-/archive/develop/openairinterface5g-develop.zip?path=doc/tutorial_resources/oai-cn5g"
   unzip ~/oai-cn5g.zip -d ~/
   mv ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g/doc/tutorial_resources/oai-cn5g ~/oai-cn5g
   rm -rf ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g ~/oai-cn5g.zip
   ```

3. **Pull OAI core Docker images**

   ```bash
   cd ~/oai-cn5g
   docker compose pull
   ```

4. **(Optional) Verify & adjust configs**

   - Check `amf_config.yaml`, `smf_config.yaml`, etc. for PLMN (MCC/MNC), IPs, subscriber definitions.
   - Ensure Core VM IP (e.g., 192.168.1.100) is reachable by gNB.

### B. Setting up the OAI gNB and nrUE (RAN)

On the RAN VM (Ubuntu 24.04):

1. **Install build dependencies (and UHD for SDR)**

   ```bash
   sudo apt update
   sudo apt install -y autoconf automake build-essential cmake ccache cpufrequtils \
      git libboost-all-dev libncurses-dev libusb-1.0-0-dev python3-dev python3-pip \
      python3-mako libforms-dev libforms-bin wireshark
   # For UHD:
   sudo apt install -y uhd-host && sudo uhd_images_downloader
   ```

2. **Clone OAI 5G RAN code**

   ```bash
   git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git ~/openairinterface5g
   cd ~/openairinterface5g
   git checkout develop
   ```

3. **Install OAI dependencies**

   ```bash
   cd ~/openairinterface5g/cmake_targets
   ./build_oai -I
   ```

4. **Build gNB & nrUE**

   ```bash
   ./build_oai -w USRP --ninja --gNB --nrUE --build-lib "nrscope" -C
   ```

5. **Resource tuning (optional)**

   - CPU performance mode, disable hyper-threading, increase network buffers, CPU pinning.

## Configuration

### 1. OAI 5G Core Configuration

- **PLMN & Tracking Area**: Match `mcc`/`mnc` in AMF config with gNB.
- **AMF Addressing**: Set gNB’s `ngap_ip_address` to Core VM IP.
- **Subscribers**: Verify default IMSI (001010000000001) and keys in `amf_config.yaml` or subscriber DB.
- **Data Network (DN)**: Confirm IP pool (e.g., 192.168.70.0/24) and routing to application servers.

### 2. OAI gNB Configuration

- Use `gnb.sa.band78.fr1.106PRB.usrpb210.conf` (band 78, 106 PRB).
- Ensure PLMN, SST (slice) match core.
- Update AMF IP in config if needed.

### 3. OAI nrUE Configuration

- **SIM Credentials**:
  ```ini
  uicc0 = {
    imsi = "001010000000001";
    key  = "fec86ba6eb707ed08905757b1bb44b8f";
    opc  = "C42449363BBAD02B66D16BC975D77CC1";
    dnn  = "oai";
    nssai_sst = 1;
  };
  ```
- **Radio Settings**: Match band/numerology/PRB with gNB via CLI flags.
- **UE Networking**: UE creates `oaitun_ue1` interface automatically.

## Deployment

1. **Launch 5G Core**

   ```bash
   cd ~/oai-cn5g
   docker compose up -d
   ```

2. **Start gNB** (RF simulator)

   ```bash
   cd ~/openairinterface5g/cmake_targets/ran_build/build
   sudo ./nr-softmodem \
     -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf \
     --gNBs.[0].min_rxtxtime 6 --rfsim
   ```

3. **Start nrUE** (RF simulator)

   ```bash
   cd ~/openairinterface5g/cmake_targets/ran_build/build
   sudo ./nr-uesoftmodem \
     -r 106 --numerology 1 --band 78 -C 3619200000 \
     --ue-fo-compensation --rfsim \
     --uicc0.imsi 001010000000001
   ```

4. **Verify Connectivity**

   ```bash
   ping 192.168.70.135 -I oaitun_ue1
   ```

## Healthcare Data Flow Simulation

1. **Wearable script** on UE VM sends UDP data:
   ```bash
   while true; do echo "HeartRate:$((60 + RANDOM % 40))" | nc -u 192.168.70.135 5000; sleep 5; done
   ```
2. **Server** on Core VM listens:
   ```bash
   nc -klu 5000
   ```
3. **Observe** incoming metrics and verify end-to-end delivery.
4. **Scale** by launching multiple UE instances with distinct IMSIs.
5. **QoS & Slicing**: Experiment with SST/QFI parameters for critical data flows.

## Network Testing & Monitoring

- **Latency & Packet Capture**: Use `wireshark` on `oaitun_ue1` or SCTP port 38412 to measure RTT and inspect signaling.
- **Ping & Throughput**: `ping`, `iperf3` between UE and server.
- **Logs**: Monitor Core containers (`docker logs -f`) and gNB/UE consoles.
- **Troubleshoot**: Validate IMSI/key, SCTP connectivity, firewall rules, use `tcpdump` as needed.

## Conclusion

You now have a virtual 5G SA network simulating a healthcare IoT scenario. This testbed can be extended to real healthcare applications, multi-cell handovers, advanced QoS, and network slicing experiments. For more details, see the official OAI documentation and recent research on 5G in healthcare.
