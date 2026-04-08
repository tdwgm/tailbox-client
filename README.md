# Tailbox Client

![Tailbox Architecture](images/header.png)

> **Work in progress.** Functional and in daily use, but still being improved. A rewrite of the wrapper script in Go is planned.

A self-contained Bash script that runs Tailscale in a rootless Podman container, exposes a SOCKS5 proxy on `localhost:1055`, and forwards all traffic to a SOCKS5 exit node on your tailnet (typically the [Tailbox Server](https://github.com/tdwgm/tailbox-server) running behind Mullvad VPN).

No system Tailscale installation required. State persists across restarts; authenticate once.

## Features

- **Kill switch** -- iptables rules inside the container drop all non-Tailscale traffic on `eth0`; if Tailscale disconnects, traffic stops rather than falling back to the host's plain network
- **Image digest pinning** -- the upstream Tailscale image is pinned by `@sha256:` digest at install time; updates require an explicit `tailbox update` and a user prompt
- **One-command install** -- `tailbox install` copies the script, pins and builds the container image, configures proxychains, and sets up shell aliases
- **Zero DNS leaks** -- DNS queries from proxied applications resolve through the exit node's network, not the host
- **Rootless auto-detection** -- uses `podman` if rootless works, falls back to `sudo podman` otherwise
- **Persistent state** -- Tailscale state is saved in `~/tailscale-container/state/`; subsequent starts connect instantly without re-authenticating

## Install

**One-liner (from GitHub Releases):**

```bash
curl -fsSL https://github.com/tdwgm/tailbox-client/releases/latest/download/tailbox \
    -o ~/.local/bin/tailbox && chmod +x ~/.local/bin/tailbox
```

**Manual:**

```bash
cp tailbox ~/.local/bin/tailbox
chmod +x ~/.local/bin/tailbox
```

## Quick Start

```bash
# 1. Start -- runs the install wizard automatically on first use
tailbox

# 2. Done -- SOCKS5 proxy is live on localhost:1055
curl --socks5-hostname localhost:1055 https://am.i.mullvad.net/connected
```

On the first run (when no `tailbox.conf` exists), the install wizard runs automatically: it pins and builds the container image, configures proxychains, sets up shell aliases, and prompts for exit node details. A login URL opens in your browser to connect the node to your tailnet.

After the first authentication, Tailscale state is saved in `~/tailscale-container/state/`. Subsequent starts connect instantly without re-authenticating.

## Commands

| Command | Description |
|---------|-------------|
| `tailbox` / `tailbox start` | Start the container, apply kill switch, start SOCKS forward (runs install wizard on first use) |
| `tailbox stop` | Remove the running container |
| `tailbox status` | Show container state, proxy status, Mullvad connectivity check |
| `tailbox exec <cmd>` | Run a `tailscale` subcommand inside the container (e.g. `tailbox exec status`) |
| `tailbox auth [key]` | Save a Tailscale auth key to `~/tailscale-container/authkey` (hidden prompt if no arg) |
| `tailbox forward` | Restart the socat forward (useful if the exit node rebooted) |
| `tailbox kill-switch` | Re-apply iptables kill switch rules inside the container |
| `tailbox update` | Fetch the current upstream digest and rebuild the local image |
| `tailbox update-check` | Query the registry for a newer digest and prompt before applying |
| `tailbox logs` | Follow container logs (`podman logs -f`) |
| `tailbox build` | Rebuild local image from pinned digest (fetches if missing) |
| `tailbox install` | Full installation: copy script, pin image, configure proxychains and shell aliases |
| `tailbox info` | Detailed configuration, image, and authentication reference |

## Browser Setup

### Firefox -- SOCKS Proxy (all traffic)

Route all Firefox traffic through the proxy:

1. Open **Settings** -> **Network Settings** -> **Manual proxy configuration**
2. Set **SOCKS Host**: `localhost`, **Port**: `1055`
3. Select **SOCKS v5**
4. Check **"Proxy DNS when using SOCKS v5"** -- this is required to prevent DNS leaks

Verify: navigate to [am.i.mullvad.net](https://am.i.mullvad.net) and confirm you see a Mullvad IP.

### Firefox -- Container Tabs (per-tab proxy)

Route only selected tabs through the proxy while keeping other tabs on your normal connection. Two extension options:

**Option A: [Multi-Account Containers](https://github.com/mozilla/multi-account-containers) + [Container Proxy](https://addons.mozilla.org/firefox/addon/container-proxy/)**

1. In Multi-Account Containers, create a new container named **Mullvad**
2. In Container Proxy settings, assign the **Mullvad** container to:
   - Proxy type: `SOCKS5`
   - Host: `localhost`
   - Port: `1055`
   - Enable **Proxy DNS**
3. Open any site in the Mullvad container -- it routes through Tailbox; all other containers use your normal connection

**Option B: [Containerise](https://github.com/kintesh/containerise)**

1. Add a rule mapping domains to a container with a SOCKS5 proxy
2. Matching URLs automatically open in the proxied container

Both approaches give you per-tab control over which traffic goes through the VPN.

### Chromium / Chrome

```bash
chromium \
    --proxy-server="socks5://localhost:1055" \
    --host-resolver-rules="MAP * ~NOTFOUND , EXCLUDE localhost"
```

The `--host-resolver-rules` flag forces DNS resolution through the proxy, preventing DNS leaks.

### System-Wide (proxychains)

`tailbox install` creates `~/.proxychains-tailscale.conf` automatically. Use it to tunnel any TCP application:

**Config** (`~/.proxychains-tailscale.conf`):

```
strict_chain
proxy_dns
tcp_read_time_out 15000
tcp_connect_time_out 8000

[ProxyList]
socks5 127.0.0.1 1055
```

**Usage:**

```bash
# SSH through the proxy
proxychains4 -q -f ~/.proxychains-tailscale.conf ssh user@host

# Any other TCP command
proxychains4 -q -f ~/.proxychains-tailscale.conf curl https://ifconfig.me

# Or use the shell aliases installed by `tailbox install` (default: `tb` / `ssht`)
tb curl https://ifconfig.me
ssht user@host
```

## Configuration

Edit `~/tailscale-container/tailbox.conf` to customize. Changes take effect on the next `tailbox stop && tailbox`. If you change `SSH_ALIAS` or `PROXY_ALIAS`, the shell functions in your rc file (`.zshrc` / `.bashrc`) are updated automatically on the next start.

| Variable | Default | Description |
|----------|---------|-------------|
| `SOCKS_PORT` | `1055` | Local port for the SOCKS5 proxy |
| `HTTP_PORT` | `1056` | Local port for the Tailscale HTTP proxy |
| `EXIT_NODE` | `tailbox-endpoint` | Tailscale hostname of the SOCKS5 exit node |
| `EXIT_NODE_PORT` | `1080` | SOCKS5 port on the exit node |
| `SSH_ALIAS` | `ssht` | Shell function name for SSH via proxy |
| `PROXY_ALIAS` | `tb` | Shell function name for any command via proxy |
| `UPDATE_CHECK_INTERVAL` | `86400` | Seconds between automatic update checks (24 h) |

`TS_AUTHKEY` and `TS_EXTRA_ARGS` can also be passed as environment variables.

## How It Works

### Socat Forwarding Chain

The client-side SOCKS5 proxy works via a multi-hop forwarding chain:

```
1. Application
   SOCKS5 connect to localhost:1055

2. Podman port mapping
   Host localhost:1055 -> container port 1081
   (-p 127.0.0.1:1055:1081)

3. socat (inside tailbox container)
   TCP4-LISTEN:1081,fork,reuseaddr
   EXEC:tailscale nc <exit-node-hostname> 1080

4. tailscale nc
   Opens a TCP-over-Tailscale stream to the named peer.
   Works in both kernel mode (tailscale0 tun device) and
   userspace mode (TS_USERSPACE=true), because tailscale nc
   uses the Tailscale daemon's own TCP proxy, not raw tun.
   Traversal via DERP relay or direct WireGuard UDP.

5. Exit node: SOCKS5 proxy (e.g. microsocks)
   Receives the proxied stream on :1080 and forwards to
   the destination via the exit node's default route
   (e.g. a Mullvad VPN tunnel).
```

The key design choice is `tailscale nc` for the forward leg: it opens a TCP stream to the exit node through Tailscale's userspace networking stack, so it works regardless of whether the container has a kernel `tailscale0` interface or is running in userspace mode.

Before starting socat, tailbox verifies exit node reachability using a TCP probe (`tailscale nc` to the SOCKS port) rather than `tailscale ping`. The disco protocol used by `tailscale ping` traverses DERP relays and can succeed even when the actual TCP path is broken, giving false positives. The TCP probe tests exactly what socat will use.

`socat` is baked into the local image at install time (not downloaded at container start), so the image is fully self-contained after `tailbox install`.

### Kill Switch

After the container connects to Tailscale, `iptables` rules are applied inside the container to drop all `eth0` (Podman bridge) traffic except what Tailscale itself needs. If Tailscale loses its connection, traffic stops rather than routing through the host's plain network.

| Chain | Interface | Match | Action |
|-------|-----------|-------|--------|
| OUTPUT | lo | any | ACCEPT |
| OUTPUT | tailscale0 / tun0 | any | ACCEPT |
| OUTPUT | eth0 | ESTABLISHED,RELATED | ACCEPT |
| OUTPUT | eth0 | udp dport 41641 | ACCEPT (WireGuard direct) |
| OUTPUT | eth0 | udp dport 3478 | ACCEPT (STUN) |
| OUTPUT | eth0 | tcp dport 443 | ACCEPT (control plane + DERP) |
| OUTPUT | eth0 | tcp dport 80 | ACCEPT (DERP probe) |
| OUTPUT | eth0 | udp/tcp dport 53 | ACCEPT (DNS for control plane) |
| OUTPUT | eth0 | any | **DROP** |
| INPUT | eth0 | ESTABLISHED,RELATED | ACCEPT |
| INPUT | eth0 | tcp dport 1081 | ACCEPT (socat from host) |
| INPUT | eth0 | tcp dport 8080 | ACCEPT (HTTP proxy from host) |
| INPUT | eth0 | any | **DROP** |

The `iptable_filter` kernel module must be loaded before the container starts. Tailbox loads it automatically via `sudo modprobe iptable_filter` when running in non-rootless mode. In rootless mode, the kill switch is skipped (Podman rootless containers do not have the capability to load kernel modules or set iptables rules by default).

## Requirements

| Package | Purpose | Install |
|---------|---------|---------|
| `podman` | Container runtime | `sudo dnf install podman` / `sudo apt install podman` |
| `proxychains-ng` | Route arbitrary TCP through the proxy | `sudo dnf install proxychains-ng` / `sudo apt install proxychains4` |

proxychains-ng is optional -- you can use `--socks5-hostname localhost:1055` directly with curl, or configure your browser as described above.

## Server

This is the client half of Tailbox. For the server component (Gluetun + Tailscale + microsocks running behind Mullvad VPN), see [tailbox-server](https://github.com/tdwgm/tailbox-server).

## License

MIT -- see [LICENSE](LICENSE).
