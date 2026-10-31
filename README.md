# Networking Labs & Solution Designs

**Mahadi Hassan Shova · IT Support & Networking · Vienna, Austria**

A documented collection of CCNA learning labs and integrated network solution designs. The focus is on explaining the problem, isolating the fault, choosing the configuration and defining how the result should be verified.

[Portfolio](https://hmahadi810.github.io/) · [Detailed visual case studies](https://hmahadi810.github.io/network-case-studies.html) · [LinkedIn](https://www.linkedin.com/in/hmahadi810/)

## Integrated case studies

| Case | Technical scope | Documentation |
| --- | --- | --- |
| Segmented office | VLANs, 802.1Q, central DHCP relay, guest ACL and DNS exceptions | [Office network](office-network.md) |
| Two-site routing | WAN state, OSPF area consistency, LAN advertisement and return routes | [OSPF troubleshooting](ospf-troubleshooting.md) |
| Resilient switching | LACP negotiation, trunk/native VLAN consistency, STP root and member-link recovery | [Switching & recovery](resilient-switching.md) |

## Guided CCNA practice

The [seven-lab portfolio](ccna-seven-labs.pdf) documents port security, EtherChannel, STP, VLAN/inter-VLAN routing, OSPF, static/default routing and DNS/DHCP troubleshooting.

## Evidence status

- **Guided practice:** screenshots document learning scenarios and objectives, including initial fault states.
- **Solution designs:** target configurations and acceptance tests are documented. Real-device or emulator validation is pending.
- Expected outcomes are not represented as captured device output or a passed certification exam.

## Using the configurations

Use isolated, clean lab devices. Match interface names and supported commands to your platform. Follow the address plan and endpoint prerequisites in each case study. Capture the actual configuration, before/after diagnostics and endpoint tests from your own run.

The examples use private lab addressing. They are learning designs, not a production deployment template.
