# 📄 Ubuntu WiFi Fix Documentation: "Ghost Hotspot" Conflict

**Issue:** WiFi shows as connected in GUI but has no internet access. Requires reboot to fix temporarily.  
**Root Cause:** The WiFi adapter (`wlp2s0`) gets stuck in **Master Mode** (acting as a Hotspot `ap0`) instead of **Managed Mode** (connecting to a router). This prevents it from obtaining an IP address from your home network.

---

## 🔍 Symptoms
- WiFi icon shows a network name (e.g., "StrangeFavour").
- No internet access (browsers time out, ping fails).
- `ifconfig` or `ip addr` shows `wlp2s0` **without** an `inet` (IP) address.
- A fake interface named `ap0` appears with an IP like `192.168.12.1`.
- `iwconfig` shows `ap0` in `Mode:Master`.

---

## 🚀 Quick Fix Script (Recommended)
Save this script to fix the issue instantly whenever it happens without rebooting.

### 1. Create the Script
Open your terminal and run:
```bash
nano ~/fix-wifi-ghost.sh
```

### 2. Paste the Following Code
```bash
#!/bin/bash
echo "🔧 Starting WiFi Recovery Process..."

# Step 1: Attempt to remove the ghost hotspot interface (ap0)
echo "⚡ Removing ghost hotspot (ap0) if exists..."
sudo ip link set ap0 down 2>/dev/null
sudo ip link delete ap0 2>/dev/null
if [ $? -ne 0 ]; then
    echo "   ℹ️  Note: ap0 not found or could not be deleted (normal if already clean)."
fi

# Step 2: Reset the physical WiFi card (wlp2s0)
WIFI_IFACE="wlp2s0"
echo "🔄 Resetting interface $WIFI_IFACE..."
sudo ip link set $WIFI_IFACE down
sleep 2
sudo ip link set $WIFI_IFACE up

# Step 3: Force NetworkManager to reconnect
echo "📡 Reconnecting via NetworkManager..."
nmcli dev disconnect $WIFI_IFACE
sleep 3
nmcli dev connect $WIFI_IFACE

# Step 4: Verify Connection
echo "⏳ Waiting for IP assignment..."
sleep 5

if ip addr show $WIFI_IFACE | grep -q "inet "; then
    echo "✅ SUCCESS! WiFi is connected and has an IP address."
    ip addr show $WIFI_IFACE | grep "inet "
else
    echo "⚠️ WARNING: Interface is up but no IP received. Try restarting NetworkManager:"
    echo "   sudo pkill -f '/usr/sbin/NetworkManager'"
fi
```

### 3. Make It Executable
```bash
chmod +x ~/fix-wifi-ghost.sh
```

### 4. How to Use
Whenever your WiFi stops working, simply run:
```bash
~/fix-wifi-ghost.sh
```

---

## 🛠️ Manual Step-by-Step Guide
If you prefer not to use a script, run these commands manually in order:

### Step 1: Kill the Ghost Hotspot
Try to delete the conflicting `ap0` interface.
*(Note: If you get "Operation not supported", ignore it and proceed to Step 2. The subsequent steps usually clear it anyway.)*
```bash
sudo ip link set ap0 down
sudo ip link delete ap0
```

### Step 2: Bounce the WiFi Adapter
Turn your physical WiFi card off and on to clear the "Master Mode" lock.
```bash
sudo ip link set wlp2s0 down
sleep 2
sudo ip link set wlp2s0 up
```

### Step 3: Force Reconnection
Tell NetworkManager to drop any stale sessions and request a new connection.
```bash
nmcli dev disconnect wlp2s0
sleep 3
nmcli dev connect wlp2s0
```

### Step 4: Verify
Check if you have an IP address now:
```bash
ip addr show wlp2s0
```
*Look for a line starting with `inet 192.168...` or `inet 10...`.*

Test internet connectivity:
```bash
ping -c 4 google.com
```

---

## 🛡️ Permanent Prevention
To reduce the chances of this happening again:

### 1. Disable WiFi Power Saving
Power saving can cause the driver to glitch into the wrong mode.
Create a config file:
```bash
sudo nano /etc/NetworkManager/conf.d/wifi-powersave.conf
```
Add these lines:
```ini
[connection]
wifi.powersave = 2
```
*(2 = Disabled, 3 = Enabled)*. Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

### 2. Check Mobile Hotspot Settings
Ensure you haven't accidentally enabled the built-in hotspot feature:
- Go to **Settings → Wi-Fi**.
- Ensure **"Turn On Wi-Fi Hotspot..."** is **OFF**.

### 3. Reload Driver on Boot (Optional)
If the issue persists after every boot, add the driver reload to your startup.
*(Replace `iwlwifi` with your actual driver found via `lspci -k`)*:
```bash
echo "sudo modprobe -r iwlwifi && sudo modprobe iwlwifi" | sudo tee -a /etc/rc.local
```

---

## 📊 Technical Explanation
- **Managed Mode**: The standard mode for a laptop connecting to a router.
- **Master Mode**: The mode used when your laptop acts as a Hotspot/Access Point.
- **The Bug**: Sometimes, due to a driver glitch or Docker/VM network conflict, the `wlp2s0` interface gets stuck creating a virtual `ap0` interface in Master Mode. While in this state, it cannot associate with external routers, resulting in "Connected" status in the UI but no data flow.
- **The Fix**: Bringing the interface `down` and `up` forces the driver to reset its state machine back to default (Managed Mode), allowing NetworkManager to negotiate a proper DHCP lease.

---

**Author:** Jbtechnix   
**Last Updated:** Today
