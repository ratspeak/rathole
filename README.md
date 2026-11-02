<div align="center">

# RATHOLE

**Security suite + transport node for [Reticulum](https://reticulum.network)**

*Run a public gateway from home with I2P*

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](LICENSE)
[![Python 3.11+](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://python.org)
[![Ratspeak](https://img.shields.io/badge/Ratspeak-ratspeak.org-purple.svg)](https://ratspeak.org)

</div>

<div align="center">
<table>
<tr>
<td align="center"><b>Overview</b><br><br><img src="docs/screenshots/overview.svg" alt="Historical overview dashboard capture" width="480"></td>
<td align="center"><b>Peers</b><br><br><img src="docs/screenshots/peers.svg" alt="Historical peers dashboard capture" width="480"></td>
</tr>
<tr><td colspan="2"></td></tr>
<tr>
<td align="center"><b>Interfaces</b><br><br><img src="docs/screenshots/interfaces.svg" alt="Historical interfaces dashboard capture" width="480"></td>
<td align="center"><b>Blackhole</b><br><br><img src="docs/screenshots/blackhole.svg" alt="Historical blackhole dashboard capture" width="480"></td>
</tr>
</table>
</div>

These dashboard captures illustrate the interface. Their addresses, counters and alerts are historical; an alert label is not a verified attack report.

Rathole runs Python Reticulum with configurable packet filters, a terminal dashboard and optional I2P peering. It is a proof of concept: the filters inspect traffic before normal RNS validation, and several monitoring features are approximate or incomplete. Review the limits below before relying on it for a public gateway.

## Install

Install Python 3.11 or newer and Git. On Debian/Ubuntu, include `python3-venv`; on macOS, Homebrew's `python3` package supplies Python. From a terminal:

```bash
git clone https://github.com/ratspeak/rathole.git
cd rathole
python3 -m venv .venv
. .venv/bin/activate
python -m pip install -e ".[all]"
```

On Windows, create the environment with `py -3 -m venv .venv` and activate it in PowerShell with `.venv\Scripts\Activate.ps1`, then run the same pip command.

Rathole hooks Python RNS internals. Its dependency declaration (`rns>=0.7.0`) is not a tested compatibility range. Record the RNS version in your working environment with `python -m pip show rns` and check compatibility before upgrading it.

## Quick Start

```bash
rat setup
rat run
```

The wizard asks for a Reticulum config directory, network interfaces, a gateway/client preset and optional I2P and registry settings. It writes `rathole.toml` in the current directory and offers to start the daemon. If you accept that offer, run `rat run` only after exiting it.

Back up an existing Reticulum configuration first: setup can change its interfaces and transport setting. Rathole also attempts to enable transport at startup, including in its **client** mode. Run it with its own Reticulum instance; attaching to another shared daemon does not put Rathole's hooks into that daemon.

Start with `observe` (gateway) or `relaxed` (client) to review filter verdicts in dry-run mode. Dry-run still starts networking, records state and may publish registry announces. Existing blackhole rules are checked before the dry-run filter override, so this is not a guarantee that all packets pass.

Use `rat status` in another activated terminal to inspect the running daemon. If you chose a nondefault control socket, supply it with `rat --socket /path/to/socket status`. Exit the dashboard or use `rat shutdown` to stop it.

`rat reset` is destructive: it can remove the data directory, config and, if selected, the Reticulum directory and identity. Read the displayed paths and keep a separate backup before confirming.

## How It Works

Rathole wraps `Transport.inbound()` and, when available, `Transport.path_request_handler()` in the Python RNS process it starts. A blackhole check precedes the global packet-rate, byte-rate and size filters. Packets then enter an announce, path-request, link-request or data pipeline.

The first blocking verdict stops processing. A `THROTTLE` verdict currently drops the packet; it does not queue it for later delivery. If filtering raises an exception, the hook attempts to pass the packet to the original RNS handler. This fallback does not guarantee continued service or delivery.

Header-derived hashes are unverified at this stage. IFAC packets skip destination-hash extraction, and announce-size estimates are heuristic. Treat alerts and reputation scores as observations to investigate, not authenticated evidence that a peer misbehaved.

## I2P Gateway Mode

The setup wizard can help install/start `i2pd`, configure its SAM interface and add an RNS I2P interface. Installation support depends on the host's package manager; follow the wizard's manual instructions if automatic setup is unavailable. Wait for the I2P interface to have a B32 address before publishing it.

I2P peering avoids a direct TCP connection between the two Reticulum peers. It does not make every connection Rathole uses anonymous. A TCP bootstrap connection such as `rns.ratspeak.org:4242` exposes your source IP to that peer; the default HTTPS registry discovery request exposes it to the registry endpoint.

Registry publishing sends a signed `rathole.gateway` announce containing the I2P address, optional node name and capabilities, version and node mode. The announce must reach the registry's Reticulum node. Discovery queries the registry over HTTPS; automatic connections require `auto_connect` and are suppressed in dry-run mode.

The client defaults are a five-minute announce interval and ten-minute discovery interval. Expiry is a server policy, not a client guarantee. To stop automatic publication, disable `registry.publish` in the file and reload. `rat registry deregister` alone clears the current publication state but does not disable future scheduled announces or immediately delete a server record.

## Filters and Their Limits

| # | Filter | Pipeline | Check | Base setting |
|---|-------|----------|-----------------|---------|
| G1 | Interface Rate Limit | Global | Packets per interface | 10.0 pkts/s |
| G2 | Bandwidth Cap | Global | Bytes per interface | 500 KB/s |
| G3 | Packet Size | Global | Oversized packets | 600 bytes max |
| A1 | Allow/Deny Lists | Announce | Hash allow/deny match | Empty lists |
| A2 | Hop Ceiling | Announce | Header hop count | 32 hops max |
| A3 | Announce Size | Announce | Estimated app-data size | 500 bytes max |
| A4 | Rate Limiter | Announce | Per-destination announce rate | 15 burst, 0.5/s refill |
| A5 | Churn Dampening | Announce | Repeated announces per destination | Off by default |
| A6 | Anomaly Detector | Announce | High announce:traffic ratio | 100.0 max ratio |
| P1 | Path Request Filter | Path | Request count and unique destinations | 30/min |
| L1 | Link Request Filter | Link | Request rate and approximate pending count | 10 burst, 50 count limit |
| D1 | Resource Guard | Data | Incomplete resource inspection | 16 MiB configured limit |

These are base settings; presets and your file can override them. Allow-list matches only bypass the deny check in that filter, not the rest of the pipeline. Link/resource counters decay periodically rather than tracking completion exactly. The raw hook does not populate resource subtypes, and the resource guard compares raw packet length rather than decoded transfer size: **do not rely on it to enforce a transfer-size limit or prevent decompression bombs**.

Filters can be toggled and tuned live:

```bash
rat filters toggle rate_limit on
rat filters set rate_limit burst 20
```

## Presets

These describe the preset values before the setup wizard applies your choices.

| Preset | Mode | Philosophy |
|--------|------|-----------|
| **observe** | Gateway | Dry-run, generous limits, 24h learning, publishes to registry |
| **balanced** | Gateway | Active filtering, moderate limits, baseline monitoring, publishes to registry |
| **fortress** | Gateway | Auto-blackhole, tight limits, additional correlator logging |
| **relaxed** | Client | Dry-run, generous limits, discovery enabled; auto-connect waits until dry-run is off |
| **standard** | Client | Active filtering, baseline monitoring, auto-connects to gateways |
| **strict** | Client | Tight rate limits, active monitoring, manual peering only |
| **lora** | Either | Ultra-tight bandwidth for LoRa gateways, registry disabled |

Inspect a preset with `rat config preset --preset-action diff --preset-name balanced` and apply it with `rat config preset --preset-action apply --preset-name balanced`. Applying a preset replaces the in-memory configuration, including general and registry settings. Review the result with `rat config show`.

Live changes are not written back to the TOML file. Edit the file to keep them across reloads and restarts; settings that create listeners or select data paths can require a restart.

## Intelligence

**Reputation** assigns scores between 0 and 1 to tracked hashes. Base rewards are 0.005 per accepted verdict, with a 0.015 drop penalty for announces. Rate-limit bucket sizes do not currently use those scores. Raw announce attribution happens before signature validation, so scores can be misleading; auto-blackholing is off in the base configuration.

**Adaptive monitoring** collects interface counters and computes rolling statistics. The current filter implementations do not consume its calculated thresholds, so enabling it does not automatically tune enforcement.

**Correlation** checks for traffic patterns at a default 30-second interval. Its current defensive response methods log findings and recommendations; they do not tighten filter thresholds automatically.

## CLI Reference

```text
# Setup
rat setup                               # interactive wizard
rat run                                 # start daemon + TUI (shortcut)

# Monitoring
rat status                              # health bar + stats
rat peers [--sort F] [--limit N]        # peer table with reputation
rat interfaces                          # per-interface breakdown
rat events [--severity S] [--type T]    # security event log
rat correlator                          # attack correlation status
rat alerts                              # alert rules status
rat adaptive                            # adaptive threshold status

# Filter management
rat filters                             # all filters by pipeline
rat filters toggle rate_limit on        # enable/disable
rat filters set rate_limit burst 20     # change parameter

# Reputation
rat reputation                          # all identities

# Blackhole
rat blackhole list                      # view blackholed identities
rat blackhole add -i <hash> -r "reason"
rat blackhole remove -i <hash>

# Config
rat config show [section]               # view config
rat config set reputation accept_reward 0.01
rat config preset --preset-action list  # list presets
rat config preset --preset-action apply --preset-name balanced
rat dry-run [on|off]                    # toggle dry-run
rat reload                              # hot-reload config

# Network
rat network add server 0.0.0.0:4242     # add TCP server interface
rat network add client host:port        # add TCP client interface
rat network remove <name>               # remove interface

# Registry
rat registry                            # registry status
rat registry discover                   # show discovered gateways
rat registry connect <hash>             # add a gateway interface
rat registry register                   # queue a gateway announce
rat registry deregister                 # clear current publication state (see above)

# Lifecycle
rat shutdown                            # stop the daemon
rat reset                               # full reset (config + state)

# Output
rat --json peers                        # JSON output for scripting
```

The current `reputation` parser does not reliably pass identity arguments to `pin` or `unpin`, and `rat reputation <hash>` is not accepted. Use `rat reputation` to inspect the full table; the per-identity CLI forms need a code fix.

## TUI Dashboard

`rathole -c rathole.toml` launches a full-screen terminal dashboard. It shows traffic, filter and registry state, with controls for live configuration.

Run standalone against a headless daemon: `rathole-tui`

## Deployment

### Linux (systemd)

[`deploy/rathole.service`](deploy/rathole.service) is a template, not a complete installation. Before enabling it:

1. Create the `rathole` service user/group and a writable `/var/lib/rathole` directory.
2. Install Rathole in a virtual environment readable by that user. Set `ExecStart` to its actual `bin/rathole` path; the template's `/usr/bin/rathole` is only a placeholder.
3. Create `/etc/rathole/rathole.toml`. Set `general.reticulum_config_path` to `/var/lib/rathole/reticulum`, `general.state_file` to `/var/lib/rathole/state.json`, `general.control_socket` to `/var/lib/rathole/control.sock` and `eventstore.db_path` to `/var/lib/rathole/events.db`.
4. Configure the dedicated Reticulum directory and ensure the service user can read/write it. The unit's `ProtectHome` prevents using a personal home directory. Its `PrivateTmp` also makes `/tmp/rathole.sock` unsuitable for host CLI access.
5. Copy the adjusted unit to `/etc/systemd/system/rathole.service`, run `sudo systemctl daemon-reload`, then `sudo systemctl enable --now rathole`. Inspect `sudo journalctl -u rathole` and query it as the service user with the explicit control-socket path.

Keep control-socket access restricted to administrators. Do not run setup or reset against a shared production Reticulum directory.

### Docker

The compose template uses host networking and expects a real `deploy/rathole.toml` file, which is not supplied. Copy `rathole.example.toml` there, then set the same `/var/lib/rathole` paths listed above. Set `general.control_socket` to `/var/lib/rathole/control.sock`. Configure the dedicated Reticulum directory in the persisted state volume before using it as a gateway.

Build and start from `deploy/` with `docker compose up -d --build`. The image runs as the `rathole` user; the state volume must be writable by it. Host networking is a deployment requirement to evaluate on your Docker host, not a promise of identical behavior on every desktop platform. Metrics are disabled by default and bind to loopback when enabled; publishing port 9777 does not itself enable them.

## Configuration

See [`rathole.example.toml`](rathole.example.toml) for the commented configuration. `rat config show` reports the daemon's in-memory settings. Live filter changes and presets are useful for tuning, but persist the desired values in the file before a restart. Keep snapshots of known-working configurations and dependency versions.

## License

[GPL-3.0-or-later](LICENSE) · [Ratspeak](https://ratspeak.org)
