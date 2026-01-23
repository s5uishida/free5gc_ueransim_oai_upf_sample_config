# free5GC 5GC & UERANSIM UE / RAN Sample Configuration - OAI-CN5G-UPF(eBPF/XDP UPF)
This describes a simple configuration for working free5GC and OAI-CN5G-UPF(eBPF/XDP UPF).
In particular, see [here](https://github.com/s5uishida/install_oai_upf) for OAI-CN5G-UPF.

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Overview of free5GC 5GC Simulation Mobile Network](#overview)
- [Changes in configuration files of free5GC 5GC, OAI-CN5G-UPF and UERANSIM UE / RAN](#changes)
  - [Changes in configuration files of free5GC 5GC C-Plane](#changes_cp)
  - [Changes in configuration files of OAI-CN5G-UPF](#changes_up)
  - [Changes in configuration files of UERANSIM UE / RAN](#changes_ueransim)
    - [Changes in configuration files of RAN](#changes_ran)
    - [Changes in configuration files of UE (IMSI-001010000000000)](#changes_ue)
- [Network settings of free5GC 5GC, OAI-CN5G-UPF and UERANSIM UE / RAN](#network_settings)
  - [Network settings of Data Network Gateway](#network_settings_up)
- [Build free5GC, OAI-CN5G-UPF and UERANSIM](#build)
- [Run free5GC 5GC, OAI-CN5G-UPF and UERANSIM UE / RAN](#run)
  - [Run OAI-CN5G-UPF](#run_up)
  - [Run free5GC 5GC C-Plane](#run_cp)
  - [Run UERANSIM](#run_ueran)
    - [Start gNB](#start_gnb)
    - [Start UE](#start_ue)
- [Ping google.com](#ping)
  - [Case for going through DN 10.60.0.0/16](#ping_1)
- [Changelog (summary)](#changelog)

---

<a id="overview"></a>

## Overview of free5GC 5GC Simulation Mobile Network

This describes a simple configuration of C-Plane, eBPF/XDP UPF and Data Network Gateway for free5GC.
**Note that this configuration is implemented with Proxmox VE VMs.**

The following minimum configuration was set as a condition.
- One UPF and Data Network Gateway
- One UE and one DNN

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The 5GC / eBPF/XDP UPF / UE / RAN used are as follows.
- 5GC - free5GC v4.2.0 (2026.01.14) - https://github.com/free5gc/free5gc
- eBPF/XDP UPF - OAI-CN5G-UPF v2.2.0 (2025.12.13) - https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf
- UE / RAN - UERANSIM v3.2.7 (2025.10.25) - https://github.com/aligungr/UERANSIM

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Mem<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | free5GC 5GC C-Plane | 192.168.0.141/24 | Ubuntu 24.04 | 1 | 2GB | 20GB |
| VM-UP | OAI-CN5G-UPF U-Plane | 192.168.0.151/24 | Ubuntu 24.04 | 1 | 6GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |
| VM2 | UERANSIM RAN (gNodeB) | 192.168.0.131/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |
| VM3 | UERANSIM UE | 192.168.0.132/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |

The network interfaces of each VM are as follows.
| VM | Device | Model | Linux Bridge | IP address | Interface | XDP |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | ens18 | VirtIO | vmbr1 | 10.0.0.141/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.141/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr4 | 192.168.14.141/24 | N4 | -- |
| VM-UP | ~~ens18~~ | ~~VirtIO~~ | ~~vmbr1~~ | ~~10.0.0.151/24~~ | ~~(NAPT NW)~~ ***down*** | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.151/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.151/24 | N3 | x |
| | ens21 | VirtIO | vmbr4 | 192.168.14.151/24 | N4 | -- |
| | ens22 | VirtIO | vmbr6 | 192.168.16.151/24 | N6 | x |
| VM-DN | ens18 | VirtIO | vmbr1 | 10.0.0.152/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.152/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr6 | 192.168.16.152/24 | N6 | -- |
| VM2 | ens18 | VirtIO | vmbr1 | 10.0.0.131/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.131/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.131/24 | N3 | -- |
| VM3 | ens18 | VirtIO | vmbr1 | 10.0.0.132/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.132/24 | (Mgmt NW) | -- |

Linux Bridges of Proxmox VE are as follows.
| Linux Bridge | Network CIDR | Interface |
| --- | --- | --- |
| vmbr1 | 10.0.0.0/24 | NAPT NW |
| mgbr0 | 192.168.0.0/24 | Mgmt NW |
| vmbr3 | 192.168.13.0/24 | N3 |
| vmbr4 | 192.168.14.0/24 | N4 |
| vmbr6 | 192.168.16.0/24 | N6 |

Subscriber Information (other information is the same) is as follows.  
**Note. Please select OP or OPc according to the setting of UERANSIM UE configuration file.**
| UE | IMSI | DNN | OP/OPc |
| --- | --- | --- | --- |
| UE | 001010000000000 | internet | OPc |

I registered these information with the free5GC WebUI.
In addition, [3GPP TS 35.208](https://www.3gpp.org/DynaReport/35208.htm) "4.3 Test Sets" is published by 3GPP as test data for the 3GPP authentication and key generation functions (MILENAGE).

The DN is as follows.
| DN | DNN | TUNnel interface of UE |
| --- | --- | --- |
| 10.60.0.0/16 | internet | uesimtun0 |

<a id="changes"></a>

## Changes in configuration files of free5GC 5GC, OAI-CN5G-UPF and UERANSIM UE / RAN

Please refer to the following for building free5GC, OAI-CN5G-UPF and UERANSIM respectively.
- free5GC v4.2.0 (2026.01.14) - https://free5gc.org/guide/
- OAI-CN5G-UPF v2.2.0 (2025.12.13) - https://github.com/s5uishida/install_oai_upf
- UERANSIM v3.2.7 (2025.10.25) - https://github.com/aligungr/UERANSIM/wiki/Installation

<a id="changes_cp"></a>

### Changes in configuration files of free5GC 5GC C-Plane

The combination of DNN and S-NSSAI parameters can be used in the logic that selects UPF as the connection destination by PFCP.

- DNN
- S-NSSAI

For the sake of simplicity, This time, only DNN will be changed. S-NSSAI of all UEs is fixed as `SST=1` and `SD=010203`.

- `free5gc/config/amfcfg.yaml`
```diff
--- amfcfg.yaml.orig    2024-10-13 05:09:24.000000000 +0900
+++ amfcfg.yaml 2025-05-04 19:42:17.462006265 +0900
@@ -5,7 +5,7 @@
 configuration:
   amfName: AMF # the name of this AMF
   ngapIpList:  # the IP list of N2 interfaces on this AMF
-    - 127.0.0.18
+    - 192.168.0.141
   ngapPort: 38412 # the SCTP port listened by NGAP
 
   # Service-based Interface (SBI) Configuration
@@ -30,22 +30,22 @@
   servedGuamiList:
     # <GUAMI> = <MCC><MNC><AMF ID>
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       amfId: cafe00 # AMF identifier (3 bytes hex string, range: 000000~FFFFFF)
 
   # the TAI (Tracking Area Identifier) list supported by this AMF
   supportTaiList:
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       tac: 000001 # Tracking Area Code (3 bytes hex string, range: 000000~FFFFFF)
 
   # the PLMNs (Public land mobile network) list supported by this AMF
   plmnSupportList:
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       snssaiList: # the S-NSSAI (Single Network Slice Selection Assistance Information) list supported by this AMF
         - sst: 1 # Slice/Service Type (uinteger, range: 0~255)
           sd: 010203 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
```
- `free5gc/config/ausfcfg.yaml`
```diff
--- ausfcfg.yaml.orig   2024-09-01 09:47:28.000000000 +0900
+++ ausfcfg.yaml        2024-09-01 09:55:54.000000000 +0900
@@ -16,10 +16,8 @@
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
   nrfCertPem: cert/nrf.pem # NRF Certificate
   plmnSupportList: # the PLMNs (Public Land Mobile Network) list supported by this AUSF
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
-    - mcc: 123 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 45  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01  # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   groupId: ausfGroup001 # ID for the group of the AUSF
   eapAkaSupiImsiPrefix: false # including "imsi-" prefix or not when using the SUPI to do EAP-AKA' authentication

```
- `free5gc/config/nrfcfg.yaml`
```diff
--- nrfcfg.yaml.orig    2024-09-01 09:47:28.000000000 +0900
+++ nrfcfg.yaml 2024-09-01 09:56:10.000000000 +0900
@@ -18,8 +18,8 @@
       key: cert/root.key
     oauth: true
   DefaultPlmnId:
-    mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-    mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+    mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   serviceNameList: # the SBI services provided by this NRF, refer to TS 29.510
     - nnrf-nfm # Nnrf_NFManagement service
     - nnrf-disc # Nnrf_NFDiscovery service
```
- `free5gc/config/nssfcfg.yaml`
```diff
--- nssfcfg.yaml.orig   2024-09-01 09:47:28.000000000 +0900
+++ nssfcfg.yaml        2024-09-01 09:56:46.000000000 +0900
@@ -18,12 +18,12 @@
   nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
   nrfCertPem: cert/nrf.pem # NRF Certificate
   supportedPlmnList: # the PLMNs (Public land mobile network) list supported by this NSSF
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   supportedNssaiInPlmnList: # Supported S-NSSAI List for each PLMN
     - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
-        mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-        mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+        mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+        mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
       supportedSnssaiList: # Supported S-NSSAIs of the PLMN
         - sst: 1 # Slice/Service Type (uinteger, range: 0~255)
           sd: 010203 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
```
- `free5gc/config/smfcfg.yaml`
```diff
--- smfcfg.yaml.orig    2024-10-13 05:09:24.000000000 +0900
+++ smfcfg.yaml 2025-05-04 21:00:46.538990509 +0900
@@ -42,16 +42,16 @@
 
   # Optional: PLMN IDs configuration.
   plmnList:
-    - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
-      mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
+    - mcc: 001 # Mobile Country Code (3 digits string, digit: 0~9)
+      mnc: 01 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
   locality: area1 # Name of the location where a set of AMF, SMF, PCF and UPFs are located
 
   # PFCP (Packet Forwarding Control Protocol) configuration for N4 interface.
   pfcp:
     # addr config is deprecated in smf config v1.0.3, please use the following config
-    nodeID: 127.0.0.1 # the Node ID of this SMF
-    listenAddr: 127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
-    externalAddr: 127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
+    nodeID: 192.168.14.141 # the Node ID of this SMF
+    listenAddr: 192.168.14.141 # the IP/FQDN of N4 interface on this SMF (PFCP)
+    externalAddr: 192.168.14.141 # the IP/FQDN of N4 interface on this SMF (PFCP)
     assocFailAlertInterval: 10s
     assocFailRetryInterval: 30s
     heartbeatInterval: 10s
@@ -63,8 +63,8 @@
         type: AN # the type of the node (AN or UPF)
       UPF: # the name of the node
         type: UPF # the type of the node (AN or UPF)
-        nodeID: 127.0.0.8 # the Node ID of this UPF
-        addr: 127.0.0.8 # the IP/FQDN of N4 interface on this UPF (PFCP)
+        nodeID: 192.168.14.151 # the Node ID of this UPF
+        addr: 192.168.14.151 # the IP/FQDN of N4 interface on this UPF (PFCP)
         sNssaiUpfInfos: # S-NSSAI information list for this UPF
           - sNssai: # S-NSSAI (Single Network Slice Selection Assistance Information)
               sst: 1 # Slice/Service Type (uinteger, range: 0~255)
@@ -91,7 +91,7 @@
         interfaces: # Interface list for this UPF
           - interfaceType: N3 # the type of the interface (N3 or N9)
             endpoints: # the IP address of this N3/N9 interface on this UPF
-              - 127.0.0.8
+              - 192.168.13.151
             networkInstances: # Data Network Name (DNN)
               - internet
 
@@ -99,7 +99,7 @@
     links:
       - A: gNB1
         B: UPF
-
+  ulcl: false
   # retransmission timer for PDU session modification command
   t3591:
     enable: true # true or false
```

<a id="changes_up"></a>

### Changes in configuration files of OAI-CN5G-UPF

See [here](https://github.com/s5uishida/install_oai_upf#conf) for the original file.

- `openair-upf/config.yaml`  
There is no change.

<a id="changes_ueransim"></a>

### Changes in configuration files of UERANSIM UE / RAN

<a id="changes_ran"></a>

#### Changes in configuration files of RAN

- `UERANSIM/config/free5gc-gnb.yaml`
```diff
--- free5gc-gnb.yaml.orig       2024-12-11 20:31:30.000000000 +0900
+++ free5gc-gnb.yaml    2025-05-04 19:47:48.421731012 +0900
@@ -1,17 +1,17 @@
-mcc: '208'          # Mobile Country Code value
-mnc: '93'           # Mobile Network Code value (2 or 3 digits)
+mcc: '001'          # Mobile Country Code value
+mnc: '01'           # Mobile Network Code value (2 or 3 digits)
 
 nci: '0x000000010'  # NR Cell Identity (36-bit)
 idLength: 32        # NR gNB ID length in bits [22...32]
 tac: 1              # Tracking Area Code
 
-linkIp: 127.0.0.1   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
-ngapIp: 127.0.0.1   # gNB's local IP address for N2 Interface (Usually same with local IP)
-gtpIp: 127.0.0.1    # gNB's local IP address for N3 Interface (Usually same with local IP)
+linkIp: 192.168.0.131   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
+ngapIp: 192.168.0.131   # gNB's local IP address for N2 Interface (Usually same with local IP)
+gtpIp: 192.168.13.131    # gNB's local IP address for N3 Interface (Usually same with local IP)
 
 # List of AMF address information
 amfConfigs:
-  - address: 127.0.0.1
+  - address: 192.168.0.141
     port: 38412
 
 # List of supported S-NSSAIs by this gNB
```

<a id="changes_ue"></a>

#### Changes in configuration files of UE (IMSI-001010000000000)

- `UERANSIM/config/free5gc-ue.yaml`
```diff
--- free5gc-ue.yaml.orig        2025-03-16 15:49:12.000000000 +0900
+++ free5gc-ue.yaml     2025-05-04 19:49:27.180000029 +0900
@@ -1,9 +1,9 @@
 # IMSI number of the UE. IMSI = [MCC|MNC|MSISDN] (In total 15 digits)
-supi: 'imsi-208930000000001'
+supi: 'imsi-001010000000000'
 # Mobile Country Code value of HPLMN
-mcc: '208'
+mcc: '001'
 # Mobile Network Code value of HPLMN (2 or 3 digits)
-mnc: '93'
+mnc: '01'
 # SUCI Protection Scheme : 0 for Null-scheme, 1 for Profile A and 2 for Profile B
 protectionScheme: 0
 # Home Network Public Key for protecting with SUCI Profile A
@@ -31,7 +31,7 @@
 
 # List of gNB IP addresses for Radio Link Simulation
 gnbSearchList:
-  - 127.0.0.1
+  - 192.168.0.131
 
 # UAC Access Identities Configuration
 uacAic:
```

<a id="network_settings"></a>

## Network settings of free5GC 5GC, OAI-CN5G-UPF and UERANSIM UE / RAN

<a id="network_settings_up"></a>

### Network settings of Data Network Gateway

See [this](https://github.com/s5uishida/install_oai_upf#setup_dn).

<a id="build"></a>

## Build free5GC, OAI-CN5G-UPF and UERANSIM

Please refer to the following for building free5GC, OAI-CN5G-UPF and UERANSIM respectively.
- free5GC v4.2.0 (2026.01.14) - https://free5gc.org/guide/
- OAI-CN5G-UPF v2.2.0 (2025.12.13) - https://github.com/s5uishida/install_oai_upf
- UERANSIM v3.2.7 (2025.10.25) - https://github.com/aligungr/UERANSIM/wiki/Installation

Install MongoDB on free5GC 5GC C-Plane machine.
[MongoDB Compass](https://www.mongodb.com/products/compass) is a convenient tool to look at the MongoDB database.

**Note. The installation guide also includes instructions on building the latest committed version.**

<a id="run"></a>

## Run free5GC 5GC, OAI-CN5G-UPF and UERANSIM UE / RAN

First run OAI-CN5G-UPF, then the 5GC and UERANSIM (UE & RAN implementation).

<a id="run_up"></a>

### Run OAI-CN5G-UPF

See [this](https://github.com/s5uishida/install_oai_upf#run).

<a id="run_cp"></a>

### Run free5GC 5GC C-Plane

Next, run free5GC 5GC C-Plane.
Create the following shell script and run it.
```bash
#!/usr/bin/env bash

PID_LIST=()

NF_LIST="nrf amf smf udr pcf udm nssf ausf chf nef bsf"

export GIN_MODE=release

for NF in ${NF_LIST}; do
    ./bin/${NF} &
    PID_LIST+=($!)
    sleep 1
done

function terminate()
{
    sudo kill -SIGTERM ${PID_LIST[${#PID_LIST[@]}-2]} ${PID_LIST[${#PID_LIST[@]}-1]}
    sleep 2
}

trap terminate SIGINT
wait ${PID_LIST}
```
The PFCP association log between OAI-CN5G-UPF and free5GC SMF is as follows.
```
[2026-01-23 22:58:10.708] [upf_n4 ] [info] handle_receive(30 bytes)
[2026-01-23 22:58:10.708] [upf_n4 ] [info] Handle SX ASSOCIATION SETUP REQUEST
[2026-01-23 22:58:10.709] [upf_n4 ] [info] handle_receive(16 bytes)
[2026-01-23 22:58:10.709] [upf_n4 ] [info] Received SX HEARTBEAT REQUEST
```

<a id="run_ueran"></a>

### Run UERANSIM

Here, the case of UE (IMSI-001010000000000) & RAN is described.
First, do an NG Setup between gNodeB and 5GC, then register the UE with 5GC and establish a PDU session.

Please refer to the following for usage of UERANSIM.

https://github.com/aligungr/UERANSIM/wiki/Usage

<a id="start_gnb"></a>

#### Start gNB

Start gNB as follows.
```
# ./nr-gnb -c ../config/free5gc-gnb.yaml
UERANSIM v3.2.7
[2026-01-23 22:58:44.623] [sctp] [info] Trying to establish SCTP connection... (192.168.0.141:38412)
[2026-01-23 22:58:44.627] [sctp] [info] SCTP connection established (192.168.0.141:38412)
[2026-01-23 22:58:44.627] [sctp] [debug] SCTP association setup ascId[9]
[2026-01-23 22:58:44.627] [ngap] [debug] Sending NG Setup Request
[2026-01-23 22:58:44.629] [ngap] [debug] NG Setup Response received
[2026-01-23 22:58:44.629] [ngap] [info] NG Setup procedure is successful
```
The free5GC C-Plane log when executed is as follows.
```
2026-01-23T22:58:44.637877925+09:00 [INFO][AMF][Ngap] [AMF] SCTP Accept from: 192.168.0.131:44021
2026-01-23T22:58:44.638561442+09:00 [INFO][AMF][Ngap] Create a new NG connection for: 192.168.0.131:44021
2026-01-23T22:58:44.639327488+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44021] Handle NGSetupRequest
2026-01-23T22:58:44.639368617+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44021] Send NG-Setup response
```

<a id="start_ue"></a>

#### Start UE

Start UE as follows. This will register the UE with 5GC and establish a PDU session.
```
# ./nr-ue -c ../config/free5gc-ue.yaml
UERANSIM v3.2.7
[2026-01-23 22:59:18.921] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2026-01-23 22:59:18.922] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2026-01-23 22:59:18.922] [nas] [info] Selected plmn[001/01]
[2026-01-23 22:59:18.922] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2026-01-23 22:59:18.922] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2026-01-23 22:59:18.922] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2026-01-23 22:59:18.922] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2026-01-23 22:59:18.924] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2026-01-23 22:59:18.924] [nas] [debug] Sending Initial Registration
[2026-01-23 22:59:18.924] [rrc] [debug] Sending RRC Setup Request
[2026-01-23 22:59:18.924] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2026-01-23 22:59:18.925] [rrc] [info] RRC connection established
[2026-01-23 22:59:18.925] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2026-01-23 22:59:18.925] [nas] [info] UE switches to state [CM-CONNECTED]
[2026-01-23 22:59:18.977] [nas] [debug] Authentication Request received
[2026-01-23 22:59:18.977] [nas] [debug] Received SQN [00000000002E]
[2026-01-23 22:59:18.977] [nas] [debug] SQN-MS [000000000000]
[2026-01-23 22:59:19.003] [nas] [debug] Security Mode Command received
[2026-01-23 22:59:19.004] [nas] [debug] Selected integrity[2] ciphering[0]
[2026-01-23 22:59:19.162] [nas] [debug] Registration accept received
[2026-01-23 22:59:19.162] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2026-01-23 22:59:19.162] [nas] [debug] Sending Registration Complete
[2026-01-23 22:59:19.162] [nas] [info] Initial Registration is successful
[2026-01-23 22:59:19.162] [nas] [debug] Sending PDU Session Establishment Request
[2026-01-23 22:59:19.163] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2026-01-23 22:59:19.368] [nas] [debug] Configuration Update Command received
[2026-01-23 22:59:19.529] [nas] [debug] PDU Session Establishment Accept received
[2026-01-23 22:59:19.529] [nas] [info] PDU Session establishment is successful PSI[1]
[2026-01-23 22:59:19.547] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
The free5GC C-Plane log when executed is as follows.
```
2026-01-23T22:59:18.906339767+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44021] Handle InitialUEMessage
2026-01-23T22:59:18.906393344+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44021] New RanUe [RanUeNgapID:1][AmfUeNgapID:1]
2026-01-23T22:59:18.906427713+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44021] 5GSMobileIdentity ["SUCI":"suci-0-001-01-0000-0-0-0000000000", err: <nil>]
2026-01-23T22:59:18.906475211+09:00 [INFO][AMF][CTX] New AmfUe [supi:][guti:00101cafe0000000001]
2026-01-23T22:59:18.906495618+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Deregistered] to [Deregistered]
2026-01-23T22:59:18.906501376+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Registration Request
2026-01-23T22:59:18.906507004+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] RegistrationType: Initial Registration
2026-01-23T22:59:18.906513203+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] MobileIdentity5GS: SUCI[suci-0-001-01-0000-0-0-0000000000]
2026-01-23T22:59:18.906523239+09:00 [INFO][AMF][Gmm] Handle event[Start Authentication], transition from [Deregistered] to [Authentication]
2026-01-23T22:59:18.906529130+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Authentication procedure
2026-01-23T22:59:18.907606810+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:18.909513818+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:18.912580410+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:18.914981418+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:18.916384469+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=AUSF |  |
2026-01-23T22:59:18.917386832+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:18.918684044+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:18.921317935+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:18.923123755+09:00 [INFO][AUSF][UeAuth] HandleUeAuthPostRequest
2026-01-23T22:59:18.923237124+09:00 [INFO][AUSF][UeAuth] Serving network authorized
2026-01-23T22:59:18.924070870+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:18.924938431+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:18.927087836+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:18.927901031+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:18.928947235+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AUSF&service-names=nudm-ueau&target-nf-type=UDM |  |
2026-01-23T22:59:18.929922129+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:18.930834800+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:18.933597173+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:18.935570183+09:00 [INFO][UDM][UEAU] Handle GenerateAuthDataRequest
2026-01-23T22:59:18.938570784+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:18.940277040+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:18.942839089+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:18.943208355+09:00 [INFO][UDM][Suci] scheme 0
2026-01-23T22:59:18.943340929+09:00 [INFO][UDM][Suci] SUPI type is IMSI
2026-01-23T22:59:18.943701437+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:18.944790024+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:18.946752601+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:18.947531987+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:18.948375580+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=UDM&target-nf-type=UDR |  |
2026-01-23T22:59:18.951453774+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |  |
2026-01-23T22:59:18.952007465+09:00 [INFO][UDM][Proc] ModifyAuthenticationSubscriptionRequest:  [{replace /sequenceNumber  { 00000000002f map[] 0 }}]
2026-01-23T22:59:18.954153280+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PATCH   | /nudr-dr/v2/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |  |
2026-01-23T22:59:18.954546047+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | POST    | /nudm-ueau/v1/suci-0-001-01-0000-0-0-0000000000/security-information/generate-auth-data |  |
2026-01-23T22:59:18.955059085+09:00 [INFO][AUSF][UeAuth] Add SuciSupiPair (suci-0-001-01-0000-0-0-0000000000, imsi-001010000000000) to map.
2026-01-23T22:59:18.955263601+09:00 [INFO][AUSF][UeAuth] Use 5G AKA auth method
2026-01-23T22:59:18.955376902+09:00 [INFO][AUSF][5gAka] XresStar = 3864303861623462393535316634323061663162333434656230633664343134
2026-01-23T22:59:18.955681214+09:00 [INFO][AUSF][GIN] | 201 |       127.0.0.1 | POST    | /nausf-auth/v1/ue-authentications |  |
2026-01-23T22:59:18.956430352+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Send Authentication Request
2026-01-23T22:59:18.956574338+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44021] Send Downlink Nas Transport
2026-01-23T22:59:18.956813782+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Start T3560 timer
2026-01-23T22:59:18.958034700+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44021] Handle UplinkNASTransport
2026-01-23T22:59:18.958177146+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44021] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2026-01-23T22:59:18.958284514+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Authentication] to [Authentication]
2026-01-23T22:59:18.958370188+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Authentication Response
2026-01-23T22:59:18.958418653+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Stop T3560 timer
2026-01-23T22:59:18.959282702+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:18.960625898+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:18.963814195+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:18.967617796+09:00 [INFO][AUSF][5gAka] Auth5gAkaComfirmRequest
2026-01-23T22:59:18.967828116+09:00 [INFO][AUSF][5gAka] res*: 3864303861623462393535316634323061663162333434656230633664343134
Xres*: 3864303861623462393535316634323061663162333434656230633664343134
2026-01-23T22:59:18.968184946+09:00 [INFO][AUSF][5gAka] 5G AKA confirmation succeeded
2026-01-23T22:59:18.969308280+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:18.970237484+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:18.973134581+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:18.974722324+09:00 [INFO][UDM][UEAU] Handle ConfirmAuthDataRequest
2026-01-23T22:59:18.975623433+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:18.976762443+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:18.979487991+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:18.981986457+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v2/subscription-data/imsi-001010000000000/authentication-data/authentication-status |  |
2026-01-23T22:59:18.982321619+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-ueau/v1/imsi-001010000000000/auth-events |  |
2026-01-23T22:59:18.982751021+09:00 [INFO][AUSF][GIN] | 200 |       127.0.0.1 | PUT     | /nausf-auth/v1/ue-authentications/suci-0-001-01-0000-0-0-0000000000/5g-aka-confirmation |  |
2026-01-23T22:59:18.983295877+09:00 [INFO][AMF][Gmm] Handle event[Authentication Success], transition from [Authentication] to [SecurityMode]
2026-01-23T22:59:18.983477867+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Security Mode Command
2026-01-23T22:59:18.983522084+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44021] Send Downlink Nas Transport
2026-01-23T22:59:18.983596898+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3560 timer
2026-01-23T22:59:18.986461325+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44021] Handle UplinkNASTransport
2026-01-23T22:59:18.986577723+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44021] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2026-01-23T22:59:18.986669907+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [SecurityMode] to [SecurityMode]
2026-01-23T22:59:18.986805188+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Security Mode Complete
2026-01-23T22:59:18.986865186+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3560 timer
2026-01-23T22:59:18.986955324+09:00 [INFO][AMF][Gmm] Handle event[SecurityMode Success], transition from [SecurityMode] to [ContextSetup]
2026-01-23T22:59:18.987026847+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle InitialRegistration
2026-01-23T22:59:18.988167172+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:18.989664690+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:18.991743952+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:18.992516792+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:18.993645564+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |  |
2026-01-23T22:59:18.994378758+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:18.995881357+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:18.998637034+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:18.999808356+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 nssai
2026-01-23T22:59:18.999840097+09:00 [INFO][UDM][SDM] Handle GetNssai
2026-01-23T22:59:19.002505087+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.003855146+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.006562791+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.008392030+09:00 [INFO][UDR][DataRepo] QueryAmDataProcedure: ueId: imsi-001010000000000, servingPlmnId: 00101
2026-01-23T22:59:19.009045037+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features= |  |
2026-01-23T22:59:19.010233569+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/nssai?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |  |
2026-01-23T22:59:19.010887857+09:00 [INFO][AMF][Gmm] RequestedNssai: &{Iei:47 Len:5 Buffer:[4 1 1 2 3]}
2026-01-23T22:59:19.011055094+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] RequestedNssai - ServingSnssai: &{Sst:1 Sd:010203}, HomeSnssai: <nil>
2026-01-23T22:59:19.012000333+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.013406094+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.015550732+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.016342086+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:19.017963309+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |  |
2026-01-23T22:59:19.020994016+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.022143314+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.024990120+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.026908568+09:00 [INFO][UDM][UECM] Handle RegistrationAmf3gppAccess
2026-01-23T22:59:19.027056078+09:00 [INFO][UDM][UECM] UEID: imsi-001010000000000
2026-01-23T22:59:19.027839069+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.029061892+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.031694915+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.034164850+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/amf-3gpp-access |  |
2026-01-23T22:59:19.034527182+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | PUT     | /nudm-uecm/v1/imsi-001010000000000/registrations/amf-3gpp-access |  |
2026-01-23T22:59:19.035666763+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.037208736+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.040179286+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.041493001+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 am-data
2026-01-23T22:59:19.041724224+09:00 [INFO][UDM][SDM] Handle GetAmData
2026-01-23T22:59:19.042551412+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.043641523+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.046247457+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.047303297+09:00 [INFO][UDR][DataRepo] QueryAmDataProcedure: ueId: imsi-001010000000000, servingPlmnId: 00101
2026-01-23T22:59:19.048029743+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features= |  |
2026-01-23T22:59:19.048432202+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/am-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |  |
2026-01-23T22:59:19.050352261+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.051552322+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.054401032+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.055734809+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 smf-select-data
2026-01-23T22:59:19.055845306+09:00 [INFO][UDM][SDM] Handle GetSmfSelectData
2026-01-23T22:59:19.056843776+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.058045233+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.060545623+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.062004734+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/smf-selection-subscription-data?supported-features= |  |
2026-01-23T22:59:19.062617144+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/smf-select-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |  |
2026-01-23T22:59:19.063879116+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.066168733+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.069930966+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.072830544+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 ue-context-in-smf-data
2026-01-23T22:59:19.072931578+09:00 [INFO][UDM][SDM] Handle GetUeContextInSmfData
2026-01-23T22:59:19.073578446+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.074789938+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.077303174+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.079150329+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/smf-registrations?supported-features= |  |
2026-01-23T22:59:19.079619554+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/ue-context-in-smf-data |  |
2026-01-23T22:59:19.080497397+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.081995723+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.085016380+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.087049092+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 sdm-subscriptions
2026-01-23T22:59:19.088158252+09:00 [INFO][UDM][SDM] Handle Subscribe
2026-01-23T22:59:19.088964289+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.090254155+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.092885495+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.096058826+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |  |
2026-01-23T22:59:19.096474412+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v2/imsi-001010000000000/sdm-subscriptions |  |
2026-01-23T22:59:19.097453289+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.099068512+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.101194485+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.101963606+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:19.103191840+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=PCF |  |
2026-01-23T22:59:19.103823232+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.104949259+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.107977157+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.110387700+09:00 [INFO][PCF][AmPol] Handle AM Policy Create Request
2026-01-23T22:59:19.111299475+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.112431759+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.114493702+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.115314338+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:19.116191269+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=UDR |  |
2026-01-23T22:59:19.117188395+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.118624769+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.121352531+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.123116144+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/policy-data/ues/imsi-001010000000000/am-data |  |
2026-01-23T22:59:19.124175722+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.125653863+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.127793756+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.128602227+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:19.130069701+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?guami=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22amfId%22%3A%22cafe00%22%7D&requester-nf-type=PCF&target-nf-type=AMF |  |
2026-01-23T22:59:19.130732988+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.132312556+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.137874181+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.140076622+09:00 [INFO][AMF][Comm] Handle AMF Status Change Subscribe Request
2026-01-23T22:59:19.140219831+09:00 [INFO][AMF][Comm] new AMF Status Subscription[1]
2026-01-23T22:59:19.140280104+09:00 [INFO][AMF][GIN] | 201 |       127.0.0.1 | POST    | /namf-comm/v1/subscriptions |  |
2026-01-23T22:59:19.140836447+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-am-policy-control/v1/policies |  |
2026-01-23T22:59:19.141368545+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Registration Accept
2026-01-23T22:59:19.141553615+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44021] Send Initial Context Setup Request
2026-01-23T22:59:19.141867394+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3550 timer
2026-01-23T22:59:19.142478745+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44021] Handle InitialContextSetupResponse
2026-01-23T22:59:19.142604092+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44021] Handle InitialContextSetupResponse (RAN UE NGAP ID: 1)
2026-01-23T22:59:19.348141972+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44021] Handle UplinkNASTransport
2026-01-23T22:59:19.348159516+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44021] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2026-01-23T22:59:19.348198331+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [ContextSetup] to [ContextSetup]
2026-01-23T22:59:19.348204175+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Registration Complete
2026-01-23T22:59:19.348209355+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3550 timer
2026-01-23T22:59:19.348228509+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Configuration Update Command
2026-01-23T22:59:19.348235336+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44021] Send Downlink Nas Transport
2026-01-23T22:59:19.348284949+09:00 [INFO][AMF][Gmm] Handle event[ContextSetup Success], transition from [ContextSetup] to [Registered]
2026-01-23T22:59:19.348480239+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44021] Handle UplinkNASTransport
2026-01-23T22:59:19.348495773+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44021] Handle UplinkNASTransport (RAN UE NGAP ID: 1)
2026-01-23T22:59:19.348533606+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Registered] to [Registered]
2026-01-23T22:59:19.348540338+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle UL NAS Transport
2026-01-23T22:59:19.348549690+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Transport 5GSM Message to SMF
2026-01-23T22:59:19.348559195+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Select SMF [snssai: {Sst:1 Sd:010203}, dnn: internet]
2026-01-23T22:59:19.351977792+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.354027630+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.356160099+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.356853626+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:19.357730621+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=NSSF |  |
2026-01-23T22:59:19.358327161+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.359654265+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.362150827+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.363679540+09:00 [INFO][NSSF][NsSel] Handle NSSelectionGet
2026-01-23T22:59:19.364095233+09:00 [WARN][NSSF][Util] No TA {"plmnId":{"mcc":"001","mnc":"01"},"tac":"000001"} in NSSF configuration
2026-01-23T22:59:19.364378056+09:00 [INFO][NSSF][GIN] | 200 |       127.0.0.1 | GET     | /nnssf-nsselection/v2/network-slice-information?nf-id=23c75b50-5359-4da0-9dde-b318fe205014&nf-type=AMF&slice-info-request-for-pdu-session=%7B%22sNssai%22%3A%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%2C%22roamingIndication%22%3A%22NON_ROAMING%22%7D&tai=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22tac%22%3A%22000001%22%7D |  |
2026-01-23T22:59:19.364897705+09:00 [WARN][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] nsiInformation is still nil, use default NRF[http://127.0.0.10:8000]
2026-01-23T22:59:19.365677043+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.367044147+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.369184816+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.369936148+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:19.371190628+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?dnn=internet&preferred-locality=area1&requester-nf-type=AMF&service-names=nsmf-pdusession&snssais=%5B%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%5D&target-nf-type=SMF&target-plmn-list=%5B%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%5D |  |
2026-01-23T22:59:19.371874924+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.373098158+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.375939368+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.377664897+09:00 [INFO][SMF][PduSess] Receive Create SM Context Request
2026-01-23T22:59:19.379649049+09:00 [INFO][SMF][PduSess] In HandlePDUSessionSMContextCreate
2026-01-23T22:59:19.380865915+09:00 [INFO][SMF][CTX] UrrPeriod: 30s
2026-01-23T22:59:19.381014610+09:00 [INFO][SMF][CTX] UrrThreshold: 500000
2026-01-23T22:59:19.383065485+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.384295633+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.386475920+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.387214156+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:19.388335397+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=UDM |  |
2026-01-23T22:59:19.389112482+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Send NF Discovery Serving UDM Successfully
2026-01-23T22:59:19.389497760+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.390606220+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.393414840+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.394839016+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 sm-data
2026-01-23T22:59:19.394983097+09:00 [INFO][UDM][SDM] Handle GetSmData
2026-01-23T22:59:19.395755183+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.396970378+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.399505905+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.399950966+09:00 [INFO][UDM][SDM] getSmDataProcedure: SUPI[imsi-001010000000000] PLMNID[00101] DNN[internet] SNssai[{"sst":1,"sd":"010203"}]
2026-01-23T22:59:19.401720019+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/sm-data?single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |  |
2026-01-23T22:59:19.402269509+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/sm-data?dnn=internet&plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D&single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |  |
2026-01-23T22:59:19.403207272+09:00 [INFO][SMF][GSM] In HandlePDUSessionEstablishmentRequest
2026-01-23T22:59:19+09:00 [INFO][NAS][Convert] ProtocolOrContainerList:  [0xc000614980 0xc0006149a0]
2026-01-23T22:59:19.403553270+09:00 [INFO][SMF][GSM] Protocol Configuration Options
2026-01-23T22:59:19.403596414+09:00 [INFO][SMF][GSM] &{[0xc000614980 0xc0006149a0]}
2026-01-23T22:59:19.403631755+09:00 [INFO][SMF][GSM] Didn't Implement container type IPAddressAllocationViaNASSignallingUL
2026-01-23T22:59:19.404563773+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.405761430+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.407812852+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.408505070+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:19.409419696+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-instance-id=23c75b50-5359-4da0-9dde-b318fe205014&target-nf-type=AMF |  |
2026-01-23T22:59:19.409806809+09:00 [INFO][SMF][Consumer] SendNFDiscoveryServingAMF ok
2026-01-23T22:59:19.409852784+09:00 [INFO][SMF][CTX] Allocated UE IP address: 10.60.0.1
2026-01-23T22:59:19.409870691+09:00 [INFO][SMF][CTX] Selected UPF: UPF
2026-01-23T22:59:19.409882903+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Allocated PDUAdress[10.60.0.1]
2026-01-23T22:59:19.410194557+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.411275279+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.413455667+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.414143889+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:19.414912199+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=BSF |  |
2026-01-23T22:59:19.415845161+09:00 [INFO][BSF][Proc] Handle GetPCFBindings
[GIN] 2026/01/23 - 22:59:19 | 204 |     750.714Âµs |       127.0.0.1 | GET      "/nbsf-management/v1/pcfBindings?dnn=internet&snssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D&supi=imsi-001010000000000"
2026-01-23T22:59:19.416703780+09:00 [INFO][SMF][Consumer] No PCF binding found in BSF, using NRF discovery for SUPI: imsi-001010000000000, DNN: internet
2026-01-23T22:59:19.417495355+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.420094792+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.422234346+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.424702127+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:19.425757449+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=SMF&target-nf-type=PCF |  |
2026-01-23T22:59:19.426389735+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.427306612+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.430043303+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.431748427+09:00 [INFO][PCF][SMpolicy] Handle CreateSmPolicy
2026-01-23T22:59:19.432542591+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.433675423+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.436091121+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.438325416+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/policy-data/ues/imsi-001010000000000/sm-data?dnn=internet&snssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |  |
2026-01-23T22:59:19.442881953+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.444050966+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.446381478+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.448150695+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/application-data/influenceData?dnns=internet&snssais=%5B%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%5D&supis=imsi-001010000000000 |  |
2026-01-23T22:59:19.448488277+09:00 [INFO][PCF][SMpolicy] Matched [0] trafficInfluDatas from UDR
2026-01-23T22:59:19.449251003+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.450379945+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.452798013+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.454151469+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v2/application-data/influenceData/subs-to-notify |  |
2026-01-23T22:59:19.455144064+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.456273121+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.458171402+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.458851876+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:19.459492724+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=BSF |  |
2026-01-23T22:59:19.460337824+09:00 [INFO][BSF][Proc] Handle CreatePCFBinding
2026-01-23T22:59:19.461669122+09:00 [INFO][BSF][CTX] PCF binding persisted to MongoDB with ID: 334479d0-2166-4e24-a131-ddb7b108fe6b, InsertedID: 334479d0-2166-4e24-a131-ddb7b108fe6b
[GIN] 2026/01/23 - 22:59:19 | 201 |    1.563308ms |       127.0.0.1 | POST     "/nbsf-management/v1/pcfBindings"
2026-01-23T22:59:19.462102562+09:00 [INFO][PCF][Consumer] Successfully registered PCF binding in BSF: 334479d0-2166-4e24-a131-ddb7b108fe6b
2026-01-23T22:59:19.462256803+09:00 [INFO][PCF][SMpolicy] Successfully registered PCF binding in BSF with ID: 334479d0-2166-4e24-a131-ddb7b108fe6b
2026-01-23T22:59:19.463261220+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-smpolicycontrol/v1/sm-policies |  |
2026-01-23T22:59:19.464458630+09:00 [INFO][SMF][PduSess] CHF Selection for SMContext SUPI[imsi-001010000000000] PDUSessionID[1]
2026-01-23T22:59:19.465309315+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.466429291+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.468541494+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.469366575+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-01-23T22:59:19.470130690+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=CHF |  |
2026-01-23T22:59:19.470538273+09:00 [INFO][SMF][Charging] Handle SendConvergedChargingRequest
2026-01-23T22:59:19.470982697+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.472032056+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.474604262+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.480110331+09:00 [INFO][CHF][ChargingPost] HandleChargingdataInitial
2026-01-23T22:59:19.480198343+09:00 [INFO][CHF][ChargingPost] SMF charging event
2026-01-23T22:59:19.480357539+09:00 [ERRO][CHF][ChargingPost] Charging gateway fail to send CDR to billing domain dial tcp 127.0.0.1:2121: connect: connection refused
2026-01-23T22:59:19.480512111+09:00 [INFO][CHF][ChargingPost] Open CDR for UE imsi-001010000000000
2026-01-23T22:59:19.480564264+09:00 [INFO][CHF][ChargingPost] NewChfUe imsi-001010000000000
2026-01-23T22:59:19.481023299+09:00 [INFO][CHF][GIN] | 201 |       127.0.0.1 | POST    | /nchf-convergedcharging/v3/chargingdata |  |
2026-01-23T22:59:19.481810971+09:00 [INFO][SMF][Charging] Send Charging Data Request[Init] successfully
2026-01-23T22:59:19.482000079+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Install PCCRule[PccRuleId-1]
2026-01-23T22:59:19.482152922+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] No srcTcData and tgtTcData. Nothing to do
2026-01-23T22:59:19.482299115+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Install PCCRule[PccRuleId-2]
2026-01-23T22:59:19.482389449+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] No srcTcData and tgtTcData. Nothing to do
2026-01-23T22:59:19.482517205+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Has default path
2026-01-23T22:59:19.483984240+09:00 [INFO][SMF][PduSess] Sending PFCP Session Establishment Request
2026-01-23T22:59:19.485899445+09:00 [INFO][SMF][PduSess] Received PFCP Session Establishment Accepted Response
2026-01-23T22:59:19.490111390+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.490967994+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 sdm-subscriptions
2026-01-23T22:59:19.491289283+09:00 [INFO][UDM][SDM] Handle Subscribe
2026-01-23T22:59:19.493608458+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.494020882+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.497348919+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.498597820+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.502808995+09:00 [INFO][AMF][Producer] Handle N1N2 Message Transfer Request
2026-01-23T22:59:19.503987422+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.505671088+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |  |
2026-01-23T22:59:19.506011824+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v2/imsi-001010000000000/sdm-subscriptions |  |
2026-01-23T22:59:19.506513468+09:00 [INFO][SMF][PduSess] SDM Subscription Successful UE: imsi-001010000000000 SubscriptionId: 2
2026-01-23T22:59:19.506590624+09:00 [INFO][SMF][GIN] | 201 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts |  |
2026-01-23T22:59:19.507343112+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] create smContext[pduSessionID: 1] Success
2026-01-23T22:59:19.507869029+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44021] Send PDU Session Resource Setup Request
2026-01-23T22:59:19.508035574+09:00 [INFO][AMF][GIN] | 200 |       127.0.0.1 | POST    | /namf-comm/v1/ue-contexts/imsi-001010000000000/n1-n2-messages |  |
2026-01-23T22:59:19.509873026+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44021] Handle PDUSessionResourceSetupResponse
2026-01-23T22:59:19.510038897+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:44021] Not comprehended IE ID 0x0079 (criticality: ignore)
2026-01-23T22:59:19.510117079+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:44021] Handle PDUSessionResourceSetupResponse (RAN UE NGAP ID: 1)
2026-01-23T22:59:19.511118658+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-01-23T22:59:19.513667508+09:00 [WARN][NRF][Token] Certificate verify: x509: certificate signed by unknown authority (possibly because of "x509: invalid signature: parent certificate cannot sign this kind of certificate" while trying to verify candidate authority certificate "free5gc")
2026-01-23T22:59:19.518387078+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-01-23T22:59:19.521372892+09:00 [INFO][SMF][PduSess] Receive Update SM Context Request
2026-01-23T22:59:19.626862831+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Accepted Response from AN UPF
2026-01-23T22:59:19.627268957+09:00 [INFO][SMF][GIN] | 200 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts/urn:uuid:117a7808-8f43-445f-b2d7-57ede53d6fb8/modify |  |
```
The PDU session establishment log of OAI-CN5G-UPF is as follows.
```
[2026-01-23 22:59:19.790] [upf_n4 ] [info] handle_receive(1105 bytes)
[2026-01-23 22:59:19.791] [upf_app] [info] Received N4_SESSION_ESTABLISHMENT_REQUEST seid 0x0 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::add(far) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::add(far) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::add(far) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::add(far) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::add(qer) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] TEID received from CP
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::set(fteid) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::get(fteid) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::add(pdr) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::add(qer) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::add(pdr) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::add(qer) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] TEID received from CP
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::set(fteid) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::get(fteid) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::add(pdr) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::add(qer) seid 0x1 
[2026-01-23 22:59:19.791] [upf_n4 ] [info] pfcp_session::add(pdr) seid 0x1 
[2026-01-23 22:59:19.791] [upf_app] [info] Establish datapath: create(pdr(s) & far(s))
[2026-01-23 22:59:19.791] [upf_app] [info] SEID 0x1: Processing 4 PDRs (Uplink: 2, Downlink: 2) in precedence order
[2026-01-23 22:59:19.791] [upf_app] [warning] FTEID is missing for PDR 4. CH bit: Not Set
[2026-01-23 22:59:19.791] [upf_app] [warning] FTEID is missing for PDR 2. CH bit: Not Set
[2026-01-23 22:59:19.791] [upf_app] [info] SEID 0x1: Loaded 4 PDRs into BPF map in precedence order
[2026-01-23 22:59:19.791] [upf_app] [info]   >> Writing to BPF m_session_pdrs map with key SEID=1 (0x1)
[2026-01-23 22:59:19.834] [upf_n4 ] [info] handle_receive(442 bytes)
[2026-01-23 22:59:19.834] [upf_app] [info] Received N4_SESSION_MODIFICATION_REQUEST seid 0x1 
[2026-01-23 22:59:19.834] [pfcp_switch] [warning] TODO check carrefully update fseid in PFCP_SESSION_MODIFICATION_REQUEST
[2026-01-23 22:59:19.834] [upf_app] [info] Modify datapath
[2026-01-23 22:59:19.834] [upf_app] [info] BPFProgram 2 is created!!!
[2026-01-23 22:59:19.834] [upf_app] [info] Initializing QER TC BPF program...
[2026-01-23 22:59:19.835] [upf_app] [info] UDP_INTERFACE = ens22
[2026-01-23 22:59:19.835] [upf_app] [info] GTP_INTERFACE = ens20
[2026-01-23 22:59:19.843] [upf_app] [info] Create Root qdisc on interface ens20 with Default Class: 65535, and r2q: 40
[2026-01-23 22:59:19.846] [upf_app] [info] Create PDU Session Class 1:1 with rate: -1000
Warning: sch_htb: quantum of class 10001 is big. Consider r2q change.
[2026-01-23 22:59:19.848] [upf_app] [info] Create QoS Flow Class 1:38 for PDU Session Parent 1:1
Warning: sch_htb: quantum of class 10026 is small. Consider r2q change.
[2026-01-23 22:59:19.851] [upf_app] [info] Create QoS Flow Class 1:75 for PDU Session Parent 1:1
Warning: sch_htb: quantum of class 1004B is small. Consider r2q change.
[2026-01-23 22:59:19.856] [upf_app] [info] Attach Section tc_filter_traffic to gtp interface
[2026-01-23 22:59:19.859] [upf_app] [info] Attach Section tc_redirect to udp interface
libbpf: Kernel error message: Exclusivity flag on, cannot modify
[2026-01-23 22:59:19.860] [upf_app] [info] Success: TC-BPF hook tc_redirect_traffic already exists for interface ens22 (Ignore: libbpf: Kernel error message))
[2026-01-23 22:59:19.860] [upf_app] [info] BPF program tc_redirect_traffic successfully attached to ens22 interface
[2026-01-23 22:59:19.860] [upf_app] [info] SEID 0x1: Modifying 4 PDRs (Uplink: 2, Downlink: 2) in precedence order
[2026-01-23 22:59:19.861] [upf_app] [info] SEID 0x1: Updated 4 PDRs in BPF map in precedence order
[2026-01-23 22:59:19.861] [upf_app] [info]   >> Writing to BPF m_session_pdrs map with key SEID=1 (0x1)
[2026-01-23 22:59:19.861] [upf_app] [info] BPFProgram 3 is created!!!
[2026-01-23 22:59:19.861] [upf_app] [info] Initializing QER TC BPF program...
[2026-01-23 22:59:19.866] [upf_app] [info] UDP_INTERFACE = ens22
[2026-01-23 22:59:19.866] [upf_app] [info] GTP_INTERFACE = ens20
[2026-01-23 22:59:19.904] [upf_app] [info] Create PDU Session Class 1:1 with rate: -1000
RTNETLINK answers: File exists
[2026-01-23 22:59:19.915] [upf_app] [error] Failed command: tc class add dev ens20 parent 1: classid 1:1 htb rate 4294966296kbit
[2026-01-23 22:59:19.915] [upf_app] [info] Create QoS Flow Class 1:38 for PDU Session Parent 1:1
RTNETLINK answers: File exists
[2026-01-23 22:59:19.918] [upf_app] [error] Failed command: tc class add dev ens20 parent 1:1 classid 1:26 htb rate 1kbit ceil 1000000kbit
[2026-01-23 22:59:19.918] [upf_app] [info] Create QoS Flow Class 1:75 for PDU Session Parent 1:1
RTNETLINK answers: File exists
[2026-01-23 22:59:19.920] [upf_app] [error] Failed command: tc class add dev ens20 parent 1:1 classid 1:4b htb rate 1kbit ceil 208000kbit
[2026-01-23 22:59:19.920] [upf_app] [info] Create QoS Flow Class 1:38 for PDU Session Parent 1:1
RTNETLINK answers: File exists
[2026-01-23 22:59:19.922] [upf_app] [error] Failed command: tc class add dev ens20 parent 1:1 classid 1:26 htb rate 1kbit ceil 1000000kbit
[2026-01-23 22:59:19.922] [upf_app] [info] Create QoS Flow Class 1:75 for PDU Session Parent 1:1
RTNETLINK answers: File exists
[2026-01-23 22:59:19.924] [upf_app] [error] Failed command: tc class add dev ens20 parent 1:1 classid 1:4b htb rate 1kbit ceil 208000kbit
[2026-01-23 22:59:19.928] [upf_app] [info] Attach Section tc_redirect to udp interface
libbpf: Kernel error message: Exclusivity flag on, cannot modify
[2026-01-23 22:59:19.928] [upf_app] [info] Success: TC-BPF hook tc_redirect_traffic already exists for interface ens22 (Ignore: libbpf: Kernel error message))
[2026-01-23 22:59:19.928] [upf_app] [info] BPF program tc_redirect_traffic successfully attached to ens22 interface
[2026-01-23 22:59:19.929] [upf_app] [info] SEID 0x1: Modifying 8 PDRs (Uplink: 4, Downlink: 4) in precedence order
[2026-01-23 22:59:19.932] [upf_app] [info] SEID 0x1: Updated 8 PDRs in BPF map in precedence order
[2026-01-23 22:59:19.932] [upf_app] [info]   >> Writing to BPF m_session_pdrs map with key SEID=1 (0x1)
```
Looking at the console log of the `nr-ue` command, UE has been assigned the IP address `10.60.0.1` from free5GC 5GC.
```
[2026-01-23 22:59:19.547] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
Just in case, make sure it matches the IP address of the UE's TUNnel interface.
```
# ip addr show
...
10: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.60.0.1/24 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::d8cf:595b:412:c627/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
...
```

<a id="ping"></a>

## Ping google.com

Specify the UE's TUNnel interface and try ping.

Please refer to the following for usage of TUNnel interface.

https://github.com/aligungr/UERANSIM/wiki/Usage

<a id="ping_1"></a>

### Case for going through DN 10.60.0.0/16

Run `tcpdump` on VM-DN and check that the packet goes through N6 (ens20).
- `ping google.com` on VM3 (UE)
```
# ping google.com -I uesimtun0 -n
PING google.com (142.250.194.110) from 10.60.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 142.250.194.110: icmp_seq=1 ttl=111 time=21.5 ms
64 bytes from 142.250.194.110: icmp_seq=2 ttl=111 time=17.4 ms
64 bytes from 142.250.194.110: icmp_seq=3 ttl=111 time=18.4 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i ens20 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens20, link-type EN10MB (Ethernet), snapshot length 262144 bytes
23:04:27.560122 IP 10.60.0.1 > 142.250.194.110: ICMP echo request, id 1692, seq 1, length 64
23:04:27.580556 IP 142.250.194.110 > 10.60.0.1: ICMP echo reply, id 1692, seq 1, length 64
23:04:28.561677 IP 10.60.0.1 > 142.250.194.110: ICMP echo request, id 1692, seq 2, length 64
23:04:28.578113 IP 142.250.194.110 > 10.60.0.1: ICMP echo reply, id 1692, seq 2, length 64
23:04:29.563165 IP 10.60.0.1 > 142.250.194.110: ICMP echo request, id 1692, seq 3, length 64
23:04:29.580654 IP 142.250.194.110 > 10.60.0.1: ICMP echo reply, id 1692, seq 3, length 64
```
You could specify the IP address assigned to the TUNnel interface to run almost any applications (iperf3 etc.) as in the following example using `nr-binder` tool.

- `curl google.com` on VM3 (UE)
```
# sh nr-binder 10.60.0.1 curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```
- Run `tcpdump` on VM-DN
```
23:05:10.236201 IP 10.60.0.1.41313 > 142.250.194.110.80: Flags [S], seq 3515789743, win 65280, options [mss 1360,sackOK,TS val 2215986113 ecr 0,nop,wscale 7], length 0
23:05:10.251554 IP 142.250.194.110.80 > 10.60.0.1.41313: Flags [S.], seq 2925662671, ack 3515789744, win 65535, options [mss 1412,sackOK,TS val 248693945 ecr 2215986113,nop,wscale 8], length 0
23:05:10.252484 IP 10.60.0.1.41313 > 142.250.194.110.80: Flags [.], ack 1, win 510, options [nop,nop,TS val 2215986129 ecr 248693945], length 0
23:05:10.252485 IP 10.60.0.1.41313 > 142.250.194.110.80: Flags [P.], seq 1:74, ack 1, win 510, options [nop,nop,TS val 2215986129 ecr 248693945], length 73: HTTP: GET / HTTP/1.1
23:05:10.268581 IP 142.250.194.110.80 > 10.60.0.1.41313: Flags [.], ack 74, win 1050, options [nop,nop,TS val 248693962 ecr 2215986129], length 0
23:05:10.417479 IP 142.250.194.110.80 > 10.60.0.1.41313: Flags [P.], seq 1:774, ack 74, win 1050, options [nop,nop,TS val 248694111 ecr 2215986129], length 773: HTTP: HTTP/1.1 301 Moved Permanently
23:05:10.418389 IP 10.60.0.1.41313 > 142.250.194.110.80: Flags [.], ack 774, win 504, options [nop,nop,TS val 2215986295 ecr 248694111], length 0
23:05:10.419292 IP 10.60.0.1.41313 > 142.250.194.110.80: Flags [F.], seq 74, ack 774, win 504, options [nop,nop,TS val 2215986295 ecr 248694111], length 0
23:05:10.434578 IP 142.250.194.110.80 > 10.60.0.1.41313: Flags [F.], seq 774, ack 75, win 1050, options [nop,nop,TS val 248694128 ecr 2215986295], length 0
23:05:10.435424 IP 10.60.0.1.41313 > 142.250.194.110.80: Flags [.], ack 775, win 504, options [nop,nop,TS val 2215986312 ecr 248694128], length 0
```
Please note that the `ping` tool does not work with `nr-binder`. Please refer to [here](https://github.com/aligungr/UERANSIM/issues/186#issuecomment-729534464) for the reason.
You could now connect to the DN and send any packets on the network using OAI-CN5G-UPF.

---

Now you could work free5GC with OAI-CN5G-UPF.
I would like to thank the excellent developers and all the contributors of free5GC, OAI-CN5G-UPF and UERANSIM.

<a id="changelog"></a>

## Changelog (summary)

- [2026.01.23] Rewrote this using OAI-CN5G-UPF built on Ubuntu 24.04.
- [2026.01.18] Initial release.
