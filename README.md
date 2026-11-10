# ansible-role-firewall

Configure an iptables firewall: default DROP, ipset whitelist, SSH rate limit,
WireGuard NAT.

Script took from [iptables-ipset](https://gitflic.ru/project/serveradmin/iptables-ipset/file?branch=master).  
Change some logic in iptables-rules.sh.j2.

## Requirements

- Debian (bullseye, bookworm), Ubuntu (focal, jammy, noble, resolute)

## Features

- Default DROP policy
- IP whitelist (ipset) with full access
- SSH rate limit per source IP (iptables `hashlimit`, no ban list)
- WireGuard NAT integration (MASQUERADE / SNAT, extra entry ports, MSS clamp)
- Optional inbound IPv6 block
- fail2ban side files: disable the stock `sshd` jail, global `ignoreip`
- Persistent rules across reboots

## Rule ordering (important)

```
ESTABLISHED, lo, ICMP, WHITELIST      -> ACCEPT
INVALID, null flags, non-SYN NEW      -> DROP
SSH NEW above firewall_ssh_ratelimit  -> DROP   (per source IP, no ban list)
WireGuard, restricted, public ports   -> ACCEPT
everything else                       -> policy DROP
```

- The SSH rate limit sits after the WHITELIST ACCEPT, so whitelisted IPs never
  hit it, and it covers the SSH port only: on CGNAT / mobile carriers a bot
  behind a shared IP cannot lock out VPN / web users on that IP.
- There are no ban lists for closed ports: the DROP policy already handles
  them. Role versions before 2026-11 had `BANPORT` / `BANDDOS` ipsets, SYN
  `DDOS_PROTECT` and portscan logging; since the public ports were ACCEPTed
  before them they only ever matched closed ports, i.e. did nothing. The
  script destroys those ipsets and the role removes their rsyslog / logrotate
  config and `/var/log/ipset/`.
- fail2ban inserts its chains at the top of INPUT, above the WHITELIST ACCEPT.
  Use `firewall_fail2ban_ignoreip` so no jail can ban a whitelisted IP. The
  script flushes every chain; the apply handler restarts fail2ban afterwards
  so its rules come back.

## Role Variables

Port variables end with the protocol (`_tcp` / `_udp`).

| Variable | Default | Description |
|----------|---------|-------------|
| `firewall_wan_interface` | auto-detect | External network interface |
| `firewall_wan_ip` | auto-detect | External IP address |
| `firewall_public_ports_tcp` | `[22]` | TCP ports open to everyone on the internet |
| `firewall_public_ports_udp` | `[]` | UDP ports open to everyone on the internet |
| `firewall_whitelist_ips` | `[]` | IPs with full access |
| `firewall_wireguard_enabled` | `false` | Enable WireGuard rules |
| `firewall_wireguard_port_udp` | `51820` | WireGuard listen port |
| `firewall_wireguard_interface` | `wg0` | WireGuard interface name |
| `firewall_wireguard_network` | `10.0.0.0/24` | WireGuard network CIDR |
| `firewall_wireguard_mss` | `0` | TCP MSS clamp for forwarded tunnel traffic (`0` = off, e.g. `1240` for inner MTU 1280); lowers MSS only, both directions |
| `firewall_wireguard_snat_source` | `""` | Egress IP for tunnel traffic: `""` = MASQUERADE (primary WAN address), an IP = `SNAT --to-source <IP>` (multi-IP hosts: clients on one address, traffic leaves from another) |
| `firewall_wireguard_extra_ports_udp` | `[]` | Extra UDP ports redirected (nat REDIRECT) to `firewall_wireguard_port_udp`, e.g. `[443]`; server keeps its real port. A port equal to `firewall_wireguard_port_udp` is skipped, so one list like `[443, 51820, 27015]` fits hosts on any of these ports |
| `firewall_ssh_ratelimit` | `""` | Drop NEW SSH connections per source IP above this rate (`hashlimit`, e.g. `6/min`); `""` = off |
| `firewall_ssh_burst` | `10` | Burst for `firewall_ssh_ratelimit` |
| `firewall_ssh_port` | `22` | Port the SSH rate limit applies to |
| `firewall_fail2ban_disable_sshd` | `false` | Write `jail.d/zz-sshd-off.local` (`[sshd] enabled = false`); use when the SSH rate limit covers SSH |
| `firewall_fail2ban_ignoreip` | `false` | Write `jail.d/zz-ignoreip.local`: `[DEFAULT] ignoreip` = loopback + whitelist + `firewall_fail2ban_ignoreip_extra` |
| `firewall_fail2ban_ignoreip_extra` | `[]` | Extra addresses fail2ban must never ban (not added to the firewall whitelist) |
| `firewall_allow_ping` | `true` | Allow ICMP ping |
| `firewall_restricted_ports_tcp` | `[]` | TCP ports open only to specific source IPs |
| `firewall_ipv6_block` | `false` | Close inbound IPv6: ip6tables INPUT/FORWARD DROP, allow only replies, `lo`, ICMPv6 (ND/RA/PTB) and DHCPv6 replies; OUTPUT stays open |
| `firewall_ipv6_allow_wireguard` | `false` | With `firewall_ipv6_block`: accept `firewall_wireguard_port_udp` over IPv6 and redirect `firewall_wireguard_extra_ports_udp` to it |
| `firewall_custom_rules` | `[]` | Additional iptables rules |

## Examples

### Basic server (SSH only)

```yaml
firewall_public_ports_tcp:
  - 22
```

### Web server

```yaml
firewall_public_ports_tcp:
  - 22
  - 80
  - 443

firewall_public_ports_udp:
  - 443
```

### VPN server (WireGuard)

```yaml
firewall_public_ports_tcp:
  - 22

firewall_wireguard_enabled: true
firewall_wireguard_port_udp: 51820
firewall_wireguard_interface: "wg0"
firewall_wireguard_network: "10.0.0.0/24"
```

### With IP whitelist

```yaml
firewall_whitelist_ips:
  - "203.0.113.10"
  - "198.51.100.0/24"
```

### SSH rate limit, fail2ban only for application jails

```yaml
firewall_ssh_ratelimit: "6/min"
firewall_ssh_burst: 10
firewall_fail2ban_disable_sshd: true
firewall_fail2ban_ignoreip: true
```

The fail2ban files are written even when fail2ban is not installed yet, so
installing it later (e.g. by another installer) picks them up.

### Port open only for specific IPs (e.g. management port)

```yaml
firewall_restricted_ports_tcp:
  - port: 27015
    sources:
      - "203.0.113.10"   # management server
```

## Persistence

Rules are saved via `netfilter-persistent` and automatically restored on boot.

## Useful Commands

```bash
# View whitelist IPs
ipset -L WHITELIST

# SSH rate-limit drops (packet counter)
iptables -vnL INPUT | grep 'limit: above'

# Per-source state of the SSH limiter
cat /proc/net/ipt_hashlimit/ssh

# View current rules
iptables -vnL

# fail2ban: active jails and global ignoreip
fail2ban-client status
fail2ban-client get <jail> ignoreip

# Save current rules
netfilter-persistent save

# Reload saved rules
netfilter-persistent reload

# Manually reapply rules
/usr/local/sbin/iptables-rules.sh
netfilter-persistent save
```

## Files

| File | Description |
|------|-------------|
| `/usr/local/sbin/iptables-rules.sh` | Main firewall script |
| `/etc/iptables/rules.v4` | Saved iptables rules |
| `/etc/iptables/ipsets` | Saved ipset lists |
| `/etc/fail2ban/jail.d/zz-sshd-off.local` | Stock `sshd` jail off (`firewall_fail2ban_disable_sshd`) |
| `/etc/fail2ban/jail.d/zz-ignoreip.local` | Global fail2ban `ignoreip` (`firewall_fail2ban_ignoreip`) |

## License

MIT
