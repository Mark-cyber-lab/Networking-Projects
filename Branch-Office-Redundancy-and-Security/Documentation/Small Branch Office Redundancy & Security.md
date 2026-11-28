# Small Branch Office Redundancy & Security

*Packet Tracer project: Small Branch Office Redundancy & Security.pkt*

> **Note:** Sections 1 to 5 are the design and the commands used to build it. Section 6 is the final configuration read from the saved project file and is the authoritative record. ACL names in Section 3 (SERVERS-IN, OUTSIDE-IN) appear as SERVER-ACL and OUTSIDE-ACL in the finished build.

## 1. The Goal

Build a small branch office network that stays online when a link or device fails, and that controls who can talk to whom.

| Objective | Feature used |
| --- | --- |
| Survive router failure | Two gateways (R1, R2) running HSRP |
| Survive switch/link failure | Dual-homed access switches, two core switches, EtherChannel, Rapid-PVST+ |
| Reach the Internet and publish a server | Dynamic NAT (pool) and static NAT |
| Block unauthorized traffic between subnets | Extended ACLs on router subinterfaces and the WAN interface |
| Stop rogue devices at the access layer | Port security, PortFast, BPDU Guard, unused ports shut |

**Success criteria**

- PC-S1 cannot ping PC-A1; PC-A1 can ping PC-S1.
- PC-S1 can browse WEB-SRV; WEB-SRV cannot start connections to Admin, Sales or Management.
- Internet-PC can browse 203.0.113.10 (static NAT) and nothing else.
- Inside PCs reach the ISP through a pool address.
- Shutting R1 or one core uplink does not break connectivity.
- A rogue PC on a protected port err-disables it.

## 2. Topology

### Devices

| Device | Model | Role |
| --- | --- | --- |
| ISP | 2911 | Simulated ISP |
| Internet-PC | PC | Outside test host |
| SW-WAN | 2960 | Joins ISP, R1 and R2 |
| R1 | 2911 | Primary gateway, HSRP active, NAT, ACLs |
| R2 | 2911 | Backup gateway, HSRP standby, NAT, ACLs |
| CORE1, CORE2 | 2960-24TT | Core/distribution |
| SW-ADMIN, SW-SALES, SW-SERVER | 2960 | Access switches |
| PC-A1, PC-A2, PC-S1, PC-S2, WEB-SRV | PC/Server | End devices |

### Addressing

| Building | VLAN | Subnet | HSRP VIP | R1 | R2 |
| --- | --- | --- | --- | --- | --- |
| Admin | 10 | 192.168.10.0/24 | .1 | .2 | .3 |
| Sales | 20 | 192.168.20.0/24 | .1 | .2 | .3 |
| Server room | 30 | 192.168.30.0/24 | .1 | .2 | .3 |
| Management | 99 | 192.168.99.0/24 | .1 | .2 | .3 |
| WAN | – | 203.0.113.0/28 | – | .2 | .3 |

ISP is 203.0.113.1. Hosts: PC-A1 .10.11, PC-A2 .10.12, PC-S1 .20.11, PC-S2 .20.12, WEB-SRV .30.10. Switch SVIs are 192.168.99.11–.15 with gateway 192.168.99.1.

### Connections

| From | To | Type |
| --- | --- | --- |
| ISP Gi0/0 | SW-WAN Fa0/1 | Access |
| R1 Gi0/0 | SW-WAN Fa0/2 | Access (203.0.113.2) |
| R2 Gi0/0 | SW-WAN Fa0/3 | Access (203.0.113.3) |
| R1 Gi0/1 | CORE1 Gi0/1 | Trunk |
| R2 Gi0/1 | CORE2 Gi0/1 | Trunk |
| CORE1 Fa0/23-24 | CORE2 Fa0/23-24 | EtherChannel trunk |
| SW-ADMIN Gi0/1 / Gi0/2 | CORE1 / CORE2 Fa0/2 | Trunk |
| SW-SALES Gi0/1 / Gi0/2 | CORE1 / CORE2 Fa0/3 | Trunk |
| SW-SERVER Gi0/1 / Gi0/2 | CORE1 / CORE2 Fa0/4 | Trunk |
| PC-A1/A2 | SW-ADMIN Fa0/1-2 | Access |
| PC-S1/S2 | SW-SALES Fa0/1-2 | Access |
| WEB-SRV | SW-SERVER Fa0/1 | Access |
| Internet-PC | ISP Gi0/1 | Access (8.8.8.0/24) |

## 3. Step-by-Step Configuration

### Step 1: VLANs (all switches)

```
vlan 10
 name ADMIN
vlan 20
 name SALES
vlan 30
 name SERVER-ROOM
vlan 99
 name MANAGEMENT
vlan 999
 name VOID
```

### Step 2: Trunks

Native VLAN 99 must match on both ends of every trunk. `switchport mode trunk` goes before `switchport nonegotiate`.

**Core to access switch (CORE1 and CORE2, adjust port and VLANs per switch)**

```
interface FastEthernet0/2
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,99
 switchport nonegotiate
interface FastEthernet0/3
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 20,99
 switchport nonegotiate
interface FastEthernet0/4
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 30,99
 switchport nonegotiate
```

**Core to router (CORE1 Gi0/1 and CORE2 Gi0/1)**

```
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,99
 switchport nonegotiate
```

**Access switch uplinks (Gi0/1 and Gi0/2), example SW-ADMIN**

```
interface range GigabitEthernet0/1 - 2
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,99
 switchport nonegotiate
```

### Step 3: EtherChannel between cores (both cores)

```
interface range FastEthernet0/23 - 24
 channel-group 1 mode on
interface Port-channel1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,99
```

### Step 4: Spanning tree

```
! All switches
spanning-tree mode rapid-pvst

! CORE1
spanning-tree vlan 10,20,30,99 root primary

! CORE2
spanning-tree vlan 10,20,30,99 root secondary
```

### Step 5: Router subinterfaces and HSRP

R1 shown. On R2 use the .3 addresses, priority 100, and omit the tracking line.

```
interface GigabitEthernet0/1
 no shutdown
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.10.2 255.255.255.0
 standby 10 ip 192.168.10.1
 standby 10 priority 110
 standby 10 preempt
interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 ip address 192.168.20.2 255.255.255.0
 standby 20 ip 192.168.20.1
 standby 20 priority 110
 standby 20 preempt
interface GigabitEthernet0/1.30
 encapsulation dot1Q 30
 ip address 192.168.30.2 255.255.255.0
 standby 30 ip 192.168.30.1
 standby 30 priority 110
 standby 30 preempt
interface GigabitEthernet0/1.99
 encapsulation dot1Q 99 native
 ip address 192.168.99.2 255.255.255.0
 standby 99 ip 192.168.99.1
 standby 99 priority 110
 standby 99 preempt
```

**Track the WAN link (R1)**

```
track 1 interface GigabitEthernet0/0 line-protocol
interface GigabitEthernet0/1.10
 standby 10 track 1 decrement 20
interface GigabitEthernet0/1.20
 standby 20 track 1 decrement 20
interface GigabitEthernet0/1.30
 standby 30 track 1 decrement 20
interface GigabitEthernet0/1.99
 standby 99 track 1 decrement 20
```

### Step 6: WAN interface and default route (R1; R2 uses .3)

```
interface GigabitEthernet0/0
 ip address 203.0.113.2 255.255.255.240
 no shutdown
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

### Step 7: NAT (both routers)

```
interface GigabitEthernet0/0
 ip nat outside
interface range GigabitEthernet0/1.10 , GigabitEthernet0/1.20 , GigabitEthernet0/1.30 , GigabitEthernet0/1.99
 ip nat inside

! Static NAT for WEB-SRV
ip nat inside source static 192.168.30.10 203.0.113.10

! Dynamic NAT for Admin and Sales
ip nat pool BRANCH 203.0.113.4 203.0.113.6 netmask 255.255.255.240
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 1 permit 192.168.20.0 0.0.0.255
ip nat inside source list 1 pool BRANCH overload
```

### Step 8: ACLs (both routers, inbound on the VLAN subinterface)

ACLs go on the **routers**, not on CORE1 or CORE2 (those are Layer 2 switches). One ACL per subinterface; keep them separate.

| Interface | ACL | Direction |
| --- | --- | --- |
| Gi0/1.10 | ADMIN-ACL | in |
| Gi0/1.20 | SALES-ACL | in |
| Gi0/1.30 | SERVERS-IN | in |
| Gi0/0 | OUTSIDE-IN | in |

**ADMIN-ACL (Admin to Servers/Sales permitted, Management SSH and ping only)**

```
ip access-list extended ADMIN-ACL
 permit udp any host 224.0.0.2 eq 1985
 permit ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit tcp 192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255 eq 22
 permit icmp 192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255
 deny ip 192.168.10.0 0.0.0.255 192.168.0.0 0.0.255.255
 permit ip 192.168.10.0 0.0.0.255 any
interface GigabitEthernet0/1.10
 ip access-group ADMIN-ACL in
```

**SALES-ACL (no new connections to Admin or Management; web and ping to WEB-SRV only)**

```
ip access-list extended SALES-ACL
 permit udp any host 224.0.0.2 eq 1985
 permit icmp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 echo-reply
 permit tcp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 established
 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.30.10 eq 80
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.30.10 eq 443
 permit icmp 192.168.20.0 0.0.0.255 host 192.168.30.10
 deny ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255
 deny ip 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit ip 192.168.20.0 0.0.0.255 any
interface GigabitEthernet0/1.20
 ip access-group SALES-ACL in
```

The `echo-reply` and `established` lines let Sales answer pings and TCP sessions that Admin starts, while Sales-initiated traffic to Admin is still denied.

**SERVERS-IN (Servers may only reply, never start connections inward)**

```
ip access-list extended SERVERS-IN
 permit udp any host 224.0.0.2 eq 1985
 permit tcp 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255 established
 permit tcp 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255 established
 permit tcp 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255 established
 permit icmp 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255 echo-reply
 permit icmp 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255 echo-reply
 permit icmp 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255 echo-reply
 deny ip 192.168.30.0 0.0.0.255 192.168.0.0 0.0.255.255 log
 permit ip 192.168.30.0 0.0.0.255 any
interface GigabitEthernet0/1.30
 ip access-group SERVERS-IN in
```

**OUTSIDE-IN (WAN side)**

The inbound ACL is checked before NAT, so match the public address 203.0.113.10. HSRP is not needed here because it only runs on the Gi0/1 subinterfaces.

```
ip access-list extended OUTSIDE-IN
 permit tcp any host 203.0.113.10 eq 80
 permit tcp any 203.0.113.0 0.0.0.15 established
 permit icmp any 203.0.113.0 0.0.0.15 echo-reply
 permit icmp any 203.0.113.0 0.0.0.15 unreachable
 permit icmp any 203.0.113.0 0.0.0.15 time-exceeded
 permit icmp host 203.0.113.1 any echo
 deny ip any any log
interface GigabitEthernet0/0
 ip access-group OUTSIDE-IN in
```

### Step 9: Port security (SW-ADMIN, SW-SALES, SW-SERVER)

**Admin ports (max 2, shutdown)**

```
interface range FastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
```

**Sales ports (max 2, restrict)**

```
interface range FastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 20
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
```

**Server port (max 1, shutdown)**

```
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 30
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
```

**Unused ports and auto-recovery (each access switch)**

```
interface range FastEthernet0/3 - 24
 switchport mode access
 switchport access vlan 999
 shutdown
errdisable recovery cause psecure-violation
errdisable recovery interval 300
```

## 4. Verification

| Check | Command | Expected |
| --- | --- | --- |
| HSRP | `show standby brief` | R1 Active, R2 Standby for groups 10, 20, 30, 99 |
| STP | `show spanning-tree` | CORE1 is root |
| EtherChannel | `show etherchannel summary` | Po1 SU, Fa0/23-24 P |
| Trunks | `show interfaces trunk` | Native 99, correct allowed VLANs |
| NAT | `show ip nat translations` | Static entry plus pool addresses .4-.6 |
| ACLs | `show access-lists` | Hit counters increase during tests |
| ACL binding | `show ip interface Gi0/1.20` | Inbound list is SALES-ACL |
| Port security | `show port-security interface fa0/1` | Sticky MAC learned; shutdown on violation |

### Test results to record

| Test | Expected |
| --- | --- |
| PC-S1 ping PC-A1 | Fails |
| PC-A1 ping PC-S1 | Succeeds |
| PC-S1 browse WEB-SRV | Succeeds |
| WEB-SRV ping PC-A1 | Fails |
| PC-A1 / PC-S1 ping ISP | Succeeds, translated to a pool address |
| Internet-PC browse 203.0.113.10 | Succeeds |
| Internet-PC ping 203.0.113.10 | Fails |
| Shut R1 Gi0/1 or Gi0/0 | PCs still reach ISP via R2 |
| Shut one core uplink | Traffic continues |
| Rogue PC on PC-A1's port | Port err-disabled |

## 5. Troubleshooting Notes

**HSRP split-brain (both routers Active on VLAN 10/20)**

- Symptom: pings from Sales to Admin alternated between failing (R1's ACL) and succeeding (R2 without the ACL), about 50% loss.
- Cause: hellos were not crossing between R1 and R2 on VLANs 10 and 20, a Layer 2 path problem. Check `show interfaces trunk`, `show vlan brief`, `show spanning-tree vlan 10`, and that CORE2's access-facing ports are trunks.
- Rule out the ACL by temporarily removing it from the subinterface, then re-apply.

**Common mistakes fixed during the build**

- ACLs cannot be applied on a 2960 switch port. They belong on the router subinterfaces.
- Trunk allowed-VLAN lists must match what each port connects to, and both ends must use native VLAN 99.
- Both routers need identical ACLs on every subinterface.
- If both routers share one NAT pool, consider giving R2 a different range to avoid duplicate translations.

## 6. Final Configuration

Read from the final saved project file. All devices are saved (running-config equals startup-config) and every test in Section 4 passed.

| Requirement | Final state |
| --- | --- |
| Gateway redundancy | HSRP groups 10, 20, 30, 99. R1 priority 105 with preempt and tracking of Gi0/0; R2 default priority 100 |
| STP | Rapid-PVST+. CORE1 is root (priority 24576), CORE2 is backup (28672) |
| Core link | LACP EtherChannel (Po1) on Fa0/23-24, trunk, native VLAN 99, VLANs 10,20,30,99 |
| Dynamic NAT | Pool POOL1: R1 203.0.113.8-.9, R2 203.0.113.6-.7, `overload`, ACL 1 permits VLANs 10 and 20 |
| Static NAT | 192.168.30.10 to 203.0.113.10 on both routers |
| ACLs | ADMIN-ACL, SALES-ACL, SERVER-ACL inbound on Gi0/1.10, .20, .30; OUTSIDE-ACL inbound on Gi0/0 |
| Port security | Sticky MAC, PortFast, BPDU Guard, `nonegotiate`. Admin and Server ports shut down on violation; Sales ports restrict |

### Routers (R1 shown; R2 uses .3 addresses, no priority line, no tracking)

```
interface GigabitEthernet0/0
 ip address 203.0.113.2 255.255.255.240
 ip nat outside
 ip access-group OUTSIDE-ACL in
interface GigabitEthernet0/1
 ip nat inside
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.10.2 255.255.255.0
 ip nat inside
 ip access-group ADMIN-ACL in
 standby 10 ip 192.168.10.1
 standby 10 priority 105
 standby 10 preempt
 standby 10 track GigabitEthernet0/0
interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 ip address 192.168.20.2 255.255.255.0
 ip nat inside
 ip access-group SALES-ACL in
 standby 20 ip 192.168.20.1
 standby 20 priority 105
 standby 20 preempt
 standby 20 track GigabitEthernet0/0
interface GigabitEthernet0/1.30
 encapsulation dot1Q 30
 ip address 192.168.30.2 255.255.255.0
 ip nat inside
 ip access-group SERVER-ACL in
 standby 30 ip 192.168.30.1
 standby 30 priority 105
 standby 30 preempt
 standby 30 track GigabitEthernet0/0
interface GigabitEthernet0/1.99
 encapsulation dot1Q 99 native
 ip address 192.168.99.2 255.255.255.0
 standby 99 ip 192.168.99.1
 standby 99 priority 105
 standby 99 preempt
 standby 99 track GigabitEthernet0/0
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

### NAT (both routers; only the pool differs)

```
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 1 permit 192.168.20.0 0.0.0.255
ip nat inside source static 192.168.30.10 203.0.113.10

! R1
ip nat pool POOL1 203.0.113.8 203.0.113.9 netmask 255.255.255.240
! R2
ip nat pool POOL1 203.0.113.6 203.0.113.7 netmask 255.255.255.240

ip nat inside source list 1 pool POOL1 overload
```

The two pools do not overlap, and the static address (.10) is identical on both routers so WEB-SRV stays published if one router fails.

### ACLs (both routers)

**ADMIN-ACL**

```
ip access-list extended ADMIN-ACL
 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
 permit tcp 192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255 eq 22
 permit icmp 192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit udp any host 224.0.0.2 eq 1985
 permit tcp 192.168.10.0 0.0.0.255 8.8.8.0 0.0.0.255 eq www
 permit tcp 192.168.10.0 0.0.0.255 8.8.8.0 0.0.0.255 eq 443
 permit ip 192.168.10.0 0.0.0.255 any
```

**SALES-ACL**

```
ip access-list extended SALES-ACL
 permit icmp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 echo-reply
 permit tcp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 established
 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.30.10 eq www
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.30.10 eq 443
 permit icmp 192.168.20.0 0.0.0.255 host 192.168.30.10
 deny ip 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit udp any host 224.0.0.2 eq 1985
 deny ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255
 permit ip 192.168.20.0 0.0.0.255 any
```

R1 also carries two extra web permits to 8.8.8.0/24 ahead of the deny to the server subnet; they do not change the result.

**SERVER-ACL**

```
ip access-list extended SERVER-ACL
 permit tcp 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255 established
 permit tcp 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255 established
 permit tcp 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255 established
 permit icmp 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255 echo-reply
 permit icmp 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255 echo-reply
 permit icmp 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255 echo-reply
 deny ip 192.168.30.0 0.0.0.255 192.168.0.0 0.0.255.255
 permit ip 192.168.30.0 0.0.0.255 any
```

**OUTSIDE-ACL**

```
ip access-list extended OUTSIDE-ACL
 permit tcp any 203.0.113.0 0.0.0.15 eq www
 permit tcp any 203.0.113.0 0.0.0.15 established
 permit icmp any 203.0.113.0 0.0.0.15 echo-reply
 permit icmp any 203.0.113.0 0.0.0.15 unreachable
 permit icmp host 203.0.113.1 any echo
 deny ip any any
```

### Switches

**Spanning tree**

```
! CORE1
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,30,99 priority 24576
! CORE2
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,30,99 priority 28672
! Access switches
spanning-tree mode rapid-pvst
```

**CORE1 trunks to the access switches**

```
interface FastEthernet0/1
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,99
 switchport mode trunk
interface FastEthernet0/2
 switchport trunk native vlan 99
 switchport trunk allowed vlan 20,99
 switchport mode trunk
interface FastEthernet0/3
 switchport trunk native vlan 99
 switchport trunk allowed vlan 30,99
 switchport mode trunk
```

**EtherChannel (CORE1 and CORE2)**

```
interface range FastEthernet0/23 - 24
 channel-protocol lacp
 channel-group 1 mode active
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,99
```

**Access ports**

```
! SW-ADMIN Fa0/2-3 (VLAN 10); violation shutdown is the default
interface range FastEthernet0/2 - 3
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable

! SW-SALES Fa0/2-3 (VLAN 20)
interface range FastEthernet0/2 - 3
 switchport mode access
 switchport access vlan 20
 switchport nonegotiate
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable

! SW-SERVER Fa0/2 (VLAN 30); maximum 1 and violation shutdown are the defaults
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 30
 switchport nonegotiate
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
```

Unused access ports are in VLAN 999 and shut down.

### Test results

All items in the Section 4 test table were run and passed: Sales to Admin ping fails, Admin to Sales works, Sales browses WEB-SRV, WEB-SRV cannot start connections inward, Internet-PC browses 203.0.113.10, inside hosts reach the ISP through the NAT pool, and the rogue-PC test err-disables the protected port.

### Known limitations

- **Single-homed access layer:** the access switches connect only to CORE1, and CORE2 is reached through the EtherChannel. A CORE1 failure takes the access switches offline; router and core-link failures are covered.
- **No errdisable auto-recovery:** this Packet Tracer version has no `errdisable recovery` command, so an err-disabled Admin or Server port is reset manually with `shutdown` then `no shutdown`.
- **HSRP tracking:** `track GigabitEthernet0/0` uses the default decrement of 10, so R1 (105) drops to 95 and R2 (100) takes over when the WAN link fails.
- **Static NAT address:** both routers answer for 203.0.113.10, which suits the lab. A production design would tie the address to HSRP.
