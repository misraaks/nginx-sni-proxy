# nginx-sni-proxy
## Overview
Lightweight SNI-based TCP proxy using nginx and gluetun (WireGuard). Incoming TLS connections are routed out through a WireGuard tunnel managed by gluetun based on the SNI hostname.

### Quick start (docker-compose)
1. Copy [.env.example](.env.example) -> `.env` and fill WireGuard values.
2. Start services:

```sh
docker compose up -d
```

### Notes
- gluetun requires WireGuard credentials and endpoint IP (no domain names for the WireGuard endpoint) — see [.env.example](.env.example).
- The resolver in [nginx.conf](nginx.conf) is set to Cloudflare (1.1.1.1 / 1.0.0.1); override if needed.

## My Use Case
I use a custom DNS provider (for example, ControlD) together with Tailscale DNS to point selected domain names at the Tailnet IP of the machine running this stack. That lets me route traffic for specific hostnames through a single WireGuard tunnel — a form of domain-based split tunneling — rather than using route-based split tunneling. It also allows multiple client devices to share the same VPN connection.

### Diagram (example: netflix.com)
A simplified flow showing the DNS resolver returning the Tailnet IP for netflix.com and how SNI-based routing sends that TLS session through the WireGuard tunnel:
```
Client                        DNS resolver                  Tailnet host (nginx + gluetun)      Internet (netflix.com)
  |                             |                               |                             |
1.| DNS query: netflix.com ---> |                               |                             |
  |                             |                               |                             |
2.| <--- A response: 100.71.2.5 |                               |                             |
  |      (Tailnet IP)           |                               |                             |
  |                             |                               |                             |
3.| --- TLS connect to 100.71.2.5 (SNI: netflix.com) ---------> |                             |
  |                             |                               |                             |
4.| <---- TCP/TLS Handshake ----------------------------------> |                             |
  |                             |                               |                             |
5.|                             |                               | (nginx reads SNI,           |
  |                             |                               |  routes via WireGuard)      |
  |                             |                               | --------------------------->|
  |                             |                               |                             |
```

If the Tailnet device running this stack already sits on the network you want outbound traffic to originate from, you can run without gluetun: nginx will route SNI-matched connections directly from that device's network.

Limitations: this approach relies on SNI (the hostname visible in the TLS handshake). It won't work for non-TLS traffic, and it may stop working as Encrypted Client Hello (ECH) becomes widespread because ECH hides the hostname.

