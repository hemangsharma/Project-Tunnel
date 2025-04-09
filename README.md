# Project Tunnel

## Overview

**Goal:** Seamlessly connect your laptop to your home network using two Raspberry Pis in a site-to-site VPN configuration that also incorporates advanced features.  
**Features:**  
- **Dynamic DNS (DDNS):** Avoid hardcoding your home IP by using a DDNS service.
- **Pi-hole:** Filter ads on your home network.
- **nftables:** Configure NAT and firewall rules with a more modern tool.
- **Web UI:** Monitor your tunnel’s status (using simple tools such as a Flask-based dashboard or another lightweight monitoring tool).
- **Auto-setup Script:** A Bash script to install and configure WireGuard on both devices.

---

## Hardware & Software Requirements

### Hardware:
- 2× Raspberry Pis (e.g., Pi 3B+ or later)
- 2× microSD cards (16GB or higher)
- 2× USB-to-Ethernet adapters (for a second Ethernet port on each)
- Ethernet cables
- A compatible router (with port forwarding on your home network)
- Access to a DDNS service (e.g., No-IP, DuckDNS)

### Software:
- Raspberry Pi OS Lite (or your preferred distro)
- WireGuard (VPN service)
- nftables (firewall/NAT)
- Pi-hole (for ad blocking, only on Home Pi)
- A web server framework (e.g., Python/Flask) for monitoring UI
- SSH access to both Pis

---

## Architecture Diagram

Below is a text-based diagram illustrating the overall connectivity:

```
     [Laptop]
         │ (Ethernet)
         │
   ┌─────────────┐
   │ Travel Pi   │
   │ (Client)    │◄──────────────┐
   └─────────────┘               │
          │                     │  (Internet via local router)
   (VPN Tunnel: wg0)             │
          │                     │  
   ┌─────────────┐       Public Internet  ──► [Dynamic DNS Service]
   │ Home Pi     │◄──────────────┐
   │ (Server,    │               │
   │ Pi-hole)    │               │
   └─────────────┘               │
          │                     │
     [Home LAN Devices]         │
```

### How It Works:
1. **Travel Pi:** Connects to the local router, establishes a WireGuard tunnel.
2. **Home Pi:** Listens for incoming connections from Travel Pi using WireGuard.
3. **Dynamic DNS:** Your home network is reached via a domain (e.g., `yourhome.ddns.net`), which always points to your current home IP.
4. **Pi-hole:** Runs on Home Pi to filter and block ads.
5. **nftables:** Handles NAT and forwarding rules.
6. **Web UI:** Runs on Home Pi to display status and tunnel statistics.

---

## Detailed Setup Procedure

### 1. Raspberry Pi System Preparation (Both Devices)

1. **Flash SD Cards:** Use Raspberry Pi Imager with Raspberry Pi OS Lite.
2. **Enable SSH:** Place an empty file named `ssh` (with no extension) on the boot partition.
3. **Update Systems:**
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```
4. **Rename the Hostnames:**
   - Home Pi:  
     ```bash
     sudo hostnamectl set-hostname tunnel-home
     ```
   - Travel Pi:  
     ```bash
     sudo hostnamectl set-hostname tunnel-travel
     ```

5. **Install Required Packages (WireGuard, nftables, etc.):**
   ```bash
   sudo apt install wireguard nftables git curl -y
   ```

---

### 2. Dynamic DNS Setup (Home Pi)

1. **Sign up for a DDNS Service:** (e.g., [DuckDNS](https://www.duckdns.org) or [No-IP](https://www.noip.com)).  
2. **Install a DDNS Update Client:**
   - For example, if using DuckDNS, follow their instructions to install the update script. Typically, you will:
     - Create a directory (e.g., `/opt/duckdns`)
     - Download the client script and create a cron job that periodically runs it, ensuring your DDNS entry remains updated.

Example cron job entry (edit with your token and domain):
   ```bash
   crontab -e
   # Add:
   */5 * * * * /usr/bin/curl -s "https://www.duckdns.org/update?domains=yourdomain&token=yourtoken&ip=" >/dev/null 2>&1
   ```

---

### 3. Installing Pi-hole on Home Pi

1. **Run the Automatic Installer:**  
   On Home Pi, execute:
   ```bash
   curl -ssl https://install.pi-hole.net | bash
   ```
2. **Follow the Installer Prompts:**  
   You will configure your network interface and upstream DNS settings. Pi-hole will block ads for your network.

---

### 4. Configuring nftables for NAT

Replace iptables rules with nftables rules. For example, on Home Pi, create `/etc/nftables.conf` and add:

```nft
table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100;
        oif "eth0" masquerade  # Change "eth0" if your interface is named differently
    }
}
```

Apply the configuration with:
```bash
sudo nft -f /etc/nftables.conf
```

Repeat similar steps on Travel Pi if needed for forwarding local laptop traffic over the VPN interface (usually “wg0”).

---

### 5. Configuring WireGuard

Follow these steps on each Raspberry Pi:

#### Generate Keypairs on Both Devices:
```bash
wg genkey | tee privatekey | wg pubkey > publickey
```

Keep the private key secret. Share the public key between the devices.

#### **Home Pi (Server) – `/etc/wireguard/wg0.conf`:**
```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <home_pi_private_key>
SaveConfig = true

[Peer]
PublicKey = <travel_pi_public_key>
AllowedIPs = 10.0.0.2/32
```
- **Dynamic DNS:** When providing the travel Pi config below, use your DDNS hostname (e.g., `yourhome.ddns.net`) in the Endpoint field.

#### **Travel Pi (Client) – `/etc/wireguard/wg0.conf`:**
```ini
[Interface]
Address = 10.0.0.2/24
PrivateKey = <travel_pi_private_key>
SaveConfig = true

[Peer]
PublicKey = <home_pi_private_key>   # Home Pi public key goes here
Endpoint = yourhome.ddns.net:51820   # Use your Dynamic DNS hostname
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Enable IP forwarding on both devices:
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
```

Start WireGuard on both devices:
```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

---

### 6. Configuring the Web UI for Tunnel Monitoring

A basic solution is to use a Python Flask application that reads the status of your WireGuard connection and system stats. For example:

1. **Install Flask on Home Pi:**
   ```bash
   sudo apt install python3-flask -y
   ```

2. **Create the Web UI Script:** Save as `/home/pi/tunnel_monitor.py`
   ```python
   #!/usr/bin/env python3
   from flask import Flask, render_template_string
   import subprocess

   app = Flask(__name__)

   TEMPLATE = """
   <!doctype html>
   <html lang="en">
     <head><title>Tunnel Status</title></head>
     <body>
       <h1>WireGuard Tunnel Status</h1>
       <pre>{{ status }}</pre>
     </body>
   </html>
   """

   def get_wg_status():
       try:
           result = subprocess.run(["sudo", "wg", "show"], capture_output=True, text=True)
           return result.stdout
       except Exception as e:
           return str(e)

   @app.route("/")
   def index():
       status = get_wg_status()
       return render_template_string(TEMPLATE, status=status)

   if __name__ == "__main__":
       app.run(host="0.0.0.0", port=5000)
   ```

3. **Make it Executable:**  
   ```bash
   chmod +x /home/pi/tunnel_monitor.py
   ```

4. **Run the Web UI:**  
   ```bash
   ./tunnel_monitor.py
   ```
Access it from your Home Pi’s IP address on port 5000 (e.g., `http://<home_pi_ip>:5000`).

For production or continuous monitoring, consider using a process manager like `systemd` or `supervisor` to manage the Flask app.

---

### 7. Auto-Setup Script for WireGuard on Both Pis

Below is a comprehensive Bash script that you can run on each Pi. Save the file as `setup-wireguard.sh` and modify the parameters where indicated (you can use command-line arguments or edit the script):

```bash
#!/bin/bash
# setup-wireguard.sh
# This script installs and configures WireGuard for Project Tunnel.
# Run it with sudo privileges.

# --- CONFIGURABLE PARAMETERS ---
MODE=""  # set to "home" or "travel"
WG_PORT=51820
WG_INTERFACE="wg0"
HOME_WG_ADDRESS="10.0.0.1/24"
TRAVEL_WG_ADDRESS="10.0.0.2/24"
# Set these values manually or pass them via environment variables
HOME_PRIVATE_KEY=""     # Fill in for home Pi (if MODE==home)
TRAVEL_PRIVATE_KEY=""   # Fill in for travel Pi (if MODE==travel)
# Public keys to be exchanged
HOME_PUBLIC_KEY=""      # Enter home public key (for travel Pi config)
TRAVEL_PUBLIC_KEY=""    # Enter travel public key (for home Pi config)
# For travel mode, DDNS hostname for home
DDNS_HOSTNAME="yourhome.ddns.net"

# --- END CONFIGURATION ---

if [[ -z "$MODE" ]]; then
    echo "Please set MODE to 'home' or 'travel' in the script."
    exit 1
fi

# Update and install dependencies
apt update && apt upgrade -y
apt install wireguard nftables -y

# Generate keys if not provided
if [[ "$MODE" == "home" && -z "$HOME_PRIVATE_KEY" ]]; then
    echo "Generating keys for Home Pi..."
    HOME_PRIVATE_KEY=$(wg genkey)
    HOME_PUBLIC_KEY=$(echo "$HOME_PRIVATE_KEY" | wg pubkey)
    echo "Home Private Key: $HOME_PRIVATE_KEY"
    echo "Home Public Key: $HOME_PUBLIC_KEY"
    # Save the keys for future reference
    echo $HOME_PRIVATE_KEY > /etc/wireguard/home_private.key
    echo $HOME_PUBLIC_KEY > /etc/wireguard/home_public.key
elif [[ "$MODE" == "travel" && -z "$TRAVEL_PRIVATE_KEY" ]]; then
    echo "Generating keys for Travel Pi..."
    TRAVEL_PRIVATE_KEY=$(wg genkey)
    TRAVEL_PUBLIC_KEY=$(echo "$TRAVEL_PRIVATE_KEY" | wg pubkey)
    echo "Travel Private Key: $TRAVEL_PRIVATE_KEY"
    echo "Travel Public Key: $TRAVEL_PUBLIC_KEY"
    # Save the keys for future reference
    echo $TRAVEL_PRIVATE_KEY > /etc/wireguard/travel_private.key
    echo $TRAVEL_PUBLIC_KEY > /etc/wireguard/travel_public.key
fi

# Create WireGuard configuration directory if needed
mkdir -p /etc/wireguard

# Write configuration based on the mode
if [[ "$MODE" == "home" ]]; then
    cat <<EOF > /etc/wireguard/${WG_INTERFACE}.conf
[Interface]
Address = ${HOME_WG_ADDRESS}
ListenPort = ${WG_PORT}
PrivateKey = ${HOME_PRIVATE_KEY}
SaveConfig = true

[Peer]
PublicKey = ${TRAVEL_PUBLIC_KEY}
AllowedIPs = 10.0.0.2/32
EOF
elif [[ "$MODE" == "travel" ]]; then
    cat <<EOF > /etc/wireguard/${WG_INTERFACE}.conf
[Interface]
Address = ${TRAVEL_WG_ADDRESS}
PrivateKey = ${TRAVEL_PRIVATE_KEY}
SaveConfig = true

[Peer]
PublicKey = ${HOME_PUBLIC_KEY}
Endpoint = ${DDNS_HOSTNAME}:${WG_PORT}
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF
fi

# Enable IP forwarding
echo "net.ipv4.ip_forward=1" | tee -a /etc/sysctl.conf
sysctl -p

# Set up nftables for NAT
cat <<EOF > /etc/nftables.conf
table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100;
EOF

if [[ "$MODE" == "home" ]]; then
cat <<EOF >> /etc/nftables.conf
        oif "eth0" masquerade
EOF
elif [[ "$MODE" == "travel" ]]; then
cat <<EOF >> /etc/nftables.conf
        oif "${WG_INTERFACE}" masquerade
EOF
fi

cat <<EOF >> /etc/nftables.conf
    }
}
EOF

nft -f /etc/nftables.conf

# Enable and start WireGuard
systemctl enable wg-quick@${WG_INTERFACE}
systemctl start wg-quick@${WG_INTERFACE}

echo "WireGuard setup completed for mode: ${MODE}"
echo "Review /etc/wireguard/${WG_INTERFACE}.conf for configuration details."
```

#### Using the Script

1. **Edit the Script:**  
   - Set the `MODE` variable to either `"home"` or `"travel"`.
   - Provide pre-generated keys or allow the script to generate them.
   - For the Travel Pi, set your `DDNS_HOSTNAME`.

2. **Run as Root:**  
   ```bash
   sudo bash setup-wireguard.sh
   ```

---

## Final Notes & Testing

1. **Testing the VPN Tunnel:**  
   - On Travel Pi, verify the WireGuard interface with:
     ```bash
     sudo wg
     ```
   - On your laptop connected to Travel Pi’s second Ethernet port, ping the Home Pi’s VPN IP (`10.0.0.1`).

2. **Web UI Monitoring:**  
   Access the monitoring page on the Home Pi by visiting:
   ```
   http://<home_pi_local_IP>:5000
   ```
3. **Troubleshooting:**  
   - Check WireGuard logs (`sudo wg show`).
   - Ensure DDNS is correctly updating your home IP.
   - Verify nftables rules with `sudo nft list ruleset`.

By following this guide and using the provided auto-setup script, you’ll have a solid foundation for your secure, feature-rich VPN tunnel project. Feel free to iterate on the Web UI and add more monitoring or automation features as your skills grow.

Happy tunneling!