# Disclaimer

SwizGuard is a privacy and censorship-resistance tool. It is designed to help users protect their network traffic from surveillance, DPI-based blocking, and other forms of network-level observation.

## Intended use

SwizGuard is intended for:

- Protecting your network traffic on untrusted networks (public Wi-Fi, hotel networks, etc.)
- Accessing geographically restricted content you have legal rights to access
- Protecting journalists, activists, and at-risk users in environments with state censorship or surveillance
- Learning about modern stealth VPN architectures
- Self-hosted privacy protection where you control every hop of the tunnel
- Security research and penetration testing in authorized contexts

## Not intended for

SwizGuard is **not** intended for:

- Evading corporate network policies without authorization
- Violating the terms of service of websites or online platforms
- Facilitating or concealing criminal activity
- Circumventing legal sanctions or export controls

## Legal considerations

**You are solely responsible for ensuring your use of SwizGuard complies with the laws of your jurisdiction and any jurisdiction whose networks you traverse.**

In some countries, operating an unauthorized VPN or circumvention tool is illegal. In others, it is fully legal or even encouraged. The author makes no representations about the legality of this tool in any specific jurisdiction. You are responsible for knowing the law that applies to you.

Specifically:
- Running circumvention tools may be illegal in China, Russia, Iran, and similar jurisdictions
- Operating a VPN service for others may require registration or licensing in some countries
- Facilitating unauthorized access to computing resources is illegal nearly everywhere

If you are unsure whether SwizGuard is legal for you to use, consult a lawyer in your jurisdiction before deploying it.

## No warranty

SwizGuard is provided "as is" without warranty of any kind. The author makes no guarantees about:

- Security against all possible adversaries
- Availability or uptime
- Protection against advanced persistent threats or nation-state adversaries
- Compatibility with future versions of its dependencies (Xray-core, sing-box, WireGuard)
- Suitability for any particular purpose

See LICENSE for the full warranty disclaimer.

## Threat model limitations

SwizGuard defends against:
- Passive traffic inspection (ISPs, network operators, corporate firewalls)
- Active network probing of the server
- TLS-in-TLS fingerprinting
- DPI-based VPN detection
- Consumer VPN blocklists

SwizGuard does NOT defend against:
- Global passive adversaries doing end-to-end traffic correlation
- VPS provider compromise, subpoena, or legal compulsion
- Endpoint compromise (malware on your device)
- Behavioral correlation (logging in with identifying accounts)
- Forensic analysis of seized devices

If your threat model includes nation-state actors specifically targeting you, a low-latency VPN is insufficient. Consider Tor with bridges, or higher-latency mixnets.

See [docs/how-it-works.md](docs/how-it-works.md) for a full threat model analysis.

## Responsible disclosure

If you discover a security vulnerability in SwizGuard, please report it responsibly. See [SECURITY.md](SECURITY.md).

## Dependencies

SwizGuard is a deployment tool that orchestrates upstream software. It does not reimplement cryptographic primitives. The actual security of your connection depends on:

- [Xray-core](https://github.com/XTLS/Xray-core) (server and desktop client)
- [sing-box](https://github.com/SagerNet/sing-box) (mobile client via SFI/SFA)
- [WireGuard](https://www.wireguard.com/) (VPN layer)
- TLS 1.3 and ChaCha20-Poly1305 cryptographic primitives

Security advisories for any of these upstream projects also affect SwizGuard deployments. Keep your installation updated.

## No affiliation

SwizGuard is not affiliated with, endorsed by, or connected to:
- The XTLS team or Xray-core project
- SagerNet or sing-box
- WireGuard LLC or the WireGuard project
- Microsoft, Apple, Cloudflare, or any other company whose domains are used as REALITY camouflage targets

It is an independent tool that composes these technologies into an easier-to-deploy package.

## Contact

For security issues: see [SECURITY.md](SECURITY.md)
For bugs and feature requests: open an issue on GitHub
For general questions: open a discussion on GitHub
