# Linux Firewalls

Netfilter provides kernel packet hooks and stateful filtering/NAT infrastructure. nftables and iptables are userspace interfaces; firewalld/UFW provide higher-level management. The installed command name alone does not identify which backend or active policy owns a packet path.

## Configuration Owner and Effective Rules

| Interface | Role | Inspection |
|---|---|---|
| `nft` | Manage nftables rules/tables | `nft list ruleset` |
| `iptables-nft` | iptables-compatible interface backed by nftables | Check tool version/backend and resulting rules |
| Legacy iptables | Older xtables backend | Inspect corresponding IPv4/IPv6 tables separately |
| firewalld | Zones/services/policies with a selected backend | Inspect active zones, runtime and permanent configuration |
| UFW | Host-policy frontend where configured | Inspect activation, rules, and actual forwarding path |

Multiple managers can create overlapping chains/hooks, bypass intended paths, or overwrite rules on reload. Identify the supported configuration owner and application/container-managed rules before changing anything. An ACCEPT at one hook/chain does not universally bypass every later policy layer.

```bash
# Read-only Linux templates; admin privileges may be required.
nft list ruleset
iptables --version
firewall-cmd --state
firewall-cmd --get-active-zones
firewall-cmd --list-all
firewall-cmd --permanent --list-all
ufw status verbose
# Tools may not be installed; absence is not proof that no kernel policy exists.
```

For firewalld, a runtime-only change can disappear after a reload/restart; a permanent-only change need not be active yet. Test both the effective runtime state and intended persistence. UFW is not automatically active merely because it is packaged on an Ubuntu/Debian host.

!!! info "External Resources"
    - [nftables Packet Hooks and Coexistence](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks) (Netfilter)
    - [firewalld Runtime versus Permanent](https://firewalld.org/documentation/configuration/runtime-versus-permanent.html) (firewalld)
    - [UFW Status and Management](https://manpages.ubuntu.com/manpages/noble/en/man8/ufw.8.html) (Ubuntu/UFW)

## Packet Paths, State, and Recovery

Use [Segmentation and Firewall Decisions](../networking/segmentation.md) for a worked ruleset, input/forward/output hooks, NAT, IPv6, and return traffic. A container's published service may traverse forwarding/NAT rather than the host INPUT path. Dropping INPUT alone is not proof that every published port is blocked.

Define permitted initiation flows and relevant reply behavior. Restrict new traffic with a default-deny policy where appropriate, but account for required DNS, management, update, node-control, and recovery paths before applying OUTPUT/FORWARD restrictions. Existing connection tracking can preserve traffic after a new-flow rule changes. IPv4 forwarding and IPv6 forwarding have separate controls.

Test permitted and denied paths from the intended vantage, same-host and routed paths, both address families, existing connections, restart persistence, and failover. Keep an authorized recovery path when changing remote-management rules. Do not copy a `flush ruleset` or blanket drop into a live host as an inspection step.
