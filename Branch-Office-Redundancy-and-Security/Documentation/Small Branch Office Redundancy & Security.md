# Small Branch Office Redundancy & Security

*Packet Tracer project: Second\_Project.pkt*

> **Note:** Sections 1 to 5 are the design and the configs given in this chat. Section 6 compares them with the running configs read from Second\_Project.pkt and lists what to fix.

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

## 6. As-Built Review (read from Second\_Project.pkt)

The saved topology differs from the design in 12 places; the ones below explain the failures you saw (inconsistent pings, HSRP split, no static NAT).

| # | Device | Found in the .pkt | Expected |
| --- | --- | --- | --- |
| 1 | R1, R2 | No static NAT line; Gi0/1.30 has no `ip nat inside` | `ip nat inside source static 192.168.30.10 203.0.113.10` |
| 2 | R1, R2 | Both use pool POOL1 203.0.113.6-.9 (overlapping) | Different ranges per router, outside .10 |
| 3 | R2 | ADMIN-ACL has no final `permit ip ... any`; SALES-ACL lacks echo-reply and Internet permits | Same ACLs as R1, so failover behaves the same |
| 4 | R1 | SALES-ACL has no final permit, so Sales cannot ping the ISP | Final `permit ip 192.168.20.0 0.0.0.255 any` |
| 5 | R1, R2 | OUTSIDE-ACL starts with `permit ip 8.8.8.0 0.0.0.255 any` | Remove: it lets the whole outside subnet reach every inside host |
| 6 | R1, R2 | HSRP: R1 priority 105 with default track decrement; R2 default 100 | Acceptable; R1 stays active until Gi0/0 fails |
| 7 | CORE1 | `spanning-tree mode pvst`, no root setting | rapid-pvst, root primary |
| 8 | CORE2 | rapid-pvst with priority 28672 on all VLANs, so CORE2 is root | CORE2 secondary |
| 9 | CORE1, CORE2 | EtherChannel uses LACP active (not `mode on`) | Fine; Section 3 Step 3 should read LACP |
| 10 | CORE2 | Fa0/1-4 and Gi0/2 are unconfigured; no access switch is attached | Trunks to the access switches, or document single uplinks |
| 11 | SW-ADMIN, SW-SALES, SW-SERVER | One uplink (Fa0/1, to CORE1); Gi0/1 and Gi0/2 are shut in VLAN 999 | Second uplink to CORE2 for redundancy |
| 12 | Access ports | Violation is `restrict` everywhere; no `errdisable recovery`; no `nonegotiate`; bare `switchport port-security` line not shown | `shutdown` on Admin and Server ports |

Cabling as built: CORE1 Fa0/1 to SW-ADMIN (VLANs 10,99), Fa0/2 to SW-SALES (20,99), Fa0/3 to SW-SERVER (30,99). Hosts sit on Fa0/2-3 of SW-ADMIN and SW-SALES and Fa0/2 of SW-SERVER.

### Fixes

**Static NAT and distinct pools (both routers)**

```
interface GigabitEthernet0/1.30
 ip nat inside
ip nat inside source static 192.168.30.10 203.0.113.10
no ip nat inside source list 1 pool POOL1 overload
no ip nat pool POOL1
```

Then on R1:

```
ip nat pool POOL1 203.0.113.4 203.0.113.6 netmask 255.255.255.240
ip nat inside source list 1 pool POOL1 overload
```

and on R2:

```
ip nat pool POOL1 203.0.113.7 203.0.113.9 netmask 255.255.255.240
ip nat inside source list 1 pool POOL1 overload
```

**Identical ACLs on R1 and R2**

```
ip access-list extended OUTSIDE-ACL
 no permit ip 8.8.8.0 0.0.0.255 any
ip access-list extended ADMIN-ACL
 permit ip 192.168.10.0 0.0.0.255 any
no ip access-list extended SALES-ACL
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

On R2, run `show access-lists` afterwards and confirm ADMIN-ACL, SERVER-ACL and OUTSIDE-ACL match R1 line for line.

**Spanning tree**

```
! CORE1
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,30,99 root primary

! CORE2
spanning-tree vlan 10,20,30,99 root secondary
```

CORE2's current `priority 28672` setting is replaced by the `root secondary` command.

**Second uplinks (CORE2, one port per access switch; match the cable you add)**

```
interface FastEthernet0/1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,99
 switchport nonegotiate
```

Repeat for the Sales (20,99) and Server (30,99) ports, then re-enable the access switches' Gi0/1 or Gi0/2 as trunks with the same native and allowed VLANs.

**Port security (Admin and Server ports shown)**

```
interface range FastEthernet0/2 - 3
 switchport nonegotiate
 switchport port-security
 switchport port-security violation shutdown
errdisable recovery cause psecure-violation
errdisable recovery interval 300
```

On SW-SERVER also set `switchport port-security maximum 1` on Fa0/2. Leave Sales ports on `restrict`.

### Re-test after the fixes

1. `show standby brief` on both routers: R1 Active, R2 Standby for groups 10, 20, 30, 99.
2. `show spanning-tree vlan 10`: CORE1 is root.
3. PC-S1 pings PC-A1: all four replies fail; PC-A1 pings PC-S1: succeeds.
4. Internet-PC browses 203.0.113.10: works; `show ip nat translations` shows the static entry.
5. Shut R1 Gi0/1, repeat the tests through R2: same results.

## 7. Revision Log: Second\_Project\_1.pkt

Only the routers changed: two NAT pool edits and one ACL line. Every switch, ISP and SW-WAN config is identical to the first file.

| # | Device | Change | Fixes item in Section 6 |
| --- | --- | --- | --- |
| 1 | R2 | `ip nat pool POOL1` changed from 203.0.113.6-.9 to 203.0.113.6-.7 | #2 (overlapping pools), partly |
| 2 | R1 | POOL1 changed from 203.0.113.6-.9 to 203.0.113.8-.9 (startup-config only, see below) | #2, completes it |
| 3 | R2 | Added `permit ip 192.168.10.0 0.0.0.255 any` as the last line of ADMIN-ACL | #3, partly (Admin can now reach the Internet through R2) |

The pools no longer overlap, with R1 at .8-.9 and R2 at .6-.7. Each pool holds only two addresses, so `overload` (already configured) is what lets all hosts share them.

```
! R1
ip nat pool POOL1 203.0.113.8 203.0.113.9 netmask 255.255.255.240
ip nat inside source list 1 pool POOL1 overload

! R2
ip nat pool POOL1 203.0.113.6 203.0.113.7 netmask 255.255.255.240
ip nat inside source list 1 pool POOL1 overload
interface GigabitEthernet0/1.10
 ip access-group ADMIN-ACL in
ip access-list extended ADMIN-ACL
 permit ip 192.168.10.0 0.0.0.255 any
```

### Check R1 first

In this file R1's running-config is the factory default (hostname Router, no interfaces), while its startup-config holds the full configuration, including the new pool. If R1 looks blank in Packet Tracer, run `copy startup-config running-config` (or `reload` without saving), then `show running-config` to confirm the subinterfaces, HSRP and ACLs are back. Then run `write memory` on R1.

CORE1, CORE2 and the three access switches also have no saved startup-config in the file; save them with `write memory` so a reload keeps your work.

### Still open from Section 6

| # | Item | Status |
| --- | --- | --- |
| 1 | Static NAT 192.168.30.10 to 203.0.113.10, and `ip nat inside` on Gi0/1.30 | Not added on either router |
| 3 | R2 SALES-ACL missing echo-reply and Internet permits; R1 SALES-ACL has no final permit | Not changed |
| 5 | `permit ip 8.8.8.0 0.0.0.255 any` in OUTSIDE-ACL | Still present on both routers |
| 7, 8 | CORE1 is pvst and not root; CORE2 is root with priority 28672 | Not changed |
| 10, 11 | CORE2 has no access-switch trunks; access switches have one uplink | Not changed |
| 12 | Port-security violation is `restrict` on all ports; no errdisable recovery | Not changed |

The fix snippets for these are in Section 6.
