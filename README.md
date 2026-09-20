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
- 5GC - free5GC v4.2.3 (2026.09.16) - https://github.com/free5gc/free5gc
- eBPF/XDP UPF - OAI-CN5G-UPF v2.2.1 (2026.09.09) - https://github.com/openairinterface/oai-cn5g-upf
- UE / RAN - UERANSIM v3.3.0 (2026.09.06) - https://github.com/aligungr/UERANSIM

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
- free5GC v4.2.3 (2026.09.16) - https://free5gc.org/guide/
- OAI-CN5G-UPF v2.2.1 (2026.09.09) - https://github.com/s5uishida/install_oai_upf
- UERANSIM v3.3.0 (2026.09.06) - https://github.com/aligungr/UERANSIM/wiki/Installation

<a id="changes_cp"></a>

### Changes in configuration files of free5GC 5GC C-Plane

The combination of DNN and S-NSSAI parameters can be used in the logic that selects UPF as the connection destination by PFCP.

- DNN
- S-NSSAI

For the sake of simplicity, This time, only DNN will be changed. S-NSSAI of all UEs is fixed as `SST=1` and `SD=010203`.

- `free5gc/config/amfcfg.yaml`
```diff
--- amfcfg.yaml.orig    2026-09-20 09:14:44.506941058 +0900
+++ amfcfg.yaml 2026-09-20 09:25:50.051605401 +0900
@@ -5,7 +5,7 @@
 configuration:
   amfName: AMF # the name of this AMF
   ngapIpList:  # the IP list of N2 interfaces on this AMF
-    - 127.0.0.18
+    - 192.168.0.141
   ngapPort: 38412 # the SCTP port listened by NGAP
 
   # Service-based Interface (SBI) Configuration
@@ -31,22 +31,22 @@
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
--- smfcfg.yaml.orig    2026-09-20 09:32:38.811265818 +0900
+++ smfcfg.yaml 2026-09-20 09:39:58.458076572 +0900
@@ -43,16 +43,16 @@
 
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
@@ -64,8 +64,8 @@
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
@@ -92,7 +92,7 @@
         interfaces: # Interface list for this UPF
           - interfaceType: N3 # the type of the interface (N3 or N9)
             endpoints: # the IP address of this N3/N9 interface on this UPF
-              - 127.0.0.8
+              - 192.168.13.151
             networkInstances: # Data Network Name (DNN)
               - internet
 
@@ -100,7 +100,7 @@
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
--- free5gc-gnb.yaml.orig       2026-09-07 02:39:26.000000000 +0900
+++ free5gc-gnb.yaml    2026-09-20 01:07:35.981147236 +0900
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
--- free5gc-ue.yaml.orig        2026-09-07 04:20:36.000000000 +0900
+++ free5gc-ue.yaml     2026-09-20 01:11:17.272141414 +0900
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
 # Home Network Public Key for protecting with SUCI
@@ -39,7 +39,7 @@
 
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
- free5GC v4.2.3 (2026.09.16) - https://free5gc.org/guide/
- OAI-CN5G-UPF v2.2.1 (2026.09.09) - https://github.com/s5uishida/install_oai_upf
- UERANSIM v3.3.0 (2026.09.06) - https://github.com/aligungr/UERANSIM/wiki/Installation

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

NF_LIST="nrf scp amf smf udr pcf udm nssf ausf chf nef bsf"

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
[2026-09-20 10:06:24.389] [upf_n4 ] [info] handle_receive(30 bytes)
[2026-09-20 10:06:24.389] [upf_n4 ] [info] Handle SX ASSOCIATION SETUP REQUEST
[2026-09-20 10:06:24.390] [upf_n4 ] [info] handle_receive(16 bytes)
[2026-09-20 10:06:24.390] [upf_n4 ] [info] Received SX HEARTBEAT REQUEST
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
UERANSIM v3.3.0
[2026-09-20 10:07:20.737] [sctp] [info] Trying to establish SCTP connection... (192.168.0.141:38412)
[2026-09-20 10:07:20.741] [sctp] [info] SCTP connection established (192.168.0.141:38412)
[2026-09-20 10:07:20.741] [sctp] [debug] SCTP association setup ascId[4]
[2026-09-20 10:07:20.741] [ngap] [debug] Sending NG Setup Request
[2026-09-20 10:07:20.742] [ngap] [debug] NG Setup Response received
[2026-09-20 10:07:20.742] [ngap] [info] NG Setup procedure is successful
```
The free5GC C-Plane log when executed is as follows.
```
2026-09-20T10:07:20.723086345+09:00 [INFO][AMF][Ngap] [AMF] SCTP Accept from: 192.168.0.131:46060
2026-09-20T10:07:20.723977094+09:00 [INFO][AMF][Ngap] Create a new NG connection for: 192.168.0.131:46060
2026-09-20T10:07:20.724018172+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:46060] Send NG-Setup response
```

<a id="start_ue"></a>

#### Start UE

Start UE as follows. This will register the UE with 5GC and establish a PDU session.
```
# ./nr-ue -c ../config/free5gc-ue.yaml
UERANSIM v3.3.0
[2026-09-20 10:08:22.430] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2026-09-20 10:08:22.431] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2026-09-20 10:08:22.431] [nas] [info] Selected plmn[001/01]
[2026-09-20 10:08:22.431] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2026-09-20 10:08:22.431] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2026-09-20 10:08:22.431] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2026-09-20 10:08:22.431] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2026-09-20 10:08:22.431] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2026-09-20 10:08:22.431] [nas] [debug] Sending Initial Registration
[2026-09-20 10:08:22.432] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2026-09-20 10:08:22.432] [rrc] [debug] Sending RRC Setup Request
[2026-09-20 10:08:22.432] [rrc] [info] RRC connection established
[2026-09-20 10:08:22.432] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2026-09-20 10:08:22.432] [nas] [info] UE switches to state [CM-CONNECTED]
[2026-09-20 10:08:22.478] [nas] [debug] Authentication Request received
[2026-09-20 10:08:22.478] [nas] [debug] Received SQN [000000000032]
[2026-09-20 10:08:22.478] [nas] [debug] SQN-MS [000000000000]
[2026-09-20 10:08:22.485] [nas] [debug] Security Mode Command received
[2026-09-20 10:08:22.485] [nas] [debug] Selected integrity[2] ciphering[0]
[2026-09-20 10:08:22.561] [nas] [debug] Registration accept received
[2026-09-20 10:08:22.561] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2026-09-20 10:08:22.562] [nas] [debug] Sending Registration Complete
[2026-09-20 10:08:22.562] [nas] [info] Initial Registration is successful
[2026-09-20 10:08:22.562] [nas] [debug] Sending PDU Session Establishment Request
[2026-09-20 10:08:22.562] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2026-09-20 10:08:22.764] [nas] [debug] Configuration Update Command received
[2026-09-20 10:08:22.872] [nas] [debug] PDU Session Establishment Accept received
[2026-09-20 10:08:22.872] [nas] [info] PDU Session establishment is successful PSI[1]
[2026-09-20 10:08:22.894] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
The free5GC C-Plane log when executed is as follows.
```
2026-09-20T10:08:22.390171792+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:46060] New RanUe [RanUeNgapID:1][AmfUeNgapID:1]
2026-09-20T10:08:22.390250338+09:00 [INFO][AMF][Ngap][ran_addr:192.168.0.131:46060] 5GSMobileIdentity ["SUCI":"suci-0-001-01-0000-0-0-0000000000", err: <nil>]
2026-09-20T10:08:22.390282791+09:00 [INFO][AMF][CTX] New AmfUe [supi:][guti:00101cafe0000000001]
2026-09-20T10:08:22.390322640+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Deregistered] to [Deregistered]
2026-09-20T10:08:22.390332364+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Registration Request
2026-09-20T10:08:22.390339171+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] RegistrationType: Initial Registration
2026-09-20T10:08:22.390345864+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] MobileIdentity5GS: SUCI[suci-0-001-01-0000-0-0-0000000000]
2026-09-20T10:08:22.390357632+09:00 [INFO][AMF][Gmm] Handle event[Start Authentication], transition from [Deregistered] to [Authentication]
2026-09-20T10:08:22.390367349+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Authentication procedure
2026-09-20T10:08:22.391449294+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.395194035+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.396067105+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.397277246+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=AUSF |  |
2026-09-20T10:08:22.398141750+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.402007452+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.403586329+09:00 [INFO][AUSF][UeAuth] HandleUeAuthPostRequest
2026-09-20T10:08:22.403724898+09:00 [INFO][AUSF][UeAuth] Serving network authorized
2026-09-20T10:08:22.404489404+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.407209782+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.407963676+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.408936612+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AUSF&service-names=nudm-ueau&target-nf-type=UDM |  |
2026-09-20T10:08:22.409831346+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.413370107+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.415052588+09:00 [INFO][UDM][UEAU] Handle GenerateAuthDataRequest
2026-09-20T10:08:22.415889480+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.419438155+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.419784468+09:00 [INFO][UDM][Suci] scheme 0
2026-09-20T10:08:22.419908239+09:00 [INFO][UDM][Suci] SUPI type is IMSI
2026-09-20T10:08:22.420260599+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.424037785+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.426654990+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.427368570+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=UDM&target-nf-type=UDR |  |
2026-09-20T10:08:22.430398809+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |  |
2026-09-20T10:08:22.430708433+09:00 [INFO][UDM][Proc] ModifyAuthenticationSubscriptionRequest:  [{replace /sequenceNumber  { 000000000033 map[] 0 }}]
2026-09-20T10:08:22.432854452+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PATCH   | /nudr-dr/v2/subscription-data/imsi-001010000000000/authentication-data/authentication-subscription |  |
2026-09-20T10:08:22.433258921+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | POST    | /nudm-ueau/v1/suci-0-001-01-0000-0-0-0000000000/security-information/generate-auth-data |  |
2026-09-20T10:08:22.433764054+09:00 [INFO][AUSF][UeAuth] Add SuciSupiPair (suci-0-001-01-0000-0-0-0000000000, imsi-001010000000000) to map.
2026-09-20T10:08:22.433836145+09:00 [INFO][AUSF][UeAuth] Use 5G AKA auth method
2026-09-20T10:08:22.434061936+09:00 [INFO][AUSF][GIN] | 201 |       127.0.0.1 | POST    | /nausf-auth/v1/ue-authentications |  |
2026-09-20T10:08:22.434572232+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Send Authentication Request
2026-09-20T10:08:22.434643612+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:46060] Send Downlink Nas Transport
2026-09-20T10:08:22.434740366+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Start T3560 timer
2026-09-20T10:08:22.436158021+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Authentication] to [Authentication]
2026-09-20T10:08:22.436311002+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Handle Authentication Response
2026-09-20T10:08:22.436445878+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:] Stop T3560 timer
2026-09-20T10:08:22.437733130+09:00 [INFO][AUSF][5gAka] Auth5gAkaComfirmRequest
2026-09-20T10:08:22.437785737+09:00 [INFO][AUSF][5gAka] 5G AKA confirmation succeeded
2026-09-20T10:08:22.438913655+09:00 [INFO][UDM][UEAU] Handle ConfirmAuthDataRequest
2026-09-20T10:08:22.441090510+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v2/subscription-data/imsi-001010000000000/authentication-data/authentication-status |  |
2026-09-20T10:08:22.441258041+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-ueau/v1/imsi-001010000000000/auth-events |  |
2026-09-20T10:08:22.441607572+09:00 [INFO][AUSF][GIN] | 200 |       127.0.0.1 | PUT     | /nausf-auth/v1/ue-authentications/suci-0-001-01-0000-0-0-0000000000/5g-aka-confirmation |  |
2026-09-20T10:08:22.441911205+09:00 [INFO][AMF][Gmm] Handle event[Authentication Success], transition from [Authentication] to [SecurityMode]
2026-09-20T10:08:22.442092679+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Security Mode Command
2026-09-20T10:08:22.442226161+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:46060] Send Downlink Nas Transport
2026-09-20T10:08:22.442276370+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3560 timer
2026-09-20T10:08:22.443481191+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [SecurityMode] to [SecurityMode]
2026-09-20T10:08:22.443505241+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Security Mode Complete
2026-09-20T10:08:22.443519560+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3560 timer
2026-09-20T10:08:22.443548297+09:00 [INFO][AMF][Gmm] Handle event[SecurityMode Success], transition from [SecurityMode] to [ContextSetup]
2026-09-20T10:08:22.443561235+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle InitialRegistration
2026-09-20T10:08:22.444721942+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.445803675+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |  |
2026-09-20T10:08:22.447126198+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.453168404+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.454273970+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 nssai
2026-09-20T10:08:22.454483786+09:00 [INFO][UDM][SDM] Handle GetNssai
2026-09-20T10:08:22.455779150+09:00 [INFO][UDR][DataRepo] QueryAmDataProcedure: ueId: imsi-001010000000000, servingPlmnId: 00101
2026-09-20T10:08:22.456494069+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features= |  |
2026-09-20T10:08:22.457628734+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/nssai?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |  |
2026-09-20T10:08:22.458203995+09:00 [INFO][AMF][Gmm] RequestedNssai: &{SNSSAIs:[{SST:1 MappedHPLMNSST:0 SD:010203 MappedHPLMNSD:}]}
2026-09-20T10:08:22.458339291+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] RequestedNssai - ServingSnssai: &{Sst:1 Sd:010203}, HomeSnssai: <nil>
2026-09-20T10:08:22.459328201+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.460535784+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=UDM |  |
2026-09-20T10:08:22.461163211+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.465114958+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.466681929+09:00 [INFO][UDM][UECM] Handle RegistrationAmf3gppAccess
2026-09-20T10:08:22.466775399+09:00 [INFO][UDM][UECM] UEID: imsi-001010000000000
2026-09-20T10:08:22.468883588+09:00 [INFO][UDR][GIN] | 204 |       127.0.0.1 | PUT     | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/amf-3gpp-access |  |
2026-09-20T10:08:22.469232071+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | PUT     | /nudm-uecm/v1/imsi-001010000000000/registrations/amf-3gpp-access |  |
2026-09-20T10:08:22.469850588+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 am-data
2026-09-20T10:08:22.469890124+09:00 [INFO][UDM][SDM] Handle GetAmData
2026-09-20T10:08:22.470397069+09:00 [INFO][UDR][DataRepo] QueryAmDataProcedure: ueId: imsi-001010000000000, servingPlmnId: 00101
2026-09-20T10:08:22.470908861+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/am-data?supported-features= |  |
2026-09-20T10:08:22.471255344+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/am-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |  |
2026-09-20T10:08:22.473452338+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 smf-select-data
2026-09-20T10:08:22.473488783+09:00 [INFO][UDM][SDM] Handle GetSmfSelectData
2026-09-20T10:08:22.474890495+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/smf-selection-subscription-data?supported-features= |  |
2026-09-20T10:08:22.475364739+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/smf-select-data?plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D |  |
2026-09-20T10:08:22.476107863+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 ue-context-in-smf-data
2026-09-20T10:08:22.476273618+09:00 [INFO][UDM][SDM] Handle GetUeContextInSmfData
2026-09-20T10:08:22.477830754+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/smf-registrations?supported-features= |  |
2026-09-20T10:08:22.478250849+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/ue-context-in-smf-data |  |
2026-09-20T10:08:22.479470963+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 sdm-subscriptions
2026-09-20T10:08:22.480007536+09:00 [INFO][UDM][SDM] Handle Subscribe
2026-09-20T10:08:22.482419594+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |  |
2026-09-20T10:08:22.482752781+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v2/imsi-001010000000000/sdm-subscriptions |  |
2026-09-20T10:08:22.483888150+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.485418148+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=AMF&supi=imsi-001010000000000&target-nf-type=PCF |  |
2026-09-20T10:08:22.486013591+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.490123179+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.492472274+09:00 [INFO][PCF][AmPol] Handle AM Policy Create Request
2026-09-20T10:08:22.493301555+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.497892959+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.500630470+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.501539911+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=UDR |  |
2026-09-20T10:08:22.502407054+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.506110550+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.507509572+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/policy-data/ues/imsi-001010000000000/am-data |  |
2026-09-20T10:08:22.508747657+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.510030355+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?guami=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22amfId%22%3A%22cafe00%22%7D&requester-nf-type=PCF&target-nf-type=AMF |  |
2026-09-20T10:08:22.510732956+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.514996333+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.516521924+09:00 [INFO][AMF][Comm] Handle AMF Status Change Subscribe Request
2026-09-20T10:08:22.516720488+09:00 [INFO][AMF][Comm] new AMF Status Subscription[1]
2026-09-20T10:08:22.516879838+09:00 [INFO][AMF][GIN] | 201 |       127.0.0.1 | POST    | /namf-comm/v1/subscriptions |  |
2026-09-20T10:08:22.517324246+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-am-policy-control/v1/policies |  |
2026-09-20T10:08:22.517812958+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Registration Accept
2026-09-20T10:08:22.517869662+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:46060] Send Initial Context Setup Request
2026-09-20T10:08:22.517959730+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Start T3550 timer
2026-09-20T10:08:22.720556942+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [ContextSetup] to [ContextSetup]
2026-09-20T10:08:22.720575920+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle Registration Complete
2026-09-20T10:08:22.720583131+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Stop T3550 timer
2026-09-20T10:08:22.720640020+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Send Configuration Update Command
2026-09-20T10:08:22.720652000+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:46060] Send Downlink Nas Transport
2026-09-20T10:08:22.720691993+09:00 [INFO][AMF][Gmm] Handle event[ContextSetup Success], transition from [ContextSetup] to [Registered]
2026-09-20T10:08:22.720733186+09:00 [INFO][AMF][Gmm] Handle event[Gmm Message], transition from [Registered] to [Registered]
2026-09-20T10:08:22.720740414+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Handle UL NAS Transport
2026-09-20T10:08:22.720746208+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Transport 5GSM Message to SMF
2026-09-20T10:08:22.720755294+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] Select SMF [snssai: {Sst:1 Sd:010203}, dnn: internet]
2026-09-20T10:08:22.721673665+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.722767965+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=AMF&target-nf-type=NSSF |  |
2026-09-20T10:08:22.723368806+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.727141567+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.728880310+09:00 [INFO][NSSF][NsSel] Handle NSSelectionGet
2026-09-20T10:08:22.729267464+09:00 [WARN][NSSF][Util] No TA {"plmnId":{"mcc":"001","mnc":"01"},"tac":"000001"} in NSSF configuration
2026-09-20T10:08:22.729718665+09:00 [INFO][NSSF][GIN] | 200 |       127.0.0.1 | GET     | /nnssf-nsselection/v2/network-slice-information?nf-id=ad3a6bf5-c7eb-4c6d-a9db-f8a267ef1e38&nf-type=AMF&slice-info-request-for-pdu-session=%7B%22sNssai%22%3A%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%2C%22roamingIndication%22%3A%22NON_ROAMING%22%7D&tai=%7B%22plmnId%22%3A%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%2C%22tac%22%3A%22000001%22%7D |  |
2026-09-20T10:08:22.730259757+09:00 [WARN][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] nsiInformation is still nil, use default NRF[http://127.0.0.10:8000]
2026-09-20T10:08:22.731057324+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.732455554+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?dnn=internet&preferred-locality=area1&requester-nf-type=AMF&service-names=nsmf-pdusession&snssais=%5B%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%5D&target-nf-type=SMF&target-plmn-list=%5B%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D%5D |  |
2026-09-20T10:08:22.733058630+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.737421875+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.741746750+09:00 [INFO][SMF][PduSess] Receive Create SM Context Request
2026-09-20T10:08:22.742781394+09:00 [INFO][SMF][PduSess] In HandlePDUSessionSMContextCreate
2026-09-20T10:08:22.743006737+09:00 [INFO][SMF][CTX] UrrPeriod: 30s
2026-09-20T10:08:22.743025769+09:00 [INFO][SMF][CTX] UrrThreshold: 500000
2026-09-20T10:08:22.743703612+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.747078149+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.747838675+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.748655319+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=UDM |  |
2026-09-20T10:08:22.749373551+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Send NF Discovery Serving UDM Successfully
2026-09-20T10:08:22.749724447+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.753415762+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.754580254+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 sm-data
2026-09-20T10:08:22.754781619+09:00 [INFO][UDM][SDM] Handle GetSmData
2026-09-20T10:08:22.754868751+09:00 [INFO][UDM][SDM] getSmDataProcedure: SUPI[imsi-001010000000000] PLMNID[00101] DNN[internet] SNssai[{"sst":1,"sd":"010203"}]
2026-09-20T10:08:22.756893106+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/subscription-data/imsi-001010000000000/00101/provisioned-data/sm-data?single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |  |
2026-09-20T10:08:22.757660040+09:00 [INFO][UDM][GIN] | 200 |       127.0.0.1 | GET     | /nudm-sdm/v2/imsi-001010000000000/sm-data?dnn=internet&plmn-id=%7B%22mcc%22%3A%22001%22%2C%22mnc%22%3A%2201%22%7D&single-nssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |  |
2026-09-20T10:08:22.758809375+09:00 [INFO][SMF][GSM] In HandlePDUSessionEstablishmentRequest
2026-09-20T10:08:22.758943843+09:00 [INFO][SMF][GSM] Protocol Configuration Options
2026-09-20T10:08:22.759136726+09:00 [INFO][SMF][GSM] &{IPv4LinkMTUReq:false DNSV4Req:true DNSV6Req:false P_CSCF_IPv4AddrReq:false UEStatus3GPPPSDataOff:0}
2026-09-20T10:08:22.760240107+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.761551941+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-instance-id=ad3a6bf5-c7eb-4c6d-a9db-f8a267ef1e38&target-nf-type=AMF |  |
2026-09-20T10:08:22.761996100+09:00 [INFO][SMF][Consumer] SendNFDiscoveryServingAMF ok
2026-09-20T10:08:22.762077424+09:00 [INFO][SMF][CTX] Allocated UE IP address: 10.60.0.1
2026-09-20T10:08:22.762102537+09:00 [INFO][SMF][CTX] Selected UPF: UPF
2026-09-20T10:08:22.762113819+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Allocated PDUAdress[10.60.0.1]
2026-09-20T10:08:22.762496397+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.763085094+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=BSF |  |
2026-09-20T10:08:22.763835127+09:00 [INFO][BSF][Proc] Handle GetPCFBindings
[GIN] 2026/09/20 - 10:08:22 | 200 |     597.282µs |       127.0.0.1 | GET      "/nbsf-management/v1/pcfBindings?dnn=internet&snssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D&supi=imsi-001010000000000"
2026-09-20T10:08:22.764627135+09:00 [INFO][SMF][Consumer] Found existing PCF binding for SUPI: imsi-001010000000000, DNN: internet
2026-09-20T10:08:22.764648976+09:00 [INFO][SMF][Consumer] Using existing PCF from BSF binding: cdef9a0d-51a1-4b1e-bad1-381ff2f7f3f9
2026-09-20T10:08:22.765329016+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.765936169+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-instance-id=cdef9a0d-51a1-4b1e-bad1-381ff2f7f3f9&target-nf-type=PCF |  |
2026-09-20T10:08:22.766301709+09:00 [WARN][SMF][Consumer] Failed to discover PCF cdef9a0d-51a1-4b1e-bad1-381ff2f7f3f9 from NRF, falling back to general PCF selection: <nil>
2026-09-20T10:08:22.766665625+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.767774071+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?preferred-locality=area1&requester-nf-type=SMF&target-nf-type=PCF |  |
2026-09-20T10:08:22.768381879+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.772403316+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.774087162+09:00 [INFO][PCF][SMpolicy] Handle CreateSmPolicy
2026-09-20T10:08:22.776144786+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/policy-data/ues/imsi-001010000000000/sm-data?dnn=internet&snssai=%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D |  |
2026-09-20T10:08:22.780748127+09:00 [INFO][UDR][GIN] | 200 |       127.0.0.1 | GET     | /nudr-dr/v2/application-data/influenceData?dnns=internet&snssais=%5B%7B%22sst%22%3A1%2C%22sd%22%3A%22010203%22%7D%5D&supis=imsi-001010000000000 |  |
2026-09-20T10:08:22.781176354+09:00 [INFO][PCF][SMpolicy] Matched [0] trafficInfluDatas from UDR
2026-09-20T10:08:22.782559859+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.785404311+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.786141229+09:00 [INFO][NRF][NFM] Handle GetNFInstanceRequest
2026-09-20T10:08:22.786810009+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-nfm/v1/nf-instances/f456042a-0f9b-4c81-8ba6-6c67443db632 |  |
2026-09-20T10:08:22.787215403+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v2/application-data/influenceData/subs-to-notify |  |
2026-09-20T10:08:22.788180629+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.789253651+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=PCF&target-nf-type=BSF |  |
2026-09-20T10:08:22.791549820+09:00 [INFO][BSF][Proc] Handle CreatePCFBinding
[GIN] 2026/09/20 - 10:08:22 | 403 |     333.994µs |       127.0.0.1 | POST     "/nbsf-management/v1/pcfBindings"
2026-09-20T10:08:22.792062783+09:00 [WARN][PCF][SMpolicy] Failed to register PCF binding in BSF: unexpected response status: 403
2026-09-20T10:08:22.792669204+09:00 [INFO][PCF][GIN] | 201 |       127.0.0.1 | POST    | /npcf-smpolicycontrol/v1/sm-policies |  |
2026-09-20T10:08:22.793835807+09:00 [INFO][SMF][PduSess] CHF Selection for SMContext SUPI[imsi-001010000000000] PDUSessionID[1]
2026-09-20T10:08:22.795781289+09:00 [INFO][NRF][DISC] Handle NFDiscoveryRequest
2026-09-20T10:08:22.796691729+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | GET     | /nnrf-disc/v1/nf-instances?requester-nf-type=SMF&target-nf-type=CHF |  |
2026-09-20T10:08:22.797278682+09:00 [INFO][SMF][Charging] Handle SendConvergedChargingRequest
2026-09-20T10:08:22.797690610+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.801355424+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.806832236+09:00 [INFO][CHF][ChargingPost] HandleChargingdataInitial
2026-09-20T10:08:22.807059381+09:00 [INFO][CHF][ChargingPost] SMF charging event
2026-09-20T10:08:22.807295360+09:00 [ERRO][CHF][ChargingPost] Charging gateway fail to send CDR to billing domain dial tcp 127.0.0.1:2121: connect: connection refused
2026-09-20T10:08:22.807413843+09:00 [INFO][CHF][ChargingPost] Open CDR for UE imsi-001010000000000
2026-09-20T10:08:22.807432953+09:00 [INFO][CHF][ChargingPost] NewChfUe imsi-001010000000000
2026-09-20T10:08:22.807878615+09:00 [INFO][CHF][GIN] | 201 |       127.0.0.1 | POST    | /nchf-convergedcharging/v3/chargingdata |  |
2026-09-20T10:08:22.808446979+09:00 [INFO][SMF][Charging] Send Charging Data Request[Init] successfully
2026-09-20T10:08:22.808614281+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Install PCCRule[PccRuleId-1]
2026-09-20T10:08:22.809097501+09:00 [INFO][SMF][Charging] [AddChargingRules] ChargingInfo bind URR[7] -> RG=1 UPF=bdc98394-3925-4178-aedb-0661f7090b69 level=0 method=OFFLINE_CHARGING
2026-09-20T10:08:22.811646835+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] No srcTcData and tgtTcData. Nothing to do
2026-09-20T10:08:22.811836702+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] Has default path
2026-09-20T10:08:22.814387182+09:00 [INFO][SMF][PduSess] Sending PFCP Session Establishment Request
2026-09-20T10:08:22.814561453+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] [BuildEstReq] UPF=bdc98394-3925-4178-aedb-0661f7090b69 urrList=6 unique_urrs=3
2026-09-20T10:08:22.816089552+09:00 [INFO][SMF][PduSess] Received PFCP Session Establishment Accepted Response
2026-09-20T10:08:22.816836087+09:00 [INFO][NRF][Token] In HTTPAccessTokenRequest
2026-09-20T10:08:22.815945985+09:00 [INFO][UDM][Consumer] TwoLayerPathHandlerFunc,  imsi-001010000000000 sdm-subscriptions
2026-09-20T10:08:22.817278206+09:00 [INFO][UDM][SDM] Handle Subscribe
2026-09-20T10:08:22.819482545+09:00 [INFO][UDR][GIN] | 201 |       127.0.0.1 | POST    | /nudr-dr/v2/subscription-data/imsi-001010000000000/context-data/sdm-subscriptions |  |
2026-09-20T10:08:22.821217376+09:00 [INFO][UDM][GIN] | 201 |       127.0.0.1 | POST    | /nudm-sdm/v2/imsi-001010000000000/sdm-subscriptions |  |
2026-09-20T10:08:22.822237292+09:00 [INFO][SMF][PduSess] SDM Subscription Successful UE: imsi-001010000000000 SubscriptionId: 2
2026-09-20T10:08:22.822360042+09:00 [INFO][SMF][GIN] | 201 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts |  |
2026-09-20T10:08:22.822983690+09:00 [INFO][AMF][Gmm][amf_ue_ngap_id:RU:1,AU:1(3GPP)][supi:SUPI:imsi-001010000000000] create smContext[pduSessionID: 1] Success
2026-09-20T10:08:22.825120326+09:00 [INFO][NRF][GIN] | 200 |       127.0.0.1 | POST    | /oauth2/token |  |
2026-09-20T10:08:22.826762348+09:00 [INFO][AMF][Producer] Handle N1N2 Message Transfer Request
2026-09-20T10:08:22.827073033+09:00 [INFO][AMF][Ngap][amf_ue_ngap_id:RU:1,AU:1(3GPP)][ran_addr:192.168.0.131:46060] Send PDU Session Resource Setup Request
2026-09-20T10:08:22.827188192+09:00 [INFO][AMF][GIN] | 200 |       127.0.0.1 | POST    | /namf-comm/v1/ue-contexts/imsi-001010000000000/n1-n2-messages |  |
2026-09-20T10:08:22.831420986+09:00 [INFO][SMF][PduSess] Receive Update SM Context Request
2026-09-20T10:08:22.831746079+09:00 [INFO][SMF][PduSess][pdu_session_id:1][supi:imsi-001010000000000] [BuildModReq] UPF=bdc98394-3925-4178-aedb-0661f7090b69 urrList=0 unique_urrs=0
2026-09-20T10:08:22.890321431+09:00 [INFO][SMF][PduSess] Received PFCP Session Modification Accepted Response from AN UPF
2026-09-20T10:08:22.890628562+09:00 [INFO][SMF][GIN] | 200 |       127.0.0.1 | POST    | /nsmf-pdusession/v1/sm-contexts/urn:uuid:6652b5cd-b049-442e-a7de-cadd2cfdd747/modify |  |
```
The PDU session establishment log of OAI-CN5G-UPF is as follows.
```
[2026-09-20 10:08:22.826] [upf_n4 ] [info] handle_receive(626 bytes)
[2026-09-20 10:08:22.826] [upf_app] [info] 
[2026-09-20 10:08:22.826] [upf_app] [info] ╔═════════════════════════════════════════════════════════════════════════════╗
[2026-09-20 10:08:22.826] [upf_app] [info] │             Received N4_SESSION_ESTABLISHMENT_REQUEST seid 0x0              │
[2026-09-20 10:08:22.826] [upf_app] [info] ╚═════════════════════════════════════════════════════════════════════════════╝
[2026-09-20 10:08:22.826] [upf_n4 ] [info] pfcp_session::add(far) seid 0x1 FAR=1
[2026-09-20 10:08:22.826] [upf_n4 ] [info]   └─ Adding new FAR 1 to session 0x1
[2026-09-20 10:08:22.826] [upf_n4 ] [info] pfcp_session::add(far) seid 0x1 FAR=2
[2026-09-20 10:08:22.826] [upf_n4 ] [info]   └─ Adding new FAR 2 to session 0x1
[2026-09-20 10:08:22.826] [upf_n4 ] [info] pfcp_session::add(qer) seid 0x1 QER=1
[2026-09-20 10:08:22.826] [upf_n4 ] [info]   └─ Adding new QER 1 to session 0x1
[2026-09-20 10:08:22.826] [upf_n4 ] [info] TEID 0x2 received from CP
[2026-09-20 10:08:22.826] [upf_n4 ] [info] pfcp_session::set(fteid) seid 0x1 
[2026-09-20 10:08:22.826] [upf_n4 ] [info] pfcp_session::get(fteid) seid 0x1 
[2026-09-20 10:08:22.826] [upf_n4 ] [info] pfcp_session::add(pdr) seid 0x1 PDR=1
[2026-09-20 10:08:22.826] [upf_n4 ] [info]   └─ Adding new PDR 1 to session 0x1
[2026-09-20 10:08:22.826] [upf_n4 ] [info] pfcp_session::add(qer) seid 0x1 QER=1
[2026-09-20 10:08:22.826] [upf_n4 ] [warning]   └─ Skipping duplicate QER 1 (QFI 1) in session 0x1 - already exists
[2026-09-20 10:08:22.826] [upf_n4 ] [info] pfcp_session::add(pdr) seid 0x1 PDR=2
[2026-09-20 10:08:22.826] [upf_n4 ] [info]   └─ Adding new PDR 2 to session 0x1
[2026-09-20 10:08:22.826] [upf_app] [info] Establish datapath: create(pdr(s), far(s), qer(s), urr(s), bar(s), mar(s))
[2026-09-20 10:08:22.826] [upf_app] [info] [eBPF] Create Pipeline - Creating pipeline for session 0x1
[2026-09-20 10:08:22.826] [upf_app] [warning] F-TEID missing for PDR 2 (CH bit: Not Set)
[2026-09-20 10:08:22.826] [upf_app] [info] Pipeline created for session 0x1 with 2 PDRs [type = IP, rules = 0x1]
[2026-09-20 10:08:22.826] [upf_app] [info] [N4] Create Session: seid 0x1 - eBPF data-path pipeline created successfully
[2026-09-20 10:08:22.843] [upf_n4 ] [info] handle_receive(228 bytes)
[2026-09-20 10:08:22.843] [upf_app] [info] 
[2026-09-20 10:08:22.843] [upf_app] [info] ╔═════════════════════════════════════════════════════════════════════════════╗
[2026-09-20 10:08:22.843] [upf_app] [info] │             Received N4_SESSION_MODIFICATION_REQUEST seid 0x1               │
[2026-09-20 10:08:22.843] [upf_app] [info] ╚═════════════════════════════════════════════════════════════════════════════╝
[2026-09-20 10:08:22.843] [pfcp_switch] [warning] TODO check carefully update fseid in PFCP_SESSION_MODIFICATION_REQUEST
[2026-09-20 10:08:22.843] [upf_n4 ] [info] pfcp_session::update(pdr) seid 0x1 PDR=2
[2026-09-20 10:08:22.843] [upf_n4 ] [info]   └─ Updating PDR 2 in session 0x1
[2026-09-20 10:08:22.843] [upf_n4 ] [info] pfcp_session::update(far) seid 0x1 FAR=2
[2026-09-20 10:08:22.843] [upf_n4 ] [info]   └─ Updating FAR 2 in session 0x1
[2026-09-20 10:08:22.843] [upf_app] [info] Modify datapath
[2026-09-20 10:08:22.843] [upf_app] [info] [eBPF] Modify Pipeline - Updating pipeline for session 0x1
[2026-09-20 10:08:22.844] [upf_app] [info] 
[2026-09-20 10:08:22.844] [upf_app] [info]   ┌───────────────────────────────────────────────────┐
[2026-09-20 10:08:22.844] [upf_app] [info]   │            QoS ENFORCEMENT SETUP                  │
[2026-09-20 10:08:22.844] [upf_app] [info]   │       Session: 0x1, Interface: ens20              │
[2026-09-20 10:08:22.844] [upf_app] [info]   └───────────────────────────────────────────────────┘
[2026-09-20 10:08:22.844] [upf_app] [info]   ┌─ N6 Interface (Non-GTP): ens22
[2026-09-20 10:08:22.844] [upf_app] [info]   └─ N3 Interface (GTP):     ens20
[2026-09-20 10:08:22.848] [upf_app] [info]   ┌─ Creating Root HTB Qdisc on ens20
[2026-09-20 10:08:22.848] [upf_app] [info]   │  • Default Class: 65535
[2026-09-20 10:08:22.848] [upf_app] [info]   │  • r2q Parameter: 1000
[2026-09-20 10:08:22.851] [upf_app] [info]   └─ ✓ Root qdisc created successfully on interface: ens20
[2026-09-20 10:08:22.851] [upf_app] [info]   ┌─ Creating PDU Session Class 1:1
[2026-09-20 10:08:22.851] [upf_app] [info]   │  • Session Rate: 4,294,966,296 kbps
[2026-09-20 10:08:22.853] [upf_app] [info]   └─ ✓ PDU session class  1:1 created successfully
[2026-09-20 10:08:22.853] [upf_app] [warning] QoS Flow missing GBR: set it to 0.8 x MBR
[2026-09-20 10:08:22.853] [upf_app] [info]   ┌─ Createing QoS Flow Class 1:38 for PDU Session Parent 1:1
[2026-09-20 10:08:22.853] [upf_app] [info]   │  • QoS Flow Rate (GBR): 800,000 kbps
[2026-09-20 10:08:22.853] [upf_app] [info]   │  • QoS Flow Ceil (MBR): 1,000,000 kbps
[2026-09-20 10:08:22.855] [upf_app] [info]   └─ ✓ QoS Flow class  1:38 created successfully for QER 1
[2026-09-20 10:08:22.859] [upf_app] [info] Attach Section tc_filter_traffic to gtp interface
[2026-09-20 10:08:22.862] [upf_app] [info] Attach Section tc_redirect to udp interface
libbpf: Kernel error message: Exclusivity flag on, cannot modify
[2026-09-20 10:08:22.862] [upf_app] [info] Success: [QERTCProgram] TC-BPF hook tc_redirect_traffic already exists for interface ens22 (Ignore: libbpf: Kernel error message))
[2026-09-20 10:08:22.862] [upf_app] [info] [QERTCProgram] TC-BPF 'tc_redirect_traffic' attached to ens22 (ingress, ifindex=6)
[2026-09-20 10:08:22.862] [upf_app] [info] 
[2026-09-20 10:08:22.862] [upf_app] [info]   ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────┐
[2026-09-20 10:08:22.862] [upf_app] [info]   │                                      QoS FLOWS - Session 0x1                                             │
[2026-09-20 10:08:22.862] [upf_app] [info]   ├──────┬─────┬──────────────┬────────────┬────────────┬────────────────────────────────────────────────────┤
[2026-09-20 10:08:22.862] [upf_app] [info]   │ QER  │ QFI │    Class     │ GBR (kbps) │ MBR (kbps) │               Flow Description                     │
[2026-09-20 10:08:22.862] [upf_app] [info]   ├──────┼─────┼──────────────┼────────────┼────────────┼────────────────────────────────────────────────────┤
[2026-09-20 10:08:22.862] [upf_app] [info]   │ 1    │ 1   │ 1:38         │ 800,000    │ 1,000,000  │ permit out ip from any to assigned                 │
[2026-09-20 10:08:22.862] [upf_app] [info]   └──────┴─────┴──────────────┴────────────┴────────────┴────────────────────────────────────────────────────┘
[2026-09-20 10:08:22.862] [upf_app] [info] 
[2026-09-20 10:08:22.862] [upf_app] [info] 
[2026-09-20 10:08:22.862] [upf_app] [info]   ┌───────────────────────────────────────────────────┐
[2026-09-20 10:08:22.862] [upf_app] [info]   │           QoS ENFORCEMENT COMPLETED               │
[2026-09-20 10:08:22.862] [upf_app] [info]   │      Session 0x1: 1 QoS Flow(s) configured        │
[2026-09-20 10:08:22.862] [upf_app] [info]   └───────────────────────────────────────────────────┘
[2026-09-20 10:08:22.862] [upf_app] [info] 
[2026-09-20 10:08:22.864] [upf_app] [info] [eBPF] Modify Pipeline - Pipeline modified for session 0x1 with 2 PDRs (1 uplink TEIDs, 1 downlink TEIDs)
[2026-09-20 10:08:22.864] [upf_app] [info] [eBPF] Modify Pipeline - Updating pipeline for session 0x1
[2026-09-20 10:08:22.864] [upf_app] [info] 
[2026-09-20 10:08:22.864] [upf_app] [info]   ┌───────────────────────────────────────────────────┐
[2026-09-20 10:08:22.864] [upf_app] [info]   │            QoS ENFORCEMENT SETUP                  │
[2026-09-20 10:08:22.864] [upf_app] [info]   │       Session: 0x1, Interface: ens20              │
[2026-09-20 10:08:22.864] [upf_app] [info]   └───────────────────────────────────────────────────┘
[2026-09-20 10:08:22.864] [upf_app] [info]   ┌─ N6 Interface (Non-GTP): ens22
[2026-09-20 10:08:22.864] [upf_app] [info]   └─ N3 Interface (GTP):     ens20
[2026-09-20 10:08:22.872] [upf_app] [info]   ┌─ Creating PDU Session Class 1:1
[2026-09-20 10:08:22.872] [upf_app] [info]   │  • Session Rate: 4,294,966,296 kbps
RTNETLINK answers: File exists
[2026-09-20 10:08:22.879] [upf_app] [error]   └─ ✗ Failed to create PDU session class 1:1
[2026-09-20 10:08:22.879] [upf_app] [warning] QoS Flow missing GBR: set it to 0.8 x MBR
[2026-09-20 10:08:22.879] [upf_app] [info]   ┌─ Createing QoS Flow Class 1:38 for PDU Session Parent 1:1
[2026-09-20 10:08:22.879] [upf_app] [info]   │  • QoS Flow Rate (GBR): 800,000 kbps
[2026-09-20 10:08:22.879] [upf_app] [info]   │  • QoS Flow Ceil (MBR): 1,000,000 kbps
RTNETLINK answers: File exists
[2026-09-20 10:08:22.881] [upf_app] [error]   └─ ✗ Failed to create QoS flow class for QER 1
[2026-09-20 10:08:22.885] [upf_app] [info] Attach Section tc_redirect to udp interface
libbpf: Kernel error message: Exclusivity flag on, cannot modify
[2026-09-20 10:08:22.885] [upf_app] [info] Success: [QERTCProgram] TC-BPF hook tc_redirect_traffic already exists for interface ens22 (Ignore: libbpf: Kernel error message))
[2026-09-20 10:08:22.885] [upf_app] [info] [QERTCProgram] TC-BPF 'tc_redirect_traffic' attached to ens22 (ingress, ifindex=6)
[2026-09-20 10:08:22.885] [upf_app] [warning] 
[2026-09-20 10:08:22.885] [upf_app] [warning]   ┌───────────────────────────────────────────────────┐
[2026-09-20 10:08:22.885] [upf_app] [warning]   │  QoS ENFORCEMENT SETUP - COMPLETED WITH WARNINGS  │
[2026-09-20 10:08:22.885] [upf_app] [warning]   │      Session 0x1: 0 QoS Flow(s) configured        │
[2026-09-20 10:08:22.885] [upf_app] [warning]   │   Some TC operations failed (see warnings above)  │
[2026-09-20 10:08:22.885] [upf_app] [warning]   └───────────────────────────────────────────────────┘
[2026-09-20 10:08:22.885] [upf_app] [info] 
[2026-09-20 10:08:22.885] [upf_app] [info] [QERTCProgram] BPF program torn down successfully
[2026-09-20 10:08:22.887] [upf_app] [info] [eBPF] Modify Pipeline - Pipeline modified for session 0x1 with 2 PDRs (1 uplink TEIDs, 1 downlink TEIDs)
[2026-09-20 10:08:22.887] [upf_app] [info] [N4] Session Modification: seid 0x1
[2026-09-20 10:08:22.887] [upf_app] [info]   └─ Updated: 1 PDR, 1 FAR, 0 QER, 0 URR, 0 BAR, 0 MAR
[2026-09-20 10:08:22.887] [upf_app] [info] [eBPF] Modify Pipeline - Updating pipeline for session 0x1
[2026-09-20 10:08:22.887] [upf_app] [info] 
[2026-09-20 10:08:22.887] [upf_app] [info]   ┌───────────────────────────────────────────────────┐
[2026-09-20 10:08:22.887] [upf_app] [info]   │            QoS ENFORCEMENT SETUP                  │
[2026-09-20 10:08:22.887] [upf_app] [info]   │       Session: 0x1, Interface: ens20              │
[2026-09-20 10:08:22.887] [upf_app] [info]   └───────────────────────────────────────────────────┘
[2026-09-20 10:08:22.887] [upf_app] [info]   ┌─ N6 Interface (Non-GTP): ens22
[2026-09-20 10:08:22.887] [upf_app] [info]   └─ N3 Interface (GTP):     ens20
[2026-09-20 10:08:22.892] [upf_app] [info]   ┌─ Creating PDU Session Class 1:1
[2026-09-20 10:08:22.892] [upf_app] [info]   │  • Session Rate: 4,294,966,296 kbps
RTNETLINK answers: File exists
[2026-09-20 10:08:22.894] [upf_app] [error]   └─ ✗ Failed to create PDU session class 1:1
[2026-09-20 10:08:22.894] [upf_app] [warning] QoS Flow missing GBR: set it to 0.8 x MBR
[2026-09-20 10:08:22.894] [upf_app] [info]   ┌─ Createing QoS Flow Class 1:38 for PDU Session Parent 1:1
[2026-09-20 10:08:22.894] [upf_app] [info]   │  • QoS Flow Rate (GBR): 800,000 kbps
[2026-09-20 10:08:22.894] [upf_app] [info]   │  • QoS Flow Ceil (MBR): 1,000,000 kbps
RTNETLINK answers: File exists
[2026-09-20 10:08:22.896] [upf_app] [error]   └─ ✗ Failed to create QoS flow class for QER 1
[2026-09-20 10:08:22.899] [upf_app] [info] Attach Section tc_redirect to udp interface
libbpf: Kernel error message: Exclusivity flag on, cannot modify
[2026-09-20 10:08:22.899] [upf_app] [info] Success: [QERTCProgram] TC-BPF hook tc_redirect_traffic already exists for interface ens22 (Ignore: libbpf: Kernel error message))
[2026-09-20 10:08:22.899] [upf_app] [info] [QERTCProgram] TC-BPF 'tc_redirect_traffic' attached to ens22 (ingress, ifindex=6)
[2026-09-20 10:08:22.899] [upf_app] [warning] 
[2026-09-20 10:08:22.900] [upf_app] [warning]   ┌───────────────────────────────────────────────────┐
[2026-09-20 10:08:22.900] [upf_app] [warning]   │  QoS ENFORCEMENT SETUP - COMPLETED WITH WARNINGS  │
[2026-09-20 10:08:22.900] [upf_app] [warning]   │      Session 0x1: 0 QoS Flow(s) configured        │
[2026-09-20 10:08:22.900] [upf_app] [warning]   │   Some TC operations failed (see warnings above)  │
[2026-09-20 10:08:22.900] [upf_app] [warning]   └───────────────────────────────────────────────────┘
[2026-09-20 10:08:22.900] [upf_app] [info] 
[2026-09-20 10:08:22.900] [upf_app] [info] [QERTCProgram] BPF program torn down successfully
[2026-09-20 10:08:22.901] [upf_app] [info] [eBPF] Modify Pipeline - Pipeline modified for session 0x1 with 2 PDRs (1 uplink TEIDs, 1 downlink TEIDs)
[2026-09-20 10:08:22.901] [upf_app] [info] [N4] Update Session: seid 0x1 - eBPF data-path pipeline updated successfully
[2026-09-20 10:08:22.901] [upf_app] [info] [N4] Session Modification: seid 0x1 - Completed successfully [Status: Session updated]
```
Looking at the console log of the `nr-ue` command, UE has been assigned the IP address `10.60.0.1` from free5GC 5GC.
```
[2026-09-20 10:08:22.894] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.60.0.1] is up.
```
Just in case, make sure it matches the IP address of the UE's TUNnel interface.
```
# ip addr show
...
6: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.60.0.1/24 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::3019:47c:940a:5485/64 scope link stable-privacy 
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
PING google.com (142.250.21.138) from 10.60.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 142.250.21.138: icmp_seq=1 ttl=106 time=18.8 ms
64 bytes from 142.250.21.138: icmp_seq=2 ttl=106 time=18.7 ms
64 bytes from 142.250.21.138: icmp_seq=3 ttl=106 time=18.6 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i ens20 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens20, link-type EN10MB (Ethernet), snapshot length 262144 bytes
10:14:18.225996 IP 10.60.0.1 > 142.250.21.138: ICMP echo request, id 1312, seq 1, length 64
10:14:18.243776 IP 142.250.21.138 > 10.60.0.1: ICMP echo reply, id 1312, seq 1, length 64
10:14:19.227985 IP 10.60.0.1 > 142.250.21.138: ICMP echo request, id 1312, seq 2, length 64
10:14:19.245579 IP 142.250.21.138 > 10.60.0.1: ICMP echo reply, id 1312, seq 2, length 64
10:14:20.229652 IP 10.60.0.1 > 142.250.21.138: ICMP echo request, id 1312, seq 3, length 64
10:14:20.247330 IP 142.250.21.138 > 10.60.0.1: ICMP echo reply, id 1312, seq 3, length 64
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
10:15:31.629392 IP 10.60.0.1.48709 > 142.250.21.139.80: Flags [S], seq 3418321718, win 65280, options [mss 1360,sackOK,TS val 13621707 ecr 0,nop,wscale 7], length 0
10:15:31.644979 IP 142.250.21.139.80 > 10.60.0.1.48709: Flags [S.], seq 2293845036, ack 3418321719, win 65535, options [mss 1412,sackOK,TS val 2601802269 ecr 13621707,nop,wscale 8], length 0
10:15:31.646073 IP 10.60.0.1.48709 > 142.250.21.139.80: Flags [.], ack 1, win 510, options [nop,nop,TS val 13621724 ecr 2601802269], length 0
10:15:31.646073 IP 10.60.0.1.48709 > 142.250.21.139.80: Flags [P.], seq 1:74, ack 1, win 510, options [nop,nop,TS val 13621724 ecr 2601802269], length 73: HTTP: GET / HTTP/1.1
10:15:31.662360 IP 142.250.21.139.80 > 10.60.0.1.48709: Flags [.], ack 74, win 1050, options [nop,nop,TS val 2601802286 ecr 13621724], length 0
10:15:31.704810 IP 142.250.21.139.80 > 10.60.0.1.48709: Flags [P.], seq 1:774, ack 74, win 1050, options [nop,nop,TS val 2601802329 ecr 13621724], length 773: HTTP: HTTP/1.1 301 Moved Permanently
10:15:31.706546 IP 10.60.0.1.48709 > 142.250.21.139.80: Flags [.], ack 774, win 504, options [nop,nop,TS val 13621784 ecr 2601802329], length 0
10:15:31.706546 IP 10.60.0.1.48709 > 142.250.21.139.80: Flags [F.], seq 74, ack 774, win 504, options [nop,nop,TS val 13621784 ecr 2601802329], length 0
10:15:31.722876 IP 142.250.21.139.80 > 10.60.0.1.48709: Flags [F.], seq 774, ack 75, win 1050, options [nop,nop,TS val 2601802347 ecr 13621784], length 0
10:15:31.723671 IP 10.60.0.1.48709 > 142.250.21.139.80: Flags [.], ack 775, win 504, options [nop,nop,TS val 13621802 ecr 2601802347], length 0
```
Please note that the `ping` tool does not work with `nr-binder`. Please refer to [here](https://github.com/aligungr/UERANSIM/issues/186#issuecomment-729534464) for the reason.
You could now connect to the DN and send any packets on the network using OAI-CN5G-UPF.

---

Now you could work free5GC with OAI-CN5G-UPF.
I would like to thank the excellent developers and all the contributors of free5GC, OAI-CN5G-UPF and UERANSIM.

<a id="changelog"></a>

## Changelog (summary)

- [2026.09.20] Updated to free5GC v4.2.3 (2026.09.16) and OAI-CN5G-UPF v2.2.1 (2026.09.09).
- [2026.01.23] Rewrote this using OAI-CN5G-UPF built on Ubuntu 24.04.
- [2026.01.18] Initial release.
