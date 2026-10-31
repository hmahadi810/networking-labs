# Segmented office network

**Solution design — device validation pending.**

VLANs · DHCP relay · guest access policy

Design a small office network for staff, CCTV and guests. Centralise address allocation in a services VLAN, retain access to DNS and isolate guest clients from internal subnets. Scope is internal connectivity; Internet access, NAT and firewall deployment are not included.

**Logical path:** Staff / CCTV / Guest → SW1 · VLANs 20/30/40 → R1 · 802.1Q gateway → Services · VLAN 10

## 01 / Network plan

| Component | Design |
| --- | --- |
| VLAN 10 / Services | 192.168.10.0/24 · R1 .1 · DHCP .10 · DNS .53 · Web .80 |
| VLAN 20 / Staff | 192.168.20.0/24 · gateway .1 · DHCP .100–.254 |
| VLAN 30 / CCTV | 192.168.30.0/24 · gateway .1 · DHCP .100–.254 |
| VLAN 40 / Guest | 192.168.40.0/24 · gateway .1 · DHCP .100–.254 |
| Physical connections | R1 Gi0/0 ↔ SW1 Gi0/1. Staff Fa0/1; camera/test PC Fa0/2; guest Fa0/3. DHCP-SRV Gi0/0 ↔ SW1 Fa0/10; DNS ↔ Fa0/11; web ↔ Fa0/12. |

## 02 / Fault isolation & solution

| Designed symptom | Diagnosis | Resolution |
| --- | --- | --- |
| Staff has no lease | Check access VLAN, trunk membership, router subinterface, helper IP and DHCP server return route. | Carry VLAN 20; relay to 192.168.10.10; give the DHCP server a route back through R1. |
| CCTV segment is unreachable | Inspect VLAN 30 on both the switch trunk and router subinterface. | Allow VLAN 30 and match dot1Q tag 30 to its gateway. |
| Guest can reach staff | Inspect ACL entries, their order and interface direction. | Apply GUEST-IN inbound on Gi0/0.40. Permit DHCP renewal and DNS exceptions before the internal-subnet deny. |

## 03 / Worked configuration

Commands below describe the target configuration. Expand each device to inspect the solution.

### R1 · router-on-a-stick + policy

```text
enable
configure terminal
hostname R1
interface gigabitEthernet0/0
 no shutdown
exit
interface gigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
exit
interface gigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip helper-address 192.168.10.10
exit
interface gigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip helper-address 192.168.10.10
exit
interface gigabitEthernet0/0.40
 encapsulation dot1Q 40
 ip address 192.168.40.1 255.255.255.0
 ip helper-address 192.168.10.10
exit
ip access-list extended GUEST-IN
 permit udp 192.168.40.0 0.0.0.255 eq bootpc host 192.168.10.10 eq bootps
 permit udp 192.168.40.0 0.0.0.255 host 192.168.10.53 eq domain
 permit tcp 192.168.40.0 0.0.0.255 host 192.168.10.53 eq domain
 deny ip 192.168.40.0 0.0.0.255 192.168.0.0 0.0.255.255
 permit ip any any
exit
interface gigabitEthernet0/0.40
 ip access-group GUEST-IN in
end
copy running-config startup-config
```

### SW1 · VLAN membership + trunk

```text
enable
configure terminal
hostname SW1
vlan 10
 name SERVICES
exit
vlan 20
 name STAFF
exit
vlan 30
 name CCTV
exit
vlan 40
 name GUEST
exit
interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 no shutdown
exit
interface fastEthernet0/1
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
exit
interface fastEthernet0/2
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown
exit
interface fastEthernet0/3
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shutdown
exit
interface fastEthernet0/10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
exit
interface fastEthernet0/11
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
exit
interface fastEthernet0/12
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
exit
end
copy running-config startup-config
```

### DHCP-SRV · IOS router used as central DHCP service

```text
enable
configure terminal
hostname DHCP-SRV
interface gigabitEthernet0/0
 ip address 192.168.10.10 255.255.255.0
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 192.168.10.1
ip dhcp excluded-address 192.168.20.1 192.168.20.99
ip dhcp pool STAFF
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 192.168.10.53
exit
ip dhcp excluded-address 192.168.30.1 192.168.30.99
ip dhcp pool CCTV
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 192.168.10.53
exit
ip dhcp excluded-address 192.168.40.1 192.168.40.99
ip dhcp pool GUEST
 network 192.168.40.0 255.255.255.0
 default-router 192.168.40.1
 dns-server 192.168.10.53
exit
end
copy running-config startup-config
```

### Endpoint / DNS service setup

```text
DNS host: 192.168.10.53/24; gateway 192.168.10.1. Enable DNS service.
Create A record: portal.lab.test -> 192.168.10.80.
Web host: 192.168.10.80/24; gateway 192.168.10.1. Enable HTTP for browser test.
Staff, camera-test PC and guest: DHCP enabled.
Use only these example addresses in an isolated lab.
```

## 04 / Acceptance tests

Expected results, not captured execution output. Add actual before/after evidence after a personal lab run.

| Test | Verification action | Expected condition |
| --- | --- | --- |
| DHCP leases | show ip dhcp binding on DHCP-SRV; inspect each client IP, mask, gateway and DNS | Each endpoint receives its own VLAN subnet and DNS 192.168.10.53. |
| Staff services | From staff: ping 192.168.10.80; nslookup portal.lab.test; open http://portal.lab.test | IP and name resolution work; the web page loads when HTTP is enabled. |
| Guest isolation | From a guest with a valid lease: ping a staff PC; test DNS separately | Guest-to-staff traffic is denied; DNS to .53 remains allowed. An unreachable guest is not evidence of an effective ACL. |
| Renewal / policy counters | Renew guest lease; inspect show access-lists GUEST-IN | DHCP renewal is allowed by the specific UDP rule; relevant permit/deny counters rise during tests. |

## 05 / Technical reasoning

DHCP across VLANs needs both a working relay path and a server return path. An isolation policy needs explicit service exceptions; a blanket deny can break renewal or DNS.
