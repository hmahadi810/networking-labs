# Two-site OSPF fault isolation

**Solution design — device validation pending.**

Layer 1/2 checks · OSPF adjacency · forward & return routes

A troubleshooting design for two sites affected by three independent faults: an administratively disabled WAN interface, mismatched OSPF areas and a missing remote LAN advertisement. The solution separates neighbor formation from route propagation.

**Logical path:** Vienna LAN · 192.168.10.0/24 → R1 · 10.0.12.1/30 → R2 · 10.0.12.2/30 → Linz LAN · 192.168.20.0/24

## 01 / Network plan

| Component | Design |
| --- | --- |
| WAN | R1 Gi0/1 ↔ R2 Gi0/1; 10.0.12.0/30; Area 0 |
| Vienna endpoint | PC-A 192.168.10.10/24; gateway 192.168.10.1; LAN to R1 Gi0/0 |
| Linz endpoint | PC-B 192.168.20.10/24; gateway 192.168.20.1; LAN to R2 Gi0/0 |
| Lab assumptions | Blank routers for full configuration below. On an existing faulted lab, first remove conflicting interface addresses and OSPF network statements. Match link masks, timers and MTU. |

## 02 / Fault isolation & solution

| Designed symptom | Diagnosis | Resolution |
| --- | --- | --- |
| No neighbor after startup | show ip interface brief; ping the directly connected WAN IP | Enable the disabled WAN port and confirm both ends are up/up. |
| WAN reachable, adjacency absent | Compare show ip ospf interface and the OSPF network statements | Use Area 0 on both WAN interfaces. Remove an obsolete Area 1 WAN statement before adding the Area 0 statement. |
| FULL neighbor, remote host unreachable | show ip route ospf on both routers; inspect endpoint gateways | Advertise both LANs; verify a forward route and a return route. |

## 03 / Worked configuration

Commands below describe the target configuration. Expand each device to inspect the solution.

### R1 · complete clean-lab configuration

```text
enable
configure terminal
hostname R1
interface gigabitEthernet0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit
interface gigabitEthernet0/1
 ip address 10.0.12.1 255.255.255.252
 no shutdown
exit
router ospf 1
 router-id 1.1.1.1
 network 10.0.12.0 0.0.0.3 area 0
 network 192.168.10.0 0.0.0.255 area 0
 passive-interface gigabitEthernet0/0
end
copy running-config startup-config
```

### R2 · complete clean-lab configuration

```text
enable
configure terminal
hostname R2
interface gigabitEthernet0/0
 ip address 192.168.20.1 255.255.255.0
 no shutdown
exit
interface gigabitEthernet0/1
 ip address 10.0.12.2 255.255.255.252
 no shutdown
exit
router ospf 1
 router-id 2.2.2.2
 network 10.0.12.0 0.0.0.3 area 0
 network 192.168.20.0 0.0.0.255 area 0
 passive-interface gigabitEthernet0/0
end
copy running-config startup-config
```

### Targeted repair on an already faulted R2

```text
enable
configure terminal
router ospf 1
 no network 10.0.12.0 0.0.0.3 area 1
 network 10.0.12.0 0.0.0.3 area 0
 network 192.168.20.0 0.0.0.255 area 0
end
copy running-config startup-config
```

## 04 / Acceptance tests

Expected results, not captured execution output. Add actual before/after evidence after a personal lab run.

| Test | Verification action | Expected condition |
| --- | --- | --- |
| WAN state | show ip interface brief; ping 10.0.12.2 from R1 | Both WAN ports up/up; directly connected peer reachable. |
| Adjacency | show ip ospf neighbor on both routers | The neighbor reaches FULL after convergence. |
| Route propagation | show ip route ospf | R1 learns 192.168.20.0/24 via 10.0.12.2; R2 learns 192.168.10.0/24 via 10.0.12.1. |
| End-to-end / return path | PC-A → PC-B ping; PC-B → PC-A ping | Both directions succeed after ARP and routing convergence. |

## 05 / Technical reasoning

A FULL neighbor is one milestone, not proof of end-to-end service. LAN advertisement, return routes and endpoint gateways must also be verified.
