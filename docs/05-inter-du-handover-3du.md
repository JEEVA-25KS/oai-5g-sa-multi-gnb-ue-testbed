# 5. Inter-DU Handover Over F1 — 3 DU Setup (Real RF, COTS UE)

## Objective
Implement and validate F1-based inter-DU handover with a real over-the-air 5G SA network, demonstrating seamless UE mobility across multiple DUs under one CU — this time with a commercial UE, not RFsim.

## Architecture
- 1 CU, 3 DUs (DU1/DU2/DU3), OAI CN5G core, COTS UE (Samsung Galaxy M13 5G)
- CU–DU split over F1: DU handles PHY/MAC, CU handles RRC + mobility control
- 4 machines: core+CU (192.168.230.94), DU1 (192.168.230.72), DU2 (192.168.230.75), DU3 (192.168.230.85)

## gNB neighbour configuration
Handover relies on the CU knowing which cells are neighbours of the serving cell. A `neighbor-config.conf` defines adjacency between DUs (intra-gNB, inter-gNB, and inter-RAT neighbour types are all supported by OAI's model, though this setup only needed inter-DU/intra-gNB).

## Config changes

All the config files ( CU, DU's (3 or 4) and neighbor config file are in the same folder path: `develop/targets/PROJECTS/GENERIC-NR-5GC/CONF/`


**Neighbour configurations** 

<img width="556" height="180" alt="image" src="https://github.com/user-attachments/assets/9050adce-c708-400f-9db4-b2bc6cbd4c9e" />


**CU** — `local_s_address`/`remote_s_address` point at the CU's own IP and a wildcard (since multiple DUs connect in):
```
gNB = ({
  local_s_address = "<CU IP>";
  remote_s_address = "0.0.0.0";
  local_s_portc = 501;  local_s_portd = 2153;
  remote_s_portc = 500; remote_s_portd = 2154;
})
```

**Each DU** — `local_n_address`/`remote_n_address` point at the DU's own IP and the CU's IP:
```
MACRLCs = ({
  num_cc = 1;
  tr_s_preference = "local_L1";
  tr_n_preference = "f1";
  local_n_if_name = "lo";
  local_n_address = "<DU IP>";
  remote_n_address = "<CU IP>";
  local_n_portc = 500;  local_n_portd = 2154;
  remote_n_portc = 501; remote_n_portd = 2153;
  pusch_FailureThres = 1000;
});
```

## Deployment sequence

```bash
# All machines — build with telnet
cd ~/openairinterface5g/cmake_targets
./build_oai -w USRP --ninja --gNB -C --build-lib telnetsrv

# Machine 1 — core + CU

# Terminal 1
cd ~/oai-cn5g && docker compose up -d

# Terminal 2
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O .../gnb-cu.sa.f1.conf --telnetsrv --telnetsrv.shrmod ci

# Machine 2 — DU1

cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O .../gnb-du.sa.band78.106PRB.usrpb210.du-1.conf \
  --gNBs.[0].min_rxtxtime 6 -E --continuous-tx

 # then power on UE and let it connect

# UE
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --ue-fo-compensation -E --uicc0.imsi 001010000000001

# Machine 3 — DU2
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O .../gnb-du.sa.band78.106PRB.usrpb210.du-2.conf \
  --gNBs.[0].min_rxtxtime 6 -E --continuous-tx

# Machine 4 — DU3
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O .../gnb-du.sa.band78.106PRB.usrpb210.du-3.conf \
  --gNBs.[0].min_rxtxtime 6 -E --continuous-tx
```

To populate the neighbour config accurately, use the CU's own RRC log after all DUs are up:
```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
cat nrRRC_stats.log
```
<img width="1091" height="303" alt="image" src="https://github.com/user-attachments/assets/c843de4a-118f-4f4f-b7c3-e5776e8df877" />


## Triggering handover
```bash
echo ci trigger_f1_ho | nc 192.168.230.94 9090 && echo    # In CU machine
```
Each invocation triggers the next handover in sequence (DU1→DU2, then close and reopen the terminal, repeat for DU2→DU3).

### DU 1 ----> DU 2
At CU terminal:
<img width="1157" height="195" alt="image" src="https://github.com/user-attachments/assets/5cf539c1-3cad-4fbe-880e-c809f6b32f85" />

### DU 2 ----> DU 3
At CU terminal:
<img width="1170" height="170" alt="image" src="https://github.com/user-attachments/assets/d2c62623-a504-4694-b326-f74b3fa15ac2" />


## Results
- UE attached to DU1 and held a stable connection
- DU1 → DU2 handover completed with **no session interruption**
- DU2 → DU3 handover subsequently completed successfully under CU control

This confirms F1 inter-DU handover is fully functional in a split OAI gNB deployment for real over-the-air UE mobility across 3 DUs — a step up from the [RFsim, control-plane-only handover](03-f1-handover-rfsim.md) validated earlier.
