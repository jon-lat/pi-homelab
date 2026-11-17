# Network

## Addressing

| Interface | Address | Notes |
|---|---|---|
| `eth0` | `192.168.1.35/24` | Primary interface, DHCP lease, gateway `192.168.1.1` |
| `tailscale0` | `100.x.y.z/32` | Tailscale |
| `br-...` (homepage) | `172.18.0.0/16` | Docker network `homepage_default` |
| `br-...` (uptime-kuma) | `172.19.0.0/16` | Docker network `uptime-kuma_default` |
| `br-...` (ntfy) | `172.20.0.0/16` | Docker network `ntfy_default` |

## DNS

Pi-hole runs natively and serves DNS on port 53 (UDP and TCP, IPv4 and IPv6).

| Setting | Value |
|---|---|
| Upstream resolvers | `1.1.1.1`, `1.0.0.1` (Cloudflare) |
| Listening mode | `LOCAL` (local networks only) |
| Query logging | Enabled |
| Blocklists | StevenBlack hosts (77,310 entries), HaGeZi Multi Pro (227,848 entries) |
| Unique blocked domains | 305,158 |

Verification:

```bash
dig @192.168.1.35 example.com +short        # resolves normally
dig @192.168.1.35 doubleclick.net +short    # returns 0.0.0.0 (blocked)
```

Clients are pointed at `192.168.1.35` for DNS. DNS filtering applies to domain lookups, so ads served from the same domain as the content are not affected.

## Firewall (UFW)

Default policy: deny incoming, allow outgoing, deny routed.

| Port | Proto | Source | Purpose |
|---|---|---|---|
| 22 | TCP | Anywhere | SSH |
| 9090 | TCP | 192.168.1.0/24 | Cockpit |
| 80, 443 | TCP | 192.168.1.0/24 | Pi-hole admin |
| 53 | TCP, UDP | 192.168.1.0/24 | Pi-hole DNS |
| 631 | TCP | 192.168.1.0/24 | CUPS |
| 3000 | TCP | 192.168.1.0/24 | Homepage |
| 8080 | TCP | 192.168.1.0/24 | ntfy |
| 631 | TCP | 172.19.0.0/16 | Uptime Kuma network to CUPS |
| 631, 3000, 9090 | TCP | 172.18.0.0/16 | Homepage network to monitored services |
| 3000, 9090 | TCP | 172.19.0.0/16 | Uptime Kuma network to monitored services |
| 5353 | UDP | Anywhere | mDNS |
| 3702 | UDP | Anywhere | WS-Discovery |
| 5357 | TCP | Anywhere | WS-Discovery |

### Container-to-host traffic

Homepage and Uptime Kuma run on their own Docker bridge networks and monitor services on the host's LAN address. UFW only allowed those ports from `192.168.1.0/24`, so the monitoring checks were dropped and the dashboard showed grey status indicators.

Diagnosis:

- Kernel log: `[UFW BLOCK] IN=br-... SRC=172.19.0.2 DST=192.168.1.35 PROTO=TCP DPT=9090`, and the same for `SRC=172.18.0.2` towards `DPT=631`
- Homepage log: `connect ETIMEDOUT 192.168.1.35:9090`, `:3000` and `:631`
- `docker exec homepage wget https://192.168.1.35:9090` timed out

Fix: allow each container network to reach only the ports it monitors.

```bash
sudo ufw allow from 172.18.0.0/16 to any port 631,3000,9090 proto tcp comment 'Homepage monitors'
sudo ufw allow from 172.19.0.0/16 to any port 3000,9090 proto tcp comment 'Uptime Kuma monitors'
```

Verification: the same `wget` now returns the Cockpit page, and the Homepage log shows no `ETIMEDOUT` entries over a 60-second window. CUPS reports as down while the service is stopped, which is expected.

## Remote access

Tailscale runs natively as the `tailscaled` systemd service. A desktop, a phone and the Pi are members of the same tailnet, so the homelab is reachable remotely over an encrypted WireGuard connection with no router port forwarding.

## Next steps

- [ ] DHCP reservation or static address for the Pi
- [ ] Router DNS pointed at Pi-hole for network-wide filtering
- [ ] Point the Pi's own DNS at Pi-hole
- [ ] Restrict SSH to the LAN and the Tailscale interface
- [ ] Tidy the mDNS and WS-Discovery rules
- [ ] Audit published container ports against the firewall policy
