# Campus Network Architecture & Deployment Guide

## 1. Architectural Overview
This document outlines the design and configuration for a 5-building enterprise campus network. The architecture utilizes a **Collapsed Core** design, featuring a central Layer 3 Multilayer Switch connecting to Layer 3 MDF switches in each building. 

**Key Technologies:**
*   **Routing:** EIGRP (AS 10) for hardware-accelerated inter-building routing.
*   **Switching:** Local VLANs to contain broadcast traffic within individual buildings.
*   **Security:** Zero Trust model using Extended ACLs, Port Security, and Device Hardening.
*   **Wireless:** Autonomous Access Points with WPA2-PSK, mapped to dedicated VLANs.

---

## 2. Global IP Addressing Scheme (IPAM)

### 2.1 Campus Backbone (Point-to-Point Links)
Used exclusively to link the Core Switch to Building MDFs and the Edge Router.
*   **Subnet Mask:** `255.255.255.252` (/30)

| Connection Path | Network Address | Core Switch IP | Destination IP |
| :--- | :--- | :--- | :--- |
| **Core Switch $\leftrightarrow$ Edge Router** | `10.1.1.0 /30` | `10.1.1.2` | `10.1.1.1` |
| **Core Switch $\leftrightarrow$ Bldg 2&3** | `10.0.0.0 /30` | `10.0.0.1` | `10.0.0.2` |
| **Core Switch $\leftrightarrow$ Admin Bldg** | `10.0.1.0 /30` | `10.0.1.1` | `10.0.1.2` |
| **Core Switch $\leftrightarrow$ Building 1** | `10.0.2.0 /30` | `10.0.2.1` | `10.0.2.2` |
| **Core Switch $\leftrightarrow$ HPSB** | `10.0.3.0 /30` | `10.0.3.1` | `10.0.3.2` |

### 2.2 Campus Server Farm
Hosted off the Campus Edge Router for shared resources.
*   **Network:** `172.16.1.0 /24`
*   **Gateway:** `172.16.1.1` (Router Interface)
*   **Web/DNS Server:** `172.16.1.10`

---

## 3. Building VLAN Allocations (Local VLANs)
Each building utilizes the same VLAN IDs for consistency but operates on unique `/24` subnets to prevent cross-campus broadcast storms. The Building MDF Layer 3 switch acts as the `.1` gateway and DHCP server for all local VLANs.
*   **Subnet Mask:** `255.255.255.0` (/24)

### Building 2&3 (Main Operations)
| VLAN ID | Network Name     | Network Address | Default Gateway | DHCP Range    |
| :------ | :--------------- | :-------------- | :-------------- | :------------ |
| **10**  | Management       | `192.168.10.0`  | `192.168.10.1`  | `.2` - `.254` |
| **20**  | Staff Offices    | `192.168.20.0`  | `192.168.20.1`  | `.2` - `.254` |
| **30**  | Labs             | `192.168.30.0`  | `192.168.30.1`  | `.2` - `.254` |
| **40**  | Staff Dorms      | `192.168.40.0`  | `192.168.40.1`  | `.2` - `.254` |
| **50**  | Student Wireless | `192.168.50.0`  | `192.168.50.1`  | `.2` - `.254` |

### Admin Building
| VLAN ID | Network Name | Network Address | Default Gateway | DHCP Range |
| :--- | :--- | :--- | :--- | :--- |
| **10** | Management | `192.168.100.0` | `192.168.100.1` | `.2` - `.254` |
| **20** | Admin Offices | `192.168.101.0` | `192.168.101.1` | `.2` - `.254` |
| **30** | Admin Labs | `192.168.102.0` | `192.168.102.1` | `.2` - `.254` |
| **50** | Guest Wireless | `192.168.103.0` | `192.168.103.1` | `.2` - `.254` |

### Building 1
| VLAN ID | Network Name | Network Address | Default Gateway | DHCP Range |
| :--- | :--- | :--- | :--- | :--- |
| **10** | Management | `192.168.110.0` | `192.168.110.1` | `.2` - `.254` |
| **20** | Staff Offices | `192.168.111.0` | `192.168.111.1` | `.2` - `.254` |
| **30** | Labs | `192.168.112.0` | `192.168.112.1` | `.2` - `.254` |
| **50** | Student Wireless | `192.168.113.0` | `192.168.113.1` | `.2` - `.254` |

### HPSB
| VLAN ID | Network Name | Network Address | Default Gateway | DHCP Range |
| :--- | :--- | :--- | :--- | :--- |
| **10** | Management | `192.168.120.0` | `192.168.120.1` | `.2` - `.254` |
| **20** | Staff Offices | `192.168.121.0` | `192.168.121.1` | `.2` - `.254` |
| **30** | Labs | `192.168.122.0` | `192.168.122.1` | `.2` - `.254` |
| **50** | Student Wireless | `192.168.123.0` | `192.168.123.1` | `.2` - `.254` |

---

## 4. Zero Trust Security Policies

### 4.1 Access Control Lists (ACLs)
Applied inbound on the SVIs (VLAN interfaces) of the Building MDF Switches.

*   **RESTRICT-STAFF (Applied to VLAN 20):**
    *   `DENY` access to VLAN 10 (Management).
    *   `PERMIT` access to all other networks.
*   **RESTRICT-UNTRUSTED (Applied to VLANs 30, 40, and 50):**
    *   `DENY` access to VLAN 10 (Management).
    *   `DENY` access to VLAN 20 (Staff Offices).
    *   `DENY` access to other untrusted VLANs (e.g., VLAN 30 cannot reach 40 or 50).
    *   `PERMIT` access to Campus Server & Internet.

### 4.2 Port Security
Applied to all Layer 2 Floor Switch ports connected to wired end-user devices (Lab PCs, Office PCs, Dorm PCs). 
*   **Rule:** `maximum 1` MAC address.
*   **Action:** `violation shutdown`.
*   *(Note: Do NOT apply to switch-to-switch trunks or Access Point ports).*

### 4.3 Device Hardening
Applied to all Routers, Core Switches, MDFs, and Floor Switches.
*   Enable Secret: `classpass123`
*   Console Password: consolepass123
*   VTY Passwords: `remotepass123`

---

## 5. Deployment Playbook (Building-by-Building Step Guide)

### Phase 1: Physical Placement & Wiring
1.  Place 1x **3650 Multilayer Switch** (Building MDF). Install AC power.
2.  Place 1x **2960 Layer 2 Switch** (Floor Switch).
3.  Place 2x **AccessPoint-PT** (Office AP, Student AP).
4.  Place End Devices (PCs, Laptops).
5.  Cable MDF $\rightarrow$ Core Switch, MDF $\rightarrow$ Floor Switch, Floor Switch $\rightarrow$ End Devices & APs.

### Phase 2: MDF Routing & EIGRP Configuration
enable 
configure terminal 
ip routing 

! Configure Uplink to Core Switch 
interface GigabitEthernet1/0/1 
 no switchport 
 ip address [10.0.x.2] 255.255.255.252 
 no shutdown 
exit 

! Enable EIGRP 
router eigrp 10 
 network 10.0.0.0 
 network [192.168.x.0] ! (Advertise all local VLAN subnets) 
 no auto-summary 
exit

### Phase 3: MDF VLAN & DHCP Setup
! Create VLAN and SVI (Repeat for 10, 20, 30, 50)
vlan 20
exit
interface vlan 20
 ip address [192.168.x.1] 255.255.255.0
 no shutdown
exit

! Create DHCP Pool (Repeat for 10, 20, 30, 50)
ip dhcp pool VLAN20-STAFF
 network [192.168.x.0] 255.255.255.0
 default-router [192.168.x.1]
exit

! Configure Trunk to Floor Switch
interface GigabitEthernet1/0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
exit
### Phase 4: Floor Switch & Wireless Configuration
- Set uplink port to `switchport mode trunk`.
- Assign end-device ports to respective VLANs (`switchport mode access`, `switchport access vlan [ID]`).
- Configure Autonomous APs via the GUI (Port 1 $\rightarrow$ SSID, WPA2-PSK, Password).

### Phase 5: Security Implementation
! Apply Passwords (Router and All Switches)
 enable 
 configure terminal 
 enable secret classpass123 
 line console 0 
  password consolepass123 
  login 
  exit 
line vty 0 4 
 password remotepass123 
 login 
 exit

! Apply Port Security (Floor Switch - PC Ports ONLY) 
enable
configure terminal

! Example: Applying to ports FastEthernet0/1 through FastEthernet0/10
interface range FastEthernet0/1 - 10
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
exit

end

! Apply ACLs (MDF Switch SVI) !!! MAKE SURE THAT YOU CHANGE THE THIRD OCTET BASE ON YOUR BUILDING IP SCHEME

enable
configure terminal

! ==========================================
! 1. CREATE THE STAFF ACL (VLAN 20)
! Blocks Staff from accessing Management infrastructure.
! ==========================================
ip access-list extended RESTRICT-STAFF
 deny ip 192.168.20.0 0.0.0.255 192.168.x.0 0.0.0.255
 permit ip any any
exit

! ==========================================
! 2. CREATE THE LABS ACL (VLAN 30)
! Blocks Labs from Management, Offices, and Dorms.
! ==========================================
ip access-list extended RESTRICT-LABS
 deny ip 192.168.30.0 0.0.0.255 192.168.x.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.x.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.x.0 0.0.0.255
 permit ip any any
exit

! ==========================================
! 3. CREATE THE DORMS ACL (VLAN 40)
! Blocks Dorms from Management, Offices, and Labs.
! ==========================================
ip access-list extended RESTRICT-DORMS
 deny ip 192.168.40.0 0.0.0.255 192.168.x.0 0.0.0.255
 deny ip 192.168.40.0 0.0.0.255 192.168.x.0 0.0.0.255
 deny ip 192.168.40.0 0.0.0.255 192.168.x.0 0.0.0.255
 permit ip any any
exit

! ==========================================
! 4. CREATE THE STUDENT WIRELESS ACL (VLAN 50)
! Blocks Students from Management, Offices, and Dorms.
! ==========================================
ip access-list extended RESTRICT-STUDENT
 deny ip 192.168.50.0 0.0.0.255 192.168.x.0 0.0.0.255
 deny ip 192.168.50.0 0.0.0.255 192.168.x.0 0.0.0.255
 deny ip 192.168.50.0 0.0.0.255 192.168.x.0 0.0.0.255
 permit ip any any
exit

! ==========================================
! 5. APPLY THE ACLs TO THE VLAN INTERFACES
! ==========================================
interface vlan 20
 ip access-group RESTRICT-STAFF in
exit

interface vlan 30
 ip access-group RESTRICT-LABS in
exit

interface vlan 40
 ip access-group RESTRICT-DORMS in
exit

interface vlan 50
 ip access-group RESTRICT-STUDENT in
exit

end

!!! MAKE SURE TO SAVE CONFIGURATION FOR ALL MDF AND FLOOR SWITCHES:
You must be in Privileged EXEC mode (the `Router#` or `Switch#` prompt, not `config` mode) to run these. You asked for both commands; they do the exact same thing, so running one immediately after the other is perfectly safe—it just overwrites the file twice.

1. ! The official command (Press Enter when it asks for the destination filename) 
	copy running-config startup-config 
 
2. ! The fast shortcut command 
	write memory

### Phase 6: Verification Testing
- **DHCP:** Toggle IP Configuration to Static, then DHCP on end devices. Ensure correct IP assignment.
- **Security:** Ping a Staff PC from a Lab/Student PC. Must return "Administratively Prohibited/Unreachable".
- **Routing:** Ping the Campus Server (`172.16.1.10`) from any building PC. Must successfully reply.