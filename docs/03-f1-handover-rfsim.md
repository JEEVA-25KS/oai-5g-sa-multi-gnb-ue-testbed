# 3. F1 Handover in RF Simulator

## Objective
Validate F1-based handover in OAI 5G SA by moving a UE between two DUs under a single CU, using RFsimulator (no physical radio).

## Setup
One CU, two DUs (DU0, DU1) connected over F1. UE initially attaches to DU0; a manual trigger (in place of measurement-based mobility, which the OAI UE doesn't yet support) forces handover to DU1.

**Build with telnet support** (needed since the UE can't trigger handover autonomously):
```bash
cd ~/openairinterface5g/cmake_targets
./build_oai --ninja --nrUE --gNB --build-lib telnetsrv
```

**Run sequence** — critical to start components in this order: DU0 → UE → DU1, since there's no channel emulation and the UE can't decode DU0's SIB1 once DU1 also on-air.

```bash
cd ~/oai-cn5g && docker compose up -d

# DU0 (RFsim server as UE side per this test config)
sudo ./nr-softmodem --rfsim -O .../gnb-du.sa.band78.106prb.rfsim.pci0.conf --rfsimulator.serveraddr server

# UE connects to DU0
sudo ./nr-uesoftmodem --rfsim --rfsimulator.serveraddr <UE-acts-as-server>

# DU1, once UE is connected to DU0
sudo ./nr-softmodem --rfsim -O .../gnb-du.sa.band78.106prb.rfsim.pci1.conf --rfsimulator.serveraddr 127.0.0.1

# Trigger handover
echo ci trigger_f1_ho | nc 127.0.0.1 9090 && echo
```

> Note: RFsimulator supports only one server with multiple clients. Since the UE needs to reach both DUs, the **UE acts as the RFsim server** and both DUs connect as clients — the reverse of the typical gNB-as-server role.

## Result
F1 handover executed successfully: UE context transferred from DU0 to DU1 under CU control, verified via RRC and F1AP signaling (RNTI change observed at handover). **User-plane/PDU-session continuity was not established** — the current implementation validates control-plane handover only.
At CU after triggering handover:
<img width="978" height="181" alt="image" src="https://github.com/user-attachments/assets/2ed5a8d2-baa7-48d8-b893-341262363720" />

The handover command makes the UE to be transferred from DU 0 to DU1, where the UE’s RNTI gets changed.

After handover,
At DU0,
<img width="1089" height="206" alt="image" src="https://github.com/user-attachments/assets/6446dacf-e2d1-4b7f-9843-b956c509af2e" />

At DU1,
<img width="1068" height="119" alt="image" src="https://github.com/user-attachments/assets/c1ca9fda-76e7-472e-a7fd-9d1958131b62" />

At UE terminal,
Before handover,
<img width="732" height="92" alt="image" src="https://github.com/user-attachments/assets/0ae31ef8-416d-461c-84e0-ee9b957770b4" />

After handover, 
<img width="867" height="71" alt="image" src="https://github.com/user-attachments/assets/dbbc9ac5-4849-47f9-89a3-338800a7e7f2" />


## Remarks / gotchas
- Start order matters: DU0 → UE → DU1 (UE must sync to DU0's SIB1 before DU1 comes online)
- "Could not open a socket" / "Could not start the RF device" errors typically mean the RFsim server role is misconfigured — the UE should be the server
- If the whole system blocks with the RFsim server at the UE, stop UE + all DUs and restart (CU can be left running)

## References
- `Handover_tutorial.md` (OAI repo)
