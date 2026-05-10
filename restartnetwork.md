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

If you recently uninstalled WARP, you might need to restart the `systemd-resolved` service to fix DNS issues:

```bash
sudo systemctl restart systemd-resolved
```

---

> ⚠️ **Important:** If you are connected via SSH, running these commands may briefly drop your connection.
