# hotspot_config

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
