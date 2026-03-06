<BS># Enterprise Network Segmentation & Zero Trust Architecture

**Author:** Peter Kiptui
**Role:** Network Security Strategist
**Infrastructure:** MikroTik RB941 (Core Router/Firewall), Huawei S2700 (Distribution Switch), UniFi AP & Cloud Key Gen 2 (Wireless), Proxmox VE (Virtualization)

## Executive Summary
This project documents the transition of a flat network into a segmented, enterprise-grade architecture. The primary objective was to implement a Zero Trust foundation by isolating traffic flows into distinct security zones (VLANs). 

This architecture allows for granular firewall control, isolating high-risk "Red Team" activities and unverified guest traffic from core virtualization infrastructure and management interfaces.

## Phase 1: Infrastructure Hardening & Recovery
**Objective:** Establish a secure baseline and failsafe access methods before applying segmentation logic.

### 1. Zero-Base Configuration
I initiated the project by resetting the MikroTik router to "No Default Configuration".
* **Purpose:** To eliminate consumer-grade presets and hidden firewall rules that could obscure traffic visibility or create security blind spots. I built the routing table from scratch to ensure total ownership of the packet flow.

### 2. The "Lifeboat" Protocol (Out-of-Band Management)
I configured physical interface `ether4` with a static IP (`192.168.88.1/24`) and excluded it from the main bridge.
* **Purpose:** To guarantee resilient management access. In the event of a VLAN filtering misconfiguration on the main trunk, this port acts as a dedicated physical key to regain control of the router without requiring a factory reset.

## Phase 2: Layer 2 Segmentation (VLAN Architecture)
**Objective:** Define the virtual security boundaries and traffic lanes.

### 1. The Unified Bridge Strategy
I deployed a single bridge (`Home_lab`) to act as the master software switch for the router's internal throughput.
* **Purpose:** To centralize VLAN filtering logic in one location, allowing the router's CPU to inspect and tag traffic efficiently across all member ports.

### 2. Zoning & VLAN Definitions
I defined the following security zones based on trust levels:
* **VLAN 10 (Stream/Trusted):** Daily driver network for trusted personal devices and the primary Parrot OS workstation.
* **VLAN 20 (Guest):** Strictly isolated network for unverified visitors.
* **VLAN 30 (Red Team):** High-risk sandbox for penetration testing targets and attack boxes.
* **VLAN 99 (Management):** A restricted control plane for MikroTik WinBox, Huawei CLI, Proxmox GUI, and UniFi Cloud Key administration.

![MikroTik VLAN Interfaces](3.jpeg)

## Phase 3: Layer 3 Services & Addressing
**Objective:** Automate network provisioning while maintaining logical separation.

### 1. Gateway Deployment
I assigned unique Layer 3 Gateways to each VLAN interface (e.g., `10.0.30.1/24` for the Red Team).
* **Purpose:** To create clear routing boundaries. By placing gateways on the VLAN interfaces rather than the physical ports, the router can route traffic between zones strictly according to Firewall rules.

![MikroTik IP Address Assignments](1.jpeg)

### 2. Independent DHCP Services
I stood up separate DHCP servers for each VLAN.
* **Purpose:** To ensure that a device plugging into a "Guest" port is immediately trapped in the Guest subnet (`10.0.20.0/24`), preventing them from even attempting to communicate with trusted assets on the main network.

## Phase 4: Wireless Security Integration
**Objective:** Extend physical segmentation to the wireless airspace.

### 1. Edge Tagging
I mapped the Wireless Interface (`wlan1`) to PVID 10.
* **Purpose:** To enforce security at the ingress point. Any packet entering the air via this radio is immediately stamped with "VLAN 10" before it touches the bridge, ensuring wireless clients cannot bypass segmentation logic.

## Phase 5: The Logic Core (VLAN Filtering)
**Objective:** Activate the segmentation engine.

### 1. Bridge VLAN Table Configuration
I configured the Bridge CPU as a "Tagged" member of all VLANs.
* **Purpose:** To establish the uplink between the router's software brain (L3) and the switching ports (L2). Without this tag, the CPU would be blind to the VLAN traffic, breaking DHCP and Routing services.

### 2. Ingress Filtering & Drop Rules
I enabled `vlan-filtering=yes` and `ingress-filtering=yes`.
* **Purpose:** To enforce strict admission control. The router now actively discards any packet that does not carry a valid VLAN tag or arrive on an authorized port, effectively preventing "VLAN Hopping" attacks.

![MikroTik Bridge Port Configuration](2.jpeg)

## Phase 6: Connectivity & NAT
**Objective:** Provide controlled internet access without exposing internal topology.

### 1. Masquerade (Source NAT)
I configured a Source NAT rule for the WAN interface (`ether1`).
* **Purpose:** To obscure the internal network structure. External servers see only the router's public IP, while the NAT engine maintains a state table to route return traffic back to the correct private VLAN.

## Phase 7: Layer 3 Firewall Policies (The Security Core)
**Objective:** Prevent default inter-VLAN routing and enforce Zero Trust principles.

By default, RouterOS routes traffic between all connected subnets. To enforce actual segmentation, I implemented strict `/ip firewall filter` rules:
1. **Drop Inter-VLAN Traffic:** Explicit rules drop all `forward` chain traffic originating from the Guest (VLAN 20) and Red Team (VLAN 30) subnets destined for the Trusted (VLAN 10) and Management (VLAN 99) subnets.
2. **Management Access Control:** The `input` chain is secured to drop all WinBox, SSH, and WebFig attempts unless they originate from authorized IPs within VLAN 10.
3. **Red Team Isolation:** VLAN 30 can access the internet, but cannot initiate connections to the Proxmox environment or any internal gateways.

## Next Steps
* **Hardware Switching:** Configuring the Huawei S2700 via Console to handle 802.1Q Trunks.
* **Threat Intelligence:** Deploying a SIEM (Wazuh) to monitor firewall logs.
* **Infrastructure as Code:** Uploading sanitized `.rsc` configuration exports for peer review.
