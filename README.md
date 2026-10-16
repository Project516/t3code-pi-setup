# Run T3 Code on a Raspberry Pi and reach it from your phone

T3 Code is a control panel for coding agents. It wraps CLIs like OpenCode, Claude Code, Codex, Cursor, Pi, and Grok Build, and lets you start and steer sessions from a web app, a desktop app, or a mobile app. This guide runs the T3 Code server on a Raspberry Pi as a background service and makes it reachable from your phone wherever you are, through T3 Connect, Tailscale, or both.

You do all the heavy lifting on the Pi. The phone is just a client. Projects, files, git state, terminals, and provider logins live on the Pi.

This guide follows T3 Code 0.1.0 and the upstream docs in [`docs/user`](https://github.com/pingdotgg/t3code/tree/main/docs/user). When they disagree, the upstream docs win.

## What you need

- A 64-bit Raspberry Pi OS (or other arm64 Linux) with systemd. T3 Code ships a `linux-arm64` executable, so 32-bit boards like the Pi Zero are out.
- `curl` or `wget`, `tar`, and `sha256sum`. T3 Code itself does not need Node. Some provider CLIs still do, so check each one you install.
- At least one provider CLI installed and authenticated on the Pi. OpenCode is a good starting point.
- The T3 Code 2.0 mobile app on your phone ([App Store](https://apps.apple.com/us/app/t3-code-remote-claude-more/id6787819824), [Google Play](https://play.google.com/store/apps/details?id=com.t3tools.t3code)). The 1.x app cannot connect to a 0.1.0 or newer server.
- For the Tailscale route only: Tailscale, with the Pi in the same tailnet as the phone.

## 1. Install T3 Code

Use the official install script:

```bash
curl -fsSL https://t3.codes/install.sh | sh
```

It downloads the self-contained `t3` executable from the GitHub release, unpacks it into `~/.t3/runtime/versions/<version>`, and puts a `t3` symlink in `~/.local/bin`. If your shell then says `command not found`, `~/.local/bin` is not on your PATH yet. The installer prints the line to add. Check it works:

```bash
t3 --version
```

Install with the script, not with `npm install -g t3`. `t3 update` repoints the `~/.local/bin/t3` symlink at each new version, so the CLI and the service always run the same build. An npm global install is a separate copy that `t3 update` never touches. It keeps running the old version while the service moves on, and the two drift apart.

If you installed with npm before, remove it and run the script once:

```bash
npm uninstall -g t3
curl -fsSL https://t3.codes/install.sh | sh
hash -r && t3 --version
```

If `t3 --version` still prints the old version, another `t3` comes earlier on your PATH. `type -a t3` shows which one.

## 2. Add a provider

T3 Code does not ship provider CLIs. You install each one on the Pi and authenticate it there, not on the phone.

For OpenCode, use OpenCode 2 (2.0.18 or newer). T3 Code 0.1.0 is built for it, and OpenCode 1.x only keeps limited support. OpenCode's own installer puts the binary in `~/.opencode/bin` and needs no Node:

```bash
curl -fsSL https://opencode.ai/v2/install | bash
opencode --version
opencode auth login
```

`npm install -g @opencode/cli` works too if you prefer npm. OpenCode 1 and 2 share one database, and the first OpenCode 2 run converts it, so do not keep both installed. If you have OpenCode 1 from npm, run `npm uninstall -g opencode-ai` first. Plugins written for OpenCode 1 (`@opencode-ai/plugin`) do not load in OpenCode 2.

For Claude Code, install it and run:

```bash
claude auth login
```

The [provider table](https://github.com/pingdotgg/t3code/blob/main/docs/user/install.md#providers) upstream lists the login command for every provider. Minimum versions for 0.1.0: Claude Code 2.1.280, Codex 0.159, OpenCode 2.0.18, Grok 1.0.13, Pi 0.80.5. **Settings, Providers** in the app warns when one is too old, and its **Update now** button updates a provider through whatever installed it.

Each provider binary must be on the PATH the service sees, or you set its **Binary path** in the provider settings. Set it explicitly for anything installed through a version manager like nvm. A provider that is not authenticated still shows in the app; it just fails when you start a session.

## 3. Install the background service

```bash
t3 service install
t3 service status
```

`t3 service install` writes a systemd user unit that runs `~/.t3/runtime/versions/<version>/t3 __service-launcher`, enables lingering so the service starts at boot and survives logout, and starts it. If enabling linger needs root, it prints the command to run. Run only that command with sudo:

```bash
sudo loginctl enable-linger "$(id -un)"
```

`t3 service status` should report `installed · t3@<version>` with no `Next:` line. If it lists a problem such as `linger-disabled` or `service-stopped`, follow the upstream [troubleshooting table](https://github.com/pingdotgg/t3code/blob/main/docs/user/background-service.md#troubleshooting). Running `t3 service install` again repairs a broken service.

The server binds to `127.0.0.1:3773`. Nothing reaches it from the network until you set up one of the routes in step 4.

### Coming from the old manual unit

Older versions of this guide had you hand-write a system unit that ran `t3 serve --host 127.0.0.1 --tailscale-serve` directly. Do not keep both. Two servers sharing one T3 home fight over port 3773 and repoint the Serve mapping under each other, and because browser session cookies are scoped by port, every flip logs every client out and demands a fresh pairing code. Remove the old unit if you have it:

```bash
sudo systemctl disable --now t3-code.service
sudo rm /etc/systemd/system/t3-code.service
sudo systemctl daemon-reload
```

### Optional watchdog

On my Pi I also run a small watchdog that kills any T3 server process outside the service's cgroup and restarts the service if its runtime state goes stale. It is what catches a second server before it can cause damage. The script lives at `~/.t3/bin/stray-guard.sh` with a matching `t3-stray-guard.service` and `t3-stray-guard.timer` user units checking every five minutes. It finds server processes with `pgrep -f "^$HOME/\.t3/runtime/versions/[^ ]+/t3 (serve|start)( |$)"`. Anchor the pattern at the start of the command line, so it never matches a shell command that only mentions the path. It is optional; the single-server rule above is the part that actually matters.

## 4. Connect your phone

There are two routes. T3 Connect is the one upstream recommends, and it needs no VPN. Tailscale keeps all traffic inside your tailnet. You can set up both. The app keeps every route for a machine and uses the first one that answers.

### T3 Connect

```bash
t3 connect
```

Over SSH it prints a browser link and a short code. Open the link on any device, check the code matches, and approve. The CLI continues on its own. Setup offers to install the background service; you already did that in step 3.

On the phone, sign in to the same T3 Connect account and pick the Pi's environment. Once connected, the app also learns the Pi's Tailscale address, and its LAN address when network access is allowed, and adds them as routes.

For push notifications and Live Activities on the phone when an agent needs you, run `t3 connect publish`. It is off by default.

`t3 connect status` shows saved setup only, not whether the Pi is online right now. `t3 connect unlink` turns exposure off and keeps your login.

### Tailscale HTTPS

`tailscale serve` normally needs root. Grant your user the right to manage Serve without sudo, once, then log out and back in:

```bash
sudo tailscale set --operator=$USER
```

Then publish the running service on your tailnet and pair in one step:

```bash
t3 pair --tailscale --label "my-phone"
```

This creates a Tailscale Serve mapping from `https://<your-machine>.<your-tailnet>.ts.net/` to the local server and prints a pairing URL and QR code. Scan the QR code with the app, or paste the URL into **Add environment**. The phone needs the Tailscale VPN on. The mapping persists across restarts and version updates. Check it with `tailscale serve status`, and remove it with:

```bash
tailscale serve --https=443 off
```

Pairing links are one-time and expire after 5 minutes by default (`--ttl` changes that). A paired device does not need the link again. Make a fresh one for each new device. The hosted web app at `https://app.t3.codes` also accepts this HTTPS pairing link.

Treat pairing URLs and approval codes like passwords. Revoke devices under **Settings, Connections** or with `t3 auth --help`.

## Daily management

```bash
t3 service status                         # Installed version, problems, log path
systemctl --user status t3code.service    # Is it running?
tail -f ~/.t3/userdata/logs/boot-service.log   # Live logs
t3 service restart                        # After changing a provider or config
t3 pair --tailscale                       # Pair another device over Tailscale
```

## Updating T3 Code

The update button at the bottom of the app's sidebar can update the Pi's server and providers remotely, from the phone or another machine. On the Pi itself:

```bash
t3 update --yes
```

This downloads the new runtime into `~/.t3/runtime/versions/`, repoints the `~/.local/bin/t3` symlink, rewrites the service unit, and restarts `t3code.service` into the new version. Restarting interrupts running agents. **Settings, General, Continue threads after restarts** resumes supported threads afterward. The `--yes` matters over SSH. `t3 update` only asks "restart now?" when both stdin and stdout are TTYs, and without `--yes` the new files land on disk while the service keeps running the old version until `t3 service restart`.

Check that the CLI and the service agree afterward:

```bash
t3 --version
t3 service status
```

Both should show the same version, and the status should have no `Next:` line. If it says `needs an update or repair`, run `t3 service install`.

Old versions stay in `~/.t3/runtime/versions/` after an update. Once the new one works, delete the old directories to free space.

### Moving to 0.1.0 (Orchestrator V2)

0.1.0 replaces the part of T3 Code that runs agents. There is no manual migration step, but check these. Upstream covers the details in [Threads from older T3 Code versions](https://github.com/pingdotgg/t3code/blob/main/docs/user/thread-migration.md).

- On first start, the server copies `~/.t3/userdata/state.sqlite` to `statev2.sqlite` and imports your threads into the copy. Messages come across. Old checkpoints, diffs, tool activity and live provider sessions do not. The old `state.sqlite` stays untouched, so you can delete it once you are sure you will not go back to 0.0.x.
- Update every client. The 2.0 mobile app, the desktop app and every other server you connect to must be on 0.1.0 or newer, or the connection is refused.
- The jump from 0.0.x is run by the old CLI, so it writes the service unit from the old template. Afterward run `t3 service status` from the new `t3`. If it says `needs an update or repair`, run `t3 service install`.
- If `t3` came from npm, replace it with the install script (step 1). Otherwise it stays on 0.0.x.
- Move OpenCode to 2.0.18 or newer (step 2).
- Any script of your own that touches `state.sqlite` or matches the old `bin.mjs serve` process needs to point at `statev2.sqlite` and the `t3 serve` executable instead.

## Notes and gotchas

- The backend binds to `127.0.0.1`. Do not change it to `0.0.0.0`. T3 Connect and Tailscale Serve are the intended ways in, and keeping the backend local cuts down accidental LAN exposure.
- Run exactly one T3 server against a given T3 home. The failure mode when two run is not an error message, it is your devices getting logged out over and over: both servers want port 3773, the loser bounces to a random port and repoints Serve, and port-scoped session cookies stop matching.
- Version mismatch shows up as a notice in the app and under **Settings, Connections**. Update the machine the notice names.
- If your device disappears from the app after an update, run `t3 --version` and `t3 service status` on the Pi. A CLI on a different version than the service usually means an old npm install of `t3` is earlier on the PATH (step 1). If the status says `needs an update or repair`, follow its `Next:` line, then check again.
- `t3 connect status` reports saved configuration only, not live reachability. Check `t3 service status` and `systemctl --user is-active t3code.service` for the live state.
- `t3 uninstall` removes the service, the `t3` symlink and every downloaded version. Projects and threads under `~/.t3/userdata` stay.

That is the whole setup. The service survives reboots and updates itself, and your phone reaches it through T3 Connect, Tailscale, or both.
