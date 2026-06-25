Technical Audit and Troubleshooting Report: "Complex Network" Project

This report describes in detail the purpose and processes of the "Complex Network" project, carried out personally and made available for consultation (refer to the README.md file for access methods).

The "Complex Network" network, originally conceived as a project during an internship, was developed with the purpose of applying advanced network device commands in order to interconnect them efficiently. The infrastructure integrates IoT devices manageable from any company workstation after connecting to the dedicated server, diversifying the topology into multiple VLANs to mitigate bottlenecks caused by broadcast messages.

A year later, the project was thoroughly reviewed, identifying significant security flaws and logical failures that, in a real context, would undermine the company's structural integrity if exploited by an attacker (e.g., possibility of lateral movement from the Guest Wi-Fi segment to production workstations, resulting in unauthorized access to the server and IoT devices). Although the Cisco Packet Tracer environment presents evident intrinsic limitations in some advanced functions — such as stable RSA key generation and encryption management — the tool proves extremely effective for the study of telecommunications and networks, allowing for the detailed examination of the data packet path.

This report proposes to analyze the critical issues of the previous system and to illustrate the resolution procedure applied to make the network secure, resilient, and efficiently fragmented (refer to the config.me file for consultation of the current network device configurations and topology).
1. Analysis of the Perimeter Firewall Status (Cisco ASA 5506-X)

The analysis of the perimeter device using the show interface ip brief command allowed for the mapping of physical security zones and highlighted a structural vulnerability in the placement of the guest network.
1.1 Active Connection Paths

The firewall operates on two main physical interfaces:

    GigabitEthernet1/1 (IP 192.168.254.1): Internal logical interface (inside). Acts as the Next-Hop for the central router's default route.

    GigabitEthernet1/2 (IP 209.165.200.226): External logical interface (outside), routed towards the public network (Modem/Internet).

1.2 Audit Evidence: Lack of a Perimeter Zone for Guests

The output shows that interfaces from GigabitEthernet1/3 to GigabitEthernet1/8 are disabled and lack an IP address.

    Detected Criticality: The guest network (GUEST_WIFI - VLAN 60) is not attached to a dedicated physical or logical interface of the firewall (e.g., a guest zone with an intermediate Security Level). Consequently, the ASA has no direct visibility on traffic originating from guest laptops if it is directed towards other company subnets (VLAN 10, VLAN 40).

    Impact on Security: Access control and guest isolation rest entirely on the central router. If restrictive ACLs are not applied on the central star router, an attacker on the Guest network can perform lateral movement and reconnaissance on the company LAN, completely bypassing the stateful controls and inspection policies of the ASA firewall.

2. Analysis of Security Policies (Audit of Access Control Lists)

The inspection of filtering rules on the ASA via show access-list identified the technical reason why network services (such as DNS resolution for cisco.com) were interrupted, despite a perfectly functioning ICMP (Ping) protocol.
2.1 Analysis of the "ACL_PER_NAT" ACL (Outbound Traffic)

The list contains the directives to allow outbound access to all subnets of the corporate network:

    Line 1 (VLAN 10): permit ip 192.168.1.0 255.255.255.0 any -> hitcnt=0

    Line 2 (VLAN 20): permit ip 172.16.1.0 255.255.255.0 any -> hitcnt=0

    Line 3 (VLAN 30): permit ip 10.1.1.0 255.255.255.0 any -> hitcnt=0

    Line 4 (VLAN 40): permit ip 192.168.40.0 255.255.255.0 any -> hitcnt=0

    Line 5 (VLAN 60): permit ip 192.168.60.0 255.255.255.0 any -> hitcnt=0

Audit Evidence: The hitcnt=0 counter on all lines proves that this list is not actively filtering traffic entering the internal interface, confirming that the placement of perimeter security rules is misaligned with the actual flow of packets.
2.2 Analysis of the "OUTSIDE_IN" ACL (Inbound Traffic)

The external perimeter interface has only one active rule:

    access-list OUTSIDE_IN line 1 extended permit icmp any any (hitcnt=11)

The Cause of the Disruption: The active counter (hitcnt=11) proves that the firewall allows exclusively the transit of ICMP packets. Any other response packet coming from the Internet — such as return packets for DNS queries (UDP port 53) or HTTP/HTTPS responses — is implicitly discarded by the ASA at the outside interface, causing the systematic timeout of commands like nslookup cisco.com on the client side.
2.3 Verification of the Modular Policy Framework (MPF) and Stateful Inspection

The show service-policy command did not return output or active policies on the perimeter device.

    Audit Evidence: The absence of an active global_policy indicates that the firewall is not performing dynamic traffic inspection (Stateful Inspection) for critical application protocols, particularly DNS (inspect dns) and HTTP/HTTPS (inspect http).

    Technical Consequence: Without stateful inspection, the ASA is unable to track outbound UDP/TCP sessions generated by internal clients. Consequently, legitimate response packets coming from external servers are considered unsolicited inbound traffic and are dropped implicitly at the Outside interface, confirming the total blockage of name resolution on the cisco.com domain.

3. Analysis of Client Side Configuration (End-Point Audit)

The final audit was conducted directly on the LAN host via the ipconfig /all command, in order to verify the correctness of the parameters distributed to network clients.
3.1 Detected Network Parameters

    IPv4 Address: 192.168.1.7 (Subnet 255.255.255.0) -> The host is correctly routed within VLAN 10.

    Default Gateway: 192.168.1.1 -> The pointer to the central star router for inter-VLAN routing is valid and active.

    DNS Servers: 208.67.220.220 -> The client points directly to the external server's public IP for domain resolution.

3.2 Evaluation of the Logical Flow of the Disruption

The host presents all correct nominal parameters for browsing. When the user attempts to reach cisco.com:

    The client generates a DNS query to the public IP 208.67.220.220.

    The packet is forwarded to the gateway 192.168.1.1 (Router1) and from there routed via default route to the internal interface of the ASA (192.168.254.1).

    The firewall performs NAT and forwards the request towards the WAN.

    The external server responds correctly, but the UDP response is discarded at the external interface of the ASA due to the absence of permit rules for port 53 in the OUTSIDE_IN ACL and the lack of a global dynamic inspection mechanism (show service-policy empty).

This misalignment on the perimeter firewall represents the primary cause of the application timeout encountered on the client side.
4. Plan for Remediation and Service Restoration

Below are the processes necessary for System Restoration:
4.1 Restoration of DNS Resolution and Web Browsing

To allow DNS (port 53) and Web (80/443) response packets to re-enter towards internal clients without exposing the network to external threats, stateful inspection has been reactivated on the ASA, integrating return Access Control Lists for authorized servers.

Configuration applied on the ASA 5506-X CLI:

    ciscoasa# configure terminal

    Restoration of Global Stateful Inspection (Modular Policy Framework)

        ciscoasa(config)# policy-map global_policy

        ciscoasa(config-pmap)# class inspection_default

        ciscoasa(config-pmap-c)# inspect dns

        ciscoasa(config-pmap-c)# inspect http

        ciscoasa(config-pmap-c)# inspect https

        ciscoasa(config-pmap-c)# exit

        ciscoasa(config-pmap)# exit

    Update of the external ACL to allow the re-entry of DNS traffic

        ciscoasa(config)# access-list OUTSIDE_IN extended permit udp host 208.67.220.220 any eq 53

        ciscoasa(config)# access-list OUTSIDE_IN extended permit tcp host 208.67.220.220 any eq 53

        ciscoasa(config)# access-group OUTSIDE_IN in interface outside

        ciscoasa(config)# write memory

4.2 Hardening and Isolation of the Guest Network (VLAN 60)

Pending a physical refactoring that attaches VLAN 60 to a dedicated firewall port, an intervention is made directly on the Central Router (central star) via extended Access Control Lists to block at the root any attempts at lateral movement towards production segments.

Configuration applied on the Router1 CLI:

    Router1# configure terminal

    access-list 160 deny ip 192.168.60.0 0.0.0.255 192.168.1.0 0.0.0.255

    access-list 160 deny ip 192.168.60.0 0.0.0.255 192.168.40.0 0.0.0.255

    access-list 160 permit ip any any

    interface Vlan60

    ip access-group 160 in

    exit

    write memory

4.3 Expected Post-Intervention Results

The nslookup cisco.com command executed from internal workstations resolves external domains nominally without interruptions, thanks to the dynamic session traceability introduced by the Modular Policy Framework.
Attempts at scanning or reconnaissance (Ping or Port Scan) originating from the GUEST_WIFI segment towards corporate machines are intercepted and discarded at the router entrance, zeroing out the internal attack surface.
5. Note on Architectural Control: WAN Flow vs Local Infrastructure

A superficial analysis of the visual topology in Packet Tracer might lead one to think that the internal host and the external server communicate simply because they are positioned within the same virtual workspace. From a logical and systems point of view, this assumption is incorrect, let's see how:
5.1 Demonstration of Geographic Transit (WAN Path)

The end-to-end connectivity is guaranteed exclusively by the traversal of a routing and security chain that simulates a real geographic infrastructure (Wide Area Network):

    Isolation of Addressing: The client operates on a private space (192.168.1.0/24), not routable on the global public network. The server resides on a remote public network (208.67.220.0/24).

    Frontier Mechanism (NAT/PAT): The firewall ASA intercepts the private packet and modifies its header, translating the source IP into the public IP of the outside path (209.165.200.226), making it suitable for transit on the Internet.

    ISP Infrastructure (Router0): The packet crosses the DSL modem and the Internet cloud to be taken over by the provider's router (Router0). This apparatus routes the traffic towards the server and, thanks to its static return routes, knows how to re-route responses towards the public IP of the company firewall.

The deactivation of the ASA external interface or the removal of routes on Router0 would cause the immediate isolation of the server, confirming the independence logical and geographic of the two networks.
6. Implementation and Logic of the True DMZ (DeMilitarized Zone)

To elevate the security requirements of the infrastructure to an Enterprise standard, a third isolated security area (physical DMZ) was introduced directly on the ASA 5506-X Firewall, attached to the GigabitEthernet1/3 physical interface.
6.1 Logical Configuration of the DMZ

    Dedicated Subnet: 192.168.50.0/24 (Gateway IP attached to the ASA interface: 192.168.50.1).

    Security Level: Configured to 50. This hierarchical setting places the DMZ in an intermediate transition zone: it possesses a lower trust level than the internal LAN (security-level 100), but higher than the public Internet perimeter (security-level 0).

    Attached Hosts: A dedicated server (Server_DMZ) configured with static IP addressing 192.168.50.2/24 and Default Gateway 192.168.50.1.

6.2 Selective Access Policy (Access Control Policy)

In compliance with the principle of Least Privilege, inter-zone transit has not been opened indiscriminately to the entire corporate infrastructure. Access to resources residing in the DMZ has been mapped through explicit directives on the firewall:

    Authorized Segments: Exclusively the Core LAN (VLAN 10, subnet 192.168.1.0/24), allowing the management endpoint usr:jakelafuria (192.168.1.7) administration and querying of application services.

    Isolation of Unauthorized Segments: The remaining corporate VLANs (Offices, Guests, IoT) do not possess explicit permit rules towards the 192.168.50.0/24 network. Connection attempts are therefore implicitly discarded by the ASA (Implicit Deny), drastically reducing the internal attack surface.

    Stateful Return Mechanism: The activation of the global_policy and dynamic inspection mechanisms on the ASA allows the automatic transit of response packets from the DMZ Server towards VLAN 10, excluding the need to configure permissive and potentially dangerous bidirectional Access Control Lists.

7. Troubleshooting DMZ Anomalies

Numerous anomalies were encountered, many of which were due to the limitations of the Cisco Packet Tracer software itself, but I guarantee that everything possible has been done to make the experience as simulative as possible.
7.1 Detected Symptoms

Subsequent to the attachment of the Server_DMZ node to the GigabitEthernet1/3 interface of the ASA, HTTP/HTTPS connection attempts originating from the usr:jakelafuria client (192.168.1.7) towards the DMZ IP (192.168.50.2) systematically failed in application timeout, despite the successful compilation of traffic classification rules.
7.2 Root Cause Analysis

The analysis of the logical flows and logs of the ASA highlighted two concurrent architectural problems:

    Asymmetry of Internal Routing: The ASA did not possess in its routing thin client an explicit return route to reach the Core LAN subnet (192.168.1.0/24), whose segments are not directly connected to the firewall but attached behind the central star (Router1 with Next-Hop IP 192.168.254.2). This caused the systematic discarding of return traffic.

    Interference of Egress NAT Directives: The global dynamic NAT policy configured on the outside interface to allow Internet exit also incorrectly intercepted flows destined for the dmz segment. The resulting alteration of packet headers compromised the integrity of the flow and the coherence of stateful inspection tables.

7.3 Applied Solution (NAT Identity / Exempt and Routing Fix)

To resolve the anomaly, a NAT Twice rule (called NAT Exempt or NAT Identity) was implemented to exclude LAN-to-DMZ traffic from any translation process, in parallel with the insertion of the correct internal Next-Hop route.

Configuration syntax applied on the ASA 5506-X CLI:

    ciscoasa# configure terminal

    object network obj-lan

    subnet 192.168.1.0 255.255.255.0

    exit

    object network obj-dmz

    subnet 192.168.50.0 255.255.255.0

    exit

    nat (inside,dmz) source static obj-lan obj-lan destination static obj-dmz obj-dmz

    route inside 192.168.1.0 255.255.255.0 192.168.254.2

    write memory

Appendix: Limitations of the Emulation Tool and Software Anomalies

During the advanced testing phases and the integration of the third security zone (DMZ), several logical inconsistencies were encountered that were not attributable to macroscopic configuration errors, but rather to structural limitations and known bugs of the Cisco Packet Tracer simulation software.

    Instability of the Cisco ASA Module (Legacy Emulation)

    Saturation of State Tables: The stateful inspection engine of the simulated ASA tends to corrupt the active connection table following repeated sequential modifications to interface ACLs. This phenomenon generates arbitrary drops on legitimate HTTP or DNS traffic, even in the presence of explicit permit directives.

    CLI Syntax Inconsistencies: Some software releases of the ASA device implemented in Packet Tracer reject standard perimeter diagnostic constructs (such as the packet-tracer input command) or incorrectly interpret the precedence of lines in advanced NAT Twice rules, effectively limiting the troubleshooting tools available to the operator.

    Corruption of Application Services (DNS and HTTP) post-Power Cycle

    Lack of Process Synchronization: Upon restoring configuration files or following a global hardware restart of the virtual environment (Power Cycle Devices), the software daemons integrated into the simulated Servers (such as the DNS service on cisco.com) show a nominal graphical status set to ON in the GUI, despite not responding to real name resolution requests forwarded by LAN clients.

    Workaround and Resolution: Such software anomalies can only be bypassed by forcing manual emptying of the local resolver cache on clients (via the ipconfig /flushdns command) or by artificially accelerating the convergence time of the virtual environment via the Fast Forward Time function.

    Technical Conclusions of the Analyzer

The increase in topological complexity (introduction of multiple VLANs, asymmetric routing, and multilevel firewalling) highlights the purely educational limit of the Packet Tracer tool.

In a real scenario operating on physical hardware, or within professional emulation environments based on hypervisor (such as EVE-NG or GNS3), routing protocols and firewall state tables would have maintained end-to-end logical coherence, without the need to apply configuration rollbacks or workarounds on client caches.

I thank you for your attention, I hope that the consultation of this educational material is useful to you.

Sayonara.
