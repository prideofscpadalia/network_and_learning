# Cisco Packet Tracer : TCP, FTP, Email Server Stimulation, DNS

A comprihensive local network simulation built in Cisco Packet Tracer demostrating core networking services including **DNS** , **SMTP/POP3 Email** , and **FTP** .

## Network Topology & Architectue 
- **Topology Type:** Star Topology centered around a managed Layer 2 Switch.
- **Media:** Copper Straight-Through cables connecting all end devices to the switch ports.
- **Device Allocation:**
  - **1x Server Node:** Acts as the centralized repository for DNS records, mailboxes, and FTP storage.
  - **4x Client Nodes:** Simulated workstations used to test web requests, email transmission, and file transfers.

## IP Addressing Scheme
All devices are configured statically to maintain predictable service resolution:
- **Server IP:** `192.168.11.5` (Static)
- **Subnet Mask:** `255.255.255.0`
- **Default Gateway / DNS Server:** `192.168.11.5` (Pointing to the local server)
- **Client IPs:** Assigned sequentially from `192.168.11.1` to `192.168.11.4`


## Service Configured
1. **DNS Service:** Resolves domain names (e.g., `www.google.com, jipanditji.com`) to the server's static IP.
2. **Email Service:** Configured local mail accounts using SMTP and POP3 protocols to send/receive internal emails.
3. **FTP Service:** Enabled file transfer capabilities allowing clients to upload (`put`) and download (`get`) files from the server via command line.

## Services Implemented

### A. DNS (Domain Name System) Service
- **Purpose:** Maps human-readable domain names to the server's IP address.
- **Configuration:** A custom DNS record was added pointing `www.google.com or jipanditji.com` to `192.168.11.5`.
- **Verification:** Clients resolve the domain name to test web and service connectivity seamlessly.

### B. Email Service (SMTP / POP3)
- **Purpose:** Handles internal electronic mail communication between client nodes.
- **Configuration:** 
  - Mail service enabled on the central server with domain mapping. [domain singa.com]
  - User accounts created for designated client workstations. 
  [
    usernames:{
        - divyanshu@singa.com
        - padalia@singa.com
    }
    password:{
        - divyanshu
        - padalia
    }
  ]
- **Protocols:** Uses **SMTP** for outgoing mail and **POP3** for incoming mail retrieval.

### C. FTP (File Transfer Protocol) Service
- **Purpose:** Provides centralized file storage and retrieval over TCP port 21.
- **Configuration:** 
  - FTP service enabled on the server.
  - Dedicated user credentials configured with read/write/delete permissions.
  [
    username:{
        - admin
    }
    password:{
        - admin
    }
  ]
- **Workflow:** Clients connect via command line (`ftp www.google.com` or 'jipanditji.com'), crrate a test file, upload test files using the `put` command, and download files using `get`.

## Simulation & Troubleshooting
- **Simulation Mode:** Used Packet Tracer's simulation mode to inspect Protocol Data Units (PDUs) layer-by-layer (checking TCP handshakes, DNS queries, and data packets).
