# WireGuard

*Using WireGuard VPN tunnels on macOS with `wg-quick` and `.conf` files.*

WireGuard is a fast, modern and secure VPN tunnel. On macOS you can drive
tunnels either with the official **WireGuard** app (App Store / menu bar) or
from the terminal with the **`wireguard-tools`** CLI (`wg` / `wg-quick`). This
note focuses on the CLI workflow with named `.conf` files.

## Install

```shell
# CLI tools: wg + wg-quick
brew install wireguard-tools

# GUI app (menu bar) — optional
brew install --cask wireguard

# Verify
wg --version
```

> The kernel module does not exist on macOS; the tunnel runs in userspace
> through `wireguard-go`, which Homebrew installs as a dependency.

---

## Configuration Files

`wg-quick` resolves a tunnel **by name** when you pass a bare interface name
instead of a path. On Apple Silicon the config directory is:

```shell
# Apple Silicon (Homebrew under /opt/homebrew)
/opt/homebrew/etc/wireguard/

# Intel (Homebrew under /usr/local)
/usr/local/etc/wireguard/
```

Create the directory and drop your `.conf` files in it. The filename (without
`.conf`) becomes the tunnel name and must be a valid interface name
(≤ 15 chars, `[a-zA-Z0-9_=+.-]`).

```shell
sudo mkdir -p /opt/homebrew/etc/wireguard
sudo install -m 600 ~/Downloads/home.conf /opt/homebrew/etc/wireguard/home.conf
```

> Config files contain a private key — keep them `chmod 600` and owned by
> `root`. `wg-quick` warns if a config is world-readable.

### Config Format

A client `.conf` looks like this:

```ini
[Interface]
# Client private key (keep secret)
PrivateKey = <CLIENT_PRIVATE_KEY>
# IP(s) assigned to this client inside the tunnel
Address = 10.0.0.2/32
# DNS servers used while the tunnel is up
DNS = 1.1.1.1, 1.0.0.1

[Peer]
# Server public key
PublicKey = <SERVER_PUBLIC_KEY>
# Optional extra layer of symmetric encryption
PresharedKey = <PRESHARED_KEY>
# Server address:port
Endpoint = vpn.example.com:51820
# Which traffic goes through the tunnel:
#   0.0.0.0/0, ::/0  -> full tunnel (all traffic)
#   10.0.0.0/24      -> split tunnel (only the VPN subnet)
AllowedIPs = 0.0.0.0/0, ::/0
# Keep NAT mappings alive (useful behind NAT/firewalls)
PersistentKeepalive = 25
```

---

## Generate Keys

If you need to create a client keypair yourself:

```shell
# Private + public key
wg genkey | tee privatekey | wg pubkey > publickey

# With an extra preshared key
wg genpsk > presharedkey
```

---

## Bring Tunnels Up and Down

```shell
# Start a tunnel by name (reads /opt/homebrew/etc/wireguard/home.conf)
sudo wg-quick up home

# Stop it
sudo wg-quick down home

# Or pass an explicit path (any location)
sudo wg-quick up ~/vpn/home.conf
sudo wg-quick down ~/vpn/home.conf
```

`wg-quick up` creates a `utunN` interface, applies the addresses, routes and
DNS from the config, then tears everything down again on `down`.

---

## Status and Inspection

```shell
# Show all active tunnels (keys, endpoints, transfer, last handshake)
sudo wg show

# Show a single tunnel
sudo wg show home

# List active WireGuard interfaces
sudo wg show interfaces

# Machine-readable dump (tab separated)
sudo wg show home dump
```

A recent `latest handshake` and growing `transfer` counters confirm the tunnel
is working.

> On macOS the kernel interface is named `utunN` (e.g. `utun5`), but `wg-quick`
> keeps a mapping so you keep referring to the tunnel by its config name. Use
> `wg show <name>` (not `wg show utun5`) — the mapping lives in
> `/var/run/wireguard/<name>.name`.

---

## Troubleshooting

```shell
# "wg-quick: `home' already exists as `utun5'"
#   -> the tunnel is already up; bring it down before starting it again
sudo wg-quick down home && sudo wg-quick up home

# "Unable to access interface: No such file or directory"
#   -> wg show was given an empty/unknown interface name.
#      Run it without arguments to list everything:
sudo wg show
```

---

## Tips

- **Full vs split tunnel** — set `AllowedIPs = 0.0.0.0/0, ::/0` to route *all*
  traffic through the VPN, or list specific subnets to only tunnel those.
- **DNS leaks** — when full-tunneling, set `DNS` in `[Interface]` so lookups go
  through the VPN.
- **Multiple tunnels** — keep one `.conf` per tunnel (e.g. `home.conf`,
  `work.conf`) and switch with `wg-quick up <name>` / `down <name>`.
- **Permissions** — `wg-quick up/down` and `wg show` need `sudo` on macOS.
- **GUI + CLI** — avoid running the same tunnel from both the app and
  `wg-quick` at once; they manage the interface independently.

---

## References

- [WireGuard — Quick Start](https://www.wireguard.com/quickstart/)
- [`wg(8)` man page](https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8)
- [`wg-quick(8)` man page](https://git.zx2c4.com/wireguard-tools/about/src/man/wg-quick.8)
