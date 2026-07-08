# Enterprise Network Simulation — Config Reference

## IP Addressing Scheme

| Segment                      | VLAN | Subnet              | Gateway           |
|------------------------------|------|---------------------|-------------------|
| HQ – HR                      | 10   | 192.168.10.0/24     | 192.168.10.1      |
| HQ – IT                      | 20   | 192.168.20.0/24     | 192.168.20.1      |
| Branch – Users               | 30   | 192.168.30.0/24     | 192.168.30.1      |
| R1-HQ <-> R2-Branch WAN link | —    | 10.0.0.0/30         | .1 (R1) / .2 (R2) |

DHCP pools exclude the first 10 addresses of each subnet for static/gateway use.

---

## 1. SW1-HQ (2960 Switch)

```
enable
configure terminal
hostname SW1-HQ

vlan 10
 name HR
vlan 20
 name IT
exit

interface range fa0/1-2
 switchport mode access
 switchport access vlan 10
exit

interface range fa0/3-4
 switchport mode access
 switchport access vlan 20
exit

interface gig0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
exit

```

## 2. SW2-Branch (2960 Switch)

Single VLAN at Branch — access ports only, no trunk needed (R2's `gig0/0` is a plain routed interface, not a sub-interface, so it expects **untagged** traffic).

**Important:** the port connecting the switch to R2-Branch must also be explicitly placed in VLAN 30. By default that port sits in VLAN 1, which puts it on a different broadcast domain than your PCs — Branch PCs would fail to reach the gateway and DHCP would fail silently. This uplink port is easy to forget because it looks like it "just needs a cable," but it still needs a VLAN assignment like every other access port.

```
enable
configure terminal
hostname SW2-Branch

vlan 30
 name BRANCH-USERS
exit

interface range fa0/1-2
 switchport mode access
 switchport access vlan 30
exit

! Uplink to R2-Branch — must be in VLAN 30 too, since R2's interface is untagged
interface gig0/1
 switchport mode access
 switchport access vlan 30
exit
```

## 3. R1-HQ (1941/2911 Router) — Router-on-a-Stick + OSPF + DHCP + ACL

```
enable
configure terminal
hostname R1-HQ

interface gig0/0
 no shutdown
exit

interface gig0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
exit

interface gig0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
exit

! WAN link to Branch (adjust interface name to what you actually connect, e.g. gig0/1 or s0/0/0)
interface gig0/1
 ip address 10.0.0.1 255.255.255.252
 no shutdown
exit

! --- OSPF ---
router ospf 1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 10.0.0.0 0.0.0.3 area 0
exit

! --- DHCP ---
ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.20.1 192.168.20.10

ip dhcp pool HR-POOL
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
exit

ip dhcp pool IT-POOL
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8
exit

! --- ACL: block Branch users from reaching the HR subnet, allow everything else ---
ip access-list extended BLOCK-BRANCH-TO-HR
 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 permit ip any any
exit

! Apply inbound on the interface facing the WAN (traffic coming FROM branch)
interface gig0/1
 ip access-group BLOCK-BRANCH-TO-HR in
exit
```

## 4. R2-Branch (1941/2911 Router) — OSPF + DHCP

```
enable
configure terminal
hostname R2-Branch

interface gig0/0
 ip address 192.168.30.1 255.255.255.0
 no shutdown
exit

interface gig0/1
 ip address 10.0.0.2 255.255.255.252
 no shutdown
exit

router ospf 1
 network 192.168.30.0 0.0.0.255 area 0
 network 10.0.0.0 0.0.0.3 area 0
exit

ip dhcp excluded-address 192.168.30.1 192.168.30.10

ip dhcp pool BRANCH-POOL
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 8.8.8.8
exit
```

## 5. PC Configuration

Set each PC's IP config to **DHCP** (not static) — this proves our DHCP pools actually work.

---

## Verification Checklist 

Run on **R1-HQ** and **R2-Branch**:
```
show ip route          ! confirm routes marked 'O' (OSPF) from the other site
show ip ospf neighbor   ! confirm adjacency is FULL between R1 and R2
show vlan brief         ! (on switches) confirm ports are in the right VLANs
show ip interface brief ! confirm all interfaces are 'up/up'
```

From an **HR PC**:
```
ping 192.168.20.x   ! should succeed (inter-VLAN routing works)
ping 192.168.30.x   ! should succeed (OSPF reaches Branch)
```

From a **Branch PC**:
```
ping 192.168.10.x   ! should FAIL (ACL is blocking Branch -> HR)
ping 192.168.20.x   ! should succeed (ACL only blocks the HR subnet)
```

---