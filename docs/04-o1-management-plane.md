# 4. Management-Plane Monitoring in a Monolithic OAI gNB (O1/Telnet)

## Objective
Enable and evaluate management-plane–style monitoring in a monolithic OAI gNB using the embedded Telnet-based O1 interface, and understand its scope relative to full O-RAN-standardized management planes.

## Background
Standard O-RAN management-plane monitoring uses NETCONF/YANG over O1 and Open Fronthaul M-Plane interfaces, with CU/DU/RU monitored independently. In a **monolithic** OAI gNB, CU/DU/RU are one process, so full O1/M-Plane exposure isn't meaningful — monitoring instead relies on OAI's embedded Telnet server with an O1 module.

## Setup

**Build with telnet + O1 support:**
```bash
cd ~/openairinterface5g/cmake_targets
./build_oai -w USRP --ninja --gNB -C --build-lib telnetsrv
```

**gNB config changes** (`gnb.sa.band78.fr1.106PRB.usrpb210.conf`):
```
NETWORK_INTERFACES:
{
    GNB_IPV4_ADDRESS_FOR_NG_AMF = "192.168.70.129/24";
    GNB_IPV4_ADDRESS_FOR_NGU   = "192.168.70.129/24";
    GNB_PORT_FOR_NGU = 2152;  # NG-U (N3) port for SA
    GNB_PORT_FOR_S1U = 2152;
};
```

**Core Docker port fix** (`oai-cn5g/docker-compose.yaml`) — avoids a UDP 2152 collision between the gNB and the UPF container:
```yaml
oai-upf:
  ports:
    - "2154:2152/udp"   # host 2154 -> container 2152, so host 2152 stays free for the gNB
  expose:
    - 2152/udp
    - 8805/udp
```
Root cause: the gNB binds host UDP 2152 directly (`GNB_PORT_FOR_NGU = 2152`); if Docker also tries to grab host 2152 for the UPF, only one process can bind it. Moving Docker's host-side mapping to 2154 (while the UPF still listens on 2152 *inside* the container) removes the collision.

**Run:**
```bash
cd ~/oai-cn5g && docker compose up -d

sudo ./nr-softmodem -O .../gnb.sa.band78.fr1.106PRB.usrpb210.conf \
  -E --continuous-tx --telnetsrv --telnetsrv.shrmod o1
```

## Monitoring

```bash
telnet 127.0.0.1 9090        # interactive: help, o1 stats, exit
echo o1 stats | nc -N 127.0.0.1 9090   # one-shot, for scripts
```

`o1 stats` returns JSON with:
- **Cell/config view** (`O1-config`): BWP, ARFCN, bandwidth, PCI, TAC, PLMN/S-NSSAI
- **Operational KPIs** (`O1-Operational`): UE count, UE RNTIs, per-UE DL/UL throughput, PRB load

Pretty-print with `jq`:
```bash
echo o1 stats | nc -N 127.0.0.1 9090 | awk '/^{$/, /^}$/' | jq .
```

Scripted polling examples:
```bash
# UE count every 5s
echo o1 stats | nc -N 127.0.0.1 9090 | awk '/^{$/, /^}$/' | jq -r '."O1-Operational"."num-ues"'

# Per-UE throughput
echo o1 stats | nc -N 127.0.0.1 9090 | awk '/^{$/, /^}$/' | jq '."O1-Operational"."ues-thp"'

# Log to file every 10s
while true; do
  echo "=== $(date) ===" >> /tmp/o1-stats.log
  echo o1 stats | nc -N 127.0.0.1 9090 | awk '/^{$/, /^}$/' >> /tmp/o1-stats.log
  sleep 10
done
```

## Results
- **Single UE**: cell config, UE count, UE identifier, and real-time DL/UL throughput all correctly reported
- **Two UEs** (two COTS phones, each with a distinct 5G SIM, core DB updated for two subscribers): both detected simultaneously with accurate per-UE operational stats
- OAI reports per-UE throughput in **bits/ms** rather than bits/s, since the MAC scheduler operates on a 1 ms cadence — this unit directly reflects PRB allocation and MCS decisions each TTI

**Conclusion**: basic management-plane-style monitoring is functional for monolithic gNB deployments in both single- and multi-UE scenarios, even without full O-RAN NETCONF/YANG support.

## References
- `common/utils/telnetsrv/DOC/telneto1.md`, `.../telnetusage.md` (OAI repo)
