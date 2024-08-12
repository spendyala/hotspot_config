# hotspot_config
To create a hotspot on a Raspberry Pi Zero W and connect it to the internet, you'll need to follow a few steps. This involves setting up the Pi as a wireless access point (AP) and bridging or routing the internet connection from another network source (such as Ethernet or a USB modem). Here's how you can do it:

### Step 1: Update and Upgrade Your Raspberry Pi
Start by updating your Raspberry Pi's package list and upgrading any out-of-date packages.

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

### Step 2: Install Necessary Software
You'll need `hostapd` to create the hotspot and `dnsmasq` to manage DHCP and DNS.

```bash
sudo apt-get install hostapd dnsmasq -y
```

After the installation, stop the services temporarily:

```bash
sudo systemctl stop hostapd
sudo systemctl stop dnsmasq
```

### Step 3: Configure a Static IP for the Wireless Interface
Edit the `dhcpcd` configuration to assign a static IP address to the `wlan0` interface.

```bash
sudo nano /etc/dhcpcd.conf
```

Add the following lines at the end of the file:

```bash
interface wlan0
    static ip_address=192.168.4.1/24
    nohook wpa_supplicant
```

Save and close the file (`Ctrl + X`, then `Y`, and `Enter`).

Restart `dhcpcd`:

```bash
sudo service dhcpcd restart
```

### Step 4: Configure `dnsmasq`
Backup the existing `dnsmasq` configuration file and create a new one.

```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
sudo nano /etc/dnsmasq.conf
```

Add the following lines to configure `dnsmasq`:

```bash
interface=wlan0      # Use the wireless interface
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
```

Save and close the file.

### Step 5: Configure `hostapd`
Edit the `hostapd` configuration file:

```bash
sudo nano /etc/hostapd/hostapd.conf
```

Add the following configuration for your hotspot:

```bash
interface=wlan0
driver=nl80211
ssid=YourNetworkName  # Replace with your desired SSID
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=YourPassword  # Replace with your desired password
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

Save and close the file.

Now, specify the location of the configuration file by editing the `hostapd` default file:

```bash
sudo nano /etc/default/hostapd
```

Find the line starting with `#DAEMON_CONF` and replace it with:

```bash
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

Save and close the file.

### Step 6: Enable IP Forwarding
Edit the `sysctl` configuration to enable IP forwarding:

```bash
sudo nano /etc/sysctl.conf
```

Uncomment the following line:

```bash
net.ipv4.ip_forward=1
```

Enable the change without rebooting:

```bash
sudo sysctl -p
```

### Step 7: Configure NAT (Network Address Translation)
Set up NAT between `wlan0` and your other network interface (e.g., `eth0` for Ethernet):

```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

If you use a different network interface for the internet connection, replace `eth0` with that interface's name.

Save the iptables rule so it persists after reboot:

```bash
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

Edit the `rc.local` file to restore iptables rules on boot:

```bash
sudo nano /etc/rc.local
```

Add the following line just above `exit 0`:

```bash
iptables-restore < /etc/iptables.ipv4.nat
```

Save and close the file.

### Step 8: Start the Services
Start `hostapd` and `dnsmasq` services:

```bash
sudo systemctl start hostapd
sudo systemctl start dnsmasq
```

Enable these services to start on boot:

```bash
sudo systemctl enable hostapd
sudo systemctl enable dnsmasq
```

### Step 9: Connect to the Hotspot
Your Raspberry Pi Zero W should now be acting as a Wi-Fi hotspot. Devices can connect to the `ssid` you set in the `hostapd.conf` file using the password you specified. These devices should also be able to access the internet if your Raspberry Pi is connected to the internet via another network interface.

### Troubleshooting Tips
- If you can't connect to the hotspot, check that the Wi-Fi channel you're using is not crowded or restricted in your country.
- Make sure that `wlan0` is up and running by checking it with `ifconfig`.
- Use `sudo journalctl -xe` to check the logs for any errors related to `hostapd` or `dnsmasq`.

This setup should get your Raspberry Pi Zero W functioning as a wireless hotspot with internet access.


To limit the hotspot to use only 10 KB/s (which is 80 Kbps) for data transfer, you can use the `tc` (traffic control) command in Linux. The `tc` command allows you to control the bandwidth, delay, packet loss, and other parameters on a network interface.

Here's how to configure it:

### Step 1: Install `tc`
The `tc` command is part of the `iproute2` package, which should already be installed on most Raspberry Pi systems. If it's not installed, you can install it with the following command:

```bash
sudo apt-get install iproute2
```

### Step 2: Set Up Bandwidth Limiting
You will apply the bandwidth limit to the `wlan0` interface, which is the wireless interface for your hotspot.

1. **Create a root qdisc (queuing discipline):**

   This step sets up a root qdisc with a bandwidth limit.

   ```bash
   sudo tc qdisc add dev wlan0 root handle 1: htb default 30
   ```

2. **Create a class under the root qdisc:**

   Now, you'll create a class under the root qdisc that defines the maximum bandwidth.

   ```bash
   sudo tc class add dev wlan0 parent 1: classid 1:1 htb rate 10kbps
   ```

   Here, `rate 10kbps` sets the maximum bandwidth to 10 Kbps (which is equivalent to 1.25 KB/s).

3. **Add a filter to match all traffic:**

   Finally, apply this class to all traffic passing through the `wlan0` interface.

   ```bash
   sudo tc filter add dev wlan0 protocol ip parent 1:0 prio 1 u32 match ip src 0.0.0.0/0 flowid 1:1
   ```

### Step 3: Verify the Setup
You can check the current `tc` configuration on the `wlan0` interface with:

```bash
sudo tc qdisc show dev wlan0
```

Or to see the details:

```bash
sudo tc class show dev wlan0
```

### Step 4: Make the Changes Persistent (Optional)
If you want to make sure these limits persist across reboots, you can add the above `tc` commands to your `/etc/rc.local` file, just above the `exit 0` line:

```bash
sudo nano /etc/rc.local
```

Add:

```bash
tc qdisc add dev wlan0 root handle 1: htb default 30
tc class add dev wlan0 parent 1: classid 1:1 htb rate 10kbps
tc filter add dev wlan0 protocol ip parent 1:0 prio 1 u32 match ip src 0.0.0.0/0 flowid 1:1
```

Save and close the file.

### Notes
- **10 Kbps** is extremely slow and may not be practical for most applications. Consider whether this limit is appropriate for your needs.
- The `tc` command is very powerful and can be used to create more complex traffic control setups, such as different limits for different clients or types of traffic.

This setup will limit all devices connected to the hotspot to a maximum data transfer rate of 10 Kbps.
