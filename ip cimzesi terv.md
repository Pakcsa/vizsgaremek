# IPv4 és IPv6 Címzési Terv — NetCore Solutions Kft.

**Projekt:** vizsgaremek-3site  
**Készítette:** Kállay Botond Zoltán & Kiss Dominik  
**Címzési architektúra:** Hierarchikus IPv4 (VLSM) és Dual-Stack IPv6 (ULA / Global Unicast szimuláció)

---

## 1. Alhálózatok és VLAN-ok áttekintése

A vállalati hálózat telephelyenként (HQ = `10.1.x.x`, Branch B = `10.2.x.x`, Branch C = `10.3.x.x`) és funkciónként szegmentált. A pont-pont (P2P) WAN összeköttetések `/30`-as (IPv4) és `/64`-es (IPv6) alhálózatokat kaptak az IP-címek hatékony kihasználása érdekében.

| Telephely / Szegmens | VLAN ID / Név | IPv4 Alhálózat | Maszk | IPv6 Előtag (Prefix) | Átjáró (IPv4 / IPv6) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Internet / ISP WAN** | — | `203.0.113.0/29` | `255.255.255.248` | `2001:db8:acad:1::/64` | `203.0.113.1` / `2001:db8:acad:1::1` |
| **HQ ↔ Branch B WAN (OSPF + VPN)** | — | `172.16.0.0/30` | `255.255.255.252` | `fd00:ffff:1::/64` | P2P összeköttetés |
| **HQ ↔ Branch C WAN (Statikus)** | — | `172.16.0.4/30` | `255.255.255.252` | `fd00:ffff:2::/64` | P2P összeköttetés |
| **HQ ASA ↔ HQ-RTR (Transit)** | — | `192.168.254.0/30` | `255.255.255.252` | — *(IPv4-only perem)* | `192.168.254.1` |
| **HQ Core ↔ HQ-ASA (Inside)** | — | `192.168.254.4/30` | `255.255.255.252` | — *(IPv4-only perem)* | `192.168.254.5` |
| **HQ Menedzsment** | `VLAN 10 (MGMT)` | `10.1.10.0/24` | `255.255.255.0` | — | `10.1.10.1` *(HSRP VIP)* |
| **HQ Dolgozók (Vezetékes)** | `VLAN 20 (STAFF)` | `10.1.20.0/24` | `255.255.255.0` | — | `10.1.20.1` *(HSRP VIP)* |
| **HQ Szerverpark** | `VLAN 30 (SERVERS)` | `10.1.30.0/24` | `255.255.255.0` | — | `10.1.30.1` *(HSRP VIP)* |
| **HQ Wi-Fi WAN (HQ-RTR ↔ WRT300N)** | — | `10.1.40.0/30` | `255.255.255.252` | — | `10.1.40.1` |
| **HQ Wi-Fi Belső LAN** | `WLAN (HQ-Corp-WiFi)` | `192.168.40.0/24` | `255.255.255.0` | — | `192.168.40.1` *(WRT300N)* |
| **Branch B Helyi hálózat** | `VLAN 1 (LAN)` | `10.2.20.0/24` | `255.255.255.0` | `fd00:2:20::/64` | `10.2.20.1` / `fd00:2:20::1` |
| **Branch B Wi-Fi WAN** | — | `10.2.40.0/30` | `255.255.255.252` | — | `10.2.40.1` |
| **Branch B Wi-Fi Belső LAN** | `WLAN (BranchB-WiFi)` | `192.168.41.0/24` | `255.255.255.0` | — | `192.168.41.1` *(WRT300N)* |
| **Branch C Helyi hálózat** | `VLAN 1 (LAN)` | `10.3.20.0/24` | `255.255.255.0` | `fd00:3:20::/64` | `10.3.20.1` / `fd00:3:20::1` |

---

## 2. Eszközök részletes interfész- és IP-kiosztása

### 2.1. Szolgáltatói (ISP) és WAN peremhálózatok

| Eszköz neve | Interfész | Kapcsolódó eszköz | IPv4 cím / Maszk | IPv6 cím / Prefix | Megjegyzés |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **ISP-RTR** | `Gi0/0/0` | `HQ-RTR (Gi0/0/0)` | `203.0.113.1 /29` | `2001:db8:acad:1::1 /64` | Publikus internet átjáró |
| **HQ-RTR** | `Gi0/0/0` | `ISP-RTR (Gi0/0/0)` | `203.0.113.2 /29` | `2001:db8:acad:1::2 /64` | NAT outside interfész + `203.0.113.3` statikus NAT |
| **HQ-RTR** | `Gi0/1/0` | `BRB-RTR (Gi0/0/0)` | `172.16.0.1 /30` | `fd00:ffff:1::1 /64` | OSPF Area 0 + IPsec VPN crypto map |
| **HQ-RTR** | `Gi0/2/0` | `BRC-RTR (Gi0/0/0)` | `172.16.0.5 /30` | `fd00:ffff:2::1 /64` | Statikus WAN kapcsolat Branch C felé |
| **HQ-RTR** | `Gi0/0` | `HQ-ASA (Gi1/1)` | `192.168.254.1 /30` | — | NAT inside, kapcsolat a tűzfal felé |
| **HQ-RTR** | `Gi0/1` | `HQ-WIFI (Internet)` | `10.1.40.1 /30` | — | Vezeték nélküli router uplink |

---

### 2.2. Budapesti Székhely (HQ) — Biztonság, Core, Access és Szerverek

| Eszköz neve | Interfész / SVI | Kapcsolódó eszköz / VLAN | IPv4 cím / Maszk | Alapértelmezett átjáró | Szerepkör / Megjegyzés |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **HQ-ASA** | `Gi1/1` *(outside)* | `HQ-RTR (Gi0/0)` | `192.168.254.2 /30` | `192.168.254.1` | Security-level 0, `outside-in` ACL |
| **HQ-ASA** | `Gi1/2` *(inside)* | `HQ-SW1 (Gi0/1)` | `192.168.254.5 /30` | — | Security-level 100 |
| **HQ-SW1** | `Gi0/1` *(routed)* | `HQ-ASA (Gi1/2)` | `192.168.254.6 /30` | `192.168.254.5` | Core uplink a tűzfal felé |
| **HQ-SW1** | `SVI Vlan 10` | `VLAN 10 (MGMT)` | `10.1.10.2 /24` | `10.1.10.1` *(VIP)* | HSRP Active (Priority 110, Preempt) |
| **HQ-SW1** | `SVI Vlan 20` | `VLAN 20 (STAFF)` | `10.1.20.2 /24` | `10.1.20.1` *(VIP)* | HSRP Active (Priority 110, Preempt) |
| **HQ-SW1** | `SVI Vlan 30` | `VLAN 30 (SERVERS)` | `10.1.30.2 /24` | `10.1.30.1` *(VIP)* | HSRP Active (Priority 110, Preempt) |
| **HQ-SW2** | `SVI Vlan 10` | `VLAN 10 (MGMT)` | `10.1.10.3 /24` | `10.1.10.1` *(VIP)* | HSRP Standby (Priority 100) |
| **HQ-SW2** | `SVI Vlan 20` | `VLAN 20 (STAFF)` | `10.1.20.3 /24` | `10.1.20.1` *(VIP)* | HSRP Standby (Priority 100) |
| **HQ-SW2** | `SVI Vlan 30` | `VLAN 30 (SERVERS)` | `10.1.30.3 /24` | `10.1.30.1` *(VIP)* | HSRP Standby (Priority 100) |
| **HQ-ACC-SW** | `SVI Vlan 10` | `VLAN 10 (MGMT)` | `10.1.10.10 /24` | `10.1.10.1` | Access switch menedzsment IP |
| **WIN-SRV** | `NIC (Fa0/1)` | `VLAN 30 (SERVERS)` | `10.1.30.10 /24` | `10.1.30.1` | AD, DNS (`corp.local`), DHCP szerver |
| **LIN-SRV** | `NIC (Fa0/2)` | `VLAN 30 (SERVERS)` | `10.1.30.11 /24` | `10.1.30.1` | Web (HTTP/HTTPS), Samba, TFTP (`203.0.113.3`) |
| **PC-HQ-1** | `NIC (Fa0/3)` | `VLAN 20 (STAFF)` | `10.1.20.10 /24` *(vagy DHCP)* | `10.1.20.1` | DNS: `10.1.30.10` |
| **PC-HQ-2** | `NIC (Fa0/4)` | `VLAN 20 (STAFF)` | `10.1.20.11 /24` *(vagy DHCP)* | `10.1.20.1` | DNS: `10.1.30.10` |
| **PRT-HQ** | `NIC (Fa0/5)` | `VLAN 20 (STAFF)` | `10.1.20.20 /24` | `10.1.20.1` | Hálózati nyomtató (fix IP) |
| **HQ-WIFI** | `Internet / LAN` | `HQ-RTR` / `WLAN` | WAN: `10.1.40.2/30` · LAN: `192.168.40.1/24` | `10.1.40.1` | SSID: `HQ-Corp-WiFi` (WPA2-PSK) |
| **LAP-HQ-WIFI** | `Wireless0` | `HQ-Corp-WiFi` | `192.168.40.100–150 /24` *(DHCP)* | `192.168.40.1` | Vezeték nélküli laptop kliens |
| **PHONE-HQ-WIFI** | `Wireless0` | `HQ-Corp-WiFi` | `192.168.40.100–150 /24` *(DHCP)* | `192.168.40.1` | Vezeték nélküli okostelefon kliens |

---

### 2.3. Debreceni Fejlesztői Iroda (Branch B)

| Eszköz neve | Interfész | Kapcsolódó eszköz | IPv4 cím / Maszk | IPv6 cím / Prefix | Átjáró / Megjegyzés |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **BRB-RTR** | `Gi0/0/0` | `HQ-RTR (Gi0/1/0)` | `172.16.0.2 /30` | `fd00:ffff:1::2 /64` | OSPF + IPsec VPN végpont |
| **BRB-RTR** | `Gi0/0` | `BRB-SW (Gi0/1)` | `10.2.20.1 /24` | `fd00:2:20::1 /64` | Fejlesztői LAN alapértelmezett átjáró |
| **BRB-RTR** | `Gi0/1` | `BRB-WIFI (Internet)` | `10.2.40.1 /30` | — | Wi-Fi router uplink |
| **BRB-SW** | `SVI Vlan 1` | `LAN` | `10.2.20.2 /24` | — | Switch menedzsment cím (`GW: 10.2.20.1`) |
| **PC-BRB-1** | `NIC (Fa0/1)` | `BRB-SW (Fa0/1)` | `10.2.20.10 /24` | `fd00:2:20::10 /64` | Fejlesztői munkaállomás (`GW: 10.2.20.1`) |
| **BRB-WIFI** | `Internet / LAN` | `BRB-RTR` / `WLAN` | WAN: `10.2.40.2/30` · LAN: `192.168.41.1/24` | — | SSID: `BranchB-WiFi` (WPA2-PSK) |
| **LAP-BRB-WIFI** | `Wireless0` | `BranchB-WiFi` | `192.168.41.100–150 /24` *(DHCP)* | — | Vezeték nélküli fejlesztői laptop |

---

### 2.4. Szegedi Értékesítési Képviselet (Branch C)

| Eszköz neve | Interfész | Kapcsolódó eszköz | IPv4 cím / Maszk | IPv6 cím / Prefix | Átjáró / Megjegyzés |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **BRC-RTR** | `Gi0/0/0` | `HQ-RTR (Gi0/2/0)` | `172.16.0.6 /30` | `fd00:ffff:2::2 /64` | Statikus alapértelmezett útvonal a HQ felé |
| **BRC-RTR** | `Gi0/0` | `BRC-SW (Gi0/1)` | `10.3.20.1 /24` | `fd00:3:20::1 /64` | Értékesítési LAN alapértelmezett átjáró |
| **BRC-SW** | `SVI Vlan 1` | `LAN` | `10.3.20.2 /24` | — | Switch menedzsment cím (`GW: 10.3.20.1`) |
| **PC-BRC-1** | `NIC (Fa0/1)` | `BRC-SW (Fa0/1)` | `10.3.20.10 /24` | `fd00:3:20::10 /64` | Értékesítői munkaállomás 1 (`GW: 10.3.20.1`) |
| **PC-BRC-2** | `NIC (Fa0/2)` | `BRC-SW (Fa0/2)` | `10.3.20.11 /24` | `fd00:3:20::11 /64` | Értékesítői munkaállomás 2 (`GW: 10.3.20.1`) |

---

## 3. DHCP és NAT/PAT Címfordítási Szabályok

* **WIN-SRV DHCP tartományok (HQ):**
  * `VLAN 10 (MGMT)` pool: `10.1.10.100 – 10.1.10.150 /24` (Átjáró: `10.1.10.1`, DNS: `10.1.30.10`)
  * `VLAN 20 (STAFF)` pool: `10.1.20.100 – 10.1.20.150 /24` (Átjáró: `10.1.20.1`, DNS: `10.1.30.10`)
* **Dinamikus PAT (Overload) a `HQ-RTR`-en:**
  * Minden belső vállalati IPv4 forgalom (`10.1.0.0/16`, `10.2.0.0/16`, `10.3.0.0/16`) az internet (`ISP-RTR`) felé a `203.0.113.2` publikus IP-címre fordítódik.
* **Statikus NAT a `HQ-RTR`-en:**
  * Belső helyi cím: `10.1.30.11` (`LIN-SRV`) $\leftrightarrow$ Külső globális cím: `203.0.113.3` (HTTP/HTTPS webszerver elérése az internet felől).
* **Tervezési megjegyzés (IPv6 perem):**
  * A WAN gerinchálózaton és a fiókirodákban (Branch B, Branch C) teljes IPv4/IPv6 *dual-stack* működik. A HQ belső hálózata és a `HQ-ASA` tűzfal közötti szegmens jelenlegi fázisban IPv4-only kialakítású, amely lehetővé teszi a NAT/PAT és a meglévő szerverinfrastruktúra egységes szűrését, miközben a peremrouterek már készen állnak a teljes IPv6-os átállásra.