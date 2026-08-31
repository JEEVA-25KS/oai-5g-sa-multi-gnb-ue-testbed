# 6. Inter-DU Handover Over F1 — 4 DU Setup (Real RF, COTS UE)

## Objective
Extend the [3-DU inter-DU handover](05-inter-du-handover-3du.md) work to 4 DUs, further validating seamless UE mobility across a larger split-gNB deployment.

## Architecture
- 1 CU, 4 DUs (DU1–DU4), OAI CN5G core, COTS UE (Samsung Galaxy M13 5G)
- UE starts on DU3; handover proceeds round-robin (DU3 → DU0/DU1 → DU2 → ...)
- Only **2 physical machines** this time: machine 1 (192.168.230.107) runs core + CU + 3 DUs, machine 2 (192.168.230.106) runs the 4th DU — since 3 DUs share a host, each needs a distinct identity:

| DU | PCI | TAC | nr_cellid |
|----|-----|-----|-----------|
| DU1 | 0 | 1 | 12345678L |
| DU2 | 1 | 2 | 12345679L |
| DU3 | 2 | 3 | 12345680L |
| DU4 | 3 | 4 | 12345689L |

<img width="1351" height="555" alt="image" src="https://github.com/user-attachments/assets/3489ce6c-c780-4720-b960-d87a9edc6e2e" />


## Config pattern
Same CU/DU `local_s_address`/`remote_s_address` and `local_n_address`/`remote_n_address` pattern as the [3-DU setup](05-inter-du-handover-3du.md#config-changes), applied across 4 DU config files, each with a unique PCI/TAC/cell ID from the table above.

## Deployment

```bash
# All machines — build with telnet
cd ~/openairinterface5g/cmake_targets
./build_oai -w USRP --ninja --gNB -C --build-lib telnetsrv

# Machine 1 — core, CU, DU1-DU3 (logged with tee for post-hoc analysis)
# Terminal 1
cd ~/oai-cn5g && docker compose up -d

# Terminal 2
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O .../gnb-cu.sa.f1.conf --telnetsrv --telnetsrv.shrmod ci \
  --log_config.global_log_options level,nocolor 2>&1 | tee $HOME/cu.log



# Terminal 3
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O .../gnb-du.sa.band78.106PRB.usrpb210.du-1.conf \
  --gNBs.[0].min_rxtxtime 6 -E --continuous-tx \
  --log_config.global_log_options level,nocolor 2>&1 | tee $HOME/du1.log


# ...same pattern for du-2.conf, du-3.conf

# Machine 2 — DU4
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O .../gnb-du.sa.band78.106PRB.usrpb210.du-4.conf \
  --gNBs.[0].min_rxtxtime 6 -E --continuous-tx \
  --log_config.global_log_options level,nocolor 2>&1 | tee $HOME/du4.log
```

Neighbour config populated the same way — from the CU's `nrRRC_stats.log` once all DUs are up.

```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
cat nrRRC_stats.log
```
<img width="780" height="404" alt="image" src="https://github.com/user-attachments/assets/4e76ba07-9a1c-45fe-b39f-33592e5873e3" />


## Triggering handover
```bash
echo ci trigger_f1_ho | nc 192.168.230.107 9090 && echo
```
First trigger moves the UE DU4 → DU1; subsequent triggers (after closing/reopening the terminal) continue the round-robin.

## Results
Experimental results confirmed **seamless UE handover across all four DUs without service interruption**, validating that F1 inter-DU handover and centralized CU-based mobility management scale beyond 3 DUs, including in a mixed topology where multiple DUs are co-located on the same physical machine and one is remote.

## Key takeaway across both handover experiments
Centralizing RRC/mobility control at the CU while distributing PHY/MAC to DUs (per the O-RAN split architecture) lets a single CU manage handover across an arbitrary number of DUs, provided:
1. Each DU has a unique PCI/TAC/cell ID
2. The neighbour config accurately reflects DU adjacency (verifiable from `nrRRC_stats.log`)
3. F1 transport addressing (`local_n_address`/`remote_n_address`, ports) is correctly set per DU
