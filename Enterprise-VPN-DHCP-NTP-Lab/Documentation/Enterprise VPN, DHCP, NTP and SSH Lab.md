# Enterprise VPN, DHCP, NTP and SSH Lab

Cisco Packet Tracer project: Enterprise-VPN-DHCP-NTP-Lab_baseline.pkt

## 1. The Goal

Build and verify a simulated enterprise network with three sites and an ISP, covering:

- **Site-to-site IPsec VPN** between HQ and Branch over a simulated internet.
- **Remote access VPN** (Easy VPN) for a remote laptop into HQ.
- **DHCP**: centralized on Server-HQ with relay for HQ, local router pool at Branch.
- **NTP**: Server-HQ as the time source, with Branch reaching it through the tunnel.
- **L3 switching**: HQ uses SVIs on a Layer 3 switch instead of router-on-a-stick.
- **SSH** management access.
- **Troubleshooting practice** by injecting and fixing faults.

### Success criteria

| Test | Expected |
| --- | --- |
| HQ PCs receive 10.1.10.x via DHCP | Pass |
| Branch PCs receive 10.2.10.x via DHCP | Pass |
| PC-HQ pings PC-BR (first packets may time out) | Pass |
| `show crypto isakmp sa` | QM\_IDLE |
| `show crypto ipsec sa` | Encaps and decaps rising |
| `show ntp status` on R-HQ and R-BR | Synchronized |
| Remote laptop connects VPN, pings 10.1.20.10 and 10.1.10.50 | Pass |

## 2. Topology

### Devices

| Site | Devices |
| --- | --- |
| HQ | R-HQ (2911), SW-HQ (3560-24PS, Layer 3), SRVR-HQ (Server-PT, DHCP + NTP), PCHQ-1, PCHQ-2 |
| Branch | R-BR (2911), SW-BR (2960-24TT), PCBR-1, PCBR-2 |
| Internet | R-ISP (2911) |
| Remote | SW-REM (2960-24TT), L-REM (Laptop-PT) |

### Connections

| Link | From | To | Subnet |
| --- | --- | --- | --- |
| HQ WAN | R-HQ Gi0/1 | R-ISP Gi0/0 | 203.0.113.0/30 (HQ .1, ISP .2) |
| Branch WAN | R-BR Gi0/1 | R-ISP Gi0/1 | 198.51.100.0/30 (BR .1, ISP .2) |
| Remote WAN | SW-REM Gi0/1 | R-ISP Gi0/2 | 192.0.2.0/29 (ISP .1, laptop .2) |
| Remote laptop | L-REM Fa0 | SW-REM Fa0/1 | 192.0.2.0/29 |
| HQ transit | R-HQ Gi0/0 | SW-HQ Gi0/1 (routed port) | 10.1.0.0/30 (R-HQ .1, SW-HQ .2) |
| HQ server | SRVR-HQ Fa0 | SW-HQ Fa0/1 | VLAN 20 |
| HQ PCs | PCHQ-1 Fa0, PCHQ-2 Fa0 | SW-HQ Fa0/2, Fa0/3 | VLAN 10 |
| Branch LAN | R-BR Gi0/0 | SW-BR Gi0/1 | 10.2.10.0/24 |
| Branch PCs | PCBR-1 Fa0, PCBR-2 Fa0 | SW-BR Fa0/1, Fa0/2 | 10.2.10.0/24 |

### Addressing

| Network | Subnet | Gateway |
| --- | --- | --- |
| HQ Users (VLAN 10) | 10.1.10.0/24 | 10.1.10.1 (SVI) |
| HQ Servers (VLAN 20) | 10.1.20.0/24 | 10.1.20.1 (SVI) |
| Branch Users | 10.2.10.0/24 | 10.2.10.1 |
| VPN client pool | 10.1.30.0/24 | n/a |

Server-HQ: 10.1.20.10 (DHCP and NTP). The ISP has no routes to private networks.

### Credentials used in the lab

| Item | Value |
| --- | --- |
| Site-to-site pre-shared key | VPNKEY123 |
| Remote VPN group / key | REMOTE / GROUPKEY123 |
| Remote VPN user | vpnuser / VPNPASS123 |
| NTP authentication | key 1, MD5, password pass@123 |
| Local admin / SSH user | macky |

## 3. Step-by-Step Configuration

### Step 1: R-ISP (simulated internet)

```
interface Gi0/0
 ip address 203.0.113.2 255.255.255.252
 no shutdown
interface Gi0/1
 ip address 198.51.100.2 255.255.255.252
 no shutdown
interface Gi0/2
 ip address 192.0.2.1 255.255.255.248
 no shutdown
```

No other routes. The ISP must not know private networks.

### Step 2: SW-HQ (Layer 3 switch with SVIs)

```
hostname SW-HQ
ip routing

vlan 10
vlan 20
vlan 999
 name VOID

interface Fa0/1
 switchport mode access
 switchport access vlan 20
interface range Fa0/2 - 3
 switchport mode access
 switchport access vlan 10
! unused ports are parked in VLAN 999 (VOID)

interface Vlan10
 ip address 10.1.10.1 255.255.255.0
 ip helper-address 10.1.20.10
 no shutdown
interface Vlan20
 ip address 10.1.20.1 255.255.255.0
 no shutdown

interface Gi0/1
 no switchport
 ip address 10.1.0.2 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.1.0.1
ip route 10.1.30.0 255.255.255.0 10.1.0.1
! also present in the saved file (redundant with the default route):
ip route 10.2.10.0 255.255.255.0 10.1.0.1
ip route 192.0.2.0 255.255.255.248 10.1.0.1
```

- `ip helper-address` is still required on the VLAN 10 SVI. DHCP discovery is a broadcast and the SVI is the Layer 3 boundary.
- Not needed on VLAN 20, since the server is in that subnet.
- The default route and the pool route are required so VPN client replies reach R-HQ.

### Step 3: R-HQ base configuration

```
hostname R-HQ
service timestamps log datetime msec
clock timezone UTC 8

interface Gi0/0
 ip address 10.1.0.1 255.255.255.252
 no shutdown
interface Gi0/1
 ip address 203.0.113.1 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 203.0.113.2
ip route 10.1.0.0 255.255.0.0 10.1.0.2
! also present in the saved file (covered by the /16):
ip route 10.1.20.0 255.255.255.0 10.1.0.2
```

The static /16 route sends all HQ LAN traffic back through SW-HQ.

### Step 4: DHCP

**HQ (Server-HQ):** Services > DHCP, service ON.

| Field | Value |
| --- | --- |
| Pool name | HQ-USERS |
| Network | 10.1.10.0 / 255.255.255.0 |
| Default gateway | 10.1.10.1 |
| DNS server | 10.1.20.10 |
| Start / end IP | 10.1.10.50 / 10.1.10.249 |
| Max users | 200 |

Relay is the `ip helper-address 10.1.20.10` on SW-HQ Vlan10 (Step 2).

**Branch (R-BR):**

```
ip dhcp excluded-address 10.2.10.1 10.2.10.20
ip dhcp pool BRANCH-POOL
 network 10.2.10.0 255.255.255.0
 default-router 10.2.10.1
 dns-server 10.1.20.10
```

Verify: `show ip dhcp pool`, `show ip dhcp binding`.

### Step 5: R-BR base configuration

```
hostname R-BR
service timestamps log datetime msec
clock timezone UTC 8

interface Gi0/0
 ip address 10.2.10.1 255.255.255.0
 no shutdown
interface Gi0/1
 ip address 198.51.100.1 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 198.51.100.2
```

SW-BR and SW-REM are left at default configuration (no VLANs, hostnames SW-BR and SW-REM).

### Step 6: Site-to-site IPsec VPN

Enable the security license on each 2911 if `crypto` commands are not recognized:

```
license boot module c2900 technology-package securityk9
write memory
reload
```

**R-HQ**

```
access-list 110 permit ip 10.1.0.0 0.0.255.255 10.2.10.0 0.0.0.255

crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
crypto isakmp key VPNKEY123 address 198.51.100.1

crypto ipsec transform-set TS esp-aes esp-sha-hmac

crypto map CMAP 10 ipsec-isakmp
 set peer 198.51.100.1
 set transform-set TS
 match address 110

interface Gi0/1
 crypto map CMAP
```

**R-BR (mirror image)**

```
access-list 110 permit ip 10.2.10.0 0.0.0.255 10.1.0.0 0.0.255.255

crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
crypto isakmp key VPNKEY123 address 203.0.113.1

crypto ipsec transform-set TS esp-aes esp-sha-hmac

crypto map CMAP 10 ipsec-isakmp
 set peer 203.0.113.1
 set transform-set TS
 match address 110

interface Gi0/1
 crypto map CMAP
```

**Order matters:** create the ACL and the transform-set first, then the crypto map, then apply it to the interface. Otherwise IOS returns "transform set with tag TS does not exist" or "Crypto Map with tag CMAP does not exist".

### Step 7: NTP

**SRVR-HQ:** Services > NTP ON, authentication enabled, key 1, password pass@123, with the correct date and time.

```
! R-HQ, R-BR and SW-HQ (on R-HQ the last line is: ntp server 10.1.20.10 key 1)
ntp authentication-key 1 md5 pass@123
ntp authenticate
ntp trusted-key 1
ntp server 10.1.20.10
```

R-BR reaches the server only through the tunnel. Packet Tracer has no `ntp source` command, so R-BR's NTP packets leave from its WAN address (198.51.100.1), which the original ACL 110 did not match. One line added to ACL 110 on each router permits NTP through the tunnel. It matches all IP between the two hosts so both routers' ACLs mirror each other exactly (a UDP port match on one side broke the match):

```
! R-BR
access-list 110 permit ip host 198.51.100.1 host 10.1.20.10
! R-HQ
access-list 110 permit ip host 10.1.20.10 host 198.51.100.1
```

Verify: `show ntp status`, `show ntp associations`, `show clock`.

### Step 8: Remote access VPN (Easy VPN)

**R-ISP:** Gi0/2 configured in Step 1.

**Laptop:** IP 192.0.2.2, mask 255.255.255.248, gateway 192.0.2.1. Confirm `ping 203.0.113.1` works before connecting.

**R-HQ (added to the existing config)**

```
aaa new-model
aaa authentication login VPN-AUTH local
aaa authorization network VPN-GROUP local
username vpnuser secret VPNPASS123

ip local pool VPN-POOL 10.1.30.10 10.1.30.50

crypto isakmp client configuration group REMOTE
 key GROUPKEY123
 pool VPN-POOL

crypto dynamic-map DYN 10
 set transform-set TS
 reverse-route

crypto map CMAP client authentication list VPN-AUTH
crypto map CMAP isakmp authorization list VPN-GROUP
crypto map CMAP client configuration address respond
crypto map CMAP 20 ipsec-isakmp dynamic DYN
```

- Both VPNs share one crypto map on Gi0/1: seq 10 static (Branch), seq 20 dynamic (remote clients).
- Packet Tracer does not support `acl` inside the ISAKMP group, so the remote VPN is a full tunnel.
- `reverse-route` installs a /32 route per connected client (for example `10.1.30.10/32 via 192.0.2.2`), so no static pool route is needed on R-HQ.
- `aaa new-model` can affect console and VTY login. Use `login local` or line passwords to avoid lockout.

**Laptop VPN client (Desktop > VPN)**

| Field | Value |
| --- | --- |
| Group Name | REMOTE |
| Group Key | GROUPKEY123 |
| Host IP | 203.0.113.1 |
| Username | vpnuser |
| Password | VPNPASS123 |

### Step 9: SSH

SSH works on both routers, R-HQ and R-BR. SW-HQ has the domain name and user but its VTY lines still use plain `login`. R-BR uses the same commands, but with login local instead of login authentication default, because it has no AAA model.

```
hostname R-HQ
ip domain-name port.local
username macky secret <password>
aaa authentication login default local
crypto key generate rsa modulus 1024
ip ssh version 2
line vty 0 4
 transport input ssh
 login authentication default
```

If `login authentication default` is rejected, use `login local`. The hostname must not be the default "Router" or "Switch" or key generation fails.

Test: `ssh -l macky 10.1.0.1` from PC-HQ and from the VPN-connected laptop (R-HQ), and ssh -l macky 10.2.10.1 from PC-HQ1 (R-BR, through the IPsec tunnel). Use R-BR's LAN address, not its WAN address 198.51.100.1, which is not covered by the tunnel. Verify with `show ip ssh`.

## 4. Verification

| Check | Command / action | Expected |
| --- | --- | --- |
| Phase 1 | `show crypto isakmp sa` | QM\_IDLE, ACTIVE for 198.51.100.1 and 192.0.2.2 |
| Phase 2 | `show crypto ipsec sa` | Encaps and decaps rising |
| Crypto map | `show crypto map` | Seq 10 (static) and seq 20 (dynamic DYN) on Gi0/1 |
| HQ DHCP | `ipconfig` on PC-HQ | 10.1.10.x |
| Branch DHCP | `ipconfig` on PC-BR | 10.2.10.x |
| Site-to-site | PC-HQ pings PC-BR | Replies after first timeouts |
| NTP | `show ntp status` | Synchronized |
| Remote VPN | Laptop `ipconfig` | Tunnel address 10.1.30.x |
| Remote reachability | Laptop pings 10.1.20.10 and 10.1.10.50 | Replies |

### Confirmed results

- Site-to-site IPsec: ISAKMP SA in QM\_IDLE between 203.0.113.1 and 198.51.100.1.
- Remote VPN: laptop received 10.1.30.10 and pinged both 10.1.20.10 and 10.1.10.50 successfully.
- NTP: R-HQ and R-BR both synchronize to 10.1.20.10 (R-BR through the tunnel). Remote to Branch is not expected to work, because ACL 110 matches only 10.1.0.0/16 and not the 10.1.30.0/24 pool.

## 5. Issues Found During the Build

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Crypto Map with tag CMAP does not exist` | Applied to interface before it was defined | Define the map first, then apply |
| `transform set with tag TS does not exist` | Transform-set not created before `set transform-set` | Create `crypto ipsec transform-set TS` first |
| `acl 120` invalid in ISAKMP group | Unsupported in Packet Tracer IOS | Skip it, use full tunnel |
| PC-HQ gets "Destination host unreachable" from 10.1.10.1 | SW-HQ had no default route to R-HQ | `ip route 0.0.0.0 0.0.0.0 10.1.0.1` |
| Laptop connected but could not reach HQ hosts | SW-HQ had no route back to 10.1.30.0/24 | Default route and pool route on SW-HQ |
| Laptop could not reach ISP/HQ | No default gateway on laptop | Set gateway 192.0.2.1 |
| `ping ... source` invalid | Not supported in this IOS | Use extended ping or skip |

## 6. Troubleshooting Faults to Inject

| # | Fault | Symptom | Verify with |
| --- | --- | --- | --- |
| 1 | Wrong pre-shared key on R-BR | Tunnel never forms | `show crypto isakmp sa` (MM\_NO\_STATE) |
| 2 | Mismatched ACL wildcard | Phase 1 up, nothing encrypted | `show crypto ipsec sa` (zero encaps) |
| 3 | Transform-set mismatch | Phase 2 fails | `debug crypto ipsec` |
| 4 | Missing `ip helper-address` on Vlan10 | HQ PCs get 169.254.x.x | `ipconfig /all` |
| 5 | Excluded range overlaps gateway or pool exhausted | Duplicate IP or no lease | `show ip dhcp binding`, `show ip dhcp conflict` |
| 6 | Crypto map not applied to Gi0/1 | No VPN | `show crypto map` |
| 7 | Wrong default route at Branch | Branch cannot reach ISP | `show ip route` |
| 8 | Wrong NTP IP or `NTP not permitted in ACL 110` | Clock unsynchronized | `show ntp associations` |
| 9 | Missing `ip routing` or SVI down on SW-HQ | HQ PCs lose gateway | `show ip interface brief` |
| 10 | NAT without excluding VPN traffic | Tunnel passes no traffic | `show ip nat translations` |
| 11 | Missing default or pool route on SW-HQ | Remote VPN connects but no HQ access | `show ip route` |
| 12 | Missing `ip domain-name` or RSA key | SSH fails | `show ip ssh` |

If NAT is added for internet access, deny VPN traffic in the NAT ACL first: `deny ip 10.1.0.0 0.0.255.255 10.2.10.0 0.0.0.255`, then `permit any`.

## 7. Final Steps

- Run `write memory` on every device.
- Save the .pkt as a clean baseline before injecting faults.
- Inject one fault at a time, fix it with its verify command, then restore the baseline.
