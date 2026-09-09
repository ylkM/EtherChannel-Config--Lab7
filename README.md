# EtherChannel + Inter-VLAN L3 Lab (ASW1/ASW2/DSW1/DSW2)

Lab topology built around two access switches (ASW1, ASW2) each dual-homed to
their respective distribution switch (DSW1, DSW2) over an EtherChannel, with
DSW1 and DSW2 connected via a routed (Layer 3) EtherChannel. End hosts and
SVI addresses are pre-configured per the diagram.

## Topology

```
        172.16.1.0/24 (VLAN 1)                         172.16.2.0/24 (VLAN 1)
   PC1 .1                                   SRV1 .1
     \  Fa0/1                                  \ Fa0/1
      \                                          \
      ASW1 --Gig0/1/Gig0/2==LACP(Po1)Trunk==Gig1/0/3/Gig1/0/4-- DSW1        DSW1 G1/0/1 ===L3 Po3=== G1/0/1 DSW2
      /                                                           |  \                                  |
   PC2 .2                                                   Gig1/0/2 x2                             VLAN1 SVI .254
     Fa0/2                                                (10.0.0.0/30, .2)                             |
                                                             uplink/WAN                                DSW2
      SRV1 side:                                                                                          |
      ASW2 --Gig0/1/Gig0/2==PAgP(Po2)Trunk==Gig1/0/3/Gig1/0/4-- DSW2                                Gig1/0/3/Gig1/0/4
```

| Device | Role                  | Management / SVI          |
|--------|-----------------------|----------------------------|
| ASW1   | L2 access switch       | VLAN1 172.16.1.0/24 (hosts .1/.2) |
| DSW1   | L3 distribution switch | VLAN1 SVI 172.16.1.254/24 |
| ASW2   | L2 access switch       | VLAN1 172.16.2.0/24 (SRV1 .1) |
| DSW2   | L3 distribution switch | VLAN1 SVI 172.16.2.254/24 |

**Assumed** transit subnet for the DSW1↔DSW2 routed EtherChannel (not
explicitly labeled on the diagram): `10.1.1.0/30` — DSW1 = `.1`, DSW2 = `.2`.
Adjust to match your actual addressing if different.

---

## 1. ASW1 ↔ DSW1 — Layer 2 EtherChannel (LACP), trunked

### ASW1
```
hostname ASW1
!
interface range GigabitEthernet0/1-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
!
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 1
!
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 1
```

### DSW1 (ASW1-facing side)
```
interface range GigabitEthernet1/0/3-4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
!
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

---

## 2. ASW2 ↔ DSW2 — Layer 2 EtherChannel (PAgP), trunked

### ASW2
```
hostname ASW2
!
interface range GigabitEthernet0/1-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 2 mode desirable
!
interface Port-channel2
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 1
```

### DSW2 (ASW2-facing side)
```
interface range GigabitEthernet1/0/3-4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 2 mode desirable
!
interface Port-channel2
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

> LACP uses **active/passive**; PAgP uses **desirable/auto**. `active` and
> `desirable` are used here on both ends so the channel forms without
> depending on the neighbor's default mode.

---

## 3. DSW1 ↔ DSW2 — Layer 3 EtherChannel (static / "on")

### DSW1
```
interface GigabitEthernet1/0/1
 no switchport
 channel-group 3 mode on
!
interface Port-channel3
 no switchport
 ip address 10.1.1.1 255.255.255.252
!
interface Vlan1
 ip address 172.16.1.254 255.255.255.0
!
ip routing
```

### DSW2
```
interface GigabitEthernet1/0/1
 no switchport
 channel-group 3 mode on
!
interface Port-channel3
 no switchport
 ip address 10.1.1.2 255.255.255.252
!
interface Vlan1
 ip address 172.16.2.254 255.255.255.0
!
ip routing
```

> Static EtherChannel ("mode on") does not run LACP/PAgP negotiation — both
> ends must be configured identically and manually, or the link stays down/
> mismatched.

---

## 4. Routes so PCs can reach SRV1

Simple static routes across the L3 EtherChannel are sufficient for this
two-subnet topology (a routing protocol such as OSPF/EIGRP would also work
and scale better, but isn't required here).

### DSW1
```
ip route 172.16.2.0 255.255.255.0 10.1.1.2
```

### DSW2
```
ip route 172.16.1.0 255.255.255.0 10.1.1.1
```

End devices (PC1, PC2, SRV1) already point their default gateway at their
local VLAN1 SVI (`172.16.1.254` / `172.16.2.254`), so no host-side changes
are needed — traffic from PC1/PC2 hits DSW1, routes over Po3 to DSW2, and
reaches SRV1, and vice versa.

---

## 5. Default EtherChannel load-balancing method

On Cisco Catalyst switches (e.g., 2960/3560/3750 series, as used by ASW/DSW
here), the **default load-balancing algorithm is `src-mac`** (source MAC
address). Verify on any switch with:

```
show etherchannel load-balance
```

---

## 6. Load-balance on source **and** destination IP address

Applied globally on each switch that participates in an EtherChannel
(ASW1, ASW2, DSW1, DSW2):

```
port-channel load-balance src-dst-ip
```

Confirm the change:
```
show etherchannel load-balance
```

---

## Verification commands

```
show etherchannel summary
show interfaces port-channel 1 trunk
show interfaces port-channel 3
show ip route
show ip interface brief
ping 172.16.2.1 source 172.16.1.1   ! from DSW1, to confirm reachability to SRV1
```
