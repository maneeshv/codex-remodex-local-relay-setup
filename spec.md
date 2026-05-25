# Remodex Self-Hosted Relay on macOS

## Goal

Run Remodex using a self-hosted relay on a Mac reachable over Tailscale, while using:

- The existing Remodex iOS app from the App Store.
- The npm-installed `remodex` bridge CLI on the Mac.
- A cloned Remodex source checkout only for the relay server.
- A macOS user LaunchAgent so the relay restarts after login and does not require an open terminal.

The iOS app does not need to be built from source for this setup.

## Architecture

There are two separate long-running pieces:

1. **Relay**
   - A Node server from the Remodex repo at `remodex/relay/server.js`.
   - Listens on `0.0.0.0:9000`.
   - Exposed to the iPhone through the Mac's Tailscale IP.
   - Managed by a custom user LaunchAgent: `com.remodex.relay`.

2. **Bridge**
   - Installed by `npm install -g remodex@latest`.
   - Started with `remodex up`.
   - Talks to Codex CLI and local repos on the Mac.
   - Managed by Remodex's own macOS LaunchAgent: `com.remodex.bridge`.
   - Connects to the relay through `REMODEX_RELAY`.

Use `ws://` over Tailscale. Tailscale encrypts the network tunnel. Use `wss://` only if the relay is behind HTTPS/TLS on a public domain.

## Prerequisites

Install or confirm:

- macOS.
- Tailscale installed and connected on the Mac.
- iPhone connected to the same Tailscale tailnet.
- Node.js and npm.
- Codex CLI installed and authenticated.
- Shell access on the Mac.

Run:

```sh
node --version
npm --version
which node
which npm
codex --version
codex login status
tailscale ip -4
tailscale status --self
```

Expected:

- `node --version` prints a version.
- `npm --version` prints a version.
- `codex login status` reports an authenticated session.
- `tailscale ip -4` prints a `100.x.x.x` address.

If Codex is not authenticated:

```sh
codex login
```

## Variables

Pick a stable workspace directory. This example uses:

```sh
export REMODEX_WORKSPACE="$HOME/Developer/personal-project/remodex-local"
export REMODEX_SOURCE_DIR="$REMODEX_WORKSPACE/remodex"
export REMODEX_RELAY_DIR="$REMODEX_SOURCE_DIR/relay"
export REMODEX_RELAY_PORT="9000"
export REMODEX_TAILSCALE_IP="$(tailscale ip -4)"
export REMODEX_RELAY="ws://${REMODEX_TAILSCALE_IP}:${REMODEX_RELAY_PORT}/relay"
export NODE_PATH_ABS="$(which node)"
```

For convenience, add the bridge relay URL to `~/.zshrc` after confirming the IP:

```sh
export REMODEX_RELAY="ws://YOUR_TAILSCALE_IP:9000/relay"
```

Then reload the shell:

```sh
source ~/.zshrc
```

Important: `~/.zshrc` helps interactive `remodex up` commands. `launchd` does not source `~/.zshrc`, so the relay LaunchAgent must contain its own `RELAY_BIND_HOST` and `PORT` values.

## Install the Remodex Bridge CLI

```sh
npm install -g remodex@latest
remodex --version
```

Expected: `remodex --version` prints a version.

## Clone and Prepare the Relay

```sh
mkdir -p "$REMODEX_WORKSPACE"
cd "$REMODEX_WORKSPACE"
git clone https://github.com/Emanuele-web04/remodex.git remodex
cd "$REMODEX_RELAY_DIR"
npm install
```

If the repo already exists:

```sh
cd "$REMODEX_SOURCE_DIR"
git pull
cd "$REMODEX_RELAY_DIR"
npm install
```

## Smoke-Test the Relay in the Foreground

Run:

```sh
cd "$REMODEX_RELAY_DIR"
RELAY_BIND_HOST=0.0.0.0 PORT=9000 npm start
```

In another terminal, verify:

```sh
curl "http://127.0.0.1:9000/health"
curl "http://${REMODEX_TAILSCALE_IP}:9000/health"
```

Expected response from both:

```json
{"ok":true}
```

Stop the foreground relay with `Ctrl-C`.

## Install the Relay LaunchAgent

Create a local copy first:

```sh
mkdir -p "$REMODEX_WORKSPACE/launchd"
```

Write this file to:

```text
$REMODEX_WORKSPACE/launchd/com.remodex.relay.plist
```

Replace:

- `__NODE_PATH_ABS__` with the output of `which node`.
- `__REMODEX_RELAY_DIR__` with the absolute path to `remodex/relay`.
- `__REMODEX_RELAY_SERVER__` with the absolute path to `remodex/relay/server.js`.

Example for Apple Silicon Homebrew:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>com.remodex.relay</string>

    <key>WorkingDirectory</key>
    <string>__REMODEX_RELAY_DIR__</string>

    <key>ProgramArguments</key>
    <array>
      <string>__NODE_PATH_ABS__</string>
      <string>__REMODEX_RELAY_SERVER__</string>
    </array>

    <key>EnvironmentVariables</key>
    <dict>
      <key>RELAY_BIND_HOST</key>
      <string>0.0.0.0</string>
      <key>PORT</key>
      <string>9000</string>
    </dict>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/tmp/remodex-relay.out.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/remodex-relay.err.log</string>
  </dict>
</plist>
```

Do not use `/opt/homebrew/bin/npm start` in the LaunchAgent. In the successful run, that failed because npm uses `#!/usr/bin/env node`, and launchd's default `PATH` does not include Homebrew. Running the absolute `node` path directly against `server.js` is more reliable.

Validate and install:

```sh
plutil -lint "$REMODEX_WORKSPACE/launchd/com.remodex.relay.plist"
mkdir -p "$HOME/Library/LaunchAgents"
install -m 644 "$REMODEX_WORKSPACE/launchd/com.remodex.relay.plist" "$HOME/Library/LaunchAgents/com.remodex.relay.plist"
```

If an older copy is loaded, unload it first:

```sh
launchctl bootout "gui/$(id -u)" "$HOME/Library/LaunchAgents/com.remodex.relay.plist"
```

It is okay if `bootout` says the service was not loaded.

Load and start:

```sh
launchctl bootstrap "gui/$(id -u)" "$HOME/Library/LaunchAgents/com.remodex.relay.plist"
launchctl kickstart -k "gui/$(id -u)/com.remodex.relay"
```

Verify:

```sh
launchctl print "gui/$(id -u)/com.remodex.relay"
curl "http://127.0.0.1:9000/health"
curl "http://${REMODEX_TAILSCALE_IP}:9000/health"
```

Expected:

- `launchctl print` shows `state = running`.
- Both health checks return `{"ok":true}`.

## Start the Bridge

Run:

```sh
REMODEX_RELAY="ws://${REMODEX_TAILSCALE_IP}:9000/relay" remodex up
```

Expected:

- Remodex writes bridge state under `~/.remodex`.
- Remodex installs/starts its own `com.remodex.bridge` LaunchAgent.
- A QR code and pairing code are printed.

Open the existing Remodex iOS app and scan the QR code.

If the QR expires, rerun:

```sh
REMODEX_RELAY="ws://${REMODEX_TAILSCALE_IP}:9000/relay" remodex up
```

Verify:

```sh
remodex status
```

Expected:

```text
[remodex] Service label: com.remodex.bridge
[remodex] Installed: yes
[remodex] Launchd loaded: yes
[remodex] Bridge state: running
[remodex] Connection: connected
```

## Acceptance Criteria

The setup is complete when all of these are true:

```sh
launchctl print "gui/$(id -u)/com.remodex.relay"
curl "http://127.0.0.1:9000/health"
curl "http://${REMODEX_TAILSCALE_IP}:9000/health"
remodex status
```

Expected:

- Relay LaunchAgent is loaded and `state = running`.
- Local relay health returns `{"ok":true}`.
- Tailscale relay health returns `{"ok":true}`.
- `remodex status` reports bridge `running` and `connected`.
- The iOS app can pair by QR and connect through the self-hosted relay.

## Maintenance

Update relay source:

```sh
cd "$REMODEX_SOURCE_DIR"
git pull
cd "$REMODEX_RELAY_DIR"
npm install
launchctl kickstart -k "gui/$(id -u)/com.remodex.relay"
```

Update bridge CLI:

```sh
npm install -g remodex@latest
REMODEX_RELAY="ws://${REMODEX_TAILSCALE_IP}:9000/relay" remodex restart
```

Stop bridge and relay:

```sh
remodex stop
launchctl bootout "gui/$(id -u)" "$HOME/Library/LaunchAgents/com.remodex.relay.plist"
```

Restart relay only:

```sh
launchctl kickstart -k "gui/$(id -u)/com.remodex.relay"
```

View relay logs:

```sh
tail -n 100 /tmp/remodex-relay.out.log
tail -n 100 /tmp/remodex-relay.err.log
```

View bridge logs:

```sh
tail -n 100 "$HOME/.remodex/logs/bridge.stdout.log"
tail -n 100 "$HOME/.remodex/logs/bridge.stderr.log"
```

## Troubleshooting

### Relay LaunchAgent exits with code 127

Check:

```sh
launchctl print "gui/$(id -u)/com.remodex.relay"
tail -n 100 /tmp/remodex-relay.err.log
```

If logs say:

```text
env: node: No such file or directory
```

then the plist is probably running `npm`, whose shebang depends on `/usr/bin/env node`. Fix the plist to run the absolute Node path directly:

```xml
<key>ProgramArguments</key>
<array>
  <string>/opt/homebrew/bin/node</string>
  <string>/absolute/path/to/remodex/relay/server.js</string>
</array>
```

Use `which node` for the real Node path. On Apple Silicon Homebrew this is commonly `/opt/homebrew/bin/node`; on Intel Homebrew it may be `/usr/local/bin/node`.

### Local health works but Tailscale health fails

Check:

```sh
tailscale status --self
tailscale ip -4
curl "http://127.0.0.1:9000/health"
curl "http://$(tailscale ip -4):9000/health"
```

The relay must bind to `0.0.0.0`, not only `127.0.0.1`.

### `remodex up` cannot write under `~/.remodex`

The bridge CLI needs to write user-level Remodex state and LaunchAgent config. Run it from a normal terminal session with user filesystem access:

```sh
REMODEX_RELAY="ws://YOUR_TAILSCALE_IP:9000/relay" remodex up
```

### iPhone cannot connect

Check:

- The iPhone is connected to the same Tailscale tailnet.
- The relay health URL works from another tailnet device.
- The QR code has not expired.
- `remodex status` shows `Connection: connected`.

## Non-Goals

Do not configure these unless you intentionally set up APNs and Remodex's push service:

```sh
REMODEX_PUSH_SERVICE_URL
REMODEX_ENABLE_PUSH_SERVICE
```

Do not build the iOS app from source unless you want custom app behavior, custom signing, or your own push/APNs setup.
