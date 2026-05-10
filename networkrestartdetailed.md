# Restarting Network Services on Ubuntu (systemd)

On modern Ubuntu systems (which use systemd), the most effective way to restart the network service is by using the `netplan apply` command or restarting the NetworkManager service.

## 1. Apply Netplan Configuration

This is the preferred method for Ubuntu (18.04 and newer) as it parses your configuration files and applies changes to the renderer.

```bash
sudo netplan apply
```

## 2. Restart via NetworkManager

If you are using a desktop version of Ubuntu or managing connections via `nmcli`, restart the NetworkManager daemon:

```bash
sudo systemctl restart NetworkManager
```

## 3. Restart the systemd-networkd service

If you are on a server environment that relies specifically on `systemd-networkd`:

```bash
sudo systemctl restart systemd-networkd
```

## 4. Refresh a Specific Interface

If you only need to reset one specific connection (e.g., Ethernet), you can "cycle" the interface:

```bash
# Turn off
sudo ip link set eth0 down

# Turn on
sudo ip link set eth0 up
```

> **Note:** Replace `eth0` with your actual interface name, which you can find by running `ip addr`.

## 5. Reset DNS Resolution

If you recently uninstalled WARP or other VPN software, you might need to restart the `systemd-resolved` service to fix DNS issues:

```bash
sudo systemctl restart systemd-resolved
```

## 6. Clear DNS Cache

Different services maintain their own DNS caches:

```bash
# For systemd-resolved (most common on modern Ubuntu)
sudo systemd-resolve --flush-caches
# or
sudo resolvectl flush-caches

# For NetworkManager
sudo systemctl reload NetworkManager

# For dnsmasq (if installed)
sudo systemctl restart dnsmasq
```

## 7. Clear ARP Cache

The ARP cache maps IP addresses to MAC addresses. Clearing it forces fresh neighbor discovery:

```bash
# Clear all ARP entries
sudo ip neigh flush all

# Or clear for a specific interface
sudo ip neigh flush dev eth0
```

## 8. Flush Routing Cache

On older kernels, you might need to clear the routing cache (modern kernels manage this automatically):

```bash
# For older systems (pre-3.6 kernels)
sudo ip route flush cache

# Modern approach - just restart networking or use
sudo sysctl -w net.ipv4.route.flush=1
```

## 9. Renew DHCP Lease

Force your system to request a fresh IP address from the DHCP server:

```bash
# Using dhclient
sudo dhclient -r eth0   # Release current lease
sudo dhclient eth0      # Request new lease

# Using nmcli (NetworkManager)
sudo nmcli connection down "Your Connection Name"
sudo nmcli connection up "Your Connection Name"

# Using netplan + dhcpcd (if applicable)
sudo dhcpcd -k eth0     # Kill existing DHCP client
sudo dhcpcd -n eth0     # Renew lease
```

## 10. Working with `ip link set` Commands

The `ip link set` command is powerful for managing network interfaces. Here are all the useful variations:

### Basic Interface Control

```bash
# Bring interface up/down
sudo ip link set eth0 up
sudo ip link set eth0 down

# Change interface name (must be down first)
sudo ip link set eth0 down
sudo ip link set eth0 name new-eth0
sudo ip link set new-eth0 up

# Change MAC address (must be down first)
sudo ip link set eth0 down
sudo ip link set eth0 address 00:11:22:33:44:55
sudo ip link set eth0 up

# Set MTU (Maximum Transmission Unit)
sudo ip link set eth0 mtu 1500

# Set interface in promiscuous mode (for packet sniffing)
sudo ip link set eth0 promisc on
sudo ip link set eth0 promisc off

# Set interface multicast flag
sudo ip link set eth0 multicast on
sudo ip link set eth0 multicast off

# Set ARP flag on/off
sudo ip link set eth0 arp on
sudo ip link set eth0 arp off

# Set interface group
sudo ip link set eth0 group 42

# Set interface alias
sudo ip link set eth0 alias "My Ethernet Interface"

# Set interface namespace
sudo ip link set eth0 netns mynamespace
```

### Specific Interface Types

```bash
# WiFi interfaces (ap0, wlan0, etc.)
sudo ip link set wlan0 up
sudo ip link set wlan0 down
sudo ip link set ap0 down   # Access point mode interface

# Virtual interfaces
sudo ip link set tun0 down
sudo ip link set tap0 down

# Bridge interfaces
sudo ip link set br0 up
sudo ip link set br0 down

# Bonding interfaces
sudo ip link set bond0 up
sudo ip link set bond0 down
```

## 11. Remove Ghost/Phantom Network Interfaces

Ghost interfaces are virtual or stale network interfaces that persist even after their parent device is removed or software is uninstalled (common after VPN uninstalls like WARP, Tailscale, Docker, or VMware).

### List all interfaces (including hidden/down ones)

```bash
# Show ALL interfaces (including down/virtual)
ip link show
ip addr show

# Show only physical interfaces
ls /sys/class/net/

# Show detailed interface info including type
sudo lshw -class network

# Show only virtual interfaces
ip -d link show | grep -B1 "veth\|tun\|tap\|docker\|br"
```

### Remove Common Ghost Interfaces

```bash
# Virtual interfaces commonly left behind by VPNs
sudo ip link delete tun0        # OpenVPN/Tailscale/WARP
sudo ip link delete tun1
sudo ip link delete tun2
sudo ip link delete utun

# WARP specific interfaces
sudo ip link delete warp
sudo ip link delete wg0         # WireGuard/WARP

# Docker interfaces
sudo ip link delete docker0
sudo ip link delete docker_gwbridge
sudo ip link delete vethXXXXXXX # Replace XXXXXXX with actual ID

# Bridge interfaces
sudo ip link delete br0
sudo ip link delete br-XXXXXXXXXXX # Docker bridge

# Virtual Ethernet pairs
sudo ip link delete veth0

# AP (Access Point) interfaces
sudo ip link delete ap0

# VMware interfaces
sudo ip link delete vmnet1
sudo ip link delete vmnet8

# VirtualBox interfaces
sudo ip link delete vboxnet0
```

### Force Remove Stubborn Interfaces

If a ghost interface won't delete normally:

```bash
# Take it down first
sudo ip link set ap0 down 2>/dev/null

# Try deletion
sudo ip link delete ap0 2>/dev/null || echo "Interface may not exist"

# For network namespaced interfaces
sudo ip netns list
sudo ip netns delete <namespace-name>

# As a last resort, restart the network stack
sudo systemctl restart systemd-networkd
```

### Clean All Ghost Interfaces Automatically

```bash
# Remove all tun/tap interfaces
for iface in $(ip link show | grep -oP 'tun[0-9]+|tap[0-9]+|ap[0-9]+'); do
    sudo ip link delete $iface 2>/dev/null
done

# Remove all bridge interfaces (except system main ones)
for iface in $(ip link show type bridge | grep -oP 'br-[a-f0-9]+|br[0-9]+'); do
    if [[ "$iface" != "br0" ]] && [[ ! "$iface" =~ ^br-[a-f0-9]{12}$ ]]; then
        sudo ip link delete $iface 2>/dev/null
    fi
done

# Clean Docker artifacts
sudo docker network prune -f 2>/dev/null
```

## 12. Reset Network Stack (Nuclear Option)

For complete network reset without rebooting:

```bash
# Stop all network services
sudo systemctl stop NetworkManager systemd-networkd systemd-resolved

# Kill any lingering dhclient processes
sudo pkill dhclient

# Flush all IP addresses and routes from all interfaces
for iface in $(ip link show | grep -oP '^[0-9]+: \K[^:]+'); do
    sudo ip addr flush dev $iface 2>/dev/null
    sudo ip route flush dev $iface 2>/dev/null
done

# Clear all iptables rules (be careful with this!)
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -t mangle -F
sudo iptables -X

# Clear all ghost interfaces
sudo ip link delete tun0 2>/dev/null
sudo ip link delete ap0 2>/dev/null
sudo ip link delete docker0 2>/dev/null

# Restart everything
sudo systemctl restart systemd-resolved systemd-networkd NetworkManager

# Re-apply netplan
sudo netplan apply
```

## 13. Check for Conflicting Services

Ensure you're not running multiple network managers simultaneously:

```bash
# Check which network services are active
systemctl status NetworkManager systemd-networkd networking

# If both NetworkManager and systemd-networkd are active, consider disabling one
# For server: disable NetworkManager
sudo systemctl disable --now NetworkManager

# For desktop: disable systemd-networkd (unless you need it)
sudo systemctl disable --now systemd-networkd
```

## 14. Verify Final Network State

After performing cleanup, verify everything is working:

```bash
# Check DNS resolution
nslookup google.com
dig google.com

# Check routing
ip route show
ip -6 route show

# Check interface status
ip addr show
ip link show

# Test connectivity
ping -c 4 8.8.8.8
ping -c 4 google.com

# Check active connections
ss -tuln

# Verify DNS cache status (systemd-resolved)
resolvectl statistics

# Check for any remaining ghost interfaces
ip -d link show | grep -E "tun|tap|docker|br-|veth"
```

## Quick One-Liner for Complete Network Refresh

```bash
# Run this to flush everything and restart (replace eth0 with your interface)
sudo ip neigh flush all && \
sudo systemd-resolve --flush-caches && \
sudo dhclient -r eth0 && \
sudo systemctl restart systemd-resolved systemd-networkd NetworkManager && \
sudo netplan apply && \
sudo dhclient eth0
```

## Troubleshooting Tips

If you're still having network issues:

```bash
# Check live logs
sudo journalctl -u NetworkManager -f
sudo journalctl -u systemd-networkd -f
sudo journalctl -u systemd-resolved -f

# Capture packets for inspection
sudo tcpdump -i eth0 -n

# Check interface statistics
sudo ip -s link show eth0
sudo netstat -i

# Test specific DNS servers
nslookup google.com 8.8.8.8
dig @1.1.1.1 google.com

# Check if any process is holding interfaces
sudo lsof | grep -E "tun|tap|docker|warp"
```

---

> ⚠️ **Important:** If you are connected via SSH, running these commands may briefly drop your connection. For remote servers, consider using a out-of-band management console (iDRAC, iLO, IPMI) or schedule maintenance downtime.

> 💡 **Pro tip:** After uninstalling VPN software like Cloudflare WARP, always run sections 11 (remove ghost interfaces) and 5-6 (reset DNS) to ensure a completely clean network state.
```

This complete documentation now includes everything:
- Basic network restarts
- DNS cache clearing
- ARP cache management
- DHCP renewal
- Comprehensive `ip link set` commands
- Ghost/phantom interface removal (including ap0 and other common ghosts)
- Nuclear reset option
- Verification steps
- Troubleshooting tips
