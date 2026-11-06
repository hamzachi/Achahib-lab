# Secure Remote Access: OpenVPN, WireGuard, BGP and Cloudflare WARP

> **Public architecture case study** · Achahib Lab · Updated November 2026  
> **Scope:** Remote-access VPN ingress, encrypted inter-site transport, selective return-path policy routing, and Cloudflare WARP internet egress.  
> **Disclosure:** All addresses, DNS names, ports, interface-specific commands, secrets and precise firewall rules have been withheld.

## Executive summary

This project combines an on-premises OpenVPN Access Server with a public VPS relay, a WireGuard tunnel, dynamic routing, and a Cloudflare WARP egress path. Remote clients connect to the VPS, while the authentication service remains inside the lab behind OPNsense. Internet-bound traffic from authenticated VPN clients returns through the WireGuard path and exits through the VPS's Cloudflare WARP connectivity.

A practical issue arose when inbound OpenVPN requests successfully reached the server, but reply traffic left the VPS through WARP rather than the public ingress interface. This asymmetric return path prevented session establishment. A **narrow, transport-specific policy routing exception** restored symmetric transport replies without sending all VPS traffic outside WARP.

## Updated logical architecture

```text
                        INTERNET
             +-------------------------+
             | Remote OpenVPN client   |
             +------------+------------+
                          |
             Encrypted VPN connection
                          |
             +------------v------------+
             | Public VPS ingress      |
             | DNAT + return-path PBR  |
             +------------+------------+
                          |
              WireGuard encrypted tunnel
                          |   BGP routes
             +------------v------------+
             | OPNsense firewall       |
             +------------+------------+
                          |
             +------------v------------+
             | OpenVPN Access Server   |
             | Directory auth + MFA    |
             +-------------------------+

VPN client internet egress (separate logical direction):
OpenVPN client -> on-prem OpenVPN -> WireGuard -> VPS
               -> Cloudflare WARP -> Internet
```

> The diagram describes logical security and forwarding roles, not physical cabling. Cloudflare **Tunnel / Access for applications** and the separate **Ubuntu Mesh for selected LAN egress** are distinct from the **VPS WARP egress** described here.

## Engineering decisions

| Layer | Purpose | Design consideration |
|---|---|---|
| OpenVPN Access Server | Authenticated remote connectivity | On-premises service remains behind the firewall; full-tunnel clients |
| Directory + MFA | User verification | Multi-factor protection complements directory-based identity |
| VPS relay | Public connection entry | Exposes a deliberate entry service without publishing the home ISP address |
| WireGuard | Encrypted inter-site transport | Carries connections between VPS and lab |
| BGP | Route exchange | Distributes the lab prefixes selected for tunnel routing |
| OPNsense | Security enforcement | Segmentation and route-policy boundary |
| Cloudflare WARP | Selected internet egress | Provides an alternate public-facing outbound path |
| Linux policy-based routing | Symmetric VPN transport | Returns only matching VPN transport replies by the ingress uplink |
| systemd | Operational persistence | Restores the rule at service startup; reboot testing pending |

## Problem and evidence

**Symptom.** Remote OpenVPN connection attempts repeatedly timed out even though the server was online and the DNS record pointed to the VPS public endpoint.

**Isolation sequence.**

1. Verified the VPS's ordinary internet traffic used WARP policy routing while its public uplink remained configured separately.
2. Confirmed the VPS could reach the on-premises OpenVPN server over WireGuard and that the expected routes were installed.
3. Verified the OpenVPN service was listening on its configured transport and that the public VPS forwarding/NAT rule existed.
4. Captured inbound VPN datagrams on the VPS public interface: requests arrived correctly.
5. Captured the same flow traversing the WireGuard interface: the on-premises OpenVPN server sent replies.
6. Captured the public interface: corresponding outbound replies were absent.
7. Captured the WARP interface: the public-uplink source address was used on replies leaving through WARP.

**Root cause.** WARP's higher-priority routing table selected the WARP interface for the OpenVPN transport reply. NAT had correctly translated the packet, but its outgoing route was inconsistent with the public ingress path. Because the service reply used the public VPS source address, its departure through WARP created an asymmetric flow.

## Remediation and validation

A narrowly scoped Linux IP rule was applied to the **VPN transport's reply packets**, directing those packets to the VPS's standard uplink routing table. Broad route changes were deliberately avoided: ordinary internet egress remained through WARP, and BGP/WireGuard paths were preserved.

The successful validation sequence was:

- VPN connection succeeded from an external network.
- Authenticated VPN client internet access was confirmed.
- Public egress observed by the VPN client matched Cloudflare WARP egress, not the VPS ingress address.
- A dedicated systemd oneshot service was installed and enabled to ensure the routing exception is present after service startup.
- Restarting that service successfully preserved the required routing policy.
- **Full VPS reboot validation remains outstanding**, deferred to a maintenance window because other services share the VPS.

## Security and operational notes

- **No operational secrets or addressing** are published in this portfolio page.
- Restrict public exposure to the minimum required services and apply firewall controls to forwarded traffic.
- Scope routing exceptions to the required transport; avoid a blanket bypass of WARP for the VPS.
- Monitor changes in Cloudflare WARP routing policy, BGP advertisements, NAT and firewall rules.
- Confirm the service starts in the correct order on a scheduled reboot, and keep out-of-band VPS console access available.
- This design obscures the home network's ISP address as the VPN endpoint; it does **not** eliminate all potential forms of origin-IP disclosure, replace authentication, or imply universal Zero Trust enforcement.

## Related documentation

- [Lab architecture and topology](../README.md)
- [Cloudflare Zero Trust and complementary paths](../Cloudflare%20Zero%20Trust/cloudflare-zero-trust-setup.md)

*This is a sanitized engineering case study. Full operational configuration and diagnostics are retained privately.*
