To reset your Wi-Fi interface's MTU to the standard default of 1500, follow these steps:
## 1. Identify your Wi-Fi interface
Run the following command to find the name of your Wi-Fi adapter (it usually starts with w, like wlan0 or wlp2s0): [1] 

ip addr | grep mtu

## 2. Reset MTU to 1500 [2] 
Replace wlan0 in the command below with your actual interface name:

sudo ip link set dev wlan0 mtu 1500

This change is immediate and should instantly improve your Wi-Fi speed by allowing standard-sized packets. [2, 3] 
## 3. Make it Permanent
Manual ip link commands reset after you reboot. Since you are on Ubuntu, the best way to make this permanent is through NetworkManager: [4, 5] 

   1. Open the connection editor:
   
   nm-connection-editor
   
   2. Select your Wi-Fi network from the list and click the Gear (Edit) icon.
   3. Go to the Ethernet or Wi-Fi tab.
   4. Find the MTU field. If it is set to 1280, change it to Automatic or type 1500.
   5. Click Save. [5, 6, 7] 

## 4. Verify the Change
Check that the interface is now correctly using 1500: [8] 

ip addr show wlan0 | grep mtu

Note: If you set the MTU via a Netplan file (located in /etc/netplan/), you must edit that .yaml file, remove the mtu: 1280 line, and run sudo netplan apply to revert it. [6, 9] 
Does your internet speed feel back to normal now that it's set to 1500?

[1] [https://askubuntu.com](https://askubuntu.com/questions/835195/how-do-i-reset-my-network-settings-to-default-ubuntu-16)
[2] [https://docs.swxtch.io](https://docs.swxtch.io/docs/how-to-set-mtu-size)
[3] [https://oneuptime.com](https://oneuptime.com/blog/post/2026-03-20-set-interface-mtu-linux/view)
[4] [https://docs.swxtch.io](https://docs.swxtch.io/docs/how-to-set-mtu-size)
[5] [https://askubuntu.com](https://askubuntu.com/questions/1393338/how-to-set-mtu-permanently-in-cli)
[6] [https://www.youtube.com](https://www.youtube.com/watch?v=UnO6Pb1WBfU)
[7] [https://askubuntu.com](https://askubuntu.com/questions/230926/how-do-i-change-the-mtu-value-on-ubuntu)
[8] [https://askubuntu.com](https://askubuntu.com/questions/1127497/how-can-i-change-mtu-size-permanently)
[9] [https://askubuntu.com](https://askubuntu.com/questions/1189722/how-to-change-the-mtu-size-permanently-in-ec2-ubuntu18-04-instance)


.................................................................................................................................................................................................





Your output shows that your Wi-Fi interface (wlp2s0) is already set to mtu 1500. In fact, all your physical and virtual interfaces are currently at the standard default.
Since your MTU is already back to normal but your speed still feels slow, the issue might be related to DNS leftovers from the WARP uninstallation or Wi-Fi power management.
Here are the two most likely fixes:
## 1. Fix DNS Lag (Most Common after WARP)
Even though the app is gone, Ubuntu might still be trying to route DNS through a non-existent Cloudflare gateway. Reset your DNS resolver:

sudo systemctl restart systemd-resolved

To test if DNS is the bottleneck, try pinging a raw IP vs. a domain:

* ping -c 4 8.8.8.8 (Tests raw speed)
* ping -c 4 google.com (Tests DNS speed)
If the first is fast and the second is slow, it's a DNS issue.

## 2. Disable Wi-Fi Power Management
Ubuntu often aggressively saves power on Wi-Fi cards, which causes "choppy" speeds.

* Check status: nmcli device show wlp2s0 | grep IP4.DNS (This also checks if WARP's 1.1.1.1 is still stuck there).
* Disable Power Save:

sudo iw dev wlp2s0 set power_save off


## 3. Clear Network Manager Cache
Sometimes NetworkManager remembers bad routing rules from the VPN session.

sudo systemctl restart NetworkManager


