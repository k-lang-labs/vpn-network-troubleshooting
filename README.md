# vpn-network-troubleshooting
Documenting the process of fixing a complete network lockout caused by VPN conflicts
# The Great VPN Network Lockout
## the solution is to just turn off all VPN's...

## The Problem
My Windows 11 laptop entirely lost internet access. Web browsers wouldn't load, and my Mullvad VPN application threw a constant error stating: `api.mullvad.net is blocked, please check firewall`. 

## Phase 1: The Firewall Rabbit Hole
Initially, I assumed Windows Defender was actively blocking Mullvad's connection to its own servers. I went to Google, researched the error, and attempted to force the connection through the system firewall manually:

1. Opened **Windows Defender Firewall** to "Allow an app or feature through Windows Defender Firewall".
2. Mullvad wasn't in the default list, so I manually browsed through Program Files and added `mullvad.exe`.
3. I repeated the process for its background resource file, `mullvad-daemon.exe`.
4. I checked the boxes to allow both executables through **Public** and **Private** networks.
5. Restarted the Mullvad service via Task Manager. 
6. **Result:** The API was still blocked. 

Assuming I needed stricter rules, I opened **Windows Defender Firewall with Advanced Security**. I manually created custom **Inbound** and **Outbound** rules specifically dictating that those two Mullvad executables were allowed to bypass the firewall. 

Even after all of this, the internet was completely dead.

## Phase 2: The Realization
I attempted to completely uninstall Mullvad, but the uninstaller failed to remove the background daemon and its Windows Filtering Platform (WFP) network locks. 

However, the firewall and Mullvad's kill-switch were only half the problem. The actual culprit hijacking the network was running silently in the background: **WireGuard**.

## The Actual Culprit: WireGuard Full Tunnel
I had a WireGuard tunnel active to connect to my home server. The configuration in the `[Peer]` section was set to a **Full Tunnel**:
`AllowedIPs = 0.0.0.0/0, ::/0`

Because the remote server endpoint was unreachable, WireGuard was acting as an unintentional, catastrophic kill-switch. The `0.0.0.0/0` command tells the operating system to route *100% of internet traffic* into the VPN tunnel. Since the tunnel led to a dead end, all Wi-Fi and Cellular traffic—including Mullvad's desperate attempts to reach its API—was being thrown into a black hole.

## The Solution: Split Tunneling
lets be honest, just turn off the damn VPN. but...
To fix the laptop (and my phone, which was suffering from the exact same cellular block), I had to stop WireGuard from hijacking the global internet connection while keeping it active for server management.

1. Opened the WireGuard configuration.
2. Deleted the full tunnel command: `0.0.0.0/0, ::/0`.
3. Replaced it with my local home network subnet (e.g., `xxx.xxx.x.x/xx`).
4. Ensured the "Block untunneled traffic" kill-switch was disabled.

### Why this works:
WireGuard now operates as a **Split Tunnel**. It only encrypts and routes traffic specifically meant for my local home server (like an SSH connection or Proxmox dashboard). 

All normal web browsing, gaming, and Mullvad VPN traffic completely bypasses WireGuard and uses the standard Wi-Fi or cellular connection perfectly. Both VPNs can now run simultaneously without fighting over the network adapter.

##on a side note : I've noticed the Wi-Fi connection is slower when loading webpages with this split tunnel. so I have decided, to leave it off unless needed anyways lmao.
