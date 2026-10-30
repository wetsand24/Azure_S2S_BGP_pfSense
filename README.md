# pfSense to Azure site-to-site VPN with BGP

These are the settings from the video. It's an IPsec tunnel from pfSense at home to an Azure VPN gateway, and BGP runs inside it, so routes on both sides come in by themselves. It uses the cheapest Azure gateway that supports BGP.

> **Cost warning:** the VPN gateway is billed every hour while it exists, even if nothing is connected. In Canada Central (October 2026), VpnGw1AZ is $0.21/h, the Standard public IP is $0.005/h and the B2ts v2 test VM is about $0.012/h. That's about **$5.44 a day** or **$165 a month**, and the gateway alone is $153 a month. Make a budget alert and delete the resource group when you're done.

## Addressing used in the video

| What | Value |
|---|---|
| Home LAN | 10.50.0.0/24 (pfSense 10.50.0.1) |
| Home second network (VLAN 51) | 10.51.0.0/24 |
| Home new network in the payoff (VLAN 52) | 10.52.0.0/24 |
| Azure VNet | 10.100.0.0/16 |
| VM subnet `snet-vms` | 10.100.1.0/24 (test VM 10.100.1.4) |
| GatewaySubnet | 10.100.255.0/27 |
| Azure BGP peer IP (read it off the gateway's Configuration page) | 10.100.255.30 |
| Tunnel (VTI) transit | 10.222.1.0/30: pfSense 10.222.1.1, Azure side 10.222.1.2 |
| ASN home / Azure | 65010 / 65515 |

## Azure portal

**Resource group:** `rg-s2s-lab`, Canada Central. Everything goes in here, so deleting it deletes everything.

**Virtual network:** `vnet-lab`, address space 10.100.0.0/16, subnet `snet-vms` 10.100.1.0/24. Then **Add a subnet** with Subnet purpose = **Virtual Network Gateway**. Azure names it `GatewaySubnet`: 10.100.255.0/27 (a /27 or bigger). Add it here, because once the VNet has a VM subnet, the 2026 gateway form can't create the gateway subnet for you (its Subnet dropdown is disabled).

**Virtual network gateway:**

| Field | Value |
|---|---|
| Name | `vpngw-lab` |
| Region | Canada Central |
| Gateway type | VPN (the 2026 form only creates route-based gateways and has no VPN type field) |
| SKU | VpnGw1AZ (Basic has no BGP; plain VpnGw1 can't be created any more) |
| Virtual network | `vnet-lab` (it picks up the GatewaySubnet by itself) |
| Public IP | create new, `pip-vpngw` |
| Active-active | Disabled |
| Configure BGP | Enabled, ASN 65515, custom APIPA field empty |

Deploying the gateway took 21 minutes on recording day (about 20 when I tested it). Microsoft says it can take 45 minutes or more.

**Local network gateway** (your home side):

| Field | Value |
|---|---|
| Name | `lng-home` |
| Endpoint | IP address = your home public IP |
| Address space | **empty** (BGP sends the networks) |
| Advanced: Configure BGP | Yes, ASN 65010, BGP peer IP 10.222.1.1 |

**Connection** (gateway → Connections → Add):

| Field | Value |
|---|---|
| Type | Site-to-site (IPsec) |
| Name | `cn-home` |
| Local network gateway | `lng-home` |
| Shared key | a long random key (the same one goes into pfSense) |
| IKE protocol | IKEv2 |
| Enable BGP | ticked |
| IPsec/IKE policy | Custom (values below) |

Custom policy:

| | Encryption | Integrity / PRF | DH / PFS group | Lifetime |
|---|---|---|---|---|
| IKE (phase 1) | GCMAES256 | SHA256 | DHGroup14 | 28800 s |
| IPsec (phase 2) | GCMAES256 | GCMAES256 | PFS2048 | 27000 s |

There's no "PFS14" in the portal's PFS list. **PFS2048 is DH group 14** (2048-bit MODP), the same as pfSense's PFS key group 14. Don't pick PFS24, that's group 24.

## OpenWrt (only if pfSense sits behind another router)

Forward UDP 500 and UDP 4500 to pfSense's WAN address (10.37.0.50 in the video). If pfSense has your public IP on its WAN, skip this.

## pfSense CE 2.9

**System → Routing → Gateways:** Default gateway IPv4 = **WANGW**, not Automatic.

**System → Package Manager → Available Packages:** install **FRR**.

**VPN → IPsec → Add P1:**

| Field | Value |
|---|---|
| Key Exchange version | IKEv2 |
| Remote Gateway | the Azure gateway's public IP |
| My identifier | IP address = **your home public IP** (needed behind NAT, otherwise Azure drops the connection) |
| Pre-Shared Key | same key as the Azure connection |
| Encryption | AES256-GCM, 128 bits, SHA256, DH group 14 |
| Life Time | 28800 |
| NAT Traversal | Auto |
| Dead Peer Detection | enabled |

**Add P2:**

| Field | Value |
|---|---|
| Mode | Routed (VTI) |
| Local Network | Network 10.222.1.1 / 30 |
| Remote Address | 10.222.1.2 |
| Encryption | AES256-GCM only, 128 bits |
| PFS key group | 14 (2048 bit) |
| Life Time | 27000 |

**Interfaces → Assignments:** add `ipsec1`, open it, enable it, name it `AZURE_VTI`, save, apply.

**System → Routing → Gateways:** edit `AZURE_VTI_VTIV4`, tick **Disable Gateway Monitoring** (Azure doesn't answer pings on the tunnel address).

**System → Routing → Static Routes:** 10.100.255.30/32 via `AZURE_VTI_VTIV4`. Without this route, pfSense sends traffic for the whole Azure network to your home router, even when BGP shows Established.

**VPN → IPsec → Advanced Settings:** IPsec Filter Mode = **Filter IPsec VTI and Transport on assigned interfaces**. With the default mode, my BGP session kept dropping.

**Firewall → Rules → AZURE_VTI:**

| Action | Protocol | Source | Destination |
|---|---|---|---|
| Pass | TCP | 10.100.255.30 | any, port 179 (BGP) |
| Pass | any | 10.100.0.0/16 | LAN subnets |

**Status → IPsec:** Connect. Phase 1 shows Established, and right below it the child SA (phase 2) shows Installed. In Azure, the connection's status changes to Connected.

**Services → FRR Global/Zebra:** Enable, set a Master Password, Default Router ID 10.50.0.1, Save.
Route Maps tab → Add: Name `ALLOW-ALL`, Action Permit, Sequence 100, Save.

**Services → FRR BGP:** Enable, Local AS 65010, Router ID 10.50.0.1, Networks to Distribute 10.50.0.0/24 and 10.51.0.0/24, Save.
Neighbors tab → Add:

| Field | Value |
|---|---|
| Name/Address | 10.100.255.30 |
| Remote AS | 65515 |
| Update Source | AZURE_VTI (resolves to 10.222.1.1) |
| EBGP Multi-Hop | 2 |
| Route Map Filters | ALLOW-ALL in, ALLOW-ALL out |

**Status → FRR → BGP:** the neighbor's State/PfxRcd column shows a number (1) instead of a word. That means the session is Established, and the one prefix is 10.100.0.0/16.

## MSS clamping: not needed here

Azure's gateway already clamps TCP MSS to 1360 in both directions. pfSense's MSS setting only applies to networks in phase 2 entries, not to routes learned over BGP. A 50 MB copy ran the same with it and without it.

## Check it

- Azure VM → network interface → **Effective routes**: 10.50.0.0/24 and 10.51.0.0/24 with next hop type *Virtual network gateway*.
- Gateway → **BGP peers**: 10.222.1.1, Connected.
- From home: `ping 10.100.1.4`, `ssh user@10.100.1.4`.

## The payoff: add a network at home

**Interfaces → VLANs → Add:** parent interface = your LAN port, VLAN tag 52, Save.
**Interfaces → Assignments:** add the new VLAN, enable it, name it `LAN3`, IPv4 static 10.52.0.1/24, Save, Apply.
**Services → FRR BGP:** add 10.52.0.0/24 to Networks to Distribute, Save.

Azure VM → network interface → Effective routes → Refresh: 10.52.0.0/24 shows up with next hop *Virtual network gateway*. Nothing changes in the portal.

## Troubleshooting

**BGP is Established, but traffic to Azure goes out your WAN.** Check `netstat -rn | grep 10.100` on pfSense. If 10.100.0.0/16 points at your WAN gateway instead of the tunnel, FRR resolved Azure's BGP address (10.100.255.30) over the default route. Two causes:
- The static route 10.100.255.30/32 via `AZURE_VTI_VTIV4` is missing. Add it (see above).
- FRR started while the tunnel was down. FRR's zebra reads pfSense's static /32 when FRR starts. If an interface apply restarts IPsec and FRR comes back before the tunnel interface does, zebra resolves 10.100.255.30 over the WAN and keeps it that way. I hit this after adding the VLAN. Fix: once the tunnel is up, run `service frr restart` (Diagnostics → Command Prompt, or SSH), then check `netstat -rn` again.

**The BGP session drops every minute or so.** Set IPsec Filter Mode to the VTI-on-assigned-interfaces option and put the rules on the `AZURE_VTI` tab (see above).

**The connection never comes up behind NAT.** Check that *My identifier* on phase 1 is your public IP, and that UDP 500 and 4500 are forwarded to pfSense.

**The Connections list in the portal says "No results" right after you create it.** Click Refresh. The list doesn't always update by itself.

## Clean up

Cost Management → Budgets: add a small monthly budget with an e-mail alert. When you're done, delete the resource group `rg-s2s-lab` (type its name to confirm). The gateway, VM and public IP go with it.
