# Corporate Network Segmentation & Security Simulation | Cisco Packet Tracer

This repository showcases a complete, production-ready enterprise network architecture designed and simulated in **Cisco Packet Tracer**. The project implements advanced network segmentation, inter-VLAN routing, and edge security filtering through a dedicated hardware firewall, enforcing strict isolation and access policies.

## 📐 Network Architecture Overview

The topology is designed around a core **Router-on-a-Stick** topology connected to segmented Layer 2 environments and an edge security boundary.

* **Core Router (Router1):** Handles inter-VLAN routing and acts as the internal default gateway for all corporate subnets.
* **Edge Firewall (Cisco ASA 5506-X):** Acts as the security gateway between the internal networks (`inside` zone) and the external network (`outside` zone).
* **Border Router (Router0):** Simulates the ISP / DMZ routing gateway that exposes public enterprise resources (e.g., DNS, corporate application servers).

---

## 🔒 VLAN Segmentation & IP Addressing Scheme

To optimize broadcast domains, maximize performance, and strictly isolate untrusted traffic (such as guests or IoT devices), the network infrastructure is divided into 5 distinct VLANs:

| VLAN ID | Department / Function | Subnet Network | Default Gateway | Security / Access Policy |
| :--- | :--- | :--- | :--- | :--- |
| **VLAN 10** | Administration / Core Staff | `192.168.1.0/24` | `192.168.1.1` | Full intranet access; Dynamic PAT to external services. |
| **VLAN 20** | Technical / Production | `172.16.1.0/24` | `172.16.1.1` | Internal production access; Dynamic PAT to external services. |
| **VLAN 30** | VoIP / IP Telephony | `10.1.1.0/24` | `10.1.1.1` | Isolated voice traffic optimization. |
| **VLAN 40** | Guest Network (Wireless) | `192.168.40.0/24` | `192.168.40.1` | **Isolated untrusted network**. Access allowed *only* to the Internet via Firewall; blocked from internal corporate VLANs. |
| **VLAN 60** | IoT Devices | `192.168.60.0/24` | `192.168.60.1` | Strictly monitored smart-devices environment. |

---

## 🛠️ Key Technical Implementations

### 1. Stateful Edge Security (Cisco ASA)
The **Cisco ASA 5506-X** separates the infrastructure into security zones:
* **Inside Zone (Security Level 100):** Trust zone containing the corporate VLANs.
* **Outside Zone (Security Level 0):** Untrusted zone facing the public web and servers.

Dynamic **PAT (Port Address Translation)** is enforced at the edge, masking internal private IPs behind the public interface address (`209.165.200.226`). Advanced **ICMP Inspection** and explicit Access Control Lists (`OUTSIDE_IN`) are configured to allow stateful return traffic while keeping the internal network stealthy to outside scans.

### 2. Precise Static Routing & Symmetric Return Paths
To prevent routing loops and asymmetric routing drops (black-holing), explicit static routes are configured at the border:
* The ASA firewall points to the internal router for all specific classful segments.
* The border ISP router (`Router0`) contains targeted summaries pointing back to the ASA outside interface, ensuring successful end-to-end ICMP Echo Requests and HTTP/DNS application traffic.

---

## 🚀 How to Run and Test the Lab

Since `.pkt` files are proprietary binaries that cannot be natively rendered on GitHub, follow these instructions to inspect the running environment:

1.  **Requirement:** Download and install **Cisco Packet Tracer** (v8.0 or later recommended) from the official [Cisco Networking Academy](https://www.netacad.com/).
2.  **Download:** Clone this repository or download the `complex network.pkt` file directly from this folder.
3.  **Simulation & Verification:**
    * Open the file in Packet Tracer.
    * Open any terminal/PC (e.g., in VLAN 10 or VLAN 40 Guest) and test end-to-end connectivity by pinging the edge server at `208.67.220.220`.
    * Access the CLI of the **ciscoasa** to inspect real-time connection states using `show access-list` and `show route`.
