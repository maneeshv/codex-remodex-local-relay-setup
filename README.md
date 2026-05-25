# Remodex Local Relay Setup

This repo is a reproducible setup guide for running [Remodex](https://github.com/Emanuele-web04/remodex) with a self-hosted relay on macOS over Tailscale.

The intended workflow is:

1. Clone this setup repo on a Mac.
2. Ask a coding agent to read and execute [spec.md](spec.md).
3. Pair the existing Remodex iOS app with the QR code printed by `remodex up`.

The iOS app does not need to be built from source for this setup.

## What This Sets Up

- A local Remodex relay cloned from the upstream Remodex repo.
- A macOS user LaunchAgent named `com.remodex.relay` so the relay runs without an open terminal.
- The npm-installed `remodex` bridge CLI.
- The Remodex bridge connected to the local relay through the Mac's Tailscale IP.

The relay and bridge are separate:

- **Relay:** transport server, managed by this repo's LaunchAgent template.
- **Bridge:** local Codex/git integration, managed by Remodex's own `remodex up` flow.

## For Coding Agents

Treat [spec.md](spec.md) as the source of truth. Execute it from the root of this cloned setup repo.

Important implementation details:

- The setup repo may be cloned anywhere.
- `spec.md` computes the setup path with `pwd -P`.
- The upstream Remodex repo is cloned into `./remodex`.
- `./remodex` is ignored by Git and is generated during setup.
- The LaunchAgent template is [launchd/com.remodex.relay.plist](launchd/com.remodex.relay.plist).
- The plist contains a `__REMODEX_RELAY_DIR__` placeholder that must be replaced with the actual absolute `remodex/relay` path before installing into `~/Library/LaunchAgents`.
- Do not install the template plist directly without replacing the placeholder.

The expected installed plist path is:

```sh
~/Library/LaunchAgents/com.remodex.relay.plist
```

The expected verification commands are in [spec.md](spec.md), but the final state should show:

```text
launchctl print gui/$(id -u)/com.remodex.relay -> state = running
curl http://127.0.0.1:9000/health -> {"ok":true}
curl http://<tailscale-ip>:9000/health -> {"ok":true}
remodex status -> Bridge state: running, Connection: connected
```

## Human Prerequisites

Before asking an agent to execute the spec, make sure the Mac has:

- Tailscale installed and connected.
- Node.js and npm installed.
- Codex CLI installed and authenticated.
- The Remodex iOS app installed on an iPhone connected to the same Tailscale tailnet.

Then ask the agent something like:

```text
Read README.md and spec.md in this repo, then execute spec.md to set up Remodex with a self-hosted local relay over Tailscale. Verify the relay and bridge status before reporting completion.
```

## Files

- [spec.md](spec.md): full executable setup spec.
- [launchd/com.remodex.relay.plist](launchd/com.remodex.relay.plist): LaunchAgent template.
- `.gitignore`: excludes generated local setup directories such as `remodex/`.
