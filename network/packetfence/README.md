# PacketFence NAC — Achahib Lab

Implemented NAC integrates PacketFence with Achahib.local Active Directory and the enterprise CA for certificate-based wired 802.1X authentication.

## Components

| Component | Role |
|---|---|
| PacketFence-NAC | RADIUS / NAC policy engine; AD domain joined |
| SRV-AD / Achahib-SRV-AD-CA | Domain identity, DNS and certificate trust |
| Cisco Catalyst 3750 | Switch-01; authenticator used for the validated test |
| Cisco SG300 | Switch-02 registered in PacketFence |
| UniFi AP-01 / AP-02 | Registered network devices; wireless validation is a separate step |
| NOC-SURFACE | Lubuntu endpoint using NetworkManager EAP-TLS |

## Confirmed authentication flow

1. NOC-SURFACE initiates EAP-TLS on Catalyst 3750 port Fa3/0/13.
2. The switch forwards the exchange to PacketFence through RADIUS.
3. PacketFence validates the client certificate against the configured CA trust.
4. PacketFence returns Access-Accept; the switch reports dot1x Authc Success and Authz Success.
5. PacketFence records the endpoint under the Machine role.

The LAN-EAPTLS wired profile and role/VLAN mapping are configured. The supplicant validates the RADIUS server identity as PacketFence-NAC.Achahib.local.

## Validation boundaries

The successful wired test does not establish deployment on every switch port or wireless endpoint. Remaining checks include role/VLAN coverage, Windows and additional Linux endpoints, wireless 802.1X, certificate expiry/revocation and reauthentication. NTLM cache and optional push/downloadable ACL indicators need review only for workflows that require them.

Based on the supplied PacketFence implementation report dated 26 August 2026. Private IPs, certificate serials and key material are omitted.

[Back to lab overview](../../README.md)
