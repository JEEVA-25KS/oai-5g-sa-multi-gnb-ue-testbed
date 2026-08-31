# OAI 5G SA Multi-gNB/UE Testbed

Implementation and evaluation of a 5G Standalone (SA) testbed built on
OpenAirInterface (OAI), covering multi-gNB / multi-UE connectivity, bandwidth
scaling, F1-based handover (single-CU/multi-DU), management-plane monitoring,
and 3-DU / 4-DU inter-DU handover over the air.

Built and tested on real USRP B210 SDR hardware (band 78, FR1, 106 PRB /
24 PRB configurations) with OAI CN5G core and NR SA gNB/UE software.

## Contents

| # | Experiment | Key result |
|---|------------|-----------|
| 1 | [Two-gNB, two-UE SA setup](docs/01-two-gnb-two-ue.md) | Simultaneous internet access validated on both UEs with independent IP/NAT config |
| 2 | [Bandwidth scaling (106 → 24 PRB)](docs/02-bandwidth-scaling.md) | PRB reduction to 24 PRBs limited throughput to ~5 Mbps, confirming proportional PRB↔throughput relationship |
| 3 | [F1 handover in RF simulator](docs/03-f1-handover-rfsim.md) | Control-plane handover (RRC/F1AP) verified between DUs under one CU; user-plane continuity not yet supported |
| 4 | [Management-plane monitoring (O1/Telnet)](docs/04-o1-management-plane.md) | Real-time cell config, UE count, per-UE DL/UL throughput visibility in monolithic gNB via O1 Telnet interface |
| 5 | [Inter-DU handover — 3 DU setup](docs/05-inter-du-handover-3du.md) | Seamless UE handover across 3 DUs under a single CU, no service interruption |
| 6 | [Inter-DU handover — 4 DU setup](docs/06-inter-du-handover-4du.md) | Extended to 4 DUs over the air with USRPs; confirmed continued mobility management at scale |

## Testbed hardware/software

- **RF hardware**: 2× USRP B210 (serial-identified via `uhd_find_devices`)
- **RAN**: OpenAirInterface 5G NR SA gNB/UE (`nr-softmodem`, `nr-uesoftmodem`), band 78, FR1
- **Core**: OAI CN5G (Docker Compose deployment)
- **Config baseline**: `gnb.sa.band78.fr1.106PRB.usrpb210.conf` (and a 24 PRB variant for bandwidth-reduction tests)

## Common setup pattern

Each experiment follows the same general flow:
1. Duplicate/modify the base gNB config file per gNB or DU (unique `gNB_ID`, IPs, cell ID)
2. Bring up the OAI CN5G core via `docker compose up -d`
3. Launch gNB(s)/DU(s)/CU via `nr-softmodem`, then UE(s) via `nr-uesoftmodem`
4. Configure IP forwarding + NAT on the core host for UE internet access
5. Validate connectivity via the `oaitun_ue*` tunnel interface and measure throughput (`speedtest`, `iperf3`)

See [`docs/`](docs/) for the full per-experiment writeups including configs, exact commands, and results, and [`configs/`](configs/) for reference config snippets.

## Known limitations (as observed)

- Limited SDR range occasionally causes both UEs to associate with the same gNB
- PDU session establishment can intermittently fail during initial UE registration
- F1 handover (RFsim) currently validates control-plane only — user-plane/PDU session continuity across DUs is not yet maintained
- The RFsimulator's single-server constraint requires the UE (not the DU) to act as the RFsim server when a UE must reach multiple DUs

## References

- [OAI CN5G](https://github.com/OPENAIRINTERFACE/oai-cn5g-fed)
- [OpenAirInterface5G GitLab](https://gitlab.eurecom.fr/oai/openairinterface5g)
- `doc/NR_SA_Tutorial_OAI_CN5G.md`, `doc/NR_SA_Tutorial_OAI_nrUE.md` (OAI repo docs)
