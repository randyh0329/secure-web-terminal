---
name: secure-web-terminal
description: Set up a self-hosted browser terminal for Claude Code (or any CLI) on a Linux host, using ttyd + tmux under systemd --user, published through a Cloudflare Tunnel and protected by Cloudflare Access. Use when the user wants to reach Claude Code from a browser, phone, or tablet; run persistent remote Claude Code sessions on a VM; expose ttyd or a web terminal safely; or debug a ttyd/tmux/cloudflared/Cloudflare Access setup (read-only terminal, sessions dying on restart, 404s behind a proxy, units not starting after logout).
---

# Claude Code web terminal behind Cloudflare Access

Goal: open `https://term.example.com` in any browser, pass a Cloudflare Access login, and land
in a **persistent** tmux session on your own Linux host where Claude Code is running. Closing the
tab, losing Wi-Fi, or restarting the web layer never kills the session.

```
Browser ─https/wss─► Cloudflare Access (login) ─► Cloudflare Tunnel ─► cloudflared on host
                                                                          │
                                                       127.0.0.1:7681  ttyd (systemd --user)
                                                                          │
                                                                tmux session (own unit)
                                                                          │
                                                                     claude / shell
```

**Security model, stated plainly:** a web terminal is remote code execution as your user, by
design. The *only* login is Cloudflare Access. Everything on the host binds to `127.0.0.1`, so
cloudflared is the only path in. Every step below protects one of those two properties. Don't
skip verification.

## Before you start: confirm with the user

- The host: Linux with systemd, a normal user with `sudo`. **Do not run any of this as root.**
  Root has no usable `systemd --user` manager on a headless VM.
- A domain on Cloudflare and access to the Zero Trust dashboard (the free plan is enough).
- **A host only the user logs into.** `127.0.0.1` keeps the network out, not other local
  accounts: any other user on the same machine can connect to the ttyd port and get this user's
  shell. On a shared host, stop and say so.
- The exact email address(es) allowed in. Get this **before** anything is published.
- Single session (one terminal) or multiple projects as tabs. Start with single. For multiple
  projects, read [references/multi-project.md](references/multi-project.md) after finishing it.

## Step 1: packages

```bash
sudo apt-get update && sudo apt-get install -y tmux
ttyd --version 2>/dev/null || true
```

Use **ttyd 1.7 or newer**. Distro packages are often 1.6.x. If so, install a **pinned** release
binary and check it against the release's published checksums before installing:

```bash
V=1.7.7 A=x86_64            # A=aarch64 on ARM; check https://github.com/tsl0922/ttyd/releases for newer
cd "$(mktemp -d)"
curl -fsSLO "https://github.com/tsl0922/ttyd/releases/download/$V/ttyd.$A"
curl -fsSL "https://github.com/tsl0922/ttyd/releases/download/$V/SHA256SUMS" | grep " ttyd.$A\$" | sha256sum -c -
sudo install -m 0755 "ttyd.$A" /usr/local/bin/ttyd     # only if the line above printed OK
ttyd --version
```

The checksum confirms the file arrived intact and matches what the release lists. It doesn't make
the upstream project trustworthy.

Install Claude Code for the same user that will own the terminal, and make sure `claude` works in
a normal SSH shell first.

## Step 2: make `systemd --user` survive logout

Without lingering, the user manager exists only while you're logged in. Your units stop when SSH
disconnects and won't start at boot.

```bash
sudo loginctl enable-linger "$USER"
loginctl show-user "$USER" --property=Linger   # expect Linger=yes
ls -d /run/user/$(id -u)                       # must exist
```

## Step 3: tmux and ttyd as two separate user units

The tmux server inherits the cgroup of whatever process starts it. If ttyd's unit starts it,
`systemctl --user restart ttyd-main` kills tmux and every shell in it. So tmux gets **its own
unit**, and ttyd only attaches.

`~/.config/systemd/user/tmux-main.service`:

```ini
[Unit]
Description=Persistent tmux session "main" for the web terminal

[Service]
Type=forking
Environment=PATH=/usr/local/bin:/usr/bin:/bin:%h/.local/bin
WorkingDirectory=%h
ExecStart=/usr/bin/env tmux new-session -d -s main
ExecStop=/usr/bin/env tmux kill-session -t =main
Restart=always
RestartSec=2

[Install]
WantedBy=default.target
```

`~/.config/systemd/user/ttyd-main.service`:

```ini
[Unit]
Description=ttyd web terminal attached to tmux session "main"
Wants=tmux-main.service
After=tmux-main.service

[Service]
Type=exec
Environment=PATH=/usr/local/bin:/usr/bin:/bin:%h/.local/bin
Environment=TERM=xterm-256color
ExecStart=/usr/bin/env ttyd -p 7681 -i 127.0.0.1 -W -O tmux new -A -s main
Restart=on-failure
RestartSec=2

[Install]
WantedBy=default.target
```

What every flag is for, so nobody "simplifies" one away:

| Piece | Why |
|---|---|
| `-i 127.0.0.1` | ttyd binds **all interfaces** by default. This flag is the loopback guarantee. Some builds want an interface name instead: use `-i lo`. |
| `-W` | Since ttyd 1.7 the terminal is **read-only** unless `-W` is passed. Without it you can see the shell but can't type. |
| `-O` | Reject WebSockets whose `Origin` doesn't match `Host`. Without it, ttyd accepts a socket opened by **any website**, so a malicious page you visit could try to drive your shell using your browser's Access session (cross-site WebSocket hijacking). cloudflared keeps the `Host` header, so same-origin connections still work. |
| *no* `-c user:pass` | Access is the login. Basic auth in front of a shell adds a weak shared secret, not real protection. |
| `tmux new -A -s main` | Attach if the session exists, create it otherwise. Reconnects resume the same session. |
| `-t =main` | `=` forces an exact tmux match. A bare `-t main` prefix-matches and could hit `main-2`. |
| `Environment=PATH=…` | The user manager's PATH is minimal and misses `/usr/local/bin` and `~/.local/bin`. |
| `RestartSec` | Without it, a crash loop hits systemd's start-rate limit and the unit stays dead. |

Enable and verify:

```bash
systemctl --user daemon-reload
systemctl --user enable --now tmux-main.service ttyd-main.service
systemctl --user status ttyd-main.service --no-pager
ss -ltnp | grep 7681              # MUST show 127.0.0.1:7681, never 0.0.0.0 or *
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:7681/   # 200
```

If `ss` shows anything other than `127.0.0.1`, stop and fix that before going further.

## Step 4: Cloudflare Access first, tunnel second

Create the lock before you open the door. If you publish the hostname first, there's a window
where anyone can get a shell.

1. **Zero Trust → Access → Applications → Add → Self-hosted.** Domain: `term.example.com`.
   Policy: **Allow**, Include → **Emails** → the exact address(es) from the user. Don't use
   "everyone" or a whole email domain unless the user explicitly asks. This login is the only
   thing in front of a shell, so:
   - Prefer an identity provider with **MFA** (Google, GitHub, Okta…) over email one-time PIN.
     With a PIN, anyone who controls the inbox gets the shell.
   - Keep the session duration short (hours, not weeks).
   - In the app's cookie settings, turn on **HttpOnly** and set **SameSite** to `Lax` or `Strict`.
2. **Zero Trust → Networks → Tunnels → Create tunnel (cloudflared).** Run the install command it
   shows on the host. That installs cloudflared as a system service with the tunnel token.
   **The token is a credential.** Don't paste it into files, chat logs or repos, and rotate it
   if it leaks.
3. On the tunnel, add a **public hostname**: `term.example.com` → service `http://127.0.0.1:7681`.
   WebSockets work through the tunnel with no extra settings.
4. On that hostname, turn on **Protect with Access** (the hostname's Access setting) and select
   the application from step 1. cloudflared then checks the Access token on every request itself.
   If the Access app is ever deleted or misconfigured, the tunnel fails closed instead of serving
   an open shell.

If the user prefers a locally managed tunnel (`cloudflared tunnel create` + `config.yml`), use the
same rule and keep the JWT check:

```yaml
ingress:
  - hostname: term.example.com
    service: http://127.0.0.1:7681
    originRequest:
      access:
        required: true
        teamName: <your-team-name>          # <team>.cloudflareaccess.com
        audTag: [<access-app-aud-tag>]      # Access app → Overview → Application Audience (AUD) tag
  - service: http_status:404
```

Keep `~/.cloudflared/*.json` (tunnel credentials) at mode 0600, and never commit it.

## Step 5: verify the perimeter (required)

```bash
# Logged out, the hostname must redirect to Access, never serve the terminal:
curl -sI https://term.example.com | grep -iE '^(HTTP|location)'
#   expect a 302 with a location on <team>.cloudflareaccess.com

# On the host: a WebSocket from a foreign Origin must be refused (proves -O is active).
curl -s -o /dev/null -w '%{http_code}\n' --max-time 3 \
  -H 'Connection: Upgrade' -H 'Upgrade: websocket' -H 'Sec-WebSocket-Version: 13' \
  -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' -H 'Sec-WebSocket-Protocol: tty' \
  -H 'Host: term.example.com' -H 'Origin: https://evil.example.net' http://127.0.0.1:7681/ws
#   expect anything except 101 (000 = connection refused/closed). 101 means -O is missing.
```

Then, by hand:

- Open the URL in a private window. You should get the Access login, sign in with the allowed
  email, and get a terminal you can **type** into.
- Sign in with a **non-allowed** email and confirm you're denied.
- Close the tab, reopen it, and confirm it's the same session (scrollback intact).
- `systemctl --user restart ttyd-main` and confirm the session survives (the tmux unit is untouched).
- From another machine, `curl http://<host-public-ip>:7681/` must fail (no listener on that interface).

Finally start Claude Code in the session: `cd ~/your-project && claude`.

## Daily use

- **Selecting text:** Claude Code and tmux use mouse reporting, so a plain drag goes to the
  program. Hold **Shift** (Option on macOS) while dragging to select locally.
- **More windows:** use tmux itself (`Ctrl-b c`, `Ctrl-b n`) for several Claude Code instances in
  one session. For separate browser tabs per project, see
  [references/multi-project.md](references/multi-project.md).
- **Phone or tablet:** works as-is. Long Claude Code output is easier to read with tmux copy mode
  (`Ctrl-b [`).

## Optional: paste screenshots into Claude Code

Claude Code accepts an image path in the prompt. A small helper on the host can turn a browser
paste into a typed path:

1. The page listens for `paste` on `document` in the **capture phase**. If
   `clipboardData.items` holds an `image/*`, call `preventDefault()` and POST the blob.
2. The server checks the file type from **magic bytes** (PNG/JPEG/GIF/WebP), not from
   `Content-Type` or the filename. It writes a server-generated name with mode `0600`.
3. The server types the path into the pane **without pressing Enter**:
   `tmux send-keys -t '=main:' -l -- '/abs/path/shot.png '`
   - `-l` sends the text literally. Without it, a payload like `Enter` or `C-c` runs as a key.
   - `--` stops a leading `-` from being parsed as a flag.
   - Pane targets need the trailing colon: `=main:`, not `=main`.

That endpoint needs the same Access protection as the terminal. A full implementation is in
the reference project linked below.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| You see the terminal but typing does nothing | ttyd ≥ 1.7 without `-W` |
| `ttyd: invalid option -- 'b'` | ttyd < 1.7. Install the release binary |
| Unit fails right away, log mentions the interface | Your ttyd wants `-i lo` instead of `-i 127.0.0.1` |
| Units stop when SSH disconnects / don't start at boot | Lingering is off (Step 2) |
| `Failed to connect to bus` from `systemctl --user` | You're root, or in `su`/`sudo -u` without a login session. Use a real login, or `machinectl shell user@` |
| Restarting ttyd kills your shells | tmux was started from inside ttyd's unit. Start `tmux-main.service` first, then restart both |
| `tmux-main` restarts in a loop | The last window was closed with `exit`. That's expected: the unit recreates the session |
| `ttyd: command not found` in the journal | Add the right directory to `Environment=PATH=` |
| Blank page or 404s behind a path prefix | Behind a reverse proxy on `/some/path`, ttyd needs `-b /some/path` |
| Access login loops or the WebSocket fails | Check that the Access app domain matches the hostname exactly, and that no other app or path rule overlaps |
| Page loads but the terminal never connects (only after adding `-O`) | Something rewrites `Host` (e.g. an `httpHostHeader` override on the tunnel, or a proxy using `changeOrigin`). Remove the rewrite. Don't remove `-O` |
| `Forbidden` / 403 from cloudflared after enabling Protect with Access | Wrong team name or AUD tag in the tunnel's Access settings |

Logs: `journalctl --user -u ttyd-main -u tmux-main -n 50 --no-pager` and
`sudo journalctl -u cloudflared -n 50 --no-pager`.

## Hard rules

- Never bind ttyd or any helper to `0.0.0.0`, and never open its port in a firewall.
- Never publish the hostname before the Access policy exists and is limited to named identities.
- Never run the units as root or use system-level units for the terminal.
- Never let the process serving the browser be the parent (cgroup) of the tmux server.
- Never drop `-O`. Never add basic auth or a shared password in place of Access.
- Never write the tunnel token, `~/.cloudflared/*.json`, or Access service tokens into a repo,
  a skill, or a chat transcript.
- Use only on a host where the user is the only account holder.

## Reference implementation

[webterm / cloud_cli](https://github.com/randyh0329/cloud_cli) is a multi-project version of this
setup: a Node app gives each project its own tab (ttyd + tmux per project) under one hostname, with
paste-to-screenshot and an end-to-end smoke test.
