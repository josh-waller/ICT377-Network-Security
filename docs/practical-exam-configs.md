# Practical Exam — Security Policy Implementation

Seven security configuration tickets implementing ACLs, zone-based firewall policies, NAT, and DMVPN fixes across the multi-site topology. All commands are paste-ready for IOS / ASA CLI.

---

## Ticket 1 — RS2-CBAC-FW1: Telnet ACL for Host Ranges

Granular ACL permitting telnet from specific host ranges on 10.7.5.0/24, plus ICMP to the management gateway.

```
! RS2-CBAC-FW1
conf t
!
ip access-list extended ACL-RS2-INSIDE
 remark *** Allow telnet from hosts .20-.23 ***
 permit tcp 10.7.5.20 0.0.0.3 any eq telnet
 remark *** Allow telnet from hosts .24-.27 ***
 permit tcp 10.7.5.24 0.0.0.3 any eq telnet
 remark *** Allow telnet from hosts .28-.29 ***
 permit tcp 10.7.5.28 0.0.0.1 any eq telnet
 remark *** Allow telnet from host .30 ***
 permit tcp host 10.7.5.30 any eq telnet
 remark *** Deny telnet from all other hosts ***
 deny   tcp 10.7.5.0 0.0.0.255 any eq telnet
 remark *** Allow ICMP to management gateway ***
 permit icmp 10.7.5.0 0.0.0.255 host 10.6.32.1
 deny   ip any any log
!
interface Ethernet0/1
 ip access-group ACL-RS2-INSIDE in
!
end
wr
```

---

## Ticket 2 — HQ-ASA-FW1: DMZ Guest Network ACL

Allow guest VLAN (172.16.99.0/24) to ping and telnet to the DMZ server only.

```
! HQ-ASA-FW1 (ASA syntax — uses subnet masks, not wildcards)
conf t
!
access-list DMZ-GUEST-ACL extended permit icmp 172.16.99.0 255.255.255.0 host 172.16.10.10 echo
access-list DMZ-GUEST-ACL extended permit tcp 172.16.99.0 255.255.255.0 host 172.16.10.10 eq telnet
access-list DMZ-GUEST-ACL extended deny   ip any any log
!
access-group DMZ-GUEST-ACL in interface DMZ-GUEST-LAN
!
end
wr
```

---

## Ticket 3 — HQ-ASA-FW1: Inbound Access to DMZ Server

Create network object for the DMZ server, configure static NAT so it is reachable from outside, and add ACL entries for inbound telnet/ICMP through both ISP links.

```
! HQ-ASA-FW1
conf t
!
! Step 1: Define the DMZ server network object
object network DMZ-SRV1-A
 host 172.16.10.10
!
! Step 2: Static NAT — map DMZ server to an outside address
! (Adjust the mapped address to match your NAT pool)
object network DMZ-SRV1-A
 nat (DMZ-SRVS-LAN,OUTSIDE-1) static 11.11.11.150
!
! Step 3: Permit inbound access through ISP101 (OUTSIDE-1)
access-list FROM_ISP101 extended permit tcp any4 host 172.16.10.10 eq telnet
access-list FROM_ISP101 extended permit icmp any4 host 172.16.10.10 echo
!
! Step 4: Permit inbound access through ISP103 (OUTSIDE-2)
access-list FROM_ISP103 extended permit tcp any4 host 172.16.10.10 eq telnet
access-list FROM_ISP103 extended permit icmp any4 host 172.16.10.10 echo
!
end
wr
```

---

## Ticket 4 — RO-ZBPF-FW1: Zone-Based Policy for VLAN 101 (Hosts 128-255)

Configure ZBPF zone-pair allowing hosts 128-255 on 10.7.1.0/24 to access the internet with limited protocols. Assumes CM-LIMITED-PROTOCOLS and CM-ALLOW-ALL-PROTOCOLS class-maps already exist.

```
! RO-ZBPF-FW1
conf t
!
! Step 1: ACL matching hosts 128-255
ip access-list extended FROM-INSIDE-HOSTS-128-255-ACL
 remark *** Hosts 128 to 255 on 10.7.1.0/24 ***
 permit ip 10.7.1.128 0.0.0.127 any
 deny   ip any any
!
! Step 2: Class-map matching limited protocols AND host range
class-map type inspect match-all CM-ALLOW-LIMITED-PROTOCOLS
 match class-map CM-LIMITED-PROTOCOLS
 match access-group name FROM-INSIDE-HOSTS-128-255-ACL
!
! Step 3: Policy-map with inspect actions
policy-map type inspect PM-INSIDE-TO-INTERNET
 class type inspect CM-ALLOW-ALL-PROTOCOLS
  inspect
 class type inspect CM-ALLOW-LIMITED-PROTOCOLS
  inspect
 class class-default
  drop log
!
! Step 4: Assign VLAN 101 subinterface to INSIDE zone with NAT
interface Ethernet0/2.101
 zone-member security INSIDE
 ip nat inside
!
! Step 5: Create zone-pair and apply service policy
zone-pair security ZP-INSIDE-TO-INTERNET source INSIDE destination OUTSIDE
 service-policy type inspect PM-INSIDE-TO-INTERNET
!
end
wr
```

---

## Ticket 5 — RS1-CBAC-FW1 / RO-ZBPF-FW1: DMVPN NHRP Fix + ACL Update

Fix NHRP network ID on RS1 spoke tunnel, update NHRP authentication on RO, and remove telnet access to RO server from RS1 inside ACL.

```
! RS1-CBAC-FW1 — Fix NHRP network-id to match hub (10001)
conf t
!
interface Tunnel11
 no shutdown
 ip nhrp network-id 10001
!
end
wr
```

```
! RO-ZBPF-FW1 — Update NHRP authentication
conf t
!
interface Tunnel11
 ip nhrp authentication <REDACTED>
!
end
wr
```

```
! RS1-CBAC-FW1 — Remove telnet access to RO server
conf t
!
ip access-list extended FROM-INSIDE-ACL
 no permit tcp 10.7.4.0 0.0.0.255 host 10.7.0.100 eq telnet
!
end
wr
```

---

## Ticket 6 — HQ-INT-L3-SW1 / RS1-CBAC-FW1: VL50 ACL + Bidirectional RS1-HQ Policy

Open VLAN 50 for general traffic. Configure RS1 inside ACL for bidirectional access between RS1 (10.7.4.0/24) and HQ VLAN 100 (10.6.100.0/24). Fix CBAC inspection direction.

```
! HQ-INT-L3-SW1 — Open VL50 for general traffic
conf t
!
no ip access-list extended VL50-ACL
ip access-list extended VL50-ACL
 permit tcp any any
 permit icmp any any
 permit udp any any
 deny   ip any any log
!
interface Vlan50
 ip access-group VL50-ACL in
!
end
wr
```

```
! RS1-CBAC-FW1 — Full bidirectional ACL for RS1 inside interface
conf t
!
no ip access-list extended FROM-INSIDE-ACL
ip access-list extended FROM-INSIDE-ACL
 remark *** RS1 to HQ VL100 ***
 permit icmp 10.7.4.0 0.0.0.255 10.6.100.0 0.0.0.255 echo
 permit tcp 10.7.4.0 0.0.0.255 10.6.100.0 0.0.0.255 eq 22
 permit tcp 10.7.4.0 0.0.0.255 10.6.100.0 0.0.0.255 eq telnet
 permit tcp 10.7.4.0 0.0.0.255 10.6.100.0 0.0.0.255 eq www
 remark *** RS1 to RO server ***
 permit icmp 10.7.4.0 0.0.0.255 host 10.7.0.100 echo
 permit tcp 10.7.4.0 0.0.0.255 host 10.7.0.100 eq 22
 remark *** HQ VL100 to RS1 (return + initiated) ***
 permit icmp 10.6.100.0 0.0.0.255 10.7.4.0 0.0.0.255
 permit tcp 10.6.100.0 0.0.0.255 10.7.4.0 0.0.0.255 eq 22
 permit tcp 10.6.100.0 0.0.0.255 10.7.4.0 0.0.0.255 eq telnet
 permit tcp 10.6.100.0 0.0.0.255 10.7.4.0 0.0.0.255 eq www
 remark *** Block other internal/private destinations ***
 deny   ip 10.7.0.0 0.0.255.255 10.0.0.0 0.255.255.255
 deny   ip 10.7.0.0 0.0.255.255 172.16.0.0 0.15.255.255
 deny   ip 10.7.0.0 0.0.255.255 192.168.0.0 0.0.255.255
 remark *** Allow ICMP to internet ***
 permit icmp 10.7.4.0 0.0.0.255 any echo
 deny   ip any any log
!
interface Ethernet0/2
 ip access-group FROM-INSIDE-ACL in
!
! Fix CBAC inspection direction — inspect outbound on inside interface
! so return traffic is automatically permitted
interface Ethernet0/2
 no ip inspect FROM_INSIDE in
 ip inspect FROM_INSIDE out
!
end
wr
```

---

## Ticket 7 — HQ-INT-L3-SW1: VL50 + VL100 Granular ACLs

Replace blanket permits with specific rules. VL50 gets SSH to RO server and inter-VLAN to VL100. VL100 gets full outbound access.

```
! HQ-INT-L3-SW1
conf t
!
! VLAN 50 — SSH to RO server + inter-VLAN to VL100
no ip access-list extended VL50-ACL
ip access-list extended VL50-ACL
 remark *** Allow ping + SSH to RO server ***
 permit icmp 10.6.50.0 0.0.0.255 host 10.7.0.100 echo
 permit tcp 10.6.50.0 0.0.0.255 host 10.7.0.100 eq 22
 remark *** Allow inter-VLAN to VL100 ***
 permit icmp 10.6.50.0 0.0.0.255 10.6.100.0 0.0.0.255 echo
 permit tcp 10.6.50.0 0.0.0.255 10.6.100.0 0.0.0.255 eq 22
 permit udp 10.6.50.0 0.0.0.255 10.6.100.0 0.0.0.255
 deny   ip any any log
!
interface Vlan50
 ip access-group VL50-ACL in
!
! VLAN 100 — Full outbound access
no ip access-list extended VL100-ACL
ip access-list extended VL100-ACL
 permit icmp 10.6.100.0 0.0.0.255 any
 permit tcp 10.6.100.0 0.0.0.255 any
 permit udp 10.6.100.0 0.0.0.255 any
 deny   ip any any log
!
interface Vlan100
 ip access-group VL100-ACL in
!
end
wr
```
