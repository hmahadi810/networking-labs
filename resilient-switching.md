# Resilient switching & recovery

**Solution design — device validation pending.**

LACP · trunk consistency · STP root control · link-failure test

Design a two-link EtherChannel between access switches. Repair passive/passive negotiation, missing VLAN membership and a native-VLAN mismatch; deliberately choose the primary STP root. Test service continuity when one physical bundle member fails.

**Logical path:** PC-A · VLAN 20 → S1 · Po1 / Gi0/1–2 → S2 · Po1 / Gi0/1–2 → PC-B · VLAN 20

## 01 / Network plan

| Component | Design |
| --- | --- |
| Links | S1 Gi0/1 ↔ S2 Gi0/1; S1 Gi0/2 ↔ S2 Gi0/2. Match member speed/duplex and local settings. |
| Endpoints | PC-A on S1 Fa0/1: 192.168.20.11/24; PC-B on S2 Fa0/1: 192.168.20.12/24. Same-VLAN test; no router required. |
| Management | S1 VLAN 10 SVI: 192.168.10.11/24; S2: 192.168.10.12/24. |
| STP design | S1 priority 4096; S2 priority 8192 for VLANs 10 and 20. Native VLAN 99 on both peers. Two physical members form one logical STP port. |

## 02 / Fault isolation & solution

| Designed symptom | Diagnosis | Resolution |
| --- | --- | --- |
| Links up, no bundle | show etherchannel summary; inspect LACP mode | Make S1 active and S2 passive or active; passive/passive does not start negotiation. |
| Bundle formed, VLAN 20 fails | show interfaces trunk; compare allowed VLANs | Carry VLAN 20 on both Port-channels and keep local member settings consistent. |
| Native mismatch / wrong root | Compare native VLANs; show spanning-tree vlan 20 | Set native VLAN 99 on both sides. Use a lower valid S1 bridge priority. Native mismatch is checked separately from LACP negotiation. |

## 03 / Worked configuration

Commands below describe the target configuration. Expand each device to inspect the solution.

### S1 · complete clean-lab configuration

```text
enable
configure terminal
hostname S1
vlan 10
 name MANAGEMENT
exit
vlan 20
 name USERS
exit
vlan 99
 name NATIVE_UNUSED
exit
interface range gigabitEthernet0/1 - 2
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,99
 channel-group 1 mode active
 no shutdown
exit
interface port-channel1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,99
exit
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20 priority 4096
interface fastEthernet0/1
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
exit
interface vlan 10
 ip address 192.168.10.11 255.255.255.0
 no shutdown
end
copy running-config startup-config
```

### S2 · complete clean-lab configuration

```text
enable
configure terminal
hostname S2
vlan 10
 name MANAGEMENT
exit
vlan 20
 name USERS
exit
vlan 99
 name NATIVE_UNUSED
exit
interface range gigabitEthernet0/1 - 2
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,99
 channel-group 1 mode passive
 no shutdown
exit
interface port-channel1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,99
exit
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20 priority 8192
interface fastEthernet0/1
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
exit
interface vlan 10
 ip address 192.168.10.12 255.255.255.0
 no shutdown
end
copy running-config startup-config
```

### Failure injection and restoration on S1

```text
enable
configure terminal
interface gigabitEthernet0/1
 shutdown
end
! Run the tests while Gi0/2 remains connected.
configure terminal
interface gigabitEthernet0/1
 no shutdown
end
```

## 04 / Acceptance tests

Expected results, not captured execution output. Add actual before/after evidence after a personal lab run.

| Test | Verification action | Expected condition |
| --- | --- | --- |
| Bundle health | show etherchannel summary | Po1 is Layer 2 / in use (commonly SU); both links bundled (commonly P). |
| VLAN / root | show interfaces trunk; show spanning-tree vlan 20 | Both peers carry VLANs 10,20,99; native VLANs match; S1 is root for the designed VLANs. |
| Single-member failure | Run PC-A → PC-B ping; shut S1 Gi0/1; inspect Po1 and repeat ping | Po1 remains operational through Gi0/2; traffic can continue after any transient loss. Available aggregate capacity is reduced. |
| Restoration | no shutdown on Gi0/1; show etherchannel summary | Both member links rejoin the bundle. |

## 05 / Technical reasoning

Port-channel health, VLAN forwarding and STP root selection are separate checks. A surviving bundle member can preserve connectivity, but a two-member bundle does not double the bandwidth of every individual flow.
