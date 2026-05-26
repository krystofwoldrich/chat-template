---
name: eas-simulator
description: Run and interact with an EAS cloud iOS simulator using eas simulator:start and agent-device CLI. Use when you need to verify an Expo app on iOS from a non-macOS environment (e.g. Linux cloud VMs).
version: 1.0.0
license: MIT
---

# EAS Cloud Simulator with agent-device

Use this skill when you need to run and test an Expo app on an iOS simulator from a cloud VM or any non-macOS environment. EAS provisions a remote iOS simulator and `agent-device` lets you interact with it programmatically.

## Prerequisites

- **EAS project linked**: `app.json` must have `extra.eas.projectId`. Run `npx eas-cli@latest init --id <project-id>` if missing.
- **iOS bundleIdentifier**: `app.json` must have `ios.bundleIdentifier` set.
- **eas.json**: Must exist with a `development-simulator` profile. See [Build Profile](#build-profile) below.
- **Simulator build**: A completed EAS build with `ios.simulator: true`. See [Build](#build) below.
- **EXPO_TOKEN**: Must be set in the environment for non-interactive EAS CLI auth.
- **agent-device**: Install globally with `npm install -g agent-device`.
- **Dev server running**: The Expo dev server must be reachable by the simulator. In cloud VMs, use a tunnel (see [Tunneling](#tunneling-the-dev-server)).

## Build Profile

Add a `development-simulator` profile to `eas.json`:

```json
{
  "build": {
    "development-simulator": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": {
        "simulator": true
      }
    }
  }
}
```

## Build

Create the simulator build (runs remotely on EAS, takes several minutes):

```bash
npx eas-cli@latest build --profile development-simulator --platform ios --non-interactive
```

Wait for the build to complete. The build artifact is automatically available to `eas simulator:start`.

## Tunneling the Dev Server

The cloud simulator cannot reach `localhost` on your VM. Use `cloudflared` to create a public tunnel:

```bash
# Install cloudflared (Linux)
curl -fsSL https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o /tmp/cloudflared
chmod +x /tmp/cloudflared
sudo mv /tmp/cloudflared /usr/local/bin/cloudflared

# Start tunnel in a background tmux session
cloudflared tunnel --url http://localhost:8081 2>&1 | tee /tmp/cloudflared.log
```

Extract the tunnel URL from the log output — look for `https://<random>.trycloudflare.com`.

Then start the Expo dev server with the tunnel URL:

```bash
export EXPO_PACKAGER_PROXY_URL=https://<random>.trycloudflare.com
bun start
# or: npx expo start
```

The dev server QR code and Metro deep link will use the tunnel URL instead of localhost.

## Starting the Simulator

Start an EAS cloud simulator session:

```bash
npx eas-cli@latest simulator:start --platform=ios --non-interactive
```

This provisions a remote iOS simulator and installs the development build. Wait for the output:

```
✔ 🎉 agent-device session is ready

🔑 Run the following in your shell to attach to the agent-device daemon:

export AGENT_DEVICE_DAEMON_BASE_URL='https://<random>.trycloudflare.com'
export AGENT_DEVICE_DAEMON_AUTH_TOKEN='<token>'

🌐 Open the following URL in your browser to preview the simulator:

https://<random>.trycloudflare.com
```

Save the `AGENT_DEVICE_DAEMON_BASE_URL` and `AGENT_DEVICE_DAEMON_AUTH_TOKEN` values — these are needed for all `agent-device` commands.

## Connecting with agent-device

Set the daemon credentials in your shell, then verify connectivity:

```bash
export AGENT_DEVICE_DAEMON_BASE_URL='https://...'
export AGENT_DEVICE_DAEMON_AUTH_TOKEN='...'

# List devices — should show an iPhone with booted=true
agent-device devices --platform ios

# List installed apps — your app should appear
agent-device apps --platform ios

# Take a screenshot
agent-device screenshot ./screenshot.png --platform ios
```

## Opening the App and Connecting to Dev Server

```bash
# Open the app (use your bundle identifier)
agent-device open dev.expo.chat --platform ios

# The Expo dev client will show "No development servers found"
# Take a snapshot to see the UI
agent-device snapshot -i --platform ios

# Tap "Enter URL manually"
agent-device press 'label="Enter URL manually"' --platform ios

# Snapshot again to find the text field ref
agent-device snapshot -i --platform ios

# Type the tunnel URL (use the cloudflared URL with exp:// scheme)
agent-device fill @e<N> "exp://<tunnel-subdomain>.trycloudflare.com" --platform ios --delay-ms 50

# Press Connect
agent-device press 'label="Connect"' --platform ios
```

Replace `@e<N>` with the actual text-field ref from the snapshot output.

## Common agent-device Commands

```bash
# Screenshot
agent-device screenshot ./path/to/file.png --platform ios

# Accessibility snapshot (all elements)
agent-device snapshot --platform ios

# Interactive elements only
agent-device snapshot -i --platform ios

# Tap by ref
agent-device press @e5 --platform ios

# Tap by label
agent-device press 'label="Settings"' --platform ios

# Tap by coordinates (when accessibility refs are missing)
agent-device press 44 97 --platform ios

# Type text into a field
agent-device fill @e12 "some text" --platform ios --delay-ms 30

# Go back
agent-device back --platform ios
```

## Stopping the Simulator

```bash
npx eas-cli@latest simulator:stop --id <session-id>
```

The session ID is printed when the simulator starts.

## Gotchas

- **Tunnel URLs are ephemeral**: `trycloudflare.com` quick tunnels can drop. If the daemon URL stops working (HTTP 530), the simulator session is likely dead — start a new one.
- **Autocorrect interference**: iOS autocorrect can modify typed text. Use `--delay-ms` on fill/type commands, and verify with a snapshot after typing.
- **Keyboard blocking UI**: The on-screen keyboard may cover elements. Use `agent-device keyboard dismiss` if supported, or tap an empty area to dismiss.
- **Send button not in accessibility tree**: Some custom UI elements (like the chat send button) may not have accessibility labels. Use coordinate-based tapping as a fallback after checking the screenshot.
- **Session provisioning time**: `eas simulator:start` can take 1-3 minutes to provision. Wait for the "agent-device session is ready" message.
- **Snapshot can be slow**: The first `agent-device snapshot` after connecting may take 10-15 seconds. Subsequent snapshots are faster (~1s).
- **Non-interactive mode**: Always use `--non-interactive` with `eas-cli` commands in cloud/CI environments to avoid TTY prompts.
