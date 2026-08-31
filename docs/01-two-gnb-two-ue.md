# 1. Two-gNB, Two-UE SA Testbed

## Objective
Set up a two-gNB, two-UE configuration and establish internet connectivity at both UEs simultaneously.

## Setup

**1. Identify USRP serial numbers**
```bash
sudo uhd_find_devices
```
```
Device 0 -> gNB1, serial: 30AD395, product: B210
Device 1 -> gNB2, serial: 30DBC30, product: B210
```

**2. Duplicate and modify gNB configs**

Base config: `gnb.sa.band78.fr1.106PRB.usrpb210.conf`

```bash
cd ~/openairinterface5g/targets/PROJECTS/GENERIC-NR-5GC/CONF/
cp gnb.sa.band78.fr1.106PRB.usrpb210.conf gnb1.conf
cp gnb.sa.band78.fr1.106PRB.usrpb210.conf gnb2.conf
```

Key fields changed per gNB (gNB1 shown; gNB2 uses `0xe02`, `192.168.70.128`, serial `30DBC30`):

```
Active_gNBs = ("gNB-OAI-1");
gNBs = ( gNB_ID = 0xe01; gNB_name = "gNB-OAI-1"; min_rxtxtime = 6;
         tracking_area_code = 1; nr_cellid = 12345678L; )

NETWORK_INTERFACES:
{
    GNB_IPV4_ADDRESS_FOR_NG_AMF = "192.168.70.129/24";
    GNB_IPV4_ADDRESS_FOR_NGU   = "192.168.70.129/24";
    GNB_PORT_FOR_S1U = 2152;
};

RUs = ({ local_rf = "yes"; nb_tx = 1; nb_rx = 1; att_tx = 12; att_rx = 12;
         bands = [78]; max_pdschReferenceSignalPower = -27; max_rxgain = 114;
         eNB_instances = [0]; clock_src = "internal"; sdr_addrs = "serial=30AD395"; });
```

> ⚠️ Uplink/downlink ARFCN frequencies are left unchanged when copying configs — watch for frequency overlap between the two gNBs since they derive from the same base file. Also confirm no IP collides with existing containers in the core's `docker-compose.yaml`.

**3. Launch core, gNBs, UEs**
```bash
# Core
cd ~/oai-cn5g && docker compose up -d

# gNB1 / gNB2 (separate terminals)
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb1.conf \
  --gNBs.[0].min_rxtxtime 6 -E --continuous-tx

# UE1 / UE2 (separate machines)
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 \
  --ue-fo-compensation -E --uicc0.imsi 001010000000001   # UE1
# UE2 uses imsi 001010000000002
```

**4. Multiple IPs on the host interface** (only if SCTP fails to bind)
```bash
sudo ip addr add 192.168.70.129/24 dev eno1   # gNB1
sudo ip addr add 192.168.70.128/24 dev eno1   # gNB2
sudo ip addr add 192.168.70.132/24 dev eno1   # Core/AMF
```
This is needed because a gNB can't bind SCTP/GTP-U to an IP that isn't assigned to any local interface — assigning it manually fixes the bind failure.

**5. Verify tunnel + enable internet access**
```bash
ip a | grep oaitun   # expect: oaitun_ue1: ... inet 10.0.0.2/24 ...
```
At the core host:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eno1 -j MASQUERADE
sudo iptables -A FORWARD -i eno1 -o oai-cn5g -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i oai-cn5g -o eno1 -j ACCEPT
```
At each UE:
```bash
sudo ip addr flush dev oaitun_ue1
sudo ip addr add 10.0.0.X/24 dev oaitun_ue1
sudo ip link set oaitun_ue1 up
sudo ip route add default via 10.0.0.1 dev oaitun_ue1
sudo resolvectl dns oaitun_ue1 8.8.8.8
sudo resolvectl domain oaitun_ue1 "~."
sudo ip link set dev oaitun_ue1 mtu 1400   # account for GTP overhead
```

## Result
Speedtest confirmed simultaneous, independent internet connectivity on both UEs through their respective gNBs.

<img width="622" height="344" alt="image" src="https://github.com/user-attachments/assets/b443121e-aeaf-4ad7-b803-73095152a900" />


## Limitations observed
- Limited SDR range occasionally causes both UEs to associate with the same gNB
- PDU session sometimes fails to establish on initial UE registration
- DNS ping may show no logs even with a working internet connection
- gNB terminal logs can become hard to parse when both gNBs run simultaneously

## References
- OAI CN5G, OpenAirInterface5G GitLab
- `doc/NR_SA_Tutorial_OAI_CN5G.md`, `doc/NR_SA_Tutorial_OAI_nrUE.md`
