# LTE Network with Minimal 4G Core using srsRAN and LimeSDR Mini

## 🧩 Abstract

This repository provides a reproducible setup for deploying a basic LTE network using the srsRAN suite (formerly srsLTE), without relying on a full 4G core like Open5GS. It creates a private mobile network composed of an eNodeB, a simplified EPC, and optionally a simulated UE. This setup is ideal for testing connectivity, understanding LTE architecture, and experimenting with SIM-based applications in a lightweight, standalone environment.

The objective is to offer a minimal yet functional LTE stack that can run on limited hardware, focusing on ease of deployment and educational use cases.

## 📂 Structure

```plaintext
/
├── README.md               ← This documentation
└── submodules/             ← Forked Git submodules for reproducibility
    ├── srsRAN_4G/          ← Forked srsRAN_4G (specific commit)
    ├── srsGUI/             ← Forked srsGUI (specific commit)
    ├── SoapySDR/           ← Forked SoapySDR (specific commit)
    └── LimeSuite/          ← Forked LimeSuite (specific commit)
```

## 🧩 Software and Versions Used

The following tools, libraries, and APIs are required or recommended for this LTE setup using srsRAN. All components are either included as Git submodules or documented for reproducibility. Versions have been validated on Ubuntu 20.04.

| Software / Library           | Version Used / Recommended         | Purpose                                                            | Source / Notes                                                         |
|-----------------------------|------------------------------------|--------------------------------------------------------------------|------------------------------------------------------------------------|
| **srsRAN_4G**               | `release_23_04`                    | LTE stack: eNodeB, EPC (MME, HSS, SGW, PGW), network tools         | [GitHub](https://github.com/srsran/srsRAN_4G)                          |
| **LimeSuite**               | `v22.09.1`                         | SDR driver and utilities for LimeSDR hardware                      | [GitHub](https://github.com/myriadrf/LimeSuite)                        |
| **SoapySDR**                | `soapy-sdr-0.8.1`                  | SDR abstraction layer, used by srsRAN to interface with LimeSDR    | [GitHub](https://github.com/pothosware/SoapySDR)                       |
| **srsGUI** (optional)       | `release_2_0_qt5`                  | Visualization of SDR signals (spectrum, constellations, etc.)     | [GitHub](https://github.com/srsran/srsGUI)                             |
| **Python**                  | `3.9+`                             | Required for sim-tools and utility scripts                         | Bundled in Ubuntu 20.04+                                               |
| **CMake**                   | `3.16+`                            | Build system for all C/C++ components                              | `sudo apt install cmake`                                               |
| **GCC / G++**               | `9.x` or `10.x`                    | C/C++ compiler for all native components                           | Default in Ubuntu 20.04 / Debian 11                                    |
| **libboost-dev**            | `>=1.71`                           | Common dependency for srsRAN and srsGUI                            | `sudo apt install libboost-all-dev`                                    |
| **libconfig++-dev**         | `1.5.x` or compatible              | Configuration parser used by srsRAN EPC tools                      | `sudo apt install libconfig++-dev`                                     |

## 💻 Minimal Hardware Requirements

This setup is intended to run on modest hardware with the following recommended minimum:

- **CPU**: Quad-core Intel (e.g., Raspberry Pi 4 / Intel NUC)
- **RAM**: 4GB minimum
- **Disk**: 16GB+ SSD recommended
- **SDR**: LimeSDR Mini (tested)
- **OS**: Ubuntu 20.04 (other versions may require adaptation)

## 🚀 Deployment

### 1. Clone with submodules

```bash
  git clone --recurse-submodules https://github.com/tuusuario/lte-minimal-srsran.git
cd lte-minimal-srsran
```

> 📦 All required components (SoapySDR, LimeSuite, srsRAN, etc.) are already included as Git submodules under the `submodules/` directory. No need to clone them manually.


### 2. Set the installation directory

```bash
export SRSRAN_INSTALL=$HOME/ran-lte
mkdir -p "$SRSRAN_INSTALL"
export LD_LIBRARY_PATH=${SRSRAN_INSTALL}/lib
export PATH=${SRSRAN_INSTALL}/bin:$PATH```

---

### 3. Build SoapySDR (from submodule)

```bash
sudo apt-get install cmake g++ libpython3-dev python3-numpy swig
cd submodules/SoapySDR
mkdir build && cd build
cmake -DCMAKE_PREFIX_PATH=${SRSRAN_INSTALL} -DCMAKE_INSTALL_PREFIX=${SRSRAN_INSTALL} ..
make -j$(nproc)
make install
```

**Test SoapySDR:**

```bash
export LD_LIBRARY_PATH=${SRSRAN_INSTALL}/lib
export PATH=${SRSRAN_INSTALL}/bin:$PATH
SoapySDRUtil --info
```

---

### 4. Build LimeSuite (from submodule)

```bash
sudo apt-get install git g++ cmake libsqlite3-dev
sudo apt-get install libi2c-dev libusb-1.0-0-dev
sudo apt-get install libwxgtk3.0-gtk3-dev freeglut3-dev
cd submodules/LimeSuite
mkdir builddir && cd builddir
cmake -DCMAKE_PREFIX_PATH=${SRSRAN_INSTALL} -DCMAKE_INSTALL_PREFIX=${SRSRAN_INSTALL} ..
make -j$(nproc)
make install
cd ../udev-rules
sudo bash install.sh
```

**Optional: launch LimeSuiteGUI**

```bash
export LD_LIBRARY_PATH=${SRSRAN_INSTALL}/lib
export PATH=${SRSRAN_INSTALL}/bin:$PATH
LimeSuiteGUI
```

**Update firmware:**

```bash
LimeUtil --update
```

## ⚙️ Configuration Overview

After building and installing srsRAN, the configuration files are placed in `~/.config/srsran/` by running the script `srsran_install_configs.sh`.

Example structure:

```bash
$ ls -l $HOME/.config/srsran/
enb.conf
epc.conf
mbms.conf
rb.conf
rr.conf
sib.conf
ue.conf
user_db.csv
```

---

### 📡 `enb.conf` — eNodeB Configuration

This file defines the base station behavior and radio hardware.

In the `[enb]` section, configure the **MCC** and **MNC** to match your SIM card:

```ini
[enb]
enb_id = 0x19B
mcc = 999
mnc = 70
```

In the `[rf]` section, specify the SDR device and its parameters:

```ini
[rf]
tx_gain = 66
rx_gain = 47
device_name = soapy
device_args = driver=lime,rxant=LNAH,txant=BAND2
```

---

### 📶 `rr.conf` — Radio Parameters

Defines cell-specific parameters including frequency.

Set the `dl_earfcn` for your chosen LTE band (e.g., Band 7, 20MHz BW = 2850):

```ini
cell_list =
(
  {
    cell_id = 0x01;
    tac = 0x0007;
    pci = 1;
    dl_earfcn = 2850;
    ho_active = false;
  }
)
```

---

### 🧠 `epc.conf` — Core Network Settings

This file combines the MME, HSS, SGW and PGW roles.

In the `[mme]` section, use the same MCC, MNC and TAC:

```ini
[mme]
mme_code = 0x1a
mme_group = 0x0001
tac = 0x0007
mcc = 999
mnc = 70
```

---

### 👤 `user_db.csv` — Subscriber Provisioning

This file defines the UEs (SIM cards) recognized by the HSS. You'll need to include your SIM's IMSI, key, and OP/OPc values:

```csv
# Name,Auth,IMSI,Key,OP_Type,OP/OPc,AMF,SQN,QCI,IP_alloc
ue1,mil,999700000131761,D387462154B78C44083E6CFC9110B081,opc,F1A54F0F0B2D5E2CA966D4EBA65250B0,8000,000000000000,7,dynamic
```

- `Auth`: Use `mil` for MILENAGE algorithm
- `AMF`: Set to `8000` for LTE
- `SQN`: Leave as zero; it will auto-increment
- `QCI`: Leave as `7` for default bearer
- `IP_alloc`: Use `dynamic` or a static IPv4 address

---

## 🚀 Starting the Network

In two separate terminal windows:

### 1. Set up the environment variable

```bash
export SRSRAN_INSTALL=${HOME}/lte-simple
```

---

### 2. Start the EPC (Core Network)

```bash
sudo LD_LIBRARY_PATH=${SRSRAN_INSTALL}/lib \
sh -c "cd ~/.config/srsran && ${SRSRAN_INSTALL}/bin/srsepc epc.conf"
```

You should see logs confirming the initialization of HSS, MME, SPGW, and the configured MCC/MNC.

---

### 3. Start the eNodeB

```bash
echo "performance" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

sudo LD_LIBRARY_PATH=${SRSRAN_INSTALL}/lib \
sh -c "cd ~/.config/srsran && ${SRSRAN_INSTALL}/bin/srsenb enb.conf"
```

Watch for messages indicating the SDR has been correctly detected and calibrated, and that the eNodeB has started.

---

## 🌐 Enable Internet Access for UEs (NAT)

If you want connected devices to reach the internet:

1. Enable IP forwarding and NAT from your host to the UE subnet.
2. Use the provided helper script, replacing `enp3s0f2` with your actual internet-connected interface:

```bash
sudo ${SRSRAN_INSTALL}/bin/srsepc_if_masq.sh enp3s0f2
```

You should see:

```bash
  Masquerading Interface enp3s0f2
```

Your LTE network is now ready for client devices to attach and access the internet.


## 🛠️ Troubleshooting

### 1. Components fail to start

Make sure you're exporting `SRSRAN_INSTALL` and running all binaries from the configured path.

### 2. SDR not detected

Check USB connection and run:

```bash
LimeUtil --find
```

Ensure `udev` rules are applied.

### 3. Permissions

```bash
sudo usermod -aG dialout $USER
sudo usermod -aG plugdev $USER
```

Reboot if needed.

### 4. Build errors

Clean build folder and try again:

```bash
rm -rf build && mkdir build && cd build
cmake ...
```

## 📱 Tested User Equipment (UE)

The following commercial devices have been tested with this LTE setup and were able to successfully attach to the network:

1. **Iphone 13** – Network not visible
2. **Sony XperiA N5** – Registered and showed LTE signal, SMS tested with applet.

---

## 🧪 Tested Configurations (Example Scenarios)

These setups have been validated and can serve as reference configurations:

### 1. iPhone 13 does not detect LTE network

iOS devices (such as iPhone 13) may not display private LTE networks, even when all RF and core components are functioning correctly. Apple enforces strict rules about which networks are visible in the manual network selection menu. These rules go beyond technical compliance and include alignment with commercial network expectations.

In our setup:
- MCC and MNC values were changed to `214/01` (Spain, Movistar) to match a real PLMN.
- The same values were correctly configured in the SIM card, `enb.conf`, and `epc.conf`.

Despite this, the iPhone 13 still does **not show the LTE network** under available networks.

This suggests that iOS performs **additional checks**, potentially validating the PLMN against its internal list of authorized operators or requiring that the network be part of a recognized carrier bundle. Since this setup is a **non-commercial private LTE network**, it does not fully align with Apple’s expected criteria.

Apple’s own documentation states:

> *"iOS devices will only display and allow manual selection of networks with valid PLMN IDs. For private LTE, this means you must use a properly allocated and configured MCC/MNC."*

However, even when following these recommendations, visibility is not guaranteed on iOS devices.

See Apple's official guidance: [Apple Platform Deployment – Cellular](https://support.apple.com/es-es/guide/deployment/depac6747317/web)

---

## 📚 References and Credits

The design and deployment of this LTE setup are based on the following sources and tools:

- **Phil Greenland – Quantulum**:
    - [https://www.quantulum.co.uk/](https://www.quantulum.co.uk/)  
      Excellent practical documentation and examples on setting up srsRAN with LimeSDR hardware. His work greatly influenced the structure of this guide.

- **srsRAN 4G Project**:
    - [https://github.com/srsran/srsRAN_4G](https://github.com/srsran/srsRAN_4G)  
      Open-source LTE stack used for eNodeB and EPC components in this setup.

- **SoapySDR Project**:
    - [https://github.com/pothosware/SoapySDR](https://github.com/pothosware/SoapySDR)  
      SDR abstraction layer used by srsRAN to interface with hardware such as LimeSDR.

- **LimeSuite Project by MyriadRF**:
    - [https://github.com/myriadrf/LimeSuite](https://github.com/myriadrf/LimeSuite)  
      Drivers, utilities, and GUI for managing LimeSDR devices.

- **GlobalPlatformPro** by Martin Paljak:
    - [https://github.com/martinpaljak/GlobalPlatformPro](https://github.com/martinpaljak/GlobalPlatformPro)  
      Used to load and manage JavaCard applets in programmable SIM cards.

- **sim-tools (forked and modified)**:
    - [https://github.com/tuusuario/sim-tools](https://github.com/tuusuario/sim-tools) *(private fork)*  
      A set of Python-based utilities to install and interact with JavaCard applets on programmable SIMs.

- **Open5GS Project** (referenced but not used in this repo):
    - [https://github.com/open5gs/open5gs](https://github.com/open5gs/open5gs)  
      An open-source EPC and 5G core network implementation, used in other parts of the TFG but not in this minimal setup.
- **Apple Platform Deployment – Cellular Network Requirements**:
    - [https://support.apple.com/es-es/guide/deployment/depac6747317/web](https://support.apple.com/es-es/guide/deployment/depac6747317/web)  
      Official documentation outlining requirements for iOS devices to detect and connect to private LTE networks.

---
## 👨‍💻 Author

- **Name**: Rafael Moreno
- **Email**: rmoreno.morcam@gmail.com
- **Project**: Final Degree Project (TFG) - Computer Engineering

---
