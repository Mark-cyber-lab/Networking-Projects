# Enterprise Multi-VLAN & Inter-VLAN Routing Network

*Documentation of First\_Project.pkt (Packet Tracer 9.0.1) – configs taken from the saved file.*

## 1. The Goal

Build a three-site enterprise network where:

- Each department sits in its own VLAN and is isolated at Layer 2.
- Inter-VLAN routing is done by **Router-on-a-Stick** (one 802.1Q trunk per site).
- **OSPF (Process 1, Area 0)** on R1, R2, R3 shares all VLAN and WAN subnets, so every department reaches every other across sites.
- **DHCP** is served centrally by R1 through `ip helper-address`.
- Switches are hardened (PortFast + BPDU Guard, port-security, unused ports in a black-hole VLAN, SSH-only management).
- An **ACL** limits Sales (VLAN 30) to HTTP/HTTPS only when reaching Servers (VLAN 60).

## 2. Topology

### Sites and devices

| Site | Devices | Departments |
| --- | --- | --- |
| HQ | R1 (2911), SW-CORE (3650, L2 role), SW-HQ1, SW-HQ2 | HR (10), Finance (20) |
| Branch A | R2 (2911), SW-A (2960) | Sales (30), Marketing (40) |
| Branch B | R3 (2911), SW-B (2960) | IT (50), Servers (60) |

### Links

| From | To | Type |
| --- | --- | --- |
| R1 Gi0/2/0 | SW-CORE Gi1/1/3 | Trunk (subinterfaces .10, .20, .99) |
| SW-CORE Gi1/1/1 | SW-HQ1 Gi6/1 | Trunk (10, 99) |
| SW-CORE Gi1/1/2 | SW-HQ2 Gi5/1 | Trunk (20, 99) |
| SW-HQ1 Fa0/1, Fa1/1–4/1 | 5 PCs | Access VLAN 10 |
| SW-HQ2 Fa0/1, Fa1/1–4/1 | 5 PCs | Access VLAN 20 |
| R2 Gi0/0 | SW-A Gi0/1 | Trunk (30, 40, 99) |
| SW-A Fa0/1–5 / Fa0/6–10 | PCs | Access VLAN 30 / 40 |
| R3 Gi0/0 | SW-B Gi0/1 | Trunk (50, 60, 99) |
| SW-B Fa0/1–5 / Fa0/6–10 | PCs / Servers | Access VLAN 50 / 60 |
| R1 Gi0/0/0 | R2 Gi0/0/0 | WAN 10.0.12.0/30 (R2 .1, R1 .2) |
| R1 Gi0/1/0 | R3 Gi0/0/0 | WAN 10.0.13.0/30 (R1 .1, R3 .2) |
| R2 Gi0/1/0 | R3 Gi0/1/0 | WAN 10.0.23.0/30 (R3 .1, R2 .2) |

### VLAN and addressing plan

| VLAN | Name | Subnet | Gateway | Site |
| --- | --- | --- | --- | --- |
| 10 | HR | 192.168.10.0/24 | .1 (R1) | HQ |
| 20 | Finance | 192.168.20.0/24 | .1 (R1) | HQ |
| 30 | Sales | 192.168.30.0/24 | .1 (R2) | Branch A |
| 40 | Marketing | 192.168.40.0/24 | .1 (R2) | Branch A |
| 50 | IT | 192.168.50.0/24 | .1 (R3) | Branch B |
| 60 | Servers | 192.168.60.0/24 | .1 (R3) | Branch B |
| 99 (HQ) | Mgmt/Native | 192.168.99.0/24 | .1 (R1) | HQ |
| 99 (Branch A) | Mgmt/Native | 192.168.199.0/24 | .1 (R2) | Branch A |
| 99 (Branch B) | Mgmt/Native | 192.168.209.0/24 | .1 (R3) | Branch B |
| 999 | Void | none | none | Unused ports |

VLAN 99 uses a **different subnet per site** so OSPF can route between management networks.

### Switch management IPs

| Switch | VLAN 99 SVI | Default gateway |
| --- | --- | --- |
| SW-CORE | 192.168.99.4 | 192.168.99.1 |
| SW-HQ1 | 192.168.99.5 | 192.168.99.1 |
| SW-HQ2 | 192.168.99.6 | 192.168.99.1 |
| SW-A | 192.168.199.2 | 192.168.199.1 |
| SW-B | 192.168.209.2 | 192.168.209.1 |

### Router IDs

R1 = 1.1.1.1, R2 = 2.2.2.2, R3 = 3.3.3.3.

## 3. Step-by-Step Configuration

Placeholders: `<user>`, `<password>`, `<enable-pass>`.

### Step 1 – Create VLANs on every switch

Create only the VLANs each switch needs. Example for SW-CORE:

```
vlan 10
 name HR
vlan 20
 name Finance
vlan 99
 name Native-Mgmt
vlan 999
 name Void
```

Per switch: SW-HQ1 → 10, 99, 999. SW-HQ2 → 20, 99, 999. SW-A → 30 Sales, 40 Marketing, 99, 999. SW-B → 50 IT, 60 Servers, 99, 999.

### Step 2 – Trunks (802.1Q, native 99, restricted VLANs)

SW-CORE:

```
interface g1/1/1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,99
interface g1/1/2
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 20,99
interface g1/1/3
 description To R1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,99
 switchport nonegotiate
```

SW-HQ1 (Gi6/1) and SW-HQ2 (Gi5/1) uplinks:

```
interface g6/1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,99
 switchport nonegotiate
```

(SW-HQ2: allowed vlan 20,99.)

SW-A:

```
interface g0/1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 30,40,99
```

SW-B:

```
interface g0/1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 50,60,99
```

### Step 3 – Access ports (PortFast, BPDU Guard, port-security)

SW-A example (VLAN 30 on Fa0/1–5, VLAN 40 on Fa0/6–10):

```
interface range f0/1-5
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
interface range f0/6-10
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
```

Repeat with: SW-B Fa0/1–5 → 50, Fa0/6–10 → 60; SW-HQ1 access ports → 10; SW-HQ2 access ports → 20.

### Step 4 – Shut down unused ports into the black-hole VLAN

```
interface range f0/11-24, g0/2
 switchport mode access
 switchport access vlan 999
 shutdown
```

SW-CORE: `interface range g1/0/1-24, g1/1/4`.

### Step 5 – Spanning Tree

SW-CORE as root:

```
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,99 priority 4096
```

Apply `spanning-tree mode rapid-pvst` on all other switches as well.

### Step 6 – Management and SSH (every switch)

```
hostname SW-A
ip domain-name corp.local
username <user> secret <password>
enable secret <enable-pass>
crypto key generate rsa modulus 1024
ip ssh version 2
interface vlan 99
 ip address 192.168.199.2 255.255.255.0
 no shutdown
ip default-gateway 192.168.199.1
line vty 0 15
 login local
 transport input ssh
 exec-timeout 5 0
```

Use the matching SVI/gateway from the management table above for each switch.

### Step 7 – Router-on-a-Stick (R1, R2, R3)

R1 (trunk on Gi0/2/0 toward SW-CORE):

```
interface g0/2/0
 no shutdown
interface g0/2/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
interface g0/2/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
interface g0/2/0.99
 encapsulation dot1Q 99 native
 ip address 192.168.99.1 255.255.255.0
```

R2 (Gi0/0 toward SW-A):

```
interface g0/0
 no shutdown
interface g0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip helper-address 10.0.12.2
interface g0/0.40
 encapsulation dot1Q 40
 ip address 192.168.40.1 255.255.255.0
 ip helper-address 10.0.12.2
interface g0/0.99
 encapsulation dot1Q 99 native
 ip address 192.168.199.1 255.255.255.0
```

R3 (Gi0/0 toward SW-B):

```
interface g0/0
 no shutdown
interface g0/0.50
 encapsulation dot1Q 50
 ip address 192.168.50.1 255.255.255.0
 ip helper-address 10.0.13.1
interface g0/0.60
 encapsulation dot1Q 60
 ip address 192.168.60.1 255.255.255.0
 ip helper-address 10.0.13.1
interface g0/0.99
 encapsulation dot1Q 99 native
 ip address 192.168.209.1 255.255.255.0
```

### Step 8 – WAN point-to-point links

R1:

```
interface g0/0/0
 ip address 10.0.12.2 255.255.255.252
 no shutdown
interface g0/1/0
 ip address 10.0.13.1 255.255.255.252
 no shutdown
```

R2:

```
interface g0/0/0
 ip address 10.0.12.1 255.255.255.252
 no shutdown
interface g0/1/0
 ip address 10.0.23.2 255.255.255.252
 no shutdown
```

R3:

```
interface g0/0/0
 ip address 10.0.13.2 255.255.255.252
 no shutdown
interface g0/1/0
 ip address 10.0.23.1 255.255.255.252
 no shutdown
```

### Step 9 – OSPF (Process 1, Area 0)

R1:

```
router ospf 1
 router-id 1.1.1.1
 passive-interface g0/2/0
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.13.0 0.0.0.3 area 0
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.99.0 0.0.0.255 area 0
```

R2:

```
router ospf 1
 router-id 2.2.2.2
 passive-interface g0/0
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
 network 192.168.30.0 0.0.0.255 area 0
 network 192.168.40.0 0.0.0.255 area 0
 network 192.168.199.0 0.0.0.255 area 0
```

R3:

```
router ospf 1
 router-id 3.3.3.3
 passive-interface g0/0
 network 10.0.13.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
 network 192.168.50.0 0.0.0.255 area 0
 network 192.168.60.0 0.0.0.255 area 0
 network 192.168.209.0 0.0.0.255 area 0
```

### Step 10 – Central DHCP on R1

```
ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.20.1 192.168.20.10
ip dhcp excluded-address 192.168.30.1 192.168.30.10
ip dhcp excluded-address 192.168.40.1 192.168.40.10
ip dhcp excluded-address 192.168.50.1 192.168.50.10
ip dhcp excluded-address 192.168.60.1 192.168.60.10
ip dhcp excluded-address 192.168.99.1
ip dhcp pool HR
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
```

Repeat the pool for FINANCE (.20), SALES (.30), MARKETING (.40), IT (.50), SERVERS (.60) and MANAGEMENT (.99). R2 and R3 relay requests using `ip helper-address` (Step 7).

### Step 11 – ACL: Sales → Servers allows only HTTP/HTTPS (R3)

```
access-list 100 permit tcp 192.168.30.0 0.0.0.255 192.168.60.0 0.0.0.255 eq www
access-list 100 permit tcp 192.168.30.0 0.0.0.255 192.168.60.0 0.0.0.255 eq 443
access-list 100 deny ip 192.168.30.0 0.0.0.255 192.168.60.0 0.0.0.255
access-list 100 permit ip any any
interface g0/0/0
 ip access-group 100 in
interface g0/1/0
 ip access-group 100 in
```

Applied inbound on both R3 WAN links, so Sales traffic is filtered no matter which path it takes.

## 4. Verification

| Check | Command | Expected |
| --- | --- | --- |
| VLANs | `show vlan brief` | Matches the VLAN plan per switch |
| Trunks | `show interfaces trunk` | Native 99, correct allowed VLANs |
| SVI | `show ip interface brief` | Vlan99 up/up |
| OSPF neighbors | `show ip ospf neighbor` (R1) | R2 and R3 in FULL |
| Routes | `show ip route ospf` | All remote VLAN and mgmt subnets |
| DHCP | `show ip dhcp binding` (R1) | Leases for every VLAN |
| ACL | `show access-lists` (R3) | Hit counters increase |
| Path | `tracert 192.168.50.x` from VLAN 10 | R1 → R3 |
| SSH | `ssh -l <user> 192.168.209.2` from VLAN 10 PC | Login prompt |
| Sales ACL | Sales PC: web page to a VLAN 60 server works, ping fails | As stated |

## 5. Observations from Your File

Things found in the saved configs that are worth fixing:

1. **R1 hostname is still `Router`.** Run `hostname R1`.
2. **SSH is incomplete on SW-HQ1, SW-HQ2 and SW-CORE.** SW-HQ1/HQ2 lack `ip domain-name`, `ip ssh version 2`, a local user and `login local` (they use `login` with a line password). SW-CORE has no local user and uses `login` with a line password. Apply Step 6 to all three.
3. **SW-B has no `enable secret`.** Add one (see earlier `enable` issue).
4. **Mixed STP modes.** Only SW-CORE runs Rapid-PVST; the others run PVST. Set `spanning-tree mode rapid-pvst` everywhere.
5. **Passive interface is set on the parent interface** (`g0/2/0`, `g0/0`). Subinterfaces may still send OSPF hellos. Safer: `passive-interface default` then `no passive-interface g0/0/0` and `g0/1/0` on each router.
6. **OSPF MD5 authentication** on WAN links (optional requirement) is not configured.
7. **SW-CORE role:** it is a 3650 used as a pure Layer 2 switch (no `ip routing`), so routing stays on R1. This is consistent with the router-on-a-stick design.
8. **Leftovers:** `switchport access vlan 99` on SW-A Gi0/1 (trunk) and port-security commands on SW-CORE trunks Gi1/1/1–2 have no useful effect. `spanning-tree guard root` on access ports is extra but harmless.
9. **VLAN 99 name** is the default `VLAN0099`; rename to `Native-Mgmt` for clarity.
10. **Credentials:** a plaintext `password` appears on SW-CORE and vty lines. Prefer `username ... secret` with `login local` and enable `service password-encryption`.
