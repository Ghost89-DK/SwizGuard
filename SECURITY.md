# Security Policy

## Reporting a vulnerability

If you discover a security vulnerability in SwizGuard, please report it responsibly.

**Do not open a public GitHub issue for security vulnerabilities.**

Instead, report it privately via one of these channels:

1. **GitHub Security Advisories** — use the "Report a vulnerability" button in the Security tab of this repository
2. **Direct contact** — reach out to the maintainer through their published contact information

Please include:
- A description of the vulnerability
- Steps to reproduce
- Potential impact (what an attacker could do)
- Any mitigating factors you're aware of
- Your contact info so we can follow up

## What counts as a vulnerability

Examples of things that ARE vulnerabilities:
- A bug in SwizGuard's scripts that leaks credentials, keys, or config
- A misconfiguration in the generated configs that undermines encryption or camouflage
- A permissions issue that lets unauthorized users read keys or logs
- An input validation bug that allows arbitrary command execution
- A flaw in the REALITY / Vision / WG chain setup that would allow a network observer to identify the connection as a VPN

Examples of things that are NOT vulnerabilities in SwizGuard specifically (but might be in upstream projects):
- Cryptographic weaknesses in TLS 1.3, ChaCha20-Poly1305, or WireGuard protocols
- Attacks against Xray-core or sing-box core — report those to the respective upstream projects
- "I can correlate traffic timing with a global passive adversary" — this is a known limitation of all low-latency VPNs, not a SwizGuard bug
- "My VPS provider can see traffic leaving" — same, this is inherent to VPS-based VPNs

## Response timeline

We aim to:
- Acknowledge reports within 72 hours
- Provide an initial assessment within 7 days
- Release a fix (or coordinate disclosure if the issue is upstream) within 30 days for critical issues

For non-critical issues, we'll work with you on a reasonable timeline.

## Supported versions

SwizGuard is distributed as a deploy-from-source tool. There are no "versions" to patch independently — the fix is always the latest commit on the main branch.

For production deployments, we recommend:
- Pin to a specific git tag or commit hash
- Subscribe to repository releases to get notified of security fixes
- Re-run `swizguard setup` on a fresh VPS after critical fixes rather than patching in-place

## Upstream dependencies

SwizGuard depends on:
- **Xray-core** — https://github.com/XTLS/Xray-core
- **sing-box** — https://github.com/SagerNet/sing-box
- **WireGuard** — https://www.wireguard.com/

Security advisories for any of these affect SwizGuard deployments. Watch those projects for releases. When they patch a security issue, update your SwizGuard installation:

```bash
# On the VPS
sudo ./swizguard setup  # re-runs with the latest xray-core binary
```

## Hardening recommendations

See [docs/how-it-works.md](docs/how-it-works.md) for the full threat model. Beyond SwizGuard's defaults:

1. **Harden the VPS first** with a hardening script (non-root user, SSH keys only, custom SSH port, UFW, fail2ban, unattended upgrades) before installing SwizGuard
2. **Rotate REALITY keys every 3-6 months** via `sudo ./swizguard rekey`
3. **Use a privacy-friendly VPS provider** in a jurisdiction that matches your threat model
4. **Pay with Monero** if you need to decouple the VPS from your real identity
5. **Don't reuse the VPS** for other public services — dedicated infrastructure reduces attack surface
6. **Keep your client devices updated** — endpoint security is a hard requirement
7. **Monitor the server** — `sudo ./swizguard status` should show expected peer behavior
8. **Treat client configs as secrets** — they contain UUIDs and keys that grant access

## Threat model reality check

SwizGuard is excellent against:
- ISP surveillance
- Corporate firewalls
- Consumer DPI systems
- State-level censorship (China, Russia, Iran) for everyday users
- VPN blocklists and streaming service geo-blocks

SwizGuard is NOT sufficient against:
- A nation-state adversary specifically targeting you
- Your VPS provider cooperating with law enforcement
- Malware on your endpoint

If your threat model includes the latter, you need more than a VPN. Consider:
- Tor with pluggable transports for anonymity
- Qubes OS or Tails for endpoint security
- Physical OPSEC for device handling
- Monero-paid anonymous VPS hosting for reducing trust surface

## Responsible use

SwizGuard is published for legitimate privacy and censorship-resistance use. Please don't use it to:
- Facilitate criminal activity
- Harass or target individuals
- Evade lawful network monitoring in contexts where that monitoring is legitimate (e.g., your employer's authorized network at work)

See [DISCLAIMER.md](DISCLAIMER.md) for the full position.
