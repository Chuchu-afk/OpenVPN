# 🔐 OpenVPN Remote Access Configuration

## Overview
This project documents the design and implementation of a **secure OpenVPN remote access solution** integrated with a segmented firewall and network environment. The goal was to provide encrypted, authenticated remote connectivity while maintaining strict network isolation and access control.

The configuration is designed to reflect **enterprise-style remote access**, emphasizing security, visibility, and maintainability.

---

## 🎯 Objectives
- Provide secure remote access over an encrypted tunnel
- Enforce least-privilege network access across VLANs
- Maintain clear separation between user, server, and management networks
- Ensure configurations are well-documented and auditable

---

## 🏗️ Architecture Summary
- **VPN Type:** OpenVPN (Full-Tunnel)
- **Deployment Model:** Virtualized firewall-based VPN
- **Authentication:** Certificate-based authentication
- **Traffic Handling:** All client traffic routed through the VPN
- **Network Segmentation:** VLAN-based access control

Remote users connect to the VPN and are securely routed into authorized network segments based on firewall rules and ACLs.

---

## 1. Create a Certificate Authority (CA)

1. Log in to OPNsense.
2. Navigate to **System → Trust → Authorities**.
3. Click **+ Add** to create a new CA.
4. Configure the CA:
   - **Method:** Create an internal Certificate Authority  
   - **Name/Description:** `OpenVPN Certificate Authority`
   - **Key Type:** `RSA 4096`
   - **Digest Algorithm:** `SHA512`
   - **Lifetime:** Default (e.g., ~825 days)
   - **Organization Info:** Country, State, City, Organization, Org Unit, Email
   - **Common Name:** `OpenVPN Certificate Authority`
5. Save the CA.

![Step 1 - Certificate Authority](https://github.com/Chuchu-afk/OpenVPN/blob/main/Project%20folder/image.png?raw=true)

---

## 2. Create a Server Certificate

1. Navigate to **System → Trust → Certificates**.
2. Click **+ Add** to create a new certificate.
3. Configure the server certificate:
   - **Type:** Server Certificate
   - **Key Type:** `RSA 4096`
   - **Digest Algorithm:** `SHA512`
   - **Issuer:** `OpenVPN Certificate Authority`
   - **Common Name:** `OpenVPN Server Certificate`
4. Save the certificate.

![Step 2 - Server Certificate](https://github.com/Chuchu-afk/OpenVPN/blob/main/Project%20folder/Screenshot%202025-10-10%20034530.png?raw=true)

---

## 3. Create a VPN User and Client Certificate

1. Navigate to **System → Access → Users**.
2. Click **+ Add** to create a new VPN user:
   - **Username:** `VPNUser`
   - **Password:** Strong password
   - (Optional) Full Name and Email
3. Add the user to the **Admins** group to allow certificate creation:
   - **System → Access → Groups → Admins**
4. Create a client certificate for the user:
   - Click the certificate icon next to the VPN user.
   - **Type:** Client Certificate
   - **Key Type:** `RSA 4096`
   - **Digest Algorithm:** `SHA512`
   - **Issuer:** `OpenVPN Certificate Authority`
   - **Common Name:** `VPNUser Certificate`
5. Save the certificate.

![Step 3 - VPN User + Client Certificate](https://github.com/Chuchu-afk/OpenVPN/blob/main/Project%20folder/image%20(1).png?raw=true)

---

## 4. Create a Static Key (TLS Key)

1. Navigate to **VPN → OpenVPN → Static Keys**.
2. Click **+ Add** to generate a new static key:
   - **Name:** `OpenVPN Static Key`
3. Click the gear icon to generate the key and save.

![Step 4 - Static Key](https://github.com/Chuchu-afk/OpenVPN/blob/main/Project%20folder/image%20(7).png?raw=true)

---

## 5. Configure the OpenVPN Server

1. Navigate to **VPN → OpenVPN → Instances → + Add**.
2. Configure the server instance:
   - **Role:** Server
   - **Protocol:** UDP (IPv4)
   - **Port:** `1194`
   - **Type:** Tunnel
   - **Server Certificate:** `OpenVPN Server Certificate`
   - **TLS Key:** `OpenVPN Static Key`
   - **Authentication:** Local Database
   - **IPv4 Local Network:** `<LAN_SUBNET>` (example: `192.168.1.0/24`)
   - **Tunnel Network:** `<TUNNEL_SUBNET>` (example: `172.16.1.0/24`)
   - **Topology:** Subnet
   - **Client to Client:** Enabled (optional—enable only if required)
   - **Push Options:** Block outside DNS, register DNS
   - **Redirect Gateway:** Default (Full Tunnel)
3. Save and **Apply Changes**.

> **Routing note (important):** Under the routing/local network settings, the “local network” should align to your internal LAN subnet (commonly `192.168.1.0/24`), not the tunnel subnet.

![Step 5 - OpenVPN Instance Settings](https://github.com/Chuchu-afk/OpenVPN/blob/main/Project%20folder/image%20(2).png?raw=true)

---

## 6. Assign the OpenVPN Interface

1. Navigate to **Interfaces → Assignments**.
2. Add the OpenVPN server interface.
3. Enable the interface and save changes.

![Step 6 - Interface Assignment](https://github.com/Chuchu-afk/OpenVPN/blob/main/Project%20folder/image%20(3).png?raw=true)

---

## 7. Configure Firewall Rules

### 7A. WAN Interface Rule (Allow incoming VPN)

1. Navigate to **Firewall → Rules → WAN → + Add**.
2. Allow UDP traffic to OpenVPN server:
   - **Protocol:** UDP
   - **Destination Port:** `1194`
   - **Source:** Any
   - **Description:** `Allow OpenVPN traffic`
3. Save and **Apply Changes**.

![Step 7A - WAN Rule](https://github.com/Chuchu-afk/OpenVPN/blob/main/Project%20folder/image%20(4).png?raw=true)

### 7B. OpenVPN Interface Rule (Allow client access)

1. Navigate to **Firewall → Rules → OpenVPN → + Add**.
2. Allow client access:
   - **Protocol:** Any
   - **Source:** `OpenVPN Server Network`
   - **Destination:** Any (tighten later if needed)
   - **Description:** `Allow client access`
3. Save and **Apply Changes**.

![Step 7B - OpenVPN Rule](https://github.com/Chuchu-afk/OpenVPN/blob/main/Project%20folder/image%20(5).png?raw=true)

---

## 8. Export Client Configuration

1. Navigate to **VPN → OpenVPN → Client Export**.
2. Select the OpenVPN server instance.
3. Export **File Only** configuration.
4. Choose the user (e.g., `VPNUser`) and download the `.ovpn` profile.
5. Import the profile into an OpenVPN client and connect (enter username/password if prompted).

> **Best practice:** Use a hostname (domain) instead of your raw WAN IP. If you own a domain in Cloudflare, use that hostname during export.

### 📸 Screenshot for this step
![Step 8 - Client Export](https://github.com/Chuchu-afk/OpenVPN/blob/main/Project%20folder/image%20(6).png?raw=true)

---
## 9. Test the Connection

1. Open the OpenVPN client and connect using the exported `.ovpn` profile.
2. Authenticate using the VPN username and password when prompted.
3. Verify the client is assigned an IP address from the VPN tunnel network.
4. Test access to authorized internal LAN resources.
5. Add the **OpenVPN Client widget** to the OPNsense dashboard to monitor active connections.

### ✅ Validation Criteria
- VPN client reports **Connected** status
- Client receives an IP from the configured tunnel subnet
- Internal resources are reachable based on firewall rules
- Active session appears in the OPNsense OpenVPN dashboard widget
## Additional Steps

### Protocol & Routing Adjustments
- Ensure the OpenVPN server instance is configured to use **UDP (IPv4)**.
- In the OpenVPN instance routing section, confirm the **Local Network** is set to the internal LAN subnet  
  (e.g., `192.168.1.0/24`) and **not** the tunnel network.

### ISP Router Port Forwarding
If OPNsense is behind an ISP router, port forwarding is required to allow inbound VPN traffic.

1. Log into the ISP router web interface.
   - The public IP can be identified using a service such as “What is my IP”.
2. Navigate to the **Advanced** or **Port Forwarding** section.
3. Create a port forwarding rule with the following parameters:
---
# Cloudflare Setup (Hostname + Optional DDNS)

## 1) Create a Cloudflare DNS Record

1. Log in to Cloudflare.
2. Go to **DNS → Records → Add Record**.
3. **Type:** `A`  
   **Name:** `connect` (or preferred subdomain)  
   **IPv4 Address:** `<YOUR_WAN_IP>`  
   **Proxy status:** **DNS Only** (gray cloud)
4. Save.
## 2) Create a Cloudflare API Token (for DDNS)

1. **Profile → API Tokens → Create Token → Custom Token**
2. **Permissions:**
   - Zone → DNS → Edit
   - Zone → Zone → Read
3. **Zone Resources:** Include → Specific Zone → `yourdomain.com`
4. Save the token securely.

### 📸 Screenshot for this step
![Cloudflare 2 - API Token](https://github.com/Chuchu-afk/OpenVPN/blob/main/Project%20folder/OPNvpn%20(1).png?raw=true)
![My OpnSense login](https://github.com/Chuchu-afk/OpenVPN/blob/main/Project%20folder/Screenshot%202025-10-10%20013343.png?raw=true)
![My OpnSense Dashboard](https://github.com/Chuchu-afk/OpenVPN/blob/main/Project%20folder/dash%20(1).png?raw=true)

---

## Troubleshooting

During implementation and testing, several issues were identified and resolved. The steps below document the root causes, remediation actions, and validation performed.

---

### Issue 1: Dynamic DNS (DDNS) Options Missing

**Symptoms**
- Dynamic DNS options were unavailable in the OPNsense UI.
- Unable to configure DDNS for a domain-based VPN endpoint.

**Root Cause**
- OPNsense was not running the required hotfix level to support DDNS functionality.
- The system was on version `2.7`, while DDNS support required `2.7.3` or later.

**Resolution**
- Updated OPNsense to the latest available version (`2.7.3`, later `2.7.4`).
- Installed the DDNS plugin:
  - `System → Firmware → Updates`
  - `System → Firmware → Plugins`
  - Installed `os-ddns` plugin
  - Rebooted OPNsense
- Verified DDNS availability under:
  - `Services → Dynamic DNS`

**Validation**
- DDNS service became available and configurable in the UI.
- Cloudflare DDNS integration was successfully enabled.

---

### Issue 2: No Domain Name for VPN Endpoint

**Symptoms**
- VPN connections required using a raw public IP address.
- Public IP exposure during VPN connection attempts.

**Root Cause**
- No domain name was configured to abstract the WAN IP.

**Resolution**
- Implemented a Cloudflare DNS record and DDNS integration (documented in the Cloudflare setup section).
- Updated OpenVPN client exports to use the domain name instead of the WAN IP.

**Validation**
- VPN clients successfully connected using the domain hostname.
- WAN IP changes no longer required client reconfiguration.

---

### Issue 3: DNS Resolution Failures on Clients

**Symptoms**
- Clients experienced errors such as:
  - `This site can’t be reached`
  - `DNS_PROBE_FINISHED_NXDOMAIN`
- Internet access partially or completely unavailable after connecting to VPN.

**Root Cause**
- Improper DNS configuration on OPNsense.
- Conflicting DHCP services (Kea / Dnsmasq vs ISC DHCP).
- DNS traffic resolving through ISP router instead of defined upstream resolvers.

**Resolution**
1. Verified system DNS configuration:
   - `System → Settings → General`
   - Configured valid upstream DNS servers (e.g., `1.1.1.1`, `8.8.8.8`)
   - Disabled:
     - “Allow DNS server list to be overridden by DHCP/PPP on WAN”
     - “Do not use the local DNS service as a nameserver for this system”

2. Validated LAN interface configuration:
   - `Interfaces → LAN`
   - Confirmed interface enabled with static IPv4 assignment

3. Ensured a single active DHCP service:
   - Verified `ISC DHCPv4` was intended to be the active DHCP server
   - Disabled `Kea DHCP` and `Dnsmasq DNS & DHCP`
   - Rebooted OPNsense to apply changes

4. Reconfigured LAN DHCP:
   - `Services → ISC DHCPv4 → LAN`
   - Enabled DHCP server on LAN interface
   - Configured IP range based on LAN subnet
   - Set DNS server to LAN IP or public DNS (e.g., `1.1.1.1`, `8.8.8.8`)
   - Saved and applied changes

5. Renewed client network configuration:
   - Disconnected and reconnected network interface
   - Ran:
     - `ipconfig /renew`
     - `nslookup google.com`

**Validation**
- DNS resolution restored successfully.
- Clients could access external websites and internal resources without errors.
- VPN connectivity remained stable.

---

### Issue 4: Domain Redirecting to ISP Router Login

**Symptoms**
- Accessing the VPN domain from inside the local network redirected to the ISP router admin page.

**Root Cause**
- Domain correctly resolved to the public WAN IP.
- ISP router presents its admin interface when accessed internally.
- This behavior does not expose the interface externally.

**Resolution**
- Confirmed remote administration on the ISP router was disabled.
- Tested access from external networks to validate the admin page was not exposed.

**Validation**
- ISP router admin interface was not accessible from outside the local network.
- Behavior considered expected and non-impacting to VPN security.


