# Tailbox Client

![Tailbox Architecture](images/header.png)

> **Work in progress.** Functional and in daily use, but still being improved. A rewrite of the wrapper script in Go is planned.

A self-contained Bash script that runs Tailscale in a **rootless** Podman container, exposes a SOCKS5 proxy on `localhost:1055`, and forwards all traffic to a SOCKS5 exit node on your tailnet (typically the [Tailbox Server](https://github.com/tdwgm/tailbox-server) running behind Mullvad VPN).

No system Tailscale installation required. State persists across restarts; authenticate once.

## Features

- **Rootless by default** -- runs under unprivileged `podman` with no sudo, no root-owned state, no capability grants. Rootful mode exists as an opt-in for the in-container kill switch but is discouraged; see *Rootful mode* below
- **Image digest pinning** -- the upstream Tailscale image is pinned by `@sha256:` digest at install time; updates require an explicit `tailbox update` and a user prompt
- **One-command install** -- `tailbox install` copies the script, pins and builds the container image, configures proxychains, and sets up shell aliases
- **Zero DNS leaks** -- DNS queries from proxied applications resolve through the exit node's network, not the host
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

### Kill Switch (rootful mode only)

In **rootful mode only** (`ROOTLESS=false` in `tailbox.conf`), after the container connects to Tailscale, `iptables` rules are applied inside the container to drop all `eth0` (Podman bridge) traffic except what Tailscale itself needs. If Tailscale loses its connection, traffic stops rather than routing through the host's plain network.

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

The `iptable_filter` kernel module must be loaded before the container starts. Tailbox loads it automatically via `sudo modprobe iptable_filter` when running in rootful mode.

### Why Rootless Mode Does Not Leak

The most common pushback on running tailbox rootless is "but then there is no kill switch, so what happens when Tailscale drops?". The answer is that the proxy chain is fail-closed by construction, not by firewall rule, so there is nothing to leak in the first place.

Look again at the path from an application to the internet, which is the same chain described in "Socat Forwarding Chain" above:

```text
app -> localhost:1055 -> podman port-map -> container:1081 -> socat -> EXEC:tailscale nc -> tailscaled -> exit node
```

There is no IP-level forwarding anywhere on this path. No `default via tailscale0`, no route injection on the host, nothing that binds an application socket to the tunnel at the kernel level. Each incoming SOCKS connection spawns a `tailscale nc` subprocess that opens a TCP stream through the running tailscaled daemon. If tailscaled is stopped, disconnected, crashed, or its unix socket has gone away, `tailscale nc` fails immediately, socat closes the client connection, and the application gets `ECONNRESET`. There is no alternate path for the traffic to take.

Compare this to a conventional VPN kill switch scenario. A host running WireGuard or OpenVPN typically has `default via <vpn-gw>` pointing at `tun0`. When the VPN drops, the default route reverts to the physical gateway and traffic silently flows via the ISP. That is exactly when an iptables kill switch is load-bearing: it is a safety net for a failure mode where the kernel will happily forward traffic the wrong way. The tailbox client has no such failure mode. The SOCKS port on `localhost:1055` is the only way in, and the socat-plus-`tailscale nc` pair cannot forward anything when tailscaled is not reachable.

### Workarounds for Rootless iptables

If you still want a packet-level kill switch in rootless mode, the Podman capability limitation is not absolute. A few options that actually work:

- **`podman unshare --rootless-netns`**. Enter the rootless container's network namespace from the host and apply rules inside it, for example `podman unshare --rootless-netns iptables -A OUTPUT -o eth0 -j DROP`. Works, but the rootless netns is recreated on every container start, so you need a wrapper script to reapply the rules after each start.
- **`--cap-add=NET_ADMIN` with pasta networking**. On Podman 4.4 and newer the pasta networking backend grants capabilities inside the user namespace in a way that can let iptables modify the container's own netfilter tables. Needs the relevant kernel modules loaded on the host and iptables userspace present in the container image.
- **A sentinel without iptables at all**. A systemd user service (or a small supervisor inside the container) that polls `tailscale status` and kills socat the moment the daemon becomes unreachable. Reactive rather than preventive, but it reaches the same end state: no proxy, no traffic.

None of these are enabled by default, because per the section above they solve a problem that does not actually exist on this client. They are listed here for anyone who needs belt-and-suspenders hardening against a compromised `tailscaled` binary (see the threat model in "Rootful mode" below); for that specific case they are viable alternatives to running the whole client as root.

### Rootful mode (discouraged)

Rootless is the default and is the recommended way to run Tailbox. Rootful mode (`ROOTLESS=false` in `~/tailscale-container/tailbox.conf`) exists only to enable the in-container kill switch above. Before opting in, be aware of the trade-off.

**What rootful gives you that rootless does not:**

- The iptables kill switch inside the container (the table above).
- Kernel-mode `tailscale0` tun device instead of Tailscale's userspace networking. This makes **no functional difference** for the SOCKS proxy, because `socat EXEC:tailscale nc` uses Tailscale's internal TCP proxy in both modes.

**What the kill switch actually protects against:**

- A compromised `tailscaled` binary making outbound connections to infrastructure other than Tailscale's control plane or DERP relays. Because the upstream image is pinned by `@sha256:` digest, this is a narrow threat (supply-chain compromise of an already-pinned digest).
- It does **not** prevent SOCKS traffic leaks if Tailscale drops, because `socat EXEC:tailscale nc` cannot forward anything when `tailscale nc` exec fails. The proxy simply stops working, which is the safe behaviour you want.
- It does **not** prevent DNS leaks, because DNS (udp/tcp 53) is whitelisted in the kill switch anyway.

**What rootful costs you:**

- A sudo password (or cached credential) on every `tailbox start`.
- State directory owned by root; migrating back to rootless requires fixing ownership (the script does this automatically when it notices).
- Alias liveness checks and any other user-space tooling that talks to `podman` must route through `sudo podman` too. Easy to get wrong; the old `podman ps`-based alias check shipped broken in rootful mode for exactly this reason.
- You can no longer run `tailbox` as the regular user who installed it without re-typing the sudo password.

**Recommendation:** leave `ROOTLESS` unset (or explicitly `true`) and accept that the client-side kill switch is not applied. The default is rootless precisely because rootful's gain (a narrow defence against a pinned-then-compromised binary) is smaller than its ongoing friction cost for a tool whose entire purpose is being lightweight and ephemeral.

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
