# UDP Broadcast Relay - Usage Guide

This document provides step-by-step howtos for configuring udpbroadcastrelay in various environments and use cases.

---

## Table of Contents

1. [Ubuntu Server: Relay UDP Broadcasts Across VLANs](#ubuntu-server-relay-udp-broadcasts-across-vlans)
2. [Relaying UDP Broadcasts Over WireGuard VPN (Site-to-Site)](#relaying-udp-broadcasts-over-wireguard-vpn)
3. [Relaying UDP Broadcasts to WireGuard Road Warrior Clients](#relaying-udp-broadcasts-to-wireguard-road-warrior-clients)

---

## Ubuntu Server: Relay UDP Broadcasts Across VLANs

### Description

This howto guides you through configuring an Ubuntu server to relay UDP broadcast packets between multiple VLANs. This is useful when you have devices on separate VLANs that need to discover each other using UDP broadcast protocols (e.g., game servers, media devices, IoT devices) but cannot communicate directly due to network segmentation.

The relay server will listen for UDP broadcast packets on a specified port across multiple VLAN interfaces and forward them to all other configured VLANs, making devices on different network segments visible to each other.

### Assumptions

- Ubuntu Server 24.04 LTS (commands may vary slightly for other versions)
- The server has a single physical network interface connected to a trunk port carrying multiple VLANs
- You have root or sudo access to the server
- VLANs are already configured on your network infrastructure (switches/router)
- The server will act as the relay point and has IP addresses assigned on each VLAN that needs relaying
- You know the UDP port(s) used by the service you want to relay
- Basic familiarity with Linux command line and networking concepts

### Example Environment

| VLAN ID | Network         | Server IP       | Purpose          |
|---------|-----------------|-----------------|------------------|
| 10      | 192.168.10.0/24 | DHCP assigned   | Main LAN         |
| 20      | 192.168.20.0/24 | DHCP assigned   | IoT devices      |
| 30      | 192.168.30.0/24 | DHCP assigned   | Gaming/Media     |

### Steps

1. **Configure the switch port as a trunk port**

   Before configuring the server, you must configure the switch port that the server is connected to as a **trunk port**. This does not happen automatically—by default, most managed switches configure ports as access ports (single VLAN, untagged).

   The exact steps depend on your switch vendor:

   | Vendor | General Approach |
   |--------|------------------|
   | **Cisco** | `switchport mode trunk` + `switchport trunk allowed vlan 10,20,30` |
   | **Ubiquiti UniFi** | Edit port profile → Set to "All" or create custom profile with tagged VLANs |
   | **Netgear** | VLAN → Port VLAN Membership → Tag VLANs 10, 20, 30 on the port |
   | **TP-Link** | VLAN → 802.1Q VLAN → Set port as Tagged for each VLAN |
   | **MikroTik** | Bridge → VLANs → Add tagged VLANs to the port |

   Ensure VLANs 10, 20, and 30 (or whichever VLANs you need) are tagged on the port. Optionally configure a native VLAN for untagged traffic.

2. **Install prerequisites and dependencies**

   Install the build tools and networking packages required to compile udpbroadcastrelay and configure VLAN interfaces:

   ```bash
   sudo apt update
   sudo apt install -y build-essential git vlan
   ```

   Enable the 8021q kernel module for VLAN tagging support:

   ```bash
   sudo modprobe 8021q
   echo "8021q" | sudo tee /etc/modules-load.d/8021q.conf
   ```

3. **Configure VLAN interfaces on Ubuntu**

   Ubuntu 24.04 uses Netplan for network configuration. First, check what configuration files already exist:

   ```bash
   ls -la /etc/netplan/
   ```

   Netplan processes files in alphabetical order, and later files can override earlier ones. You have two options:

   **Option A: Create a separate VLAN config file (recommended)**

   If a file already exists that configures your main interface (e.g., `00-installer-config.yaml` or `50-cloud-init.yaml`), create a separate file for VLANs:

   ```bash
   sudo nano /etc/netplan/60-vlans.yaml
   ```

   Add only the VLAN configurations (the parent interface is already defined in the other file):

   ```yaml
   network:
     version: 2
     vlans:
       eth0.10:
         id: 10
         link: eth0
         dhcp4: yes
       eth0.20:
         id: 20
         link: eth0
         dhcp4: yes
       eth0.30:
         id: 30
         link: eth0
         dhcp4: yes
   ```

   **Option B: Single configuration file**

   If no existing netplan file configures your interface, or you prefer a single file, create or edit:

   ```bash
   sudo nano /etc/netplan/01-netcfg.yaml
   ```

   Add both the ethernet and VLAN interface configurations. Replace `eth0` with your actual physical interface name (use `ip link` to find it):

   ```yaml
   network:
     version: 2
     ethernets:
       eth0:
         dhcp4: no
     vlans:
       eth0.10:
         id: 10
         link: eth0
         dhcp4: yes
       eth0.20:
         id: 20
         link: eth0
         dhcp4: yes
       eth0.30:
         id: 30
         link: eth0
         dhcp4: yes
   ```

   Apply the Netplan configuration:

   ```bash
   sudo netplan apply
   ```

   Verify the VLAN interfaces are created:

   ```bash
   ip addr show
   ```

   You should see `eth0.10`, `eth0.20`, and `eth0.30` interfaces with their assigned IP addresses.

   > **Note:** The number of IP addresses assigned depends on your configuration. With the example above, you'll get 3 IPs (one per VLAN interface). If your existing netplan file has `dhcp4: yes` on `eth0`, you'll also get an IP on the native/untagged VLAN.

4. **Download and compile udpbroadcastrelay**

   Clone the repository and compile the program:

   ```bash
   cd /opt
   sudo git clone https://github.com/udp-broadcast-relay/udp-broadcast-relay.git udpbroadcastrelay
   cd udpbroadcastrelay
   sudo make
   ```

   Copy the compiled binary to a system directory:

   ```bash
   sudo cp udpbroadcastrelay /usr/local/bin/
   sudo chmod +x /usr/local/bin/udpbroadcastrelay
   ```

5. **Test udpbroadcastrelay manually**

   Run a quick test to ensure the relay works. This example relays UDP port 1900 (SSDP/UPnP discovery) across all three VLANs:

   ```bash
   sudo /usr/local/bin/udpbroadcastrelay \
       --id 1 \
       --port 1900 \
       --dev eth0.10 \
       --dev eth0.20 \
       --dev eth0.30 \
       -f
   ```

   The `-f` flag runs the relay in the foreground so you can see output. Press `Ctrl+C` to stop.

   Common UDP ports you may want to relay:
   - `1900` - SSDP/UPnP (Chromecast, smart TVs, etc.)
   - `5353` - mDNS (with `--multicast 224.0.0.251`)
   - `27015` - Source engine games (Steam)
   - `7359` - Jellyfin server discovery
   - `32410` - Plex GDM discovery

6. **Create a systemd service for automatic startup**

   Create a systemd service file to run udpbroadcastrelay at boot:

   ```bash
   sudo nano /etc/systemd/system/udpbroadcastrelay.service
   ```

   Add the following content (adjust the port and interfaces for your needs):

   ```ini
   [Unit]
   Description=UDP Broadcast Relay Service
   After=network-online.target
   Wants=network-online.target

   [Service]
   Type=simple
   ExecStart=/usr/local/bin/udpbroadcastrelay --id 1 --port 1900 --dev eth0.10 --dev eth0.20 --dev eth0.30
   Restart=on-failure
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   ```

   > **Note:** If you need to relay multiple ports, create additional service files (e.g., `udpbroadcastrelay-mdns.service`) with different `--id` values for each instance.

7. **Enable and start the service**

   Reload systemd to recognize the new service, then enable and start it:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable udpbroadcastrelay.service
   sudo systemctl start udpbroadcastrelay.service
   ```

   Check the service status:

   ```bash
   sudo systemctl status udpbroadcastrelay.service
   ```

8. **Verify the relay is working**

   Check that the service is running and listening:

   ```bash
   ps aux | grep udpbroadcastrelay
   ```

   View the service logs:

   ```bash
   sudo journalctl -u udpbroadcastrelay.service -f
   ```

   To test the relay functionality:
   - Use a device on one VLAN to perform a discovery (e.g., open a Chromecast-enabled app)
   - Check if devices on other VLANs are discovered
   - Monitor the journal logs to see packets being relayed

9. **Troubleshooting common issues**

   **Service fails to start:**
   - Check logs: `sudo journalctl -u udpbroadcastrelay.service -e`
   - Verify interfaces exist: `ip link show`
   - Ensure the binary is executable: `ls -la /usr/local/bin/udpbroadcastrelay`

   **Devices not discovering each other:**
   - Verify firewall rules allow UDP traffic on the relayed port:
     ```bash
     sudo ufw allow 1900/udp
     ```
   - Check that packets are being received (run in foreground with `-d` for debug output):
     ```bash
     sudo /usr/local/bin/udpbroadcastrelay --id 1 --port 1900 --dev eth0.10 --dev eth0.20 --dev eth0.30 -f -d
     ```
   - Ensure the server has IP addresses on all VLAN interfaces
   - Verify the switch trunk port is correctly tagging VLANs

   **Duplicate packets or loops:**
   - Ensure each udpbroadcastrelay instance uses a unique `--id` value (1-63)
   - Use `--blockid` to prevent relay loops between multiple relay servers

   **Permission denied errors:**
   - The relay must run as root to create raw sockets
   - Verify the service file does not include `User=` directive

### Sub-Example: HDHomeRun Discovery Across VLANs

HDHomeRun TV tuners use UDP broadcast on port **65001** for device discovery. If your HDHomeRun device is on a different VLAN than your media server or viewing devices (e.g., Plex, Jellyfin, or the HDHomeRun app), you'll need to relay this port.

**Manual test command:**

```bash
sudo /usr/local/bin/udpbroadcastrelay \
    --id 2 \
    --port 65001 \
    --dev eth0.10 \
    --dev eth0.20 \
    -f -d
```

**Systemd service file** (`/etc/systemd/system/udpbroadcastrelay-hdhomerun.service`):

```ini
[Unit]
Description=UDP Broadcast Relay for HDHomeRun Discovery
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/udpbroadcastrelay --id 2 --port 65001 --dev eth0.10 --dev eth0.20
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**Enable and start:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable udpbroadcastrelay-hdhomerun.service
sudo systemctl start udpbroadcastrelay-hdhomerun.service
```

**Firewall rule:**

```bash
sudo ufw allow 65001/udp
```

> **Note:** Each running instance of `udpbroadcastrelay` requires a unique `--id` value (1-63). The ID is embedded in relayed packets to prevent relay loops—without unique IDs, multiple relay servers or instances could re-relay each other's packets infinitely. If you're already running the main SSDP relay with `--id 1` (from Step 6), use `--id 2` for HDHomeRun, `--id 3` for mDNS, etc.

---

## Relaying UDP Broadcasts Over WireGuard VPN

### Description

This section explains how to relay UDP broadcast packets between your local network and a remote network connected via WireGuard VPN. This is useful when you want devices on a remote site (e.g., a vacation home, office, or mobile device) to discover services on your home network, or vice versa.

Common use cases:
- Access Plex/Jellyfin servers from a remote location without manual IP configuration
- Discover Chromecast or AirPlay devices across VPN-connected sites
- Allow HDHomeRun tuners to be discovered by clients on remote networks
- Enable game server discovery across multiple locations

### Prerequisites

- A working WireGuard VPN connection between sites
- The relay server must have the WireGuard interface configured and active
- Both ends of the VPN must allow UDP traffic on the ports you want to relay

### Example Environment

| Location | Network | WireGuard IP | Interface |
|----------|---------|--------------|-----------|
| Home (server) | 192.168.10.0/24 | 10.0.0.1 | wg0 |
| Remote site | 192.168.20.0/24 | 10.0.0.2 | wg0 |

In this example, the relay server is at home and has:
- `eth0` or `eth0.10` on the local LAN (192.168.10.x)
- `wg0` for the WireGuard tunnel (10.0.0.1)

### Configuration

1. **Verify WireGuard is running and the interface exists**

   ```bash
   ip link show wg0
   ip addr show wg0
   ```

   You should see the `wg0` interface with its assigned IP address.

2. **Ensure WireGuard routing includes the remote network**

   WireGuard uses `AllowedIPs` to control routing. For the relay to send and receive packets to/from the remote network, the remote subnet must be included in your peer configuration.

   Check your WireGuard config:

   ```bash
   sudo nano /etc/wireguard/wg0.conf
   ```

   The peer section should include the remote subnet in `AllowedIPs`:

   ```ini
   [Peer]
   PublicKey = <remote-public-key>
   AllowedIPs = 10.0.0.2/32, 192.168.20.0/24
   Endpoint = remote.example.com:51820
   PersistentKeepalive = 25
   ```

   - `10.0.0.2/32` — The peer's WireGuard tunnel IP
   - `192.168.20.0/24` — The remote LAN (required for routing relayed packets)

   Without the remote subnet, packets relayed by `udpbroadcastrelay` won't be routed through the tunnel, and incoming packets from the remote network will be dropped.

   If you modified the config, restart WireGuard:

   ```bash
   sudo systemctl restart wg-quick@wg0
   ```

3. **Test udpbroadcastrelay with WireGuard**

   Run a test relay that includes the WireGuard interface:

   ```bash
   sudo /usr/local/bin/udpbroadcastrelay \
       --id 1 \
       --port 1900 \
       --dev eth0 \
       --dev wg0 \
       -f -d
   ```

   > **Note:** Use `eth0` for your local LAN interface, or `eth0.10` if using VLANs.

4. **Create a systemd service for WireGuard relay**

   Create a dedicated service file:

   ```bash
   sudo nano /etc/systemd/system/udpbroadcastrelay-vpn.service
   ```

   Add the following content:

   ```ini
   [Unit]
   Description=UDP Broadcast Relay over WireGuard VPN
   After=network-online.target wg-quick@wg0.service
   Wants=network-online.target
   Requires=wg-quick@wg0.service

   [Service]
   Type=simple
   ExecStart=/usr/local/bin/udpbroadcastrelay --id 10 --port 1900 --dev eth0 --dev wg0
   Restart=on-failure
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   ```

   > **Note:** The service depends on `wg-quick@wg0.service` to ensure WireGuard is up before the relay starts. Adjust the interface name if your WireGuard config uses a different name.

5. **Enable and start the service**

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable udpbroadcastrelay-vpn.service
   sudo systemctl start udpbroadcastrelay-vpn.service
   ```

6. **Configure firewall rules**

   Allow UDP traffic on the relayed port through the WireGuard interface:

   ```bash
   sudo ufw allow in on wg0 to any port 1900 proto udp
   sudo ufw allow in on eth0 to any port 1900 proto udp
   ```

### Relaying Multiple Ports Over VPN

If you need to relay multiple services (e.g., SSDP, Plex, and HDHomeRun), create separate service files with unique IDs:

| Service | Port | ID | Service File |
|---------|------|----|--------------|
| SSDP/UPnP | 1900 | 10 | udpbroadcastrelay-vpn-ssdp.service |
| Plex GDM | 32410 | 11 | udpbroadcastrelay-vpn-plex.service |
| HDHomeRun | 65001 | 12 | udpbroadcastrelay-vpn-hdhomerun.service |

Example for Plex:

```bash
sudo nano /etc/systemd/system/udpbroadcastrelay-vpn-plex.service
```

```ini
[Unit]
Description=UDP Broadcast Relay for Plex over WireGuard
After=network-online.target wg-quick@wg0.service
Wants=network-online.target
Requires=wg-quick@wg0.service

[Service]
Type=simple
ExecStart=/usr/local/bin/udpbroadcastrelay --id 11 --port 32410 --dev eth0 --dev wg0
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Combining VLANs and WireGuard

If your local network uses VLANs and you want to relay across both VLANs and the VPN, include all interfaces:

```bash
sudo /usr/local/bin/udpbroadcastrelay \
    --id 1 \
    --port 1900 \
    --dev eth0.10 \
    --dev eth0.20 \
    --dev eth0.30 \
    --dev wg0 \
    -f
```

This allows devices on any local VLAN to discover (and be discovered by) devices on the remote VPN network.

### Troubleshooting WireGuard Relay

**Relay not receiving packets from VPN:**
- Verify WireGuard is connected: `sudo wg show`
- Check that the remote subnet is in `AllowedIPs`
- Ensure `PersistentKeepalive` is set to keep the tunnel active

**Packets not reaching remote devices:**
- Verify IP forwarding is enabled on the relay server:
  ```bash
  sudo sysctl net.ipv4.ip_forward=1
  echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
  ```
- Check firewall rules on both ends of the VPN
- Ensure the remote device's routing table includes the path back

**Service fails to start (interface not found):**
- The WireGuard interface may not be ready when the service starts
- Verify the service has `Requires=wg-quick@wg0.service` in the `[Unit]` section
- Check if the interface name matches your WireGuard config

**Bidirectional relay:**
- For discovery to work both ways, you may need to run udpbroadcastrelay on *both* ends of the VPN tunnel
- Use different `--id` values on each end and consider using `--blockid` to prevent loops

---

## Relaying UDP Broadcasts to WireGuard Road Warrior Clients

### Description

This section covers the "road warrior" or "hub-and-spoke" VPN topology, where individual devices (phones, laptops, tablets) connect directly to a central WireGuard server. This differs from site-to-site VPN where you connect entire networks.

Common use cases:
- Mobile devices discovering home media servers (Plex, Jellyfin) while away
- Laptops finding network printers or Chromecast devices through VPN
- Remote workers accessing office services that use UDP broadcast discovery

### How Road Warrior Differs from Site-to-Site

| Aspect | Site-to-Site | Road Warrior |
|--------|--------------|---------------|
| **Peers** | 1 peer = 1 remote network | 1 peer = 1 device |
| **AllowedIPs** | Includes remote subnet (e.g., `192.168.20.0/24`) | Only the device's tunnel IP (e.g., `10.0.0.2/32`) |
| **Typical use** | Connecting two offices/homes | Mobile devices, laptops |
| **Relay runs on** | Both ends (optionally) | Server only |

### Example Environment

**Server (home):**
- LAN: `192.168.10.0/24` on `eth0`
- WireGuard: `10.0.0.1/24` on `wg0`

**Clients:**
| Device | WireGuard IP | AllowedIPs (on server) |
|--------|--------------|------------------------|
| Alice's phone | 10.0.0.2 | 10.0.0.2/32 |
| Bob's laptop | 10.0.0.3 | 10.0.0.3/32 |
| Carol's tablet | 10.0.0.4 | 10.0.0.4/32 |

### Server Configuration

1. **Configure WireGuard with a proper subnet**

   The server's WireGuard interface should use a subnet (e.g., `/24`), not a single IP:

   ```bash
   sudo nano /etc/wireguard/wg0.conf
   ```

   ```ini
   [Interface]
   PrivateKey = <server-private-key>
   Address = 10.0.0.1/24
   ListenPort = 51820

   [Peer]
   # Alice's phone
   PublicKey = <alice-public-key>
   AllowedIPs = 10.0.0.2/32

   [Peer]
   # Bob's laptop
   PublicKey = <bob-public-key>
   AllowedIPs = 10.0.0.3/32

   [Peer]
   # Carol's tablet
   PublicKey = <carol-public-key>
   AllowedIPs = 10.0.0.4/32
   ```

   > **Note:** The `AllowedIPs` for each peer is `/32` (single IP) because it controls routing, not the client's network. This is correct for road warrior setups.

2. **Configure clients with matching subnet**

   Each client's WireGuard config should use the same subnet mask as the server:

   ```ini
   [Interface]
   PrivateKey = <client-private-key>
   Address = 10.0.0.2/24
   DNS = 192.168.10.1

   [Peer]
   PublicKey = <server-public-key>
   AllowedIPs = 10.0.0.0/24, 192.168.10.0/24
   Endpoint = home.example.com:51820
   PersistentKeepalive = 25
   ```

   Key points:
   - `Address = 10.0.0.2/24` — The `/24` subnet allows the client to receive broadcasts on the `10.0.0.x` network
   - `AllowedIPs` includes `10.0.0.0/24` (VPN subnet) and `192.168.10.0/24` (home LAN)

3. **Run udpbroadcastrelay on the server**

   The relay configuration is the same as site-to-site:

   ```bash
   sudo /usr/local/bin/udpbroadcastrelay \
       --id 1 \
       --port 1900 \
       --dev eth0 \
       --dev wg0 \
       -f -d
   ```

   When the relay sends a broadcast packet to `wg0`, it goes to the WireGuard subnet's broadcast address (e.g., `10.0.0.255`). All connected clients on that subnet will receive it.

4. **Create a systemd service**

   ```bash
   sudo nano /etc/systemd/system/udpbroadcastrelay-roadwarrior.service
   ```

   ```ini
   [Unit]
   Description=UDP Broadcast Relay for Road Warrior VPN Clients
   After=network-online.target wg-quick@wg0.service
   Wants=network-online.target
   Requires=wg-quick@wg0.service

   [Service]
   Type=simple
   ExecStart=/usr/local/bin/udpbroadcastrelay --id 20 --port 1900 --dev eth0 --dev wg0
   Restart=on-failure
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   ```

   Enable and start:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable udpbroadcastrelay-roadwarrior.service
   sudo systemctl start udpbroadcastrelay-roadwarrior.service
   ```

### What Clients Will Discover

With this configuration:

| From | Can Discover |
|------|--------------|
| VPN client (10.0.0.x) | Devices on home LAN (192.168.10.x) |
| Home LAN device (192.168.10.x) | VPN clients (10.0.0.x) |
| VPN client | Other VPN clients (if connected simultaneously) |

### Combining with VLANs

If your home network uses VLANs, include all VLAN interfaces:

```bash
sudo /usr/local/bin/udpbroadcastrelay \
    --id 1 \
    --port 1900 \
    --dev eth0.10 \
    --dev eth0.20 \
    --dev eth0.30 \
    --dev wg0 \
    -f
```

This allows road warrior clients to discover devices on all your home VLANs.

### Troubleshooting Road Warrior Relay

**Clients not receiving broadcasts:**
- Verify the client's `Address` uses a subnet (e.g., `/24`), not `/32`
- Check that the server's WireGuard interface also uses `/24`
- Ensure the client is actually connected: `sudo wg show` on the server

**Discovery works one way only:**
- Verify the client's `AllowedIPs` includes both the VPN subnet and your home LAN
- Check that `PersistentKeepalive` is set on the client to keep the tunnel active

**Broadcasts reaching some clients but not others:**
- All clients must be on the same WireGuard subnet (e.g., `10.0.0.0/24`)
- Verify each client has a unique IP address
- Check that clients are online and connected when the broadcast is sent

**Performance considerations:**
- Each broadcast is sent to all connected clients, even if they don't need it
- For many clients, this may increase bandwidth usage on the server's upload
- Consider limiting which ports you relay to only essential services

### mDNS Limitations Over WireGuard VPN

mDNS (used by Apple Bonjour, Avahi, and many IoT devices) is **problematic over WireGuard** and may not work reliably with udpbroadcastrelay or other solutions.

#### Why mDNS is different from regular broadcasts:

| Aspect | Broadcast (e.g., SSDP) | mDNS |
|--------|------------------------|------|
| **Destination** | Broadcast address (255.255.255.255) | Multicast address (224.0.0.251) |
| **Port** | Varies (e.g., 1900) | 5353 |
| **Scope** | Subnet | Link-local by design |
| **TTL handling** | Usually forwarded | Often rejected if routed |

#### The core problem:

WireGuard doesn't support multicast. It routes packets based on `AllowedIPs`, which maps destination IPs to specific peers. Multicast addresses like `224.0.0.251` aren't in any peer's `AllowedIPs`, so the packet has nowhere to go.

```
mDNS packet → 224.0.0.251 on wg0 → WireGuard doesn't know where to route it → dropped
```

This affects both udpbroadcastrelay and Avahi reflector equally.

#### Partial workaround with udpbroadcastrelay:

The relay can convert multicast to broadcast, which *might* help:

```bash
sudo /usr/local/bin/udpbroadcastrelay \
    --id 5 \
    --port 5353 \
    --dev eth0 \
    --dev wg0 \
    --multicast 224.0.0.251 \
    -f -d
```

When a multicast mDNS packet arrives on `eth0`, the relay sends it as a broadcast to `wg0` (e.g., `10.0.0.255`). If clients have a `/24` subnet configured, they may receive it.

**Limitations:**
- Responses from clients also need to make it back
- Some mDNS implementations reject responses that have been routed
- Apple devices (AirPlay, AirPrint) are particularly strict about link-local scope
- Results vary significantly by device and service

#### Better alternatives for service discovery over VPN:

| Solution | How it works | Reliability |
|----------|--------------|-------------|
| **Use SSDP instead of mDNS** | Many apps support both; SSDP uses broadcast, not multicast | Good — Plex, Jellyfin, Chromecast support SSDP |
| **Direct IP configuration** | Skip discovery; configure server IPs directly in client apps | Best — works always |
| **Tailscale** | Built-in MagicDNS translates mDNS to unicast DNS | Excellent — but requires switching VPN solutions |
| **Unicast DNS-SD** | Advertise services via regular DNS (requires DNS server config) | Good — no multicast needed |

#### Recommendation:

For reliable service discovery with road warrior clients:

1. **Prefer SSDP/UPnP over mDNS** where applications support both
2. **Configure server IPs directly** in mobile apps (most media apps allow manual server entry)
3. **Don't rely on mDNS** for critical functionality over VPN

---
