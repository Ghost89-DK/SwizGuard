# How SwizGuard Works

This is the deep technical breakdown. If you just want to use the tool, the README is enough. If you want to understand exactly what's happening on the wire and why, read on.

## The problem I was trying to solve

Plain WireGuard is fast and well-audited. I love it as a VPN protocol. But it has a fingerprinting problem — every modern DPI system can identify WireGuard traffic in milliseconds without decrypting anything, because the handshake pattern is unique. The first packet is always a specific size, the response is always a specific size, the keepalive interval is consistent. Once that's identified, the connection can be blocked, throttled, or flagged.

I ran into this in real life. I'd connect to my self-hosted WireGuard from a hotel in some country, get a few minutes of connectivity, and then the network would figure out what was happening and start dropping my packets. Sometimes the captive portal would actively block known VPN protocols. Sometimes the firewall would just rate-limit anything UDP that didn't look like normal browsing.

VLESS+REALITY is the answer to the "make it not look like a VPN" problem. REALITY in particular is a TLS-based protocol that camouflages your connection as a normal HTTPS connection to a major website. Even better, when someone actively probes your server, REALITY proxies the entire TLS handshake to the real website and returns the real certificate. There's nothing fake to detect.

But VLESS+REALITY by itself has a subtle weakness called **TLS-in-TLS fingerprinting**. When your browser tunnels HTTPS through a VLESS+REALITY proxy, the inner TLS handshake (your browser talking HTTPS to the destination site) has deterministic packet sizes that appear inside the encrypted outer tunnel. Observers can't read what's inside, but they can see the pattern of "TLS-shaped packets wrapped inside another TLS session." China's GFW has been using this detection technique since around 2022.

The Xray team fixed this with **Vision flow** (`xtls-rprx-vision`), which injects random padding into the inner handshake records and adds a kernel splice optimization for bulk transfer.

SwizGuard combines all of this — WireGuard for the inner VPN, VLESS as the proxy framing, REALITY for the outer camouflage, Vision for the fingerprint hardening — into a single deployable stack. The novel part isn't any individual protocol. It's that I tunneled WireGuard inside the VLESS+REALITY+Vision chain, which gives you a defense-in-depth setup where even if someone broke one layer, two more remain. And I made it actually deployable on iPhone via SFI, which nobody had a turnkey solution for.

## The full architecture

```
┌───────────────────────────────────────────────────────────────┐
│ YOUR DEVICE (one process: Xray on desktop, sing-box on mobile) │
│                                                                │
│  1. Browser → SOCKS5 on 127.0.0.1:10808 (desktop) or           │
│               TUN interface (mobile, via SFI)                  │
│  2. Routing rule sends traffic to wireguard-out                │
│  3. WireGuard outbound (userspace, gVisor) encrypts with       │
│     ChaCha20-Poly1305                                          │
│  4. The wireguard outbound dials its peer at 127.0.0.1:51821   │
│     BUT with sockopt.dialerProxy (Xray) / detour (sing-box)    │
│     → that dial happens THROUGH the VLESS+REALITY outbound     │
│  5. VLESS+REALITY+Vision outbound:                             │
│     - Wraps the WireGuard UDP as TLS 1.3 to www.microsoft.com  │
│     - REALITY's stolen-handshake camouflage activates          │
│     - Vision flow pads the inner handshake for fingerprint     │
│       resistance                                               │
│     - uTLS makes the ClientHello look exactly like Chrome      │
│  6. Packet leaves the device as HTTPS :443 to your VPS         │
└───────────────────────────┬──────────────────────────────────┘
                            │
                       [ INTERNET ]
                            │
                 What an observer sees:
                 - TLS 1.3 connection to :443
                 - SNI: www.microsoft.com
                 - Valid Microsoft certificate (if probed)
                 - Chrome uTLS ClientHello fingerprint
                 - Random-padded inner handshake
                 - Normal HTTPS browsing shape
                 - NO VPN signature
                 - NO WireGuard fingerprint
                 - NO TLS-in-TLS tell
                            │
┌───────────────────────────┴──────────────────────────────────┐
│ YOUR VPS                                                      │
│                                                               │
│  7. Xray VLESS+REALITY inbound on :443                        │
│  8. REALITY validates the x25519 pre-shared auth              │
│     - If invalid (a probe): proxies to real Microsoft         │
│     - If valid (your client): completes the handshake locally │
│  9. Xray unwraps the VLESS layer (Vision validates)           │
│  10. freedom outbound forwards the decrypted UDP              │
│  11. Hits the local WireGuard server on 127.0.0.1:51821       │
│  12. WireGuard decrypts the inner tunnel                      │
│  13. Routes to the actual destination via eth0 (public IP)    │
│  14. Response travels back through the same chain in reverse  │
└──────────────────────────────────────────────────────────────┘
```

## The layers from innermost to outermost

When you load a website through SwizGuard, your data passes through several encryption and framing layers:

1. **Application data** — your browser's HTTPS, your SSH, whatever
2. **WireGuard encryption** — ChaCha20-Poly1305 over UDP
3. **VLESS framing** — lightweight proxy protocol, no extra encryption
4. **Vision flow** — handshake padding, kernel splice optimization
5. **REALITY / TLS 1.3** — outer encryption and camouflage
6. **TCP** — actual wire transport

Your traffic is encrypted by both WireGuard AND TLS 1.3. Both use ChaCha20-family AEAD ciphers. An attacker would need to break both layers to read your data, and breaking either one is not known to be possible with current cryptography.

## REALITY in detail

REALITY is the magic that makes the outer layer invisible. Here's what it actually does.

### Why normal proxy TLS gets caught

Traditional proxies like Trojan or Shadowsocks-with-TLS use either self-signed certificates or real certificates from a CA you control. A censor or DPI system can:

- Check if the cert matches a known proxy pattern
- Actively probe your server and see it's not actually a real website
- Fingerprint the TLS implementation, which is usually different from real browsers

### How REALITY avoids all of that

REALITY doesn't use your own certificate at all. Here's the handshake flow:

1. Your client sends a TLS ClientHello with SNI set to `www.microsoft.com` and Chrome's exact uTLS fingerprint
2. The server receives it and checks for a special authentication token embedded in what looks like random padding in the handshake
3. **If the token is missing or invalid** (meaning it's a censor, scanner, or someone who typed your IP into a browser): the server proxies the entire TLS handshake to the real `www.microsoft.com` and returns Microsoft's actual certificate. The prober sees a legitimate website and moves on.
4. **If the token is valid** (it's your real client, derived from the pre-shared x25519 key): the server completes the handshake locally and switches to proxy mode.

The result:
- Active probing returns a real website with a real certificate
- Passive observation sees standard TLS 1.3 traffic to Microsoft
- There's no fake certificate, no custom CA, nothing synthetic to fingerprint
- The x25519 authentication is buried in fields that look like random TLS padding

I tested this. From a separate machine I ran:

```bash
curl -I --resolve www.microsoft.com:443:MY_VPS_IP https://www.microsoft.com
```

And got:
```
HTTP/2 200
server: AkamaiGHost
content-type: text/html
```

That response is coming from Akamai's actual CDN infrastructure, served via the REALITY proxy. The cert in the TLS session is a real DigiCert-signed cert for `*.microsoft.com`. From a prober's perspective, my server is mechanically indistinguishable from a Microsoft edge node because, well, that's what the bytes on the wire actually are.

### Why x25519

REALITY uses x25519 (Curve25519) for the pre-shared authentication because:
- The keys look like random bytes — indistinguishable from TLS random fields
- The authentication happens within the existing TLS handshake structure
- No extra round trips or unusual packet patterns required

## Vision flow in detail

Vision (`xtls-rprx-vision`) is a flow mode within VLESS that does two things at once.

### Inner handshake padding

When you tunnel TLS through VLESS+REALITY, the inner TLS handshake has very recognizable sizes:
- ClientHello: ~517 bytes
- ServerHello + certificate chain: ~2-4KB
- Change cipher spec: ~64 bytes

Those sizes are deterministic and appear in the encrypted outer tunnel as a recognizable pattern. Vision detects the inner TLS handshake records and injects random-sized padding bytes into each one, breaking the deterministic sizing.

After the handshake completes, Vision stops padding because bulk data transfer doesn't have the same fingerprint problem.

### Kernel splice optimization

Once the inner handshake is done, Vision tells the Linux kernel to `splice(2)` raw TCP bytes directly between the client socket and the upstream socket, bypassing Xray's userspace buffer copying entirely. This gives you:

- Lower CPU on the server
- Higher throughput (near line-rate)
- Lower latency on bulk transfers

It's one of those rare features that's both more secure AND faster.

## The chained outbound trick

The architecture hinges on a single config field that tells the WireGuard outbound to dial its peer endpoint THROUGH another outbound (the VLESS+REALITY one). Different libraries call this different things.

### Xray (desktop) — `sockopt.dialerProxy`

```json
{
  "tag": "wireguard-out",
  "protocol": "wireguard",
  "settings": { ... },
  "streamSettings": {
    "sockopt": {
      "dialerProxy": "proxy"
    }
  }
}
```

When the WireGuard outbound needs to send UDP packets to its peer, Xray intercepts the dial and routes it through the outbound tagged `"proxy"` (the VLESS+REALITY one).

### sing-box (iOS / Android) — `detour`

```json
{
  "type": "wireguard",
  "tag": "wg-out",
  "detour": "proxy",
  ...
}
```

Same concept under a different name. `detour` is a first-class field that chains any outbound through another. This is the field that makes mobile parity possible — without it, you can't get the full chain on iPhone.

### What this gets you

WireGuard's encrypted UDP packets never leave your device as UDP. They're transported as TLS frames over TCP, wrapped as HTTPS to microsoft.com, with all the REALITY + Vision magic applied to the outer layer. The WireGuard layer exists only inside your device and inside your VPS — it never appears on the public wire as WireGuard.

## Why userspace WireGuard

SwizGuard runs WireGuard in userspace (via Xray's gVisor TUN or sing-box's internal WG stack) instead of using the kernel WireGuard module. The reasons:

- **No sudo to start the tunnel** — Xray and sing-box run as regular users
- **No routing conflicts** — there's no system-level WG tunnel grabbing the default route
- **No kernel module required** — works on macOS and iOS where kernel WG isn't available
- **Easier multi-instance setup** — you could run two SwizGuard clients side by side without them fighting

The trade-off is slightly lower peak throughput than kernel WG (5-15% depending on workload). In practice the bottleneck is the REALITY layer and the internet, not WireGuard, so the loss is invisible during real use.

## Why mobile uses sing-box instead of Xray

Desktop clients use Xray-core because:
- It's the reference implementation of all the protocols
- It has the best `sockopt.dialerProxy` support
- It's what the VPS also runs
- `brew install xray` is one command

Mobile clients use sing-box because:
- **Xray has no maintained mobile apps with full JSON config import**
- Shadowrocket and Hiddify are too opinionated — they can't express chained outbounds
- **SFI (Sing-Box For iOS)** and **SFA (Sing-Box For Android)** import raw sing-box JSON directly
- Sing-box's `detour` field is the clean equivalent of Xray's chaining

The two config formats are different (sing-box uses `endpoints` for WireGuard in its modern schema, Xray uses `outbounds` with `streamSettings.sockopt`), but they talk to the same server and produce the same wire behavior. SwizGuard generates both from a single template so you don't have to maintain them separately.

## MTU math

WireGuard inside TLS 1.3 inside TCP has layered overhead:

- WireGuard adds ~60 bytes per packet (header + auth tag)
- VLESS adds ~20 bytes per record (framing)
- TLS 1.3 adds ~22 bytes per record (header + auth tag)
- TCP adds 20 bytes

Total per-packet overhead is roughly 120 bytes. With a standard 1500-byte Ethernet MTU, that leaves ~1380 bytes of payload. SwizGuard sets the WireGuard MTU to 1280 (the IPv6 minimum, a safe universal choice) to avoid fragmentation while leaving headroom for the encapsulation.

## Performance characteristics

Compared to vanilla WireGuard:

- **Latency:** +1-5ms depending on server distance (mostly REALITY handshake cost; splice handles the rest)
- **Throughput:** ~70-80% of raw WireGuard in userspace mode
- **CPU:** moderate — ChaCha20 is hardware-accelerated on modern devices, splice optimization helps
- **Battery (mobile):** slightly higher than plain VPN due to the extra encryption layer, but mitigated by sing-box's efficiency

In practice — for browsing, streaming, video calls, general use — the overhead is imperceptible. I run this on my daily-driver Mac and I don't notice it.

## Threat model

I want to be honest about what this defends against and what it doesn't. Most VPN tools oversell their own capabilities. I'd rather you understand the tradeoffs.

### What SwizGuard defends against

**Passive network observation:** Solid. Your ISP, corporate firewall, or nation-state bulk collection sees TLS 1.3 to Microsoft with a Chrome fingerprint. There's nothing to flag.

**Active probing:** Solid. REALITY returns the real Microsoft website to any prober. There's no way to distinguish your server from a real Microsoft edge node via network probing alone.

**TLS-in-TLS fingerprinting:** Solid with Vision flow enabled. The known GFW detection technique is mitigated.

**Commercial VPN blocklists:** Solid. Your VPS IP isn't on any "known VPN" list because it's not a commercial VPN provider.

**Consumer DPI firewalls (Fortinet, Cisco, hotel networks, corporate guest Wi-Fi):** Solid. They see HTTPS.

**Corporate network restrictions:** Solid. Port 443 is always open. Your traffic looks like browsing.

### What SwizGuard does NOT defend against

**Global passive adversaries doing end-to-end timing correlation.** If someone has visibility into your ISP traffic AND your VPS's upstream provider, they can correlate "bytes enter your ISP" with "bytes exit the VPS to destination X" purely from timing and volume patterns. This is a fundamental limitation of low-latency VPNs. Only high-latency mixnets (Tor at scale, Nym, etc.) defeat this. If your threat model includes nation-state actors specifically targeting you, you need Tor with bridges, not a VPN.

**VPS provider compromise or legal compulsion.** If your VPS provider is compromised or subpoenaed, the attacker gets root on the box and can extract your REALITY private key, your WG server key, and decrypt traffic in real time. Mitigations:
- Pick a jurisdiction that matches your threat model
- Use a provider with a good privacy track record (Njalla, Mullvad's own VPS, 1984 Hosting)
- Pay with Monero for anonymity
- Rotate keys periodically with `swizguard rekey`

**Endpoint compromise.** Malware on your iPhone or Mac can read your traffic before encryption or after decryption. No VPN helps here. Keep your devices clean.

**Behavioral deanonymization.** Logging into Google with your real name while connected to SwizGuard still identifies you to Google. The VPN hides your network location, not your identity. Two different things.

## How SwizGuard compares

| | WireGuard alone | Consumer VPN (Nord, etc.) | SwizGuard |
|---|---|---|---|
| Encrypted | Yes | Yes | Yes (multi-layer) |
| Fast | Very | Moderate | Fast (near-native with splice) |
| Fingerprintable by DPI | Yes | Yes | **No** |
| Blockable by firewall | Yes | Yes | **No** (would have to block Microsoft) |
| Active probe resistant | No | No | **Yes** (returns real Microsoft) |
| Needs domain / cert | No | N/A (they handle it) | **No** (REALITY steals it) |
| Unique IP per user | Yes | No (shared) | Yes |
| Third-party trust | Just VPS | VPN company + VPS | **Just VPS** |
| Self-hosted | Yes | No | **Yes** |
| Logs by default | Yes (system logs) | Their policy | **None** (access logs disabled) |
| Works in China/Russia/Iran | No | Mostly blocked | **Yes** |
| One-command deploy | No | N/A | **Yes** |

## Operational security notes

A few things I do that aren't strictly required but I recommend:

- **Rotate REALITY keys every 3-6 months** with `sudo ./swizguard rekey`. Clients need to be regenerated after.
- **Disable access logs** (already default in SwizGuard). The server has no record of what sites you visit.
- **Use a dedicated VPS** for SwizGuard. Don't run other public services on the same IP — they create fingerprintable patterns and widen your attack surface.
- **Pick a camouflage target geographically close to your VPS** for lower probe latency. Microsoft has edge presence everywhere, which is why I use them as the default.
- **If your VPS doesn't fully support IPv6, disable IPv6 on the client.** Half-configured IPv6 leaks are worse than no IPv6 at all.
- **Test your camouflage:** from another machine, run `curl -I --resolve www.microsoft.com:443:YOUR_VPS_IP https://www.microsoft.com`. You should get Microsoft's real response back. If you do, the camouflage is working. If you don't, something is broken or your VPS is compromised.
