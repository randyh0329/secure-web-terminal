# Multi-project: one tab per project, one hostname

Read this after the single-session setup in `SKILL.md` works and has been verified.

## Option A: just use tmux

Often enough: one tmux session with a window per project (`Ctrl-b c`, rename with `Ctrl-b ,`).
There's nothing new to build or secure. Suggest this first.

## Option B: separate browser tabs per project

### Key design decision: one hostname, path-based fan-out

Don't create a Cloudflare hostname or tunnel ingress rule per project. Point the tunnel at **one**
local app (`127.0.0.1:3000`). That app routes `/term/<slug>/*` to the right per-project ttyd port.
Adding or removing a project never touches Cloudflare config or the Access policy.

```
term.example.com/            → static UI (tab bar)
term.example.com/api/*       → project CRUD
term.example.com/term/<slug>/ (HTTP + WebSocket) → 127.0.0.1:<port of slug>  (ttyd -b /term/<slug>)
```

### Per-project units: use a systemd template

`~/.config/systemd/user/ttyd@.service`, instantiated as `ttyd@my-app.service`:

```ini
[Service]
Type=exec
EnvironmentFile=%h/webterm/env/%i.env        # PORT, IFACE, CWD written by the app
Environment=PATH=/usr/local/bin:/usr/bin:/bin:%h/.local/bin
Environment=TERM=xterm-256color
ExecStart=/usr/bin/env ttyd -p ${PORT} -i ${IFACE} -b /term/%i -W -O tmux new -A -s %i -c ${CWD}
Restart=on-failure
RestartSec=2s
```

A missing env file makes an orphaned instance fail fast instead of starting on a random port.

### Invariants that are easy to break

1. **Validate the slug strictly**, e.g. `^[a-z0-9](?:[a-z0-9-]{0,30}[a-z0-9])?$`, plus a reserved
   list (`api`, `term`, …). The slug ends up in a file path, a unit name and a tmux target.
   Re-validate whenever you load persisted state.
2. **Match the slug in the proxy path raw, before percent-decoding.** Then `%2e%2e`, `%2f`
   and `%00` fail the character check instead of decoding into something dangerous. Forward the
   rest of the path **unchanged**: ttyd runs with `-b /term/<slug>` and expects the prefix.
3. **Mount the proxy before any body parser** (e.g. `express.json()`). Otherwise the parser
   consumes the stream and ttyd receives an empty body.
4. **WebSocket upgrades bypass the HTTP framework.** Any auth or routing check must be applied
   to **both** the HTTP path and the `server.on('upgrade')` path. The WebSocket is the shell, so
   a gate that only covers HTTP protects nothing.
5. **Keep the tmux server out of every unit's cgroup.** Create each project's session *before*
   starting its ttyd, and start the first session (which starts the server) in a transient scope:
   `systemd-run --user --scope --collect --unit=tmux-server -- tmux new-session -d -s <slug> -c <cwd>`.
   Otherwise restarting the app kills every shell, or deleting one project kills the rest.
6. **Use exact tmux targets:** `=<slug>` for sessions, `=<slug>:` for panes. Use `send-keys -l --`
   for anything typed programmatically.
7. **Serialize registry writes** (one promise queue, write to a temp file, then atomic `rename`),
   so concurrent creates can't get the same port.
8. **Spawn processes with argv arrays only**, never through a shell string.
9. **Make lifecycle operations idempotent**: "up" on a live project and "down" on a missing one
   both succeed.
10. **Refuse to start** if the app is configured to bind a non-loopback address.
11. **Keep the `Host` header when proxying** (no `changeOrigin`), so ttyd's `-O` origin check
    still passes for your own page. Pass `-O` to every ttyd instance.
12. **Same-origin checks on everything that changes state.** Refuse non-GET API requests and
    every WebSocket upgrade whose `Origin` doesn't match `Host` (or that are marked
    `Sec-Fetch-Site: cross-site`). Access authenticates the browser, so cross-site requests carry
    a valid session. Add `X-Frame-Options: DENY` and a CSP with `frame-ancestors 'none'`, so a
    hostile page can't trick a click onto a "delete project" control.
13. **For defense in depth, verify the Access JWT in the app**: check the `Cf-Access-Jwt-Assertion`
    header against your team's certs (`https://<team>.cloudflareaccess.com/cdn-cgi/access/certs`)
    and the app's AUD tag, on both the HTTP and the upgrade path. Or turn on
    "Protect with Access" on the tunnel hostname. Accept RS256 only, fail closed if the keys
    can't be fetched, and don't exempt loopback (cloudflared connects from loopback).

### Frontend notes

- Talk to ttyd's protocol directly instead of iframing its page: `GET <base>/token`, then a
  WebSocket with subprotocol `tty` whose first message is `{"AuthToken", "columns", "rows"}`, then
  binary frames tagged by their first byte. Iframes make shared focus and paste handling across
  tabs painful.
- To switch tabs, hide panes (`display:none`). Never destroy them, so scrollback and sockets stay
  live. Create each terminal on first activation, because a fit addon can't measure a hidden
  element.
- **No global keyboard shortcuts.** Claude Code uses many modifier combinations, so switching tabs
  should be click-only.
- Honor ttyd flow control (pause above ~100 kB queued, resume when drained). Otherwise a flood of
  output freezes the browser.

### Reference

[webterm / cloud_cli](https://github.com/randyh0329/cloud_cli) implements all of the above as a small,
framework-free Node app with three dependencies, and ships a smoke test that checks each invariant
against a real host.
