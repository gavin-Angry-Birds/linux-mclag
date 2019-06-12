﻿# MC-LAG high level design for SONiC

<!-- TOC -->

- [MC-LAG high level design for SONiC](#mc-lag-high-level-design-for-sonic)
- [1. Document History](#1-document-history)
- [2. Abbreviations](#2-abbreviations)
- [3. Terminologies](#3-terminologies)
- [4. Design considerations](#4-design-considerations)
- [5. Brief introduction of ICCPd](#5-brief-introduction-of-iccpd)
  - [5.1. Use-cases supported by ICCP lite](#51-use-cases-supported-by-iccp-lite)
  - [5.2. ICCPd State machines](#52-iccpd-state-machines)
  - [5.3. Role election](#53-role-election)
  - [5.4. Information sync up with ICCP](#54-information-sync-up-with-iccp)
  - [5.5. ICCP Heartbeat sanity check](#55-iccp-heartbeat-sanity-check)
  - [5.6. ICCP consistence check](#56-iccp-consistence-check)
- [6. Typical configurations](#6-typical-configurations)
  - [6.1. Syntax of configuration](#61-syntax-of-configuration)
  - [6.2. MC-LAG L3 scenario configuration](#62-mc-lag-l3-scenario-configuration)
  - [6.3. MC-LAG L2 scenario configuration](#63-mc-lag-l2-scenario-configuration)
- [7. Typical Data diagram for MCLAG](#7-typical-data-diagram-for-mclag)
  - [7.1. MC-LAG L3 scenario](#71-mc-lag-l3-scenario)
    - [7.1.1. MC-LAG enabled interface up](#711-mc-lag-enabled-interface-up)
    - [7.1.2. MC-LAG enabled interface down](#712-mc-lag-enabled-interface-down)
    - [7.1.3. Link between peers](#713-link-between-peers)
    - [7.1.4. ARP sync-up between MC-LAG peers](#714-arp-sync-up-between-mc-lag-peers)
    - [7.1.5. L3 multicast](#715-l3-multicast)
  - [7.2. MC-LAG L2 scenario](#72-mc-lag-l2-scenario)
    - [7.2.1. MC-LAG enabled interface up](#721-mc-lag-enabled-interface-up)
    - [7.2.2. MC-LAG enabled interface down](#722-mc-lag-enabled-interface-down)
    - [7.2.3. Link between peers](#723-link-between-peers)
    - [7.2.4. MAC sync-up between MC-LAG peers](#724-mac-sync-up-between-mc-lag-peers)
    - [7.2.5. Peer link MAC learning](#725-peer-link-mac-learning)
    - [7.2.6. L2 BUM flooding and port isolation](#726-l2-bum-flooding-and-port-isolation)
  - [7.3. Peer connection down](#73-peer-connection-down)
  - [7.4. Peer link down](#74-peer-link-down)
  - [7.5. Peer link is Vxlan tunnel](#75-peer-link-is-vxlan-tunnel)
- [8. SONiC system diagram for MCLAG](#8-sonic-system-diagram-for-mclag)
- [9. Design changes](#9-design-changes)
  - [9.1. The schema changes](#91-the-schema-changes)
    - [9.1.1. Add MCLAG configuration](#911-add-mclag-configuration)
    - [9.1.2. Add Acl_table and Acl_rule_table in app-db](#912-add-acltable-and-aclruletable-in-app-db)
  - [9.2. Add mclagsyncd process](#92-add-mclagsyncd-process)
  - [9.3. CLI design](#93-cli-design)
  - [9.4. aclorch changes](#94-aclorch-changes)
  - [9.5. portmgr changes](#95-portmgr-changes)
  - [9.6. portsorch changes](#96-portsorch-changes)
  - [9.7. intfmgr changes](#97-intfmgr-changes)
  - [9.8. intfsorch changes](#98-intfsorch-changes)
  - [9.9. routeorch changes](#99-routeorch-changes)
  - [9.10. fdborch changes](#910-fdborch-changes)
  - [9.11. vxlanmgr changes](#911-vxlanmgr-changes)
  - [9.12. vxlanorch changes](#912-vxlanorch-changes)
  - [9.13. vnetorch changes](#913-vnetorch-changes)
  - [9.14. warm-reboot consideration](#914-warm-reboot-consideration)
  - [9.15. teammgr changes](#915-teammgr-changes)

<!-- /TOC -->

# 1. Document History

| Version | Date       | Author                 | Description                                      |
|---------|------------|------------------------|--------------------------------------------------|
| v.01    | 07/30/2018 |jian                    | Initial version from nephos                      |
| v.02    | 10/19/2018 |jian/shine/jeffrey      | Revised per review meeting with SONIC community  |
| v.03    | 10/27/2018 |jeffrey                 | Minor update to clarify the behavior.            |
| v.04    | 03/01/2019 |jian/shine/jeffrey      | Revised per review meeting with SONIC community  |
| v.05    | 04/26/2019 |jian                    | Add pull request on chapter 9                    |
| v.06    | 06/10/2019 |jian,jeffrey,shine            | Add L2 forwarding description and more use cases.|

# 2. Abbreviations

|**Term** |**Definition**                                    |
|---------|--------------------------------------------------|
|MCLAG  |Multi-Chassis Link Aggregation Group              |
|ICCP  |Inter-Chassis Communication Protocol              |
|FDB     |Layer-2 (MAC) based forwarding table              |

# 3. Terminologies

![Diagram 3](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_3.png)
Diagram 3

# 4. Design considerations

- MCLAG domain consists of only two systems.
- Each system only join one MC-LAG domain
- Supports Known Unicast and BUM traffic
- L3 interface on MLAG ports will have vMAC generated from VRRP algorithm using the same IP address assigned to the L3 LIF (logical interface)；(Not supported currently)
- ARP resolution and ARP response packet sync-up between MC-LAG peers
- FDB sync-up between MC-LAG peers
- Support MC-LAG routed port or MC-LAG L2 port joining L3 vlan interface

> [caveat: vlan interface does not go down when the last port in the
> vlan goes down, need further discussion in the community with
> regarding the 1Q or 1D model]

# 5. Brief introduction of ICCPd

- ICCP(Inter-Chassis Communication Protocol) is defined in RFC7275，our MLAG control plane implements the light version of ICCP. RFC7275 has very complicated state machines, information to be synced up between peers are very heavy, sanity checks are accordingly too many. So-called Lite Version is that MLAG ICCPd only implements a subset of the state machines defined in RFC7275 without compromising the usefulness and integrity.

- ICCP protocol using TCP port 8888 to make connection between peers, ICCP lite will do configuration consistence check, sync up ARP table and MAC address of related interfaces.

## 5.1. Use-cases supported by ICCP lite

RFC7275 describes 4 use-cases, this ICCPd or ICCP lite support all of these use-cases:

![Diagram 5.1.1](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_5.1.1.png)
In the scenario of figure 2, the PEs within an RG (Redundancy Group) are co-located in the same physical location, e.g., point of presence (POP) or central office (CO). Furthermore, dedicated links provide the interconnect for ICCP among the PEs.

![Diagram 5.1.2](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_5.1.2.png)
In the scenario of figure 3, the PEs within an RG (Redundancy Group) are co-located in the same physical location (POP, CO). However, unlike the previous scenario, there are no dedicated links between the PEs. The interconnect for ICCP is provided through the core network to which the PEs are connected. Figure 3 depicts this model.

![Diagram 5.1.3](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_5.1.3.png)
In the scenario of figure 4, the PEs within an RG (Redundancy Group) are located in different physical locations to provide geographic redundancy. A dedicated interconnect is provided to link the PEs. The resiliency mechanisms for the interconnect are similar to those highlighted in the co-located interconnect counterpart.

![Diagram 5.1.4](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_5.1.4.png)
In the scenario of figure 5, the PEs of an RG (Redundancy Group) are located in different physical locations and the interconnect for ICCP is provided over the PSN network to which the PEs are connected. This interconnect option is more likely to be the one used for geo-redundancy, as it is more economically appealing compared to the geo-redundant dedicated interconnect option.

RFC7275 says that information sync and heartbeat checking can take a path that is different from the one for carrying data traffic.
Will MC-LAG work if there is no physical connection between the peers? In theory, if the peer IP address is reachable, there is no requirement to have physical connection between the peers. For example, the peer-link can be ais Vxlan tunnel. ICCP may use this VXLAN tunnel if user configured the system properly.

## 5.2. ICCPd State machines

Lite version support complete “ICCP Connection State Machine”，shown as followings:
The ICCP Connection state machine is defined to have six states, as below. Please note although LDP is mentioned in the below section, it’s not used for ICCPd.

- NONEXISTENT: This state is the starting point for the state machine. It indicates that no ICCP connection exists and that there’s no LDP session established between the PEs.

- INITIALIZED: This state indicates that an LDP session exists between the PEs but LDP ICCP capability information has not yet been exchanged between them.

- CAPSENT: This state indicates that an LDP session exists between the PEs and that the local PE has advertised LDP ICCP capability to its peer.

- CAPREC: This state indicates that an LDP session exists between the PEs and that the local PE has both received and advertised LDP ICCP capability from/to its peer.

- CONNECTING: This state indicates that the local PE has initiated an ICCP connection to its peer and is awaiting its response.

- OPERATIONAL: This state indicates that the ICCP connection is operational.

![Diagram 5.2](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_5.2.png)
Diagram 5.2

RFC7275 also describes so called “Application Connection State Machine”, which are designed for application exclusively, and each application maintains its own state machines.

In this version, when ICCP state machine becomes OPERATIONAL, application state machine will be OPERATIONAL immediately.

In RFC7275 original requirements, ICCP will sync up many kinds of information with LACP protocol etc. This lite version does not implement the connection between ICCP and teamd to avoid the modification in teamd, there is no information exchange between ICCPd and teamd neither. ICCP work with teamd indirectly, for example, when ICCPd detects the change of MAC address for any MLAG member portchannel, ICCPd will inform Linux kernel and teamd will be hence notified by kernel.  In this way, teamd can notify LACP so that LACP in the server will form teamd interface.

## 5.3. Role election

ICCP peer must be in active mode or standby mode. The ipv4 address is used to determine which role the peer has Local ip and remote ip is configured manually, system compares these two ip addresses to determine its role, the one with large IP address is the standby, and the other one is active. Active is the client, standby is the server. The client connect to the server actively.

The role of active or standby is the concept of control plane. In the data plane, each peer determines the data forward path individually, no matter the role it is.

Another way to specify the active/standby role is via static configuration from config db. Currently static configuration from config db is not supported, may support it in the future.

## 5.4. Information sync up with ICCP

Following information are sync'ed up between ICCP lite peers.

1 system configuration：Sync up system MAC address. This is to serve LACP to derive the correct system ID. When the standby receives a system MAC from active, it will change local system ID to the peer system MAC.

2 Aggregator configuration: Sync up AGG_ID (portchannel ID, name, MAC etc) of MC-LAG enabled PortChannel, this is to record portchannel information of each peer. This info is used for sanity check as well as enabling packet forwarding in the peer link when a member link of MC-LAG goes down.

3 Aggregator State: Mainly exchanges MC-LAG enabled portchannel state (up/down) with AGG_ID. Generally, if the same named portchannels in each peer are both up, the data forwarding in the peer-link for that portchannel is disabled to prevent the data stream pass-through. When a portchannel of one peer is down, the peer-link is enabled for that portchannel to let the data stream pass-through.

4 ARP information: ARP entries learned from kernel will be synced up with peer. The peer will update kernel as dynamically learned ARP once the ARP is synced via ICCP.

5 FDB information: FDB entries learned from MC-LAG enabled port-channel will be synced up with peer.

ICCP lite only supports port-configuration/state sync up for MC-LAG enabled ports. We don’t sync the static IP address, if a static IP address is configured on MC-LAG enabled PortChannel in one peer, the same IP address must be configured on the same PortChannel in another peer.

## 5.5. ICCP Heartbeat sanity check

RFC7275 does not specify how to do the heartbeat check, but just suggest to use BFD or IP reachability monitoring to check if peer is reachable. This ICCP lite version defines a proprietary heartbeat message sent every 1s. If no such message is received within 15 consecutive intervals, then peer is declared as lost.

## 5.6. ICCP consistence check

This lite version does consistence check for the following contents:
1 Peer IP, local IP will be checked against with message contents.
2 Enable MC-LAG portchannel interface check: if it is L3 interface, then the IP address assigned must be the same. If they join a vlan, then the vlan must be the same etc.
3 Peer-link in both devices must be the same type.

# 6. Typical configurations

## 6.1. Syntax of configuration

```bash
"MC_LAG": {
    "{{MC-LAG_domain_id}}": {
        "local_ip": {{ipv4_address}},
        "peer_link": {{interface_name}},
        "peer_ip": {{ipv4_address}},
        "mclag_interface": {{interface_name}}
    }
}

MC-LAG domain ID, must be from 1 to 65535.
"local_ip" is the ip address of this device to set TCP connection.
"peer_ip" is the ip address of peer device to set TCP connection.
"peer_link" is the name of interface that act as interconnection. For L3 scenario, this is optional.
"mclag_interface" is the name of PortChannel interfaces that has MC-LAG enabled, it can be a list of portChannel interfaces separated by commas ','.
```

## 6.2. MC-LAG L3 scenario configuration

![Diagram 6.2](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_6.2.png)
Diagram 6.2

In the L3 scenario, peer-link PortChannel0002 and MC-LAG enabled interface PortChannel0001 are both L3 interface.

## 6.3. MC-LAG L2 scenario configuration

![Diagram 6.3](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_6.3.png)
Diagram 6.3

In the L2 scenario, peer-link PortChannel0002 and MC-LAG enabled interface PortChannel0001 are both the member of vlan100. PortChannel0002 uses L2 forwarding.

# 7. Typical Data diagram for MCLAG

## 7.1. MC-LAG L3 scenario

### 7.1.1. MC-LAG enabled interface up

![Diagram 7.1.1](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_7.1.1.png)
Diagram 7.1.1

- In the above diagram, PortChannel0001 and PortChannel0002 areis mclag MC-LAG enabled interfaces, status is up,  PortChannel00032 is peer -link.
- The data flow path is presented by the red line.
- The data flow path from PA to CE1: When the traffic reach PEER1, it will match the direct route, such as 10.1.1.0/24, and forwarded through PortChannel0001.
- The data flow path from CE1 to PA: CE1 may send the traffic to PEER1 or PEER2. PEER2 must has route entry that can reach PA. This route entry is installed by routing protocol.

### 7.1.2. MC-LAG enabled interface down

![Diagram 7.1.2](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_7.1.2.png)
Diagram 7.1.2

- In the above diagram, PortChannel0001 is MC-LAG enabled interface, status is down, PortChannel0002 is peer-link.
- The data flow path is presented by the yellow line for traffic from PA to CE1.
- The data flow path from PA to CE1: When PortChannel0001 in PEER1 is down, the direct route 10.1.1.0/24 will be deleted, the routing protocol (or a static route configuration) will make sure there is a backup route reach CE1 via peer -link. When the traffic reach PEER1, it will match the backup route, and forwarded through PortChannel0002.
- The data flow path from CE1 to PA: CE1 will detect the interface connecting to PEER1 is down, the data will send to PA via PEER2. The routing protocol will make sure there is a path to reach PA.

### 7.1.3. Link between peers

- In L3 scenario user needn't configure peer-link. Peer-link is chosen by routing protocol or static configured route.
- The peer-link can be either Ethernet , PortChannel or a Vxlan tunnel. The peer-link can be used to carry data traffic when one of the MC-LAG member link is down.
- routing protocol or static route configured manually provides backup path to reach MC-LAG enabled subnet via peer link.
- In this scenario, the direct-connected L3 peer link connecting the two peer devices is not required. When the MC-LAG member link is up, the direct route has the highest priority. If one of the MC-LAG member link is down, the direct route will be deleted, and the backup route will take effect.

### 7.1.4. ARP sync-up between MC-LAG peers

- If one peer learned an ARP entry, it will send the ARP entry to the other peer via ICCP. For example, PEER1 learned ARP entry of CE1 from PortChannel0001, it will send this ARP to PEER2 via ICCP. PEER2 receives this ARP entry, and install it into Linux kernel, the learned interface name is PortChannel0001. This means the name of MC-LAG enabled PortChannel interface in both peer devices must be the same.
- Unlike with OSPF, ICCP don’t flood ARP entry to peer periodically. To prevent the ARP entry from aging, ICCP uses Netlink Socket to monitor ARP reply received by Linux kernel. For example, when an ARP entry in PEER2 is aged, the Linux kernel will send an ARP request via PortChannel0001. CE1 receives the ARP request, and send back one ARP reply. For CE1, PEER1 and PEER2 are viewed as the same device, the ARP reply may send to PEER2 or PEER1. If PEER2 receives the ARP reply, the ARP entry is learned again and information is updated in the kernel. At the same time, PEER2 will notify PEER1 via ICCP sync message. If PEER1 receives the ARP reply, since the ARP entry already exists in the kernel, kernel will use Netlink to send the ARP packet to its applications, ICCP will collect the ARP information from the ARP reply packet and send to PEER2, so PEER2 can update the ARP entry in the Linux kernel.

### 7.1.5. L3 multicast

- Since the control plane does not support multicast protocols, e.g. IGMP, PIM, so multicast L3 forwarding is not supported.

## 7.2. MC-LAG L2 scenario

### 7.2.1. MC-LAG enabled interface up

![Diagram 7.2.1](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_7.2.1.png)
Diagram 7.2.1

- In the above diagram, PortChannel0001 is MC-LAG enabled port, status is up, PortChannel0002 is peer link. PortChannel0001 and PortChannel0002 are in same VLAN. Just like portChannel0001, PortChannel0002 uses L2 forwarding. Assume PA to PEER1 connection is part of the same VLAN as PortChannel0001.
- The data flow path is presented by the red line.
- The data flow path from PA to CE1: When the traffic reach PEER1, it will match the MAC of CE1, and forwarded through PortChannel0001. If the MAC of CE1 is not learned in PEER1 yet, the traffic will be flooded in the VLAN. PEER2 will also receive the flooded packet, but it will not forward the packet to MC-LAG enabled ports.
- The data flow path from CE1 to PA: CE1 may send the traffic to PEER1 or PEER2. PEER2 must has PA’s MAC entry for unicast forwarding, but ICCPD does not sync up PA’s MAC address because PA to PEER1 connection is not part of the MC-LAG port, so the packet from CE1 to PA will be flooded by PEER2 to PEER1 first, then PEER1 will unicast to PA. In this path, if packet flooding is not desired, PA must connect to PEER2 to form another LAG.
- Peer link data forwarding is disabled for MC-LAG enabled interfaces. In the above diagram, if PEER2 receives traffic from PEER1 via PortChannel0002, the traffic will not be forwarded to PortChannel0001 since port isolation logic is applied.
- MAC learning in peer link is disabled. In the above diagram, MAC learning in PortChannel0002 is disabled in both PEER1 and PEER2.

### 7.2.2. MC-LAG enabled interface down

![Diagram 7.2.2](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_7.2.2.png)
Diagram 7.2.2

- In the above diagram, PortChannel0001 is MC-LAG enabled interface, status is down, PortChannel0002 is the peer link. Assume PA to PEER1 connection is part of the same VLAN as PortChannel0001.
- The data flow path is presented by the yellow line.
- The data flow path from PA to CE1: When PortChannel0001 in PEER1 is down, ICCP reprograms the nexthop of MAC entry to point to peer link PortChannel0002 and make the MAC entry in the ASIC table. When the traffic reaches PEER1, it will match the updated MAC entry, and forwarded through PortChannel0002.
- When PortChannel0001 in PEER1 is down, the portChannel state is synced to PEER2. ICCP in PEER2 enables the peer link forwarding for MC-LAG enabled interfaces corresponding to portChannel0001. In the above diagram, if PEER2 receives traffic from PEER1 via PortChannel0002, the traffic will be forwarded to CE1 via PortChannel0001.
- MAC learning of peer link is disabled. In the above diagram, MAC learning of PortChannel0002 is disabled in both PEER1 and PEER2.
- The data flow path from CE1 to PA: CE1 will detect the interface connecting to PEER1 is down, the data will send to PA via PEER2. PEER2 will do L2 flooding and sends the packet to PEER1 where L2 bridging is performed to reach PA.

### 7.2.3. Link between peers

- In L2 scenario user must configure peer link
- The peer link can be either Ethernet, a PortChannel or a Vxlan tunnel. The peer link is used to carry data traffic when one of the MC-LAG member link is down and MAC forwarding reroute is needed.
- Although data traffic and ICCP control traffic can take different path between the two peers, the suggestion is to use the same physical link to prevent loop.

### 7.2.4. MAC sync-up between MC-LAG peers

- If one peer learned an MAC entry, it will send this MAC to other peer via ICCP. For example, PEER1 learned MAC entry of CE1 from PortChannel0001, it will send this MAC to PEER2 via ICCP. PEER2 receives this MAC, and install the MAC into Linux kernel, the learned interface is also PortChannel0001. This means the name of MC-LAG enabled PortChannel interface in both peer devices must be the same.
- Unlike with OSPF, ICCP don’t flood MAC entry to peer periodically. To prevent the MAC entry from aging, ICCP defines two flags for each MAC entry, MAC_AGE_LOCAL and MAC_AGE_PEER. MAC_AGE_LOCAL indicates the MAC entry in my device is aged, and MAC_AGE_PEER indicates the same MAC entry in peer device is aged. The MAC entry will be deleted from my FDB only when the two flags are both set for this MAC. For example, if the MAC of CE1 aged in PEER2, it will be set MAC_AGE_LOCAL. If this MAC is not set MAC_AGE_PEER flag at the same time (because PEER1 did not aging the MAC entry, hence did not tell PEER2 to set the flag), it will be installed back to the ASIC. Then PEER2 notifies the MAC age event to PEER1, PEER1 will set MAC_AGE_PEER for the same MAC.

### 7.2.5. Peer link MAC learning

- When the MC-LAG enabled interface is up, peer link is the backup link for data traffic. MAC learning must be disabled on peer link to prevent data traffic from forwarding. If the learning is enabled, the same MAC (e.g. MAC of CE1) may be learned via MC-LAG port or peer link, and the output port of this MAC will keep toggling.
- When the MC-LAG enabled interface is down, MAC learning is also disabled in peer link, dynamic MAC entries will be installed to FDB pointing to peer link as the next hop, so traffic destined to those dynamic MAC entries will take the peer link path.

### 7.2.6. L2 BUM flooding and port isolation

- In the L2 scenario, MC-LAG enabled port and peer link are in same VLAN. When both ports are up, BUM may be flooded before the MAC is learned.
- To prevent CE from receiving duplicated traffic, peer link port must be isolated from MC-LAG enabled port. In the diagram of 7.2.1, if PEER2 receives traffic from PEER1 via PortChannel0002, the traffic will not be forwarded to PortChannel0001. The following ASIC functions achieve this goal.
- In Linux kernel, ebtable is used to isolate peer link port and from MC-LAG port. The Linux command is like ‘ebtables –A FORWARD -i PortChannel0002 -o PortChannel0001 -j DROP’. PortChannel0002 is the peer-link input interface, and PortChannel0001 is the output interface.
- In ASIC, ACL rule is used to isolate peer link from MC-LAG port. The rule is when the traffic is received from peer link and the output port is MC-LAG member port, the traffic must be dropped. It’s suggested to use isolation group, although isolation group is defined in SAI, but orchagent does not support it currently. Also isolation group may not be supported by all ASIC vendors. Using ACL is a more generic way to support the isolation function. We will refine this function to isolation group later if it’s required.

## 7.3. Peer connection down

![Diagram 7.3](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_7.3.png)
Diagram 7.3

- When peer connection is down, no action is taken by ICCPD. Data forwarding continues as usual, eventually the system will become unstable since information cannot be exchanged between the two peers.

## 7.4. Peer link down

![Diagram 7.4](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_7.4.png)
Diagram 7.4

- When peer link is down, as shown above, no action is taken by ICCPD. Data forwarding continues as usual. If ICCP connection uses this peer link interface, eventually the system will become unstable since information cannot be exchanged between the two peers. If one MC-LAG enabled port is down, data traffic may get lost since the peer link as a backup path is down.

## 7.5. Peer link is Vxlan tunnel

- The peer link can be Vxlan tunnel. This scenario is test in progress.

# 8. SONiC system diagram for MCLAG

The MC-LAG code will run in a separate container. The following chart is with added ICCPd container. The blocks below are just illustrating the concept flow.

Note: ICCPd docker container doesn't start by default, it could be started on demand. Linux command `sudo systemctl start iccpd` can be used to start ICCPd docker container, `sudo systemctl enable iccpd` will start ICCPd docker automatic after rebooting.

![Diagram 8](https://github.com/shine4chen/SONiC/blob/mclag/images/mclag_hld/MCLAG_HLD_8.png)
Diagram 8

Flow 1: ICCPd reads MCLAG related configuration from CONFIG_DB

Flow 2: Updating kernel with ARP learned. Syncing up kernel with changed MAC to update LACP with system ID for the portchannel.

Flow 3: ICCPd learns events such as L2 interface creation, L3 interface creation, MAC of local interfaces and its modification, IP address and its modification, ARP learning, VLAN creation and its member’s association/de-association, interface admin/link state.

Flow 4: ICCPd update teamd with system ID to help LACP to form the portchannel for the MLAG ports. Please note teamds in the two peers are independent.

Flow 5: mclagctld receives CLIs, pass it on to ICCPd, MC-LAG information is returned by ICCPd so that it can be displayed.

Flow 6: ICCPd exchange information with APP_DB via mclagsyncd，for example, the information to stop L2 learning on peer-link, flush software FDB table of MC-LAG vlans、isolate peer link from port-channel in the flood domain, associate MAC to its L3 interface.

Flow 7: same as 6

Flow 8: After configure/update APP_DB, OrchAgent monitors and procesess the related information, then update ASIC_DB.

# 9. Design changes

## 9.1. The schema changes

### 9.1.1. Add MCLAG configuration

```jason
  "MC_LAG": {
            "{{MC-LAG_domain_id}}": {
              "local_ip": {{ipv4_address}},
              "peer_link": {{interface_name}},
              "peer_ip": {{ipv4_address}},,
              "mclag_interface": {{interface_name}}
          }
      },

  MC-LAG domain ID, must be from 1 to 65535.
  "local_ip" is the ip address of this device to set TCP connection.
  "peer_ip" is the ip address of peer device to set TCP connection.
  "peer_link" is the name of interface that act as interconnection.
  "mclag_interface" is the name of PortChannel interfaces that MC-LAG enabled, can be multiple  interfaces separated by commas ‘,’.
  The example configuration is:

  "MC_LAG": {
        "1": {
            "local_ip": "2.2.2.1",
            "mclag_interface": "PortChannel0001,PortChannel0002",
            "peer_link": "PortChannel0022",
            "peer_ip": "2.2.2.2"
        }
    },
```

### 9.1.2. Add Acl_table and Acl_rule_table in app-db

Jason definition is exactly same as acl_table and acl_rule_table in config-db. MCLAG uses acl to isolate peer-link from mclag-enabled port-channel. so We add acl_table and acl_rule_table in app-db. Other apps which need to configure acl dynamicly can reuse this mechanism.
The acl hld has defined app_acl_table and app_acl_rule_table but don't implement it. Please refer [acl_hld](https://github.com/Azure/SONiC/blob/master/doc/acl/ACL-High-Level-Design.md#31211-acl-tables-table) for detail.

```jason
    "ACL_TABLE": {
        "mclag": {
            "policy_desc" : "Mclag egress port isolate acl",
            "type" : "L3",
            "stage" : "egress"
            "ports" : [
                "PortChannel0100"
                ]
        }
    },
    "ACL_RULE": {
       "mclag|mclag": {
            "OUT_PORTS" : "Ethernet2,Ethernet1,Ethernet0",
            "IP_TYPE" : "ANY",
            "PACKET_ACTION" : "DROP"
        }
    }
```

## 9.2. Add mclagsyncd process

Mclagsyncd will work with APP_DB to take care of following actions:

- Disable L2 MAC learning for peer-link ports.
- Flush FDB table contents before disable L2 MAC learning.
- Updating MAC association with its L3 interface. For example, PortChannel0001 is configured ip address, it is L3 interface. When changing the MAC of L2 port PortChannel0001, the MAC of L3 interface must be change to the same MAC at the same time. If not, data forwarding is blocked via L3 interface.
- When the MC-LAG enabled port-channel is down, update the nexthop of MAC entries learned from this PortChannel to point to peer link.
- Isolate peer-link from MC-LAG enabled port-channel.
Mclagsyncd will work with ASIC_DB to take care of following actions:
- Read MACs from ASIC_DB, and notify the MAC changes to ICCPd. FDB entries learned from MC-LAG enabled port-channel will be synced up with peer.

## 9.3. CLI design

Add mclagctld process to support new defined CLIs so that the ICCPd related information can be displayed.

The new CLIs are:

- `mclagdctl -i <mclag-id> dump arp`
Display the local ARP entries learned from the port-channels that mclag is enabled.

- `mclagdctl -i <mclag-id> dump mac`
Display the local MAC entries learned from the port-channels that mclag is enabled.

- `mclagdctl -i <mclag-id> dump portlist local`
Display the local portlist, include the port-channels and their member ports.

- `mclagdctl -i <mclag-id> dump portlist peer`
Display the peer portlist, include the port-channels and their member ports.

- `mclagdctl -i <mclag-id> dump state`
Display the local configuration and state.

## 9.4. aclorch changes

Adding the following logic:([PR#810](https://github.com/Azure/sonic-swss/pull/810))

- ACL table can be matched with OUT_PORTS.
- Aclorch as the consumer to monitor the events of APP_ACL_TABLE_NAME and APP_ACL_RULE_TABLE_NAME tables in APP_DB . The definition of APP_ACL_TABLE_NAME and APP_ACL_RULE_TABLE_NAME are in sonic-swss-common/common/schema.h.
- ICCPd set ACL rule in APP_DB for port isolation.

## 9.5. portmgr changes

Adding the following logics:

- If the port has the attribute ‘learn_mode’ in CFG_DB, read the attribute and set this attribute in APP_DB. If ‘learn_mode’ is set to ‘disabled’, the MAC learning of this port is disbled.

## 9.6. portsorch changes

Adding the following logics:

- Listening new APP_DB events, enable or disable to learn interface MACs. If ‘learn_mode’ attribute of one port is set to ‘disabled’, the MAC learning of this port is disbled.([PR#809](https://github.com/Azure/sonic-swss/pull/809))
- Add LAG name map to counter table, so that ‘show mac’ can display the MACs learned from LAG ports. ([PR#808](https://github.com/Azure/sonic-swss/pull/808))

## 9.7. intfmgr changes

Adding the following logics:

- If the L3 interface has the attribute ‘mac_addr’ in CFG_DB, read the attribute and set this attribute in APP_DB. Attribute ‘mac_addr’ specify the MAC of this L3 interface.

## 9.8. intfsorch changes

Adding the following logics:

- Listening MAC modification of L3 interfaces,  updating it to ASIC.([PR#814](https://github.com/Azure/sonic-swss/pull/814))
- Removing the function of adding/delete direct connected network FIB update triggered by creation of L3 interface in orchAgent. (Such direct connected network FIB does not respond to the state change, which is wrong, it only responds to the ip address addition or deletion）[PR#878](https://github.com/Azure/sonic-swss/pull/878))

## 9.9. routeorch changes

Adding the following logics:

- For the creation of the directly connected network, the code is moved from intfsorch to routeorch. Like other types of route, the direct connect network is informed by Zebra. This is to fix a common issue, not only for MC-LAG support.([PR#878](https://github.com/Azure/sonic-swss/pull/878))

## 9.10. fdborch changes

Adding the following logics:

- If the FDB entry is already exist when the event is ADD, remove the entry first, then add the entry.
- FDB MAC can be learnt from or set to VXLAN tunnel in ASIC.
([PR#877](https://github.com/Azure/sonic-swss/pull/877))

## 9.11. vxlanmgr changes

Adding the following logics:

- When create Vxlan tunnel map (encap and decap) , create the L2 interface in Linux kernel at the same time. The format of L2 interface name is ‘VxlanTunnelName-VNI’, such as VTTNL0001-1000. The prefix of Vxlan tunnel name in CFG_DB must be 'VTTNL'.

## 9.12. vxlanorch changes

Adding the following logics:

- When creating vxlan tunnel, create its bridge port.
- Add tunnel name map to counter table, so that ‘show mac’ can display the MACs learned from Vxlan tunnel.
- Listening new APP_DB events，enable or disable to learn VXLAN tunnel interface MACs. If ‘learn_mode’ is set to ‘disabled’, the MAC learning of this vxlan tunnel is disabled.

## 9.13. vnetorch changes

- Vnetorch create the bridge port of Vxlan tunnel in function VNetBitmapObject::getBridgeInfoByVni(), this is not the best place to create bridge port. In vxlanorch, when creating Vxlan tunnel, create its bridge port at the same time.

## 9.14. warm-reboot consideration

- Teamd saves the last LACP PDU received from LAG peer in one file per port in directory '/var/warmboot/teamd/'. During warm-reboot, the routes or MACs in ASIC are not changed.
- In MC-LAG scenario, two peer devices form one end point of a LAG, these two devices must have the same MAC address since it’s used for LACP. During warm-reboot, this MAC must not be changed. For this reason, if the last reboot is warm-reboot, when some ports are added to the existing portchannel, before the first LACP PDU is sent out via those newly added ports, teamd gets the MAC from the saved LACP PDU and updates the port MAC accordingly.

## 9.15. teammgr changes

Adding the following logics:

- If the PortChannel has the attribute 'learn_mode' in CFG_DB, read the attribute and set this attribute in APP_DB. If 'learn_mode' is set to 'disabled', the MAC learning of LAG port is disbled.
- Attribute 'fast_rate' can be set for LAG port.
