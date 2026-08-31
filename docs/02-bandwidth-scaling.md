# 2. Changing the Bandwidth of the Testbed Setup

## Objective
Reduce the bandwidth of a one-gNB/one-UE setup by reducing the number of PRBs, and analyze the resulting network connectivity and data rate.

## Setup
- Config: `gnb.sa.band78.fr1.24PRB.usrpb210.conf` (24 PRB variant of the base config)
- File path: `develop/targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.24PRB.usrpb210.conf`

```
plmn_list = ({ mcc = 001; mnc = 01; mnc_length = 2; snssaiList = ({ sst = 1 }) });
```

```bash
# Core
cd ~/oai-cn5g && docker compose up -d

# gNB (24 PRB)
sudo ./nr-softmodem -O .../CONF/gnb.sa.band78.fr1.24PRB.usrpb210.conf \
  --gNBs.[0].min_rxtxtime 6 --continuous-tx --ssb 40

# UE
sudo ./nr-uesoftmodem -r 24 --numerology 1 --band 78 -C 3604320000 --ssb 40 \
  --ue-fo-compensation --uicc0.imsi 001010000000001
```

IP forwarding/NAT and UE tunnel configuration follow the same pattern as the [two-gNB setup](01-two-gnb-two-ue.md).

## Result
With PRBs reduced from 106 → 24, measured throughput dropped to **~5 Mbit/s** — proportionally in line with the reduced allocation, confirming the direct PRB↔throughput relationship.

## Real-time monitoring tools used
- `ifstat -i oaitun_ue1` — simple RX/TX rate readout
- `nload oaitun_ue1` — graphical incoming/outgoing rate view (download-heavy pattern observed, as expected for typical browsing traffic — outgoing traffic stayed near-flat, dominated by ACKs/DNS)
- `iftop -i oaitun_ue1` — per-flow breakdown of traffic by remote host, with TX/RX/TOTAL cumulative and peak rates

## Result summary
Reducing PRBs significantly reduced UE data rate, confirming that PRB allocation directly bounds available bandwidth in the SA setup.
<img width="759" height="176" alt="image" src="https://github.com/user-attachments/assets/32d269e6-0551-4839-aa11-e8c2cd1d6f12" />


## References
- OpenAirInterface5G GitLab, `doc/NR_SA_Tutorial_OAI_CN5G.md`, `doc/NR_SA_Tutorial_OAI_nrUE.md`
- `CONF/gnb.sa.band78.fr1.24PRB.usrpb210.conf`
