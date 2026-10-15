# browser-manager

A single-owner dashboard and `browserctl` CLI for several named, signed-in browser desktops on one host, reachable from
a phone through a Cloudflare Tunnel behind Cloudflare Access. Each browser keeps its own persistent profile, so logins
made inside it stay there; closing the viewer never stops the browser. Scripts can drive a running browser through a
loopback CDP bridge, inheriting its sessions and its standing with bot protection.

It is single-owner, not a multi-tenant service: separate profiles isolate browser sessions, but are not separate OS
security boundaries. The browsers have the host user's normal network and filesystem access. Only that owner should
have Access permission.

- **Linux host:** each browser runs headed in its own Xvfb display, viewed through noVNC over private Unix sockets.
- **macOS host:** native headed Chrome per browser, streamed as page images over an authenticated WebSocket (below).

## Security model

- The dashboard and every viewer WebSocket require a Cloudflare Access JWT, verified against the team's keys, for the
  configured audience and owner address. Mutations also require the expected `Origin`.
- The application listens only on `127.0.0.1:3210`. VNC and the CLI use private Unix sockets (the CLI's is owner-only),
  so there is no public bearer token and no TCP listener besides the loopback port.
- Unknown instance names cannot become network or filesystem targets. There is no profile-deletion command, no
  arbitrary executable launcher and no public CDP endpoint. Browser processes stay sandboxed.
- An expired Access session loses its live desktop connection and must sign in again.

## 1. Provision the tunnel and Access application

Outside this repository (Terraform, the dashboard or the API), create:

1. A **Cloudflare Tunnel** with one public hostname, for example `browsers.example.com`, whose service is
   `http://127.0.0.1:3210`. Keep its connector token private.
2. A **Cloudflare Access application** for exactly that hostname, with a policy that admits only the owner.
3. Optionally, an **Access service token** for a machine viewer (see "OpenClaw viewers"), admitted only by that
   application.

Then write the environment file, owned by the browser user, mode `0600`:

```bash
BM_BASE_URL=https://browsers.example.com
BM_ACCESS_ISSUER=https://<team>.cloudflareaccess.com
BM_ACCESS_AUD=<the Access application's audience tag>
BM_OWNER_EMAIL=you@example.com
BM_ACCESS_SERVICE_TOKEN_ID=            # optional: <client id>.access of the viewer's service token
BM_PORT=3210
BM_BROWSER_BIN=/usr/bin/google-chrome   # only if Chrome is not at a standard path
```

Nothing deployment-specific has a default: the server refuses to start until all of these are set. Never put the tunnel
token in this file; it goes to `cloudflared` alone, as a separate file.

## 2. Install on a Linux host

Node.js 22.13+, an installed, sandbox-capable Chromium or Chrome, the desktop prerequisites and a current
[`cloudflared`](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/downloads/) with
`--token-file` support:

```bash
sudo apt-get install xvfb x11vnc openbox x11-xserver-utils x11-utils wmctrl python3 dbus libglib2.0-bin at-spi2-core python3-dbus
git clone https://github.com/ejc3/browser-manager && cd browser-manager
npm ci --ignore-scripts
npm run build
./scripts/install.sh /absolute/path/browser-manager.env /absolute/path/tunnel-token
```

Run the installer as the browser owner, not root. It installs two **user** systemd units, `browser-manager.service` and
`browser-manager-tunnel.service`, and `~/.local/bin/browserctl`, and restarts only those. The checkout must stay at the
same path; to update, pull, rebuild and rerun the installer. For persistence across logout and reboot, an administrator
can run `loginctl enable-linger <browser-user>`. Do not disable the Chrome sandbox to make installation pass; the
installer never downloads a browser or imports any browser's credentials.

State and profiles live under `~/.local/state/browser-manager`, configuration and the token under
`~/.config/browser-manager`, both private to the owner.

## 3. Use it

```bash
browserctl start claude --url https://claude.ai
browserctl start research --url https://example.com
browserctl list
browserctl url claude
browserctl rename claude "Claude · Personal"
browserctl stop research
```

Open your hostname, sign in through Access as the owner, then open a desktop (for example `/browsers/claude`). Up to 32
names can be registered. Stop keeps a name and its profile; Start reuses them. On service restart, previously running
desktops are restored. **Rename** changes only the display label, never the name, URL, profile or connection.

To reuse an existing profile without copying it, close the browser using it, then
`browserctl start <name> --profile /absolute/profile/path`. Only the CLI accepts a path; profiles with Chrome locks are
refused.

### Automating a signed-in browser (CDP bridge)

A script that launches its own copy of a profile is a new visitor to bot protection (ChatGPT answered one with a
Cloudflare "verify you are human" check); the running desktop is not. `browserctl cdp <name>` prints a loopback CDP
endpoint for a running browser, for Playwright's `connect_over_cdp` or any CDP client. It is handed out only through the
owner-only control socket, behind a random token. Each client gets its own browser-level session: it sees and drives
only the tabs it attaches, never the desktop's own sessions, and everything it set (emulation, auto-attach) is undone
when it disconnects. The bridge refuses `Browser.close` (and crash commands) and acknowledges without forwarding the
browser-wide download setting, so a script can open and close its own tabs but never end or reconfigure the person's
browser.

`browserctl job <name> <job>` runs a job from `jobs/` in a new tab of that browser and closes only that tab. A job is a
Python file defining `run(page, args) -> dict`; the result is printed as JSON:

```bash
python3 -m venv ~/pw-venv && ~/pw-venv/bin/pip install playwright   # once; no browser download, jobs drive yours
browserctl job codex codex-code-review.py list
browserctl job codex codex-code-review.py enable owner/repo
```

A job never solves a human-verification challenge: it reports it, and you clear it once in the live desktop.

### The viewer

- **Back** beside the desktop title goes back in the remote browser's history (it follows the active tab's native
  toolbar state through the desktop's private accessibility bus, polled every 500 ms in the foreground; no history URLs
  are returned). The top-left arrow returns to the dashboard.
- **Fit to screen** scales per viewer; off means logical actual size. **Phone** switches the remote browser itself to a
  narrow responsive layout (bounded to 320–500 × 480–900 CSS pixels on a phone, 390 × 844 elsewhere) without restarting
  or reloading it; every viewer of that browser sees the change. It changes the viewport, not the user agent.
- **Keyboard** opens text, paste and special-key controls. On touch screens, a one-finger swipe scrolls (with a short
  glide unless reduced motion is set), a tap clicks, and tap-then-drag drags.
- The Linux desktop renders at 2× density (Desktop is 2880 × 1800 pixels). VNC uses 24-bit color at JPEG quality 9,
  dropping to 6 with stronger compression when the browser reports Data Saver, 2G/3G or under 2 Mbps.
- A disconnected viewer retries while visible and focused (1, 2, 4, 8, then every 10 s), and at once when the tab
  returns. Retries never bypass Access sign-in.

### OpenClaw viewers

`scripts/configure-openclaw-viewer.mjs` lets an OpenClaw browser profile view a browser-manager
host without a person signing in. It reads the viewer's service credential as JSON on stdin
(`client_id`, `client_secret`, `base_url`, `issuer`, `audience`), exchanges it with exactly that host over HTTPS
(redirects refused), verifies the returned Access JWT and a cookie-only API request, and installs only the short-lived,
host-scoped Access cookie in the profile. The long-lived secret never enters the browser, argv, a file or the output.
The allowed hosts, each pinned to its own dedicated profile, and the Access team come from the environment:

```bash
openclaw browser create-profile --name secure-browser-viewer
openclaw browser --browser-profile secure-browser-viewer open about:blank
export BM_VIEWER_PROFILES='{"https://browsers.example.com":"secure-browser-viewer"}'
export BM_ACCESS_ISSUER=https://<team>.cloudflareaccess.com
<command that prints the credential JSON> | node scripts/configure-openclaw-viewer.mjs secure-browser-viewer
```

Run it again when the Access session expires. The origin independently checks that the JWT's `common_name` is the
configured `BM_ACCESS_SERVICE_TOKEN_ID`.

## Native browsers on macOS

The macOS backend starts installed, sandboxed native headed Chrome with a dedicated profile per browser, streams page
images and accepts only bounded tab, navigation, pointer, key and text commands through the authenticated WebSocket.
browser-manager supplies the tabs and address bar; Chrome's own toolbar, file dialogs and the rest of the desktop are not
streamed. Downloads are disabled. Chrome debugging uses inherited private pipes, never a listening port. No Screen
Recording or Accessibility permission is requested, and your normal Chrome profile is not touched.

Give the Mac its own tunnel and Access application: never run two hosts' connectors on one tunnel, or Cloudflare
load-balances between two different browser lists. From a checkout at a permanent path, with Chrome, Node.js 22.13+ and
`brew install cloudflared`:

```bash
npm ci --ignore-scripts
npm run build
node scripts/install-macos.mjs /absolute/private/browser-manager.env /absolute/private/tunnel-token
browserctl start personal --url https://example.com
```

The installer owns only two user LaunchAgents (`com.ejc3.browser-manager` and `com.ejc3.browser-manager-tunnel`). They
start at that user's login, not before FileVault unlock. Always use this GUI login path: Chrome started directly over
SSH cannot reach the Keychain (`errSecInteractionNotAllowed`) and cannot keep encrypted cookies.

## Operations

- Diagnostics: `systemctl --user status browser-manager browser-manager-tunnel` and
  `journalctl --user -u browser-manager -u browser-manager-tunnel` (prefix
  `DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u)/bus` in a shell without a user bus).
- A 302 to the Access login proves only that Cloudflare answers; check the tunnel's connections to know the host is up.
- Rotating the tunnel token: install the new token file atomically (mode `0600`) and restart only
  `browser-manager-tunnel.service`; the desktops keep running. After a suspected exposure, also invalidate the tunnel's
  existing connections ([Cloudflare's guide](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/remote-tunnel-permissions/#rotate-a-compromised-token)).

## Development

`npm test`, `npm run typecheck` and `npm run build` are the CI gate on Linux and macOS (install `x11vnc` first on
Linux; the native VNC check is skipped on macOS). Tests cover signed Access identities, Origin enforcement, socket
routing, expiry, the CDP bridge and lifecycle, without a Cloudflare account.
`BM_BROWSER_BIN=/path/to/chrome npm run test:live` runs two real sandboxed desktops and the production UI on loopback with
an ephemeral signed test identity (no production auth bypass); screenshots go to `~/browser-manager-ui-artifacts`.

Pull requests are reviewed by Codex, CodeRabbit and Greptile.

## License

MIT
