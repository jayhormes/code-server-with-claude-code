---
name: code-server-setup
description: Set up Claude Code for remote browser use by running code-server on the user's Mac behind a Cloudflare Tunnel. Trigger when the user wants to use Claude Code from a restricted work laptop via their home Mac, wants the Claude Code VS Code extension accessible through a browser, needs to install the `claude` CLI inside code-server, hits Claude Code auth / MCP / WebSocket issues on code-server, or asks about trycloudflare / cloudflared quick tunnels for remote dev. Covers both the quick route (random trycloudflare.com URL, no domain required) and the production route (named tunnel on a user-owned domain with auto-start on boot), plus Claude-Code-specific setup and troubleshooting inside the code-server environment.
---

# code-server + Cloudflare Tunnel Setup (macOS)

Goal: install code-server on the user's Mac and expose it through Cloudflare Tunnel so the user can reach Claude Code from any browser.

## Execution Rules

1. Before each step, state what you are about to do. Pause for confirmation on sudo commands and browser-auth steps.
2. For any variable listed under "Required Variables", ask the user. Never fabricate values.
3. Substitute real values into commands — never execute a command with `<placeholder>` still in it.
4. Verify each step before moving on. On failure, stop and report.
5. Back up existing files under `~/.cloudflared/` or `/etc/cloudflared/` before overwriting.

## First Decision: Does the User Own a Domain?

Ask up front: "Do you have a domain you own and can point at Cloudflare?"

- **Has a domain** → use the **Production Route** (fixed URL, auto-start on boot).
- **No domain / just validating** → use the **Quick Route** (random `*.trycloudflare.com` URL, no domain or Cloudflare login required).

The routes do not conflict. The user can validate with the Quick Route first and migrate later.

---

## Quick Route: Quick Tunnel (No Domain)

Use when: user has no domain, wants to validate, or only needs temporary access. Limitations:

- URL is random `https://<random>.trycloudflare.com` and changes on every restart
- Not intended for long-term production; no Cloudflare SLA
- No boot-time auto-start

### Q1. Install and Start code-server

```bash
brew install code-server cloudflared
brew services start code-server
cat ~/.config/code-server/config.yaml   # record the password
code-server --install-extension anthropic.claude-code
```

### Q2. Start the Quick Tunnel

```bash
cloudflared tunnel --url http://127.0.0.1:8080
```

Output includes:

```
Your quick Tunnel has been created! Visit it at:
https://random-words-xxxx.trycloudflare.com
```

Hand that URL to the user; open in browser, log in with the code-server password.

### Q3. (Optional) Background Mode

Quick Tunnel dies when the terminal closes. To keep it running:

```bash
nohup cloudflared tunnel --url http://127.0.0.1:8080 > ~/cloudflared-quick.log 2>&1 &
sleep 3
grep -o 'https://[a-z0-9-]*\.trycloudflare\.com' ~/cloudflared-quick.log | head -1
```

Stop: `pkill -f 'cloudflared tunnel --url'`

### Q4. Upgrading to the Production Route

Once the user obtains a domain, jump to Step 4 of the Production Route. The Quick Tunnel can remain running or be stopped without interference.

---

## Production Route: Named Tunnel (Owned Domain)

### Required Variables

- `DOMAIN`: domain the user owns and has delegated to Cloudflare (e.g. `example.com`)
- `SUBDOMAIN`: subdomain to use (default `code`; final URL `code.example.com`)
- `TUNNEL_NAME`: tunnel name (default `mac-mini`)
- `MAC_USERNAME`: obtain via `whoami`
- Homebrew path: Apple Silicon → `/opt/homebrew/bin`; Intel → `/usr/local/bin`

### Prerequisite Checks

```bash
which brew          # Homebrew installed
whoami              # username
uname -m            # arm64 | x86_64
```

Domain side must already be:

- Cloudflare account created
- Nameservers switched to the ones Cloudflare assigned
- Cloudflare dashboard shows the domain as **Active**

If not, direct the user to <https://dash.cloudflare.com> first.

### Step 1: Install Packages

```bash
brew install code-server cloudflared
```

### Step 2: Start code-server and Record Password

```bash
brew services start code-server
cat ~/.config/code-server/config.yaml
```

Expected:

```yaml
bind-addr: 127.0.0.1:8080
auth: password
password: <random password>
cert: false
```

Offer to replace with a user-chosen strong password. If changed, run `brew services restart code-server`.

### Step 3: Install the Claude Code Extension

```bash
code-server --install-extension anthropic.claude-code
```

### Step 4: Cloudflare Login and Tunnel Creation

```bash
cloudflared login
cloudflared tunnel create "$TUNNEL_NAME"
cloudflared tunnel route dns "$TUNNEL_NAME" "$SUBDOMAIN.$DOMAIN"
```

Capture `TUNNEL_ID` from that output or `cloudflared tunnel list`.

### Step 5: Write `~/.cloudflared/config.yml`

```yaml
tunnel: <TUNNEL_ID>
credentials-file: /Users/<MAC_USERNAME>/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: <SUBDOMAIN>.<DOMAIN>
    service: http://127.0.0.1:8080
  - service: http_status:404
```

### Step 6: Copy Config to System Paths

The LaunchDaemon runs as root and will not read `~/.cloudflared/`.

```bash
sudo mkdir -p /etc/cloudflared
sudo cp ~/.cloudflared/config.yml /etc/cloudflared/config.yml
sudo cp ~/.cloudflared/<TUNNEL_ID>.json /etc/cloudflared/<TUNNEL_ID>.json
```

### Step 7: Create the LaunchDaemon

> Verify `cloudflared` path with `which cloudflared` before writing. Apple Silicon default: `/opt/homebrew/bin/cloudflared`.

```bash
sudo tee /Library/LaunchDaemons/com.cloudflare.cloudflared.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.cloudflare.cloudflared</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/cloudflared</string>
        <string>tunnel</string>
        <string>--config</string>
        <string>/etc/cloudflared/config.yml</string>
        <string>run</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Library/Logs/com.cloudflare.cloudflared.out.log</string>
    <key>StandardErrorPath</key>
    <string>/Library/Logs/com.cloudflare.cloudflared.err.log</string>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
EOF

sudo launchctl load /Library/LaunchDaemons/com.cloudflare.cloudflared.plist
```

Do **not** use `sudo cloudflared service install` — the plist it generates omits `tunnel run`, so the service starts but does nothing.

### Step 8: Verify

```bash
cloudflared tunnel info "$TUNNEL_NAME"                                  # ≥1 connection
curl -sL -o /dev/null -w "%{http_code}\n" "https://$SUBDOMAIN.$DOMAIN"   # 200
```

Then have the user open `https://<SUBDOMAIN>.<DOMAIN>` and log in with the Step 2 password.

---

## Claude Code Inside code-server

The tunnel + code-server work above only delivers a browser-based VS Code. To actually use Claude Code from that browser, the following Claude-Code-specific setup and checks apply.

### C1. Install the `claude` CLI Inside code-server

The VS Code extension alone is not enough for terminal-driven workflows. Install the CLI **inside the remote Mac** (so it executes there, not on the user's client laptop):

```bash
# prerequisites: Node.js 18+ on the Mac
brew install node                         # skip if already installed
npm i -g @anthropic-ai/claude-code        # installs the `claude` CLI
claude --version                           # sanity check
```

If the user prefers not to use `npm -g`, `brew install anthropic/claude/claude-code` is an alternative where available.

### C2. First-Time Authentication

Auth must be done from a terminal **inside code-server** (not the user's local terminal), because tokens are stored on the Mac where `claude` will actually run.

```bash
claude        # triggers OAuth; opens a URL in the code-server browser tab
```

Troubleshooting:

- If the OAuth redirect fails to reach `localhost` (common on remote browser setups), copy the shown URL manually and paste back the code when prompted.
- Tokens live under `~/.config/claude/` on the Mac. If the user ever restores a different Mac, re-run `claude` to re-auth — do not copy tokens blindly.

### C3. Using the VS Code Extension vs CLI

Both should point at the same workspace. Inside code-server:

- **Extension (sidebar)**: installed in Step 3 of the Production Route (or Q1 of the Quick Route). Opens chat panel; uses the same auth as the CLI once `claude` has logged in once on this Mac.
- **CLI in terminal**: `claude` from any code-server terminal. Preferred for scripted or multi-step work.

If the extension shows "not authenticated" but CLI works, have the user run the extension's **Claude: Sign In** command — it reuses the CLI token when available.

### C4. MCP Servers Under code-server

MCP servers launched by Claude Code run as child processes on the Mac. Notes:

- MCP configs live in `~/.claude/` on the Mac — edit them over the code-server terminal, not the client laptop.
- Servers that need local GUI access (e.g. anything opening windows) will fail because code-server has no display. Prefer headless MCP servers.
- `stdio` transport works fine; `sse`/`http` transports pointing at `localhost` also work (all on the same Mac).
- After editing `mcp.json`, reload the Claude Code extension or restart the CLI session.

### C5. Known Issues in the Browser Client

| Symptom | Cause | Fix |
|---------|-------|-----|
| WebSocket drops every few minutes | Cloudflare idle-timeout on Free plan | Enable "Always keep connection alive" in code-server settings, or add a small keep-alive in the browser tab (some extensions do this). Zero Trust users can raise the idle timeout. |
| ⌘/Ctrl shortcuts collide with host browser | Host browser (Chrome/Safari) eats the shortcut before code-server sees it | In code-server: Settings → Keyboard Shortcuts, rebind the Claude Code commands; or use the Command Palette. |
| "Paste" into Claude Code chat loses formatting | Browser clipboard permission not granted | Click the address-bar lock → allow clipboard for the site. |
| Extension stuck on "Loading…" | code-server cached an old extension version | `code-server --uninstall-extension anthropic.claude-code && code-server --install-extension anthropic.claude-code`, then reload the browser tab. |
| `claude` CLI hangs on start | Stale lockfile in `~/.config/claude/` | Check for and remove stale `*.lock` files, then retry. |

### C6. Updating Claude Code

```bash
npm update -g @anthropic-ai/claude-code                       # CLI
code-server --install-extension anthropic.claude-code --force # extension (re-pulls latest)
```

Restart the code-server service after extension updates:

```bash
brew services restart code-server
```

---

## Common Failures (Infrastructure)

| Symptom | Cause | Fix |
|---------|-------|-----|
| `curl` returns 530 or DNS error | `cloudflared tunnel route dns` not run | Re-run the last line of Step 4 |
| Service runs but `tunnel info` shows 0 connections | Using plist from `service install` | Rewrite the plist per Step 7 |
| System service cannot find config | Files not copied to `/etc/cloudflared/` | Re-run Step 6 |
| Login page does not load | code-server is not running | `brew services list`; `brew services restart code-server` |

## Security Recommendations

- Replace the default code-server password with a user-chosen strong one
- Add a Cloudflare Zero Trust layer (Email OTP or Google SSO) for stronger access control
- Do not leave the password in plaintext in `config.yaml` for long-term use

## Completion Report Template

```
code-server remote environment is ready
- URL: https://<SUBDOMAIN>.<DOMAIN>        (or trycloudflare URL for Quick Route)
- Password: see ~/.config/code-server/config.yaml (or user-chosen)
- Tunnel name: <TUNNEL_NAME>                (Production Route only)
- Tunnel ID: <TUNNEL_ID>                    (Production Route only)
- LaunchDaemon: /Library/LaunchDaemons/com.cloudflare.cloudflared.plist   (Production Route only)
- Logs: /Library/Logs/com.cloudflare.cloudflared.{out,err}.log             (Production Route only)
```
