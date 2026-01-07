# ansible-role-firewall

Configure iptables firewall with ipset, DDoS protection, and portscan blocking.

## Requirements

- Debian/Ubuntu

## Features

- Default DROP policy
- DDoS protection with automatic IP banning
- Portscan detection and blocking
- WireGuard NAT integration
- IP whitelist support
- Separate log files with rotation
- Persistent rules across reboots

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `firewall_wan_interface` | auto-detect | External network interface |
| `firewall_wan_ip` | auto-detect | External IP address |
| `firewall_ports_tcp` | `[22]` | Allowed TCP ports |
| `firewall_ports_udp` | `[]` | Allowed UDP ports |
| `firewall_whitelist_ips` | `[]` | IPs with full access |
| `firewall_wireguard_enabled` | `false` | Enable WireGuard rules |
| `firewall_wireguard_port` | `51820` | WireGuard listen port |
| `firewall_wireguard_interface` | `wg0` | WireGuard interface name |
| `firewall_wireguard_network` | `10.0.0.0/24` | WireGuard network CIDR |
| `firewall_ddos_enabled` | `true` | Enable DDoS protection |
| `firewall_ddos_rate` | `10/sec` | Rate limit for DDoS |
| `firewall_ddos_burst` | `20` | Burst limit for DDoS |
| `firewall_ban_timeout` | `600` | Ban duration (seconds) |
| `firewall_logging_enabled` | `true` | Enable logging |
| `firewall_portscan_protection` | `true` | Enable portscan protection |
| `firewall_allow_ping` | `true` | Allow ICMP ping |
| `firewall_custom_rules` | `[]` | Additional iptables rules |

## Examples

### Basic server (SSH only)

```yaml
firewall_ports_tcp:
  - 22
```

### Web server

```yaml
firewall_ports_tcp:
  - 22
  - 80
  - 443

firewall_ports_udp:
  - 443
```

### VPN server (WireGuard)

```yaml
firewall_ports_tcp:
  - 22

firewall_wireguard_enabled: true
firewall_wireguard_port: 51820
firewall_wireguard_interface: "wg0"
firewall_wireguard_network: "10.0.0.0/24"
```

### With IP whitelist

```yaml
firewall_whitelist_ips:
  - "203.0.113.10"
  - "198.51.100.0/24"
```

## Persistence

Rules are saved via `netfilter-persistent` and automatically restored on boot.

## Useful Commands

```bash
# View banned IPs
ipset -L BANPORT
ipset -L BANDDOS

# Clear all bans
ipset flush BANPORT
ipset flush BANDDOS

# Unban specific IP
ipset del BANPORT 1.2.3.4

# View current rules
iptables -L -n -v

# View logs
tail -f /var/log/ipset_ddos.log
tail -f /var/log/ipset_portscan.log

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
| `/etc/rsyslog.d/ipset.conf` | Rsyslog configuration |
| `/etc/logrotate.d/ipset-iptables` | Logrotate configuration |
| `/var/log/ipset_ddos.log` | DDoS attack log |
| `/var/log/ipset_portscan.log` | Portscan log |

## License

MIT
