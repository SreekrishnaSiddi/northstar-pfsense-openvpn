# pfSense Firewall, Network Segmentation & OpenVPN Remote Access

**Company:** Northstar Information Systems (fictional) · **Domain:** corp.northstar.local

## Goal
Put a real firewall in front of an existing Active Directory lab, split the network into a **corporate** zone and an isolated **guest** zone, and give remote IT staff secure access through a **certificate-based OpenVPN** tunnel.

## Network design
```
                         INTERNET
                            │
                 VirtualBox NAT (192.168.50.0/24)
                            │  WAN em0  192.168.50.15 (DHCP)
                      ┌─────┴─────┐
                      │   FW01    │  pfSense CE 2.9.2
                      └─┬───────┬─┘
             LAN em1    │       │   OPT1 em2 "GUESTNET"
          10.0.2.1/24   │       │   10.0.20.1/24
   ┌────────────────────┴──┐ ┌──┴──────────────────────┐
   │ CorpNet               │ │ GuestNet                │
   │ DC01      10.0.2.10   │ │ pfSense DHCP .100–.200  │
   │ CLIENT01  10.0.2.100  │ │ Internet only, no CorpNet│
   └───────────▲───────────┘ └─────────────────────────┘
               │ OpenVPN tunnel 10.0.8.0/24 (TCP 1194)
          My PC (remote IT tech)
```

| Host | Role | Address |
|---|---|---|
| FW01 | pfSense firewall, gateway, OpenVPN server | WAN 192.168.50.15 · LAN 10.0.2.1 · GUESTNET 10.0.20.1 |
| DC01 | Windows Server 2022: AD DS, DNS, DHCP (CorpNet) | 10.0.2.10 (static) |
| CLIENT01 | Windows 11 Enterprise, domain-joined | 10.0.2.100 (DHCP from DC01) |
| VPN clients | Remote IT staff | 10.0.8.0/24 |

**Key decision:** CorpNet reuses the existing 10.0.2.0/24 subnet, and pfSense takes the old gateway address 10.0.2.1. DC01 and CLIENT01 moved behind the firewall with **no re-addressing**.

---

## Step 1 – Build the FW01 VM
| Setting | Value |
|---|---|
| Installer | Netgate Installer (AMD64 ISO), pfSense **CE 2.9.2** |
| Resources | 1 GB RAM · 1 CPU · 16 GB disk · EFI off |
| Adapter 1 (em0) | NAT → **WAN** |
| Adapter 2 (em1) | Internal Network "CorpNet" → **LAN** |
| Adapter 3 (em2) | Internal Network "GuestNet" → **OPT1 / GUESTNET** |

**Avoiding a subnet overlap:** VirtualBox's NAT hands out 10.0.2.0/24 by default, which is the same subnet as CorpNet. pfSense won't run a WAN and LAN on overlapping subnets, so I moved FW01's NAT range before configuring the LAN:
```powershell
VBoxManage modifyvm "FW01" --natnet1 "192.168.50.0/24"
```

## Step 2 – Configure interfaces
From the pfSense console (option 2), set **LAN = 10.0.2.1/24** with **pfSense DHCP off**, because DC01 already provides DHCP for CorpNet and two DHCP servers on one network would conflict.

**Setup Wizard choices**
- Hostname **FW01**, domain **corp.northstar.local**, DNS 8.8.8.8 / 1.1.1.1, time zone America/Chicago
- **"Allow DNS to be overridden by WAN" unchecked**, so pfSense uses its own predictable DNS
- **WAN "Block RFC1918" and "Block bogon" unchecked.** The WAN sits on a private 192.168.50.x network inside VirtualBox, and inbound VPN traffic arrives from a private address. In a real deployment with a public WAN, these would stay **on**.

## Step 3 – Move the domain behind the firewall
Switched DC01 and CLIENT01 from the old VirtualBox NAT network to the **CorpNet** internal network, then verified:
- DC01: `ping 10.0.2.1`, `ping 8.8.8.8`, `nslookup google.com` all succeeded (AD DNS forwarders reach the internet through pfSense)
- CLIENT01: `ipconfig /renew` → 10.0.2.100 from DC01; `Test-ComputerSecureChannel` → **True**

---

## Step 4 – Guest network segmentation
**Interface:** OPT1 renamed **GUESTNET**, static 10.0.20.1/24.
**DHCP:** pfSense serves 10.0.20.100–200 with public DNS (8.8.8.8 / 1.1.1.1), so guests learn nothing about the internal domain.

**Firewall rules on GUESTNET (top to bottom; first match wins):**
| # | Action | Source | Destination | Purpose |
|---|---|---|---|---|
| 1 | ❌ Block | GUESTNET subnets | LAN subnets | No access to corporate servers or PCs |
| 2 | ❌ Block | GUESTNET subnets | This Firewall (self) | No access to firewall admin |
| 3 | ✅ Pass | GUESTNET subnets | Any | Internet access |

### Isolation test
I temporarily moved CLIENT01 onto GuestNet as a "guest laptop":
| Test | Result |
|---|---|
| IP address | 10.0.20.100 from pfSense DHCP ✅ |
| `ping 8.8.8.8` / `ping google.com` | Replies ✅ internet works |
| `ping 10.0.2.10` (DC01) | Timed out ❌ blocked as intended |
| `Test-NetConnection 10.0.2.10 -Port 445` | False ❌ no file shares |
| `https://10.0.20.1` (firewall admin) | Won't load ❌ blocked as intended |

---

## Step 5 – OpenVPN remote access
### PKI (certificates)
| Item | Name | Purpose |
|---|---|---|
| Certificate Authority | **Northstar-CA** | Internal CA that signs all VPN certificates |
| Server certificate | **FW01-OpenVPN** (type: Server) | Proves the VPN server's identity |
| User certificate | **maria.lopez** | Per-user certificate, revocable individually |

### Server settings
| Setting | Value | Why |
|---|---|---|
| Mode | Remote Access (SSL/TLS + User Auth) | Certificate (something you **have**) + password (something you **know**) |
| Interface / port | WAN · **TCP 1194** | See Troubleshooting for why TCP rather than UDP |
| Tunnel network | 10.0.8.0/24 | Doesn't overlap CorpNet, GuestNet, or WAN |
| Local network | 10.0.2.0/24 | Only CorpNet is reachable through the tunnel |
| Redirect gateway | Off (**split tunnel**) | The user's normal internet traffic stays off the VPN |
| DNS | 10.0.2.10, domain corp.northstar.local | VPN users resolve internal names through DC01 |

### Client
- Exported an inline `.ovpn` profile with the **openvpn-client-export** package
- VirtualBox port forward on FW01's NAT adapter: **TCP 127.0.0.1:1194 → 192.168.50.15:1194**, bound to localhost only, so nothing else on my home network can reach the lab firewall
- Imported into **OpenVPN Connect** on my PC → assigned **10.0.8.2**

### Verification over the VPN
| Test | Result |
|---|---|
| `ping 10.0.2.1` (pfSense LAN) | ✅ |
| `nslookup dc01.corp.northstar.local 10.0.2.10` | ✅ resolves 10.0.2.10 |
| `Test-NetConnection 10.0.2.10 -Port 445` (DC01 SMB) | ✅ True |
| `Test-NetConnection 10.0.2.100 -Port 3389` (CLIENT01 RDP) | ✅ True |
| RDP to CLIENT01 as an IT user (CORP\SG-IT member) | ✅ Connected |

This connects to the earlier labs: an off-site IT user connects through the firewall's VPN and gets the remote access that **Group Policy only allows for SG-IT**.

---

## Troubleshooting log
| Problem | How I found it | Cause | Fix |
|---|---|---|---|
| WAN got 10.0.2.15, the same subnet as CorpNet | pfSense console | VirtualBox NAT default range is 10.0.2.0/24 | `VBoxManage modifyvm FW01 --natnet1 192.168.50.0/24` |
| Guests could reach CorpNet | Reviewed rule order | **Pass** rule sat above the Block rules (first match wins) | Moved both Block rules above the Pass rule |
| Client export: *"Could not locate the CA reference for the server certificate"* | Edited the VPN server's crypto settings | Server was using the **GUI default** self-signed cert, not one signed by Northstar-CA | Created FW01-OpenVPN as a **Server** certificate signed by Northstar-CA and assigned it to the server |
| VPN stuck on "Trying to connect…" over UDP | Client log: 66 packets sent, **0 received**. pfSense OpenVPN log: no connection attempts. Firewall log: no WAN blocks | UDP wasn't making it through the VirtualBox NAT port forward to FW01 | Switched server, WAN rule, and port forward to **TCP**. `Test-NetConnection 127.0.0.1 -Port 1194` → True proved the path; re-exported the profile → connected |
| Over VPN, CLIENT01 reachable but **DC01 not** (DNS and SMB timed out) | `Get-NetConnectionProfile` on DC01 | Network profile was **Public** after the network change, so the firewall blocked cross-subnet traffic | Set NLA `AlwaysExpectDomainController=1`, NLA start = delayed-auto, rebooted → **DomainAuthenticated** |
| DC01 *still* unreachable from the VPN | `ipconfig /all` on DC01 | Subnet mask was **255.0.0.0 (/8)**: DC01 treated 10.0.8.2 as local and sent replies directly instead of via the gateway | `Set-NetIPAddress -IPAddress 10.0.2.10 -PrefixLength 24` → DNS and SMB worked immediately |

**Lesson:** when one host works and another doesn't over the same tunnel, the VPN is fine. Look at the host that's failing: its firewall profile, then its own IP configuration.

## Skills demonstrated
- pfSense installation, interface assignment, and firewall rule design (rule ordering, aliases like *LAN subnets* / *This Firewall*)
- Network segmentation with an isolated guest network and per-zone DHCP
- Internal PKI: CA, server, and user certificates
- OpenVPN remote access with split tunneling and pushed DNS
- Integrating a firewall into an existing AD/DNS/DHCP environment without re-addressing
- Methodical troubleshooting with client and server logs, `Test-NetConnection`, `ipconfig`, and `Get-NetConnectionProfile`

