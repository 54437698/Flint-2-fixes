# GLI Open WRT Flint-2-fixes
A collection of fixes for GLI Flint two 


The Phantom Config: A Manifesto & Hotfix Guide for GL.iNet Firmware

Every software release is a game of Jenga. You stack new features on top of old foundations, hoping the whole tower doesn’t sway. But in recent firmware releases for the Flint 2 (GL-MT6000), the integration team left legacy instructions, phantom blocks, and broken scripts running in the background.

While the Flint 2's massive 1GB of DDR4 RAM and quad-core processor can easily brute-force its way through this un-optimized code, relying on high-end hardware to hide sloppy software is a sin.

Below is the technical breakdown of the bugs plaguing the logs, followed by the exact hotfixes to clean up your system.
🛠️ Bug 1: The AdGuard Home Phantom DHCP Panic
The Problem

When you disable the DHCP server inside the AdGuard Home UI, the GL.iNet initialization scripts still blindly inject a hollow, unconfigured dhcp: block into the active config file on boot. The AdGuard engine hits these blank parameters, panics, and screams err="dhcpv4: invalid IP is not an IPv4 address" before failing over.
The Permanent Fix

We must strip the unconfigured junk out of the YAML config so the core engine knows the DHCP sub-module is completely disabled without trying to parse blank variables.

    SSH into your router:
    Bash

    ssh root@192.168.8.1

    Open the active configuration file (note: GL.iNet names this config.yaml, not AdGuardHome.yaml):
    Bash

    vi /etc/AdGuardHome/config.yaml

    Scroll down to the dhcp: section. Delete the empty lines (gateway_ip: "", subnet_mask: "", etc.) all the way down to the filtering: block. Modify it to look exactly like this:
    YAML

    dhcp:
      enabled: false
    filtering:
      blocking_ipv4: ""
      blocking_ipv6: ""

    Press Esc, type :wq, and hit Enter to save and exit.

    Restart the service:
    Bash

    /etc/init.d/adguardhome restart

🛠️ Bug 2: Legacy Firewall Script Ghosts
The Problem

Upon running a firewall cycle (fw3 restart), the system forces the execution of legacy optimization scripts that either do not exist or call targets (ETHERNET_TTL, parental_control) that the modern kernel completely rejects. This causes a wall of iptables v1.8.7 (legacy): Couldn't load target execution faults and exit-code failures.
The Permanent Fix

Since these are legacy remnants that your hardware completely bypasses anyway, removing them forces OpenWrt to gracefully skip the execution check without throwing a fault code.

    Run these commands via SSH to safely remove the legacy, broken script files:
    Bash

    rm /etc/firewall.security
    rm /etc/firewall.ethernet_ttl

    Rebuild and restart your firewall tables:
    Bash

    fw3 restart

📊 The Results

After applying these fixes, check your system logs:
Bash

logread | grep -E "AdGuardHome|firewall|iptables"

You will see your multi-WAN interfaces map, your WireGuard VPN tunnels lock in with their proper NAT6 masquerading, and AdGuard Home boot up in total silence. No phantom panics, no dead script execution failures.

Clean up the initialization scripts, strip the dead weight, and let the Flint 2 run at the peak efficiency its hardware actually deserves.

The Phantom Config: A Manifesto & Hotfix Guide for GL.iNet Firmware

Every software release is a game of Jenga. You stack new features on top of old foundations, hoping the whole tower doesn’t sway. But in recent firmware releases for the Flint 2 (GL-MT6000), the integration team left legacy instructions, phantom blocks, and broken scripts running in the background.

While the Flint 2's massive 1GB of DDR4 RAM and quad-core processor can easily brute-force its way through this un-optimized code, relying on high-end hardware to hide sloppy software is a sin.

Below is the technical breakdown of the bugs plaguing the logs, followed by the exact hotfixes to clean up your system.
🛠️ Bug 1: The AdGuard Home Phantom DHCP Panic
The Problem

When you disable the DHCP server inside the AdGuard Home UI, the GL.iNet initialization scripts still blindly inject a hollow, unconfigured dhcp: block into the active config file on boot. The AdGuard engine hits these blank parameters, panics, and screams err="dhcpv4: invalid IP is not an IPv4 address" before failing over.
The Permanent Fix

We must strip the unconfigured junk out of the YAML config so the core engine knows the DHCP sub-module is completely disabled without trying to parse blank variables.

    SSH into your router:
    Bash

    ssh root@192.168.8.1

    Open the active configuration file (note: GL.iNet names this config.yaml, not AdGuardHome.yaml):
    Bash

    vi /etc/AdGuardHome/config.yaml

    Scroll down to the dhcp: section. Delete the empty lines (gateway_ip: "", subnet_mask: "", etc.) all the way down to the filtering: block. Modify it to look exactly like this:
    YAML

    dhcp:
      enabled: false
    filtering:
      blocking_ipv4: ""
      blocking_ipv6: ""

    Press Esc, type :wq, and hit Enter to save and exit.

    Restart the service:
    Bash

    /etc/init.d/adguardhome restart

🛠️ Bug 2: Legacy Firewall Script Ghosts
The Problem

Upon running a firewall cycle (fw3 restart), the system forces the execution of legacy optimization scripts that either do not exist or call targets (ETHERNET_TTL) that the modern kernel completely rejects. This causes a wall of iptables v1.8.7 (legacy): Couldn't load target execution faults and exit-code failures.
The Permanent Fix

Since these are legacy remnants that your hardware completely bypasses anyway, removing them forces OpenWrt to gracefully skip the execution check without throwing a fault code.

    Run these commands via SSH to safely remove the legacy, broken script files:
    Bash

    rm /etc/firewall.security
    rm /etc/firewall.ethernet_ttl

    Rebuild and restart your firewall tables:
    Bash

    fw3 restart

🛠️ Bug 3: Disabled DPI / Parental Control Bloat (Netify)
The Problem

If you don't use the native Parental Controls or the real-time client traffic tracking, GL.iNet turns off the Netifyd Deep Packet Inspection engine. However, the firewall init framework still calls /etc/netifyd/iptables-init.sh.

Because the main DPI engine is turned off, the required firewall targets like parental_control don't actually exist in RAM. The script starts blindly firing blind packet rules into empty space, generating a massive, recurring wall of junk log errors:
Plaintext

iptables v1.8.7 (legacy): Couldn't load target `parental_control':No such file or directory
iptables: Bad rule (does a matching rule exist in that chain?).

The Permanent Fix

If you are using AdGuard Home for your filtering and don't use GL.iNet's native parental app tracking, you can completely stop this broken init script from running and clogging up your CPU cycles.

    Disable the Netify service so it stops initializing on boot:
    Bash

    /etc/init.d/netifyd disable

    Clear out the broken firewall initialization script so it stops throwing parental_control chain faults:
    Bash

    rm /etc/netifyd/iptables-init.sh

    Cycle the firewall one last time to apply the clean layout:
    Bash

    fw3 restart

📊 The Results

After applying these fixes, check your system logs:
Bash

logread | grep -E "AdGuardHome|firewall|iptables"

You will see your multi-WAN interfaces map, your WireGuard VPN tunnels lock in with their proper NAT6 masquerading, and your routing tables initialize in total silence. No phantom panics, no dead script execution failures, and no orphaned DPI tracking errors.

Clean up the initialization scripts, strip the dead weight, and let the Flint 2 run at the peak efficiency its hardware actually deserves.
