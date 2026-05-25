# Remodex Self-Hosted Relay Setup Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Run Remodex with a self-hosted relay reachable over Tailscale, while using the existing iOS app and the npm-installed Mac bridge CLI.

**Architecture:** The relay runs as a long-lived Node process from a `remodex/` source checkout inside this workspace and listens on the Tailscale interface. The Remodex bridge runs on the same Mac through the `remodex` CLI and connects to that relay with `REMODEX_RELAY`. The iOS app pairs by scanning the QR code; no custom iOS build is required for this setup.

**Tech Stack:** macOS `launchd`, Node.js/npm, Remodex relay source checkout, Remodex npm CLI, Codex CLI, Tailscale.

---

### Task 1: Confirm prerequisites

**Files:**
- No file changes.

- [ ] **Step 1: Confirm Node and npm are available**

```sh
node --version
npm --version
```

Expected: Node.js is v18 or newer, and npm prints a version.

- [ ] **Step 2: Confirm Codex CLI is installed and authenticated**

```sh
codex --version
codex login status
```

Expected: `codex login status` succeeds. If it does not, run:

```sh
codex login
```

- [ ] **Step 3: Confirm Tailscale address**

```sh
tailscale ip -4
tailscale status --self
```

Expected: the Mac prints a `100.x.x.x` Tailscale IP. Earlier context showed `100.88.103.107`; use the current command output if it differs.

### Task 2: Install or update the Remodex bridge CLI

**Files:**
- No repo file changes.

- [ ] **Step 1: Install the bridge CLI**

```sh
npm install -g remodex@latest
```

- [ ] **Step 2: Verify the CLI**

```sh
remodex --version
```

Expected: a Remodex version prints.

### Task 3: Clone and prepare the relay source

**Files:**
- Create source checkout: `/Users/maneesh/Developer/personal-project/remodex-local/remodex`

- [ ] **Step 1: Clone the public repo**

```sh
cd /Users/maneesh/Developer/personal-project/remodex-local
git clone https://github.com/Emanuele-web04/remodex.git remodex
```

- [ ] **Step 2: Install relay dependencies**

```sh
cd /Users/maneesh/Developer/personal-project/remodex-local/remodex/relay
npm install
```

- [ ] **Step 3: Smoke-test the relay in the foreground**

```sh
cd /Users/maneesh/Developer/personal-project/remodex-local/remodex/relay
RELAY_BIND_HOST=0.0.0.0 PORT=9000 npm start
```

Expected: relay starts and stays running. In another terminal:

```sh
curl http://127.0.0.1:9000/health
curl http://100.88.103.107:9000/health
```

Expected: both return:

```json
{"ok":true}
```

Stop the foreground relay with `Ctrl-C` after the smoke test.

### Task 4: Run the relay as a macOS LaunchAgent

**Files:**
- Create: `/Users/maneesh/Library/LaunchAgents/com.remodex.relay.plist`
- Logs: `/tmp/remodex-relay.out.log`, `/tmp/remodex-relay.err.log`

- [ ] **Step 1: Create the LaunchAgent plist**

Write this exact file to `/Users/maneesh/Library/LaunchAgents/com.remodex.relay.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>com.remodex.relay</string>

    <key>WorkingDirectory</key>
    <string>/Users/maneesh/Developer/personal-project/remodex-local/remodex/relay</string>

    <key>ProgramArguments</key>
    <array>
      <string>/opt/homebrew/bin/node</string>
      <string>/Users/maneesh/Developer/personal-project/remodex-local/remodex/relay/server.js</string>
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

- [ ] **Step 2: Load and start the relay**

```sh
launchctl bootstrap gui/$(id -u) /Users/maneesh/Library/LaunchAgents/com.remodex.relay.plist
launchctl kickstart -k gui/$(id -u)/com.remodex.relay
```

- [ ] **Step 3: Verify the daemonized relay**

```sh
launchctl print gui/$(id -u)/com.remodex.relay
curl http://127.0.0.1:9000/health
curl http://100.88.103.107:9000/health
```

Expected: `launchctl print` shows the service, and both health checks return `{"ok":true}`.

### Task 5: Start the Remodex bridge against the self-hosted relay

**Files:**
- Remodex bridge state is managed by the CLI under the user's home directory.

- [ ] **Step 1: Start the bridge using the Tailscale relay URL**

```sh
REMODEX_RELAY="ws://100.88.103.107:9000/relay" remodex up
```

Expected: on macOS, `remodex up` writes/starts the bridge LaunchAgent, waits for a pairing payload, and prints a QR code.

- [ ] **Step 2: Pair the iOS app**

Open the existing Remodex iOS app and scan the QR from inside the app.

Expected: the first scan trusts the Mac. Later reconnects should reuse the same trusted Mac through the relay.

- [ ] **Step 3: Verify bridge status**

```sh
remodex status
```

Expected: status shows the macOS bridge service is loaded/running and has a recent pairing payload.

### Task 6: Maintenance commands

**Files:**
- No file changes.

- [ ] **Step 1: Restart the relay after source updates**

```sh
cd /Users/maneesh/Developer/personal-project/remodex-local/remodex
git pull
cd relay
npm install
launchctl kickstart -k gui/$(id -u)/com.remodex.relay
```

- [ ] **Step 2: Restart the bridge after CLI updates**

```sh
npm install -g remodex@latest
REMODEX_RELAY="ws://100.88.103.107:9000/relay" remodex restart
```

- [ ] **Step 3: Stop services if needed**

```sh
remodex stop
launchctl bootout gui/$(id -u) /Users/maneesh/Library/LaunchAgents/com.remodex.relay.plist
```

### Task 7: Known non-goals and caveats

**Files:**
- No file changes.

- [ ] **Step 1: Leave managed push disabled**

Do not set these unless you intentionally configure APNs and the relay-side push service:

```sh
REMODEX_PUSH_SERVICE_URL
REMODEX_ENABLE_PUSH_SERVICE
```

- [ ] **Step 2: Do not build the iOS app unless you need custom behavior**

Use the App Store app for this setup. Build from source only if you want to modify the app, use your own signing defaults, or operate your own push/APNs configuration.

- [ ] **Step 3: Keep private deployment values out of Git**

Do not commit real relay hostnames, private IPs, APNs credentials, or local plist files containing private deployment details.
