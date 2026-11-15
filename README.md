# KESTREL Router Session Harvester

ARP man-in-the-middle harvester for authenticated router admin pages that
expose subscription email addresses and password hashes.

The tool captures a victim's router session on a plaintext LAN, extracts the
subscription pair from the router's own WAN list responses, and prints the
Hashcat command to crack the password hash. It targets the Huawei-style admin
flow behind `/CustomApp/userguideframe.asp` (`/login.cgi`,
`/asp/GetRandCount.asp`, `/html/bbsp/common/wan_list.asp`), and also handles
pages with labeled email and hash fields.

> Authorized testing only. Use this on networks you own or have written
> permission to test.

Current build: `2026-11-01.8`

## Features

- ARP poisoning of every host on the subnet through the gateway and target
- TCP stream reassembly and HTTP/1.x parsing
- Login credential capture from POST bodies, query strings, and Basic auth
- Router session cookie capture
- Live response scanning that extracts subscription pairs from the victim's
  own `wan_list.asp` traffic
- Guide page grab with same-origin `.asp` source following
- Login replay with CSRF token refresh and automatic logout
- `--auto` IP takeover for sessions bound to the victim IP
- Forwarding preflight, FORWARD DROP handling, and full state restore
- Per-run loot rotation
- Hashcat hint for mode 20800

## Requirements

- Linux on the victim LAN with root
- Python 3.9 or newer
- `scapy` and `requests`
- A real bridged NIC. NAT and WSL2 mirrored networking do not carry victim
  traffic into the VM.

## Install

```bash
pip install -r requirements.txt
```

## Quick start

Default capture flow:

```bash
sudo python3 exploit.py --iface eth0 --debug
```

Takeover flow for routers that bind sessions to the client IP:

```bash
sudo python3 exploit.py --iface eth0 --auto --debug
```

Wait for `running, ctrl-c to stop`, have the victim log in to the router over
plain HTTP, then stop with one Ctrl-C.

## How it works

1. Discovers LAN hosts and resolves the gateway and target MAC addresses.
2. Opens forwarding and inserts a temporary FORWARD ACCEPT rule when the
   firewall drops forwarded traffic.
3. ARP-poisons the hosts and the server addresses to become the relay.
4. Reassembles TCP streams and parses HTTP/1.x in both directions.
5. Captures login credentials, Basic auth, and router session cookies.
6. Scans router response bodies for subscription pairs.
7. On a session cookie, fetches the guide page and its `.asp` data sources,
   then replays the login if the cookie is rejected. In `--auto`, the victim
   IP is added temporarily and the fetch is bound to it.
8. Prints the email and hash, the ISP note, and the Hashcat command.
9. Removes the added IP, restores ARP, removes the firewall rule, and restores
   sysctls.

## Options

| Flag | Default | Description |
|------|---------|-------------|
| `--iface` | `eth4` | LAN interface |
| `--gateway` | auto | Gateway IP |
| `--subnet` | interface network | CIDR to poison |
| `--target-url` | `http://192.168.1.1/CustomApp/userguideframe.asp` | Authenticated leak page |
| `--interval` | `2` | ARP poison resend seconds |
| `--rescan` | `60` | Host rescan seconds |
| `--sweep-timeout` | `2` | ARP sweep timeout seconds |
| `--flow-idle` | `300` | Drop idle TCP streams after seconds |
| `--timeout` | `10` | HTTP request timeout seconds |
| `--max-hosts` | `256` | Maximum hosts to poison |
| `--ports` | `80` | Plaintext HTTP ports to sniff |
| `--loot` | `loot` | Output directory |
| `--keep-loot` | off | Append to existing loot instead of rotating |
| `--no-replay` | off | Skip credential replay |
| `--no-forward` | off | Leave forwarding unchanged |
| `--capture-self` | off | Process this host's own traffic |
| `--auto` | off | Victim IP takeover on captured cookie |
| `--auto-delay` | `2` | Seconds between takeover steps |
| `--auto-retries` | `3` | Takeover fetch attempts |
| `--debug` | off | Log every HTTP message and grab source |
| `--check` | off | Print interface, forwarding, firewall, hosts, exit |
| `--restore-only` | off | Send truthful ARP mappings and exit |
| `--selftest` | off | Offline parser tests, no root needed |

## Outputs

A non-empty `loot/` is rotated to `loot.bak-<timestamp>` at startup. Use
`--keep-loot` to append instead.

- `creds.jsonl`: captured credentials
- `cookies.jsonl`: captured router cookies with their source
- `leaks.jsonl`: extracted pairs, source URLs, saved files, and Hashcat commands
- HTML and JS dumps of every fetched page and data source

## Hashcat

Each pair prints the ISP note and a cracking command:

```text
note: many ISPs use an 8-digit numeric password
hashcat: hashcat -m 20800 -a 3 <hash> '?d?d?d?d?d?d?d?d'
```

Mode 20800 is `sha256(md5($pass))`. The mask covers the 8-digit numeric
passwords many ISPs assign to subscriptions.

## Auto mode

`--auto` replaces the cookie grab with an IP takeover:

1. Captures the victim's router cookie.
2. Adds the victim IP as a `/32` on the interface.
3. Fetches the leak page and WAN sources with the cookie from that source IP.
4. Removes the address and restores ARP.

The victim IP is held only for the fetch, but a brief IP conflict with the
victim device is possible.

## Safety and restore

- Forwarding, firewall, ARP, and sysctls are restored on Ctrl-C and normal exit.
- The poison thread is joined before ARP restoration, and corrective ARP
  frames are sent in hypervisor-safe variants plus gratuitous announcements.
- A SIGKILL leaves only ARP timeout as recovery. Run `--restore-only` to send
  corrections.
- `--check` prints the environment without changing it.

## Limitations

- Plaintext HTTP only. HTTPS bodies are not parsed.
- IPv4 only.
- WSL2 mirrored networking does not forward other hosts' traffic into the VM.
- Windows hosts with static ARP entries are not captured.
- Routers that allow a single admin session may reject a replay login while
  the victim holds a session. `--auto` avoids that by taking over the IP.
- The IP takeover can briefly conflict with the victim device.

## Troubleshooting

- `no IPv4 address on <iface>`: wrong interface name. Check `ip -br addr`.
- `ARP resolve failed`: wrong gateway, subnet, or interface.
- LAN loses connectivity: forwarding was blocked. Stop, then run `--check`.
- Device offline after a stop: run `--restore-only`, or reconnect its network.
- No credentials: the victim is not logging in over plain HTTP.
- No leak: the session was rejected and replay was unavailable. Try `--auto`.

## Repository layout

- `exploit.py`: the harvester
- `requirements.txt`: Python dependencies
- `README.md`: this file

## Note

developed with deepseek 4.1 flash

## Legal

This project is for authorized security testing. Using it against networks
without permission is illegal.
