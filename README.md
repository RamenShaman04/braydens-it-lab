# CCNA Mega Lab: Three-Tier Enterprise Network (Cisco Packet Tracer)

A full build of a two-office, three-tier (Access / Distribution / Core) enterprise network in Cisco Packet Tracer. I completed Jeremy's IT Lab's **CCNA Mega Lab** and documented every task with the exact commands and the reasoning behind them, so this repo shows how I configure and verify a network, not only that I passed the exam.

> **Credit:** Lab design by [Jeremy's IT Lab](https://www.youtube.com/@JeremysITLab). The configuration, documentation, and verification here are my own work.

**Author:** [Your Name] | **Certification:** CCNA | **Tool:** Cisco Packet Tracer | [LinkedIn](https://www.linkedin.com/in/your-profile)

---

## Topology

![Network topology](<img width="844" height="540" alt="image" src="https://github.com/user-attachments/assets/78a77103-2a2d-474c-b913-4ec7124d20d4" />
)

- **Office A** and **Office B**, each with two Distribution switches and three Access switches
- **Core:** two Layer 3 switches (CSW1, CSW2) in an EtherChannel
- **Edge:** R1 with dual ISP uplinks (primary + floating backup)
- **Services:** SRV1 (DNS, Syslog, FTP) and WLC1 with two lightweight APs

## Skills Demonstrated

| Area | What I configured |
|---|---|
| Switching | VLANs, 802.1Q trunking, DTP disabled, native VLAN hardening, VTPv2 |
| Link aggregation | Layer 2 EtherChannel (PAgP and LACP), Layer 3 EtherChannel |
| Redundancy | HSRPv2 with per-VLAN active/standby load balancing, preemption |
| Spanning tree | Rapid PVST+, root bridge aligned with HSRP active router, PortFast, BPDU Guard |
| Routing | Single-area OSPF, passive interfaces, point-to-point networks, recursive and floating static routes, default route origination |
| IP services | DHCP server and relay, DNS, NTP with authentication, SNMP, Syslog, FTP IOS upgrade |
| Security | SSH v2 with VTY ACL, extended ACLs, port security, DHCP Snooping, Dynamic ARP Inspection, type 9 secrets, disabling unused ports |
| NAT | Static NAT and pool-based PAT |
| IPv6 | Dual-stack addressing, EUI-64, IPv6 default routes |
| Wireless | WLC configuration, dynamic interface, WPA2-PSK WLAN, AP association |

## Repository Structure

```
.
├── README.md
├── CCNA-Mega-Lab.pkt        # Packet Tracer file (open with Cisco Packet Tracer)
├── images/                  # Topology and verification screenshots
├── configs/                 # Final running-configs (one .txt per device)
└── docs/
    └── walkthrough.md       # Full task-by-task write-up
```

## IP Addressing Summary

### VLANs and HSRP

| Office | VLAN | Name | Subnet | HSRP VIP | HSRP group | Active / Root |
|---|---|---|---|---|---|---|
| A | 10 | PCs | 10.1.0.0/24 | 10.1.0.1 | 2 | DSW-A1 |
| A | 20 | Phones | 10.2.0.0/24 | 10.2.0.1 | 3 | DSW-A2 |
| A | 40 | Wi-Fi | 10.6.0.0/24 | 10.6.0.1 | 4 | DSW-A2 |
| A | 99 | Management | 10.0.0.0/28 | 10.0.0.1 | 1 | DSW-A1 |
| B | 10 | PCs | 10.3.0.0/24 | 10.3.0.1 | 2 | DSW-B1 |
| B | 20 | Phones | 10.4.0.0/24 | 10.4.0.1 | 3 | DSW-B2 |
| B | 30 | Servers | 10.5.0.0/24 | 10.5.0.1 | 4 | DSW-B2 |
| B | 99 | Management | 10.0.0.16/28 | 10.0.0.17 | 1 | DSW-B1 |

### Loopbacks (OSPF router IDs)

| Device | Loopback0 |
|---|---|
| R1 | 10.0.0.76/32 |
| CSW1 | 10.0.0.77/32 |
| CSW2 | 10.0.0.78/32 |
| DSW-A1 | 10.0.0.79/32 |
| DSW-A2 | 10.0.0.80/32 |
| DSW-B1 | 10.0.0.81/32 |
| DSW-B2 | 10.0.0.82/32 |

---

## Walkthrough Highlights

The complete step-by-step log is in [`docs/walkthrough.md`](docs/walkthrough.md). Below are the key configurations from each part.

### Part 1: Initial Setup

Hostnames, enable secrets, a local user, and a hardened console line. I used **type 9 (scrypt)** hashing on devices that support it and fell back to **type 5 (MD5)** on those that don't.

```
enable algorithm-type scrypt secret <password>   ! type 9 where supported
enable secret <password>                          ! type 5 fallback
username <user> algorithm-type scrypt secret <password>

line console 0
 login local
 exec-timeout 30 0
 logging synchronous
```

**Why:** Stronger hashing protects credentials if a config is leaked, and a 30-minute timeout stops an unattended console session from staying open.

### Part 2: VLANs and Layer 2 EtherChannel

- Office A uses **PAgP** (`desirable`), Office B uses **LACP** (`active`).
- All Access-to-Distribution links are static trunks with DTP disabled and an unused native VLAN.
- Distribution switches act as VTPv2 servers, Access switches as clients.
- Unused ports are administratively shut down.

```
interface range g1/0/1-2
 channel-group 1 mode desirable        ! PAgP (Office A); use "active" for LACP (Office B)

interface port-channel 1
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,40,99
```

**Why:** Splitting traffic into VLANs reduces broadcast domains and keeps guest Wi-Fi isolated from management. Moving the native VLAN off VLAN 1 and disabling DTP closes off VLAN-hopping tricks.

### Part 3: Layer 3 Addressing and HSRP

Routed point-to-point /30 links, a Layer 3 EtherChannel between the core switches, and HSRPv2 on every Distribution SVI. Active routers are split per VLAN (priority 105 plus `preempt`) so both distribution switches carry traffic.

```
ip routing

interface vlan 10
 ip address 10.1.0.2 255.255.255.0
 standby version 2
 standby 2 ip 10.1.0.1
 standby 2 priority 105
 standby 2 preempt
```

**Why:** HSRP gives hosts one virtual gateway that survives a switch failure. Alternating the active role per VLAN uses both switches instead of leaving one idle.

### Part 4: Rapid PVST+

Root bridges were aligned with the HSRP active router for each VLAN so Layer 2 and Layer 3 forwarding paths match.

```
spanning-tree mode rapid-pvst
spanning-tree vlan 10,99 priority 0        ! DSW-A1 (root, HSRP active)
spanning-tree vlan 20,40 priority 4096     ! DSW-A1 (secondary)

interface f0/1
 spanning-tree portfast
 spanning-tree bpduguard enable
```

**Why:** If STP and HSRP disagree, traffic takes a longer path across the inter-switch link. PortFast plus BPDU Guard lets hosts come up instantly while blocking rogue switches.

### Part 5: Routing

OSPF process 1, area 0, router IDs set to loopbacks, `network` statements matching exact interface addresses on switches, and point-to-point network type on all physical links (no DR/BDR election). R1 originates a default route with a recursive primary and a floating backup (AD 2).

```
router ospf 1
 router-id 10.0.0.79
 passive-interface Loopback0
 passive-interface Vlan10
 network 10.0.0.46 0.0.0.0 area 0

interface range g1/1/1-2
 ip ospf network point-to-point

! On R1
ip route 0.0.0.0 0.0.0.0 203.0.113.1          ! primary ISP (recursive)
ip route 0.0.0.0 0.0.0.0 203.0.113.5 2        ! floating backup ISP
router ospf 1
 default-information originate
```

**Why:** Passive interfaces stop OSPF hellos from leaking toward hosts. A floating static route stays out of the routing table until the primary path fails.

### Part 6: Network Services

DHCP on R1 with relay on the Distribution switches, DNS on SRV1, NTP with MD5 authentication, SNMP read-only, Syslog, an IOS upgrade over FTP, SSH v2 locked down with an ACL, static NAT for SRV1, and PAT for user subnets.

```
! DHCP relay (Distribution SVIs)
interface vlan 10
 ip helper-address 10.0.0.76

! NTP authentication
ntp authentication-key 1 md5 <key>
ntp trusted-key 1
ntp server 10.0.0.76 key 1

! SSH hardening
ip domain-name jeremysitlab.com
crypto key generate rsa                        ! largest modulus supported
ip ssh version 2
access-list 1 permit 10.1.0.0 0.0.0.255
line vty 0 15
 access-class 1 in
 login local
 transport input ssh

! NAT on R1
ip nat inside source static 10.5.0.4 203.0.113.113
access-list 2 permit 10.1.0.0 0.0.0.255
ip nat pool POOL1 203.0.113.200 203.0.113.207 netmask 255.255.255.248
ip nat inside source list 2 pool POOL1 overload
```

**Why:** DHCP relay lets one server serve many subnets. Restricting VTY access to one subnet and allowing only SSH removes Telnet and limits who can even attempt a login. `overload` is what turns NAT into PAT.

### Part 7: Security

An extended ACL blocks everything except ICMP from Office A PCs to Office B PCs, applied **inbound on the interface closest to the source**. Layer 2 protections stack on the Access switches.

```
ip access-list extended OfficeA_to_OfficeB
 permit icmp 10.1.0.0 0.0.0.255 10.3.0.0 0.0.0.255
 deny ip 10.1.0.0 0.0.0.255 10.3.0.0 0.0.0.255
 permit ip any any

! Port security
switchport port-security
switchport port-security violation restrict
switchport port-security mac-address sticky

! DHCP Snooping and DAI
ip dhcp snooping
ip dhcp snooping vlan 10,20,40,99
no ip dhcp snooping information option
ip arp inspection vlan 10,20,40,99
ip arp inspection validate dst-mac src-mac ip
```

**Why:** Extended ACLs go near the source to drop unwanted traffic early; standard ACLs go near the destination. DHCP Snooping builds the binding table that DAI uses to stop ARP spoofing.

### Part 8: IPv6

```
ipv6 unicast-routing

interface g0/0/0
 ipv6 address 2001:db8:a::2/64
interface g0/0
 ipv6 address 2001:db8:a1::/64 eui-64
interface port-channel 1
 ipv6 enable

ipv6 route ::/0 2001:db8:a::1                      ! recursive
ipv6 route ::/0 g0/1/0 2001:db8:b::1 2             ! fully specified, floating
```

### Part 9: Wireless

Configured the WLC from its web GUI: a dynamic interface on VLAN 40 and a WPA2-PSK (AES) WLAN, then confirmed both LWAPs joined the controller.

> **Note:** The video's WLC dynamic interface address (10.6.0.2) duplicates DSW-A1's VLAN 40 SVI. I used **10.6.0.4** instead to avoid the conflict.

---

## Verification

Screenshots and output proving the network works.

| Check | Command | Screenshot |
|---|---|---|
| VTP domain joined | `show vtp status` | `images/verify-vtp.png` |
| EtherChannels up | `show etherchannel summary` | `images/verify-etherchannel.png` |
| HSRP roles | `show standby brief` | `images/verify-hsrp.png` |
| STP roots | `show spanning-tree summary` | `images/verify-stp.png` |
| OSPF neighbors | `show ip ospf neighbor` | `images/verify-ospf.png` |
| Routing table | `show ip route` | `images/verify-routes.png` |
| DHCP leases | `show ip dhcp binding` | `images/verify-dhcp.png` |
| NAT translations | `show ip nat translations` | `images/verify-nat.png` |
| Internet reachability | `ping jeremysitlab.com` from a PC | `images/verify-ping.png` |
| Wireless APs joined | WLC > Wireless tab | `images/verify-wlc.png` |

## What I Learned

- **Documentation habits matter.** Writing the *why* next to each command exposed gaps in my understanding that exam questions never did.
- **Layer 2 and Layer 3 have to agree.** Aligning STP roots with HSRP active routers made the design make sense end to end.
- **Verify, don't assume.** Catching the duplicate WLC address came from reading the addressing plan instead of copying the video.
- *(Add one or two real problems you hit, what the symptom was, and how you fixed it.)*

## How to Use This Lab

1. Install [Cisco Packet Tracer](https://www.netacad.com/cisco-packet-tracer) (free with a Networking Academy account).
2. Download `CCNA-Mega-Lab.pkt` from this repo and open it.
3. Follow the task list in [`docs/walkthrough.md`](docs/walkthrough.md).

> Passwords in this repo (such as the enable secret and WPA2 key) are specified by the lab and exist only in a simulation.

## Next Steps

- [ ] Rebuild the lab in GNS3 or EVE-NG with real IOS images
- [ ] Add network automation (Python/Netmiko) to push configs
- [ ] Study for CCNP ENCOR

## Contact

[Your Name] | [LinkedIn](https://www.linkedin.com/in/your-profile) | [your.email@example.com](mailto:your.email@example.com)
