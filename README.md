# Run T3 Code on a Raspberry Pi and reach it from your phone

T3 Code is a control panel for coding agents. It wraps CLIs like OpenCode, Claude Code, Codex, Cursor, and Grok Build, and lets you start and steer sessions from a web app, a desktop app, or a mobile app. This guide runs the T3 Code server on a Raspberry Pi and makes it reachable from outside your home through Tailscale, so you can control agents from your phone wherever you are.

You do all the heavy lifting on the Pi. The phone is just a client. Projects, files, git state, terminals, and provider logins live on the Pi.

## What you need

- A 64-bit Raspberry Pi OS (or other arm64 Linux) with systemd. The background service step relies on systemd.
- Tailscale, with the Pi in the same tailnet as the phone.
- At least one provider CLI installed and authenticated on the Pi. OpenCode is a good starting point.
- The T3 Code 2.0 mobile app on your phone. The 1.x app cannot connect to a T3 Code 0.1.0 or newer server.

T3 Code itself no longer needs Node. Some provider CLIs still do, so check each one you install.

## 1. Install Tailscale

If Tailscale is already installed, skip this part. If not, install the package for your Pi and join the tailnet:

```bash
tailscale up
```

Confirm the Pi shows up in the tailnet:

```bash
tailscale ip -4
```

This prints the tailnet IP, for example `100.101.102.103`. Note the machine name too, since you use it for the HTTPS address. You get it from:

```bash
tailscale status
```

## 2. Install T3 Code

Use the official install script. It downloads the self-contained `t3` executable from the GitHub release, unpacks it into `~/.t3/runtime/versions/<version>`, and puts a `t3` symlink in `~/.local/bin`:

```bash
curl -fsSL https://t3.codes/install.sh | sh
```

Make sure `~/.local/bin` is on your PATH, then check it works:

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

## 3. Add a provider

T3 Code does not ship provider CLIs. You install each one and authenticate it.

For OpenCode, use OpenCode 2 (2.0.18 or newer). T3 Code 0.1.0 is built for it, and OpenCode 1.x only keeps limited support. Install it with the official script, which puts the binary in `~/.opencode/bin`:

```bash
curl -fsSL https://opencode.ai/v2/install | bash
opencode --version
opencode auth login
```

OpenCode 1 and 2 share one database, and the first OpenCode 2 run converts it. Do not keep both installed. If you have OpenCode 1 from npm, run `npm uninstall -g opencode-ai` before installing OpenCode 2. Plugins written for OpenCode 1 (`@opencode-ai/plugin`) do not load in OpenCode 2.

For Claude Code you install that separately and run:

```bash
claude auth login
```

Minimum versions for T3 Code 0.1.0: Claude Code 2.1.280, Codex 0.159, OpenCode 2.0.18, Grok 1.0.13, Pi 0.80.5. Settings, Providers in the app warns when one is too old.

Run these logins on the Pi, not on the phone. Each provider binary must sit on the PATH that the systemd service uses, or you set the binary path explicitly in the T3 Code settings.

A provider that is not authenticated still shows in the app; it just fails when you start a session. You can add providers later without restarting anything.

## 4. Allow the user to configure Tailscale Serve

The `tailscale serve` command normally needs root. You grant your user the right to manage Serve without sudo once:

```bash
sudo tailscale set --operator=$USER
```

Log out and back in after doing this. With the grant you can set the Serve mapping yourself, which is all this setup needs: one static rule that keeps working across reboots and server updates.

## 5. Install the background service

T3 Code ships a service installer. It creates a systemd user unit that runs `~/.t3/runtime/versions/<version>/t3 __service-launcher`. The launcher starts and self-updates the actual server from the same directory. The server binds to localhost only on port 3773, then Tailscale Serve is the single public door.

```bash
t3 service install
```

User services only start at boot if linger is on for your account, so turn that on once:

```bash
sudo loginctl enable-linger $USER
```

Then start it and mark it enabled:

```bash
systemctl --user enable --now t3code.service
```

Set the Serve mapping. This is done once; the rule persists in tailscaled until you remove it:

```bash
tailscale serve --bg http://127.0.0.1:3773
```

Check it is up:

```bash
systemctl --user status t3code.service
tailscale serve status
```

`tailscale serve status` should show an HTTPS address that proxies to `http://127.0.0.1:3773`, and the port stays fixed across restarts and version updates.

### Coming from the old manual unit

Older versions of this guide had you hand-write a system unit that ran `t3 serve --host 127.0.0.1 --tailscale-serve` directly. Do not keep both. Two servers sharing one T3 home fight over port 3773 and repoint the Serve mapping under each other, and because browser session cookies are scoped by port, every flip logs every client out and demands a fresh pairing code. Remove the old unit if you have it:

```bash
sudo systemctl disable --now t3-code.service
sudo rm /etc/systemd/system/t3-code.service
sudo systemctl daemon-reload
```

### Optional watchdog

On my Pi I also run a small watchdog that kills any T3 server process outside the service's cgroup and restarts the service if its runtime state goes stale. It is what catches a second server before it can cause damage. The script lives at `~/.t3/bin/stray-guard.sh` with a matching `t3-stray-guard.service` and `t3-stray-guard.timer` user units checking every five minutes. It finds server processes with `pgrep -f "^$HOME/\.t3/runtime/versions/[^ ]+/t3 (serve|start)( |$)"`. Anchor the pattern at the start of the command line, so it never matches a shell command that only mentions the path. It is optional; the single-server rule above is the part that actually matters.

## 6. Pair your phone

With the server running, mint a pairing token. The pairing link carries the token in its fragment, so treat it like a password:

```bash
t3 auth pairing create --ttl 30d --label "my-phone" \
  --base-url "https://<your-machine>.<your-tailnet>.ts.net"
```

You get back a pairing URL such as:

```text
https://<your-machine>.<your-tailnet>.ts.net/pair#token=<PAIRING_TOKEN>
```

Open that link on the phone with the T3 Code app open. The app exchanges the token, creates a session, and remembers the environment from then on. You do not need the token again unless you pair another gadget.

To reach it from a browser as well, the same HTTPS address works from any device on the tailnet, and the hosted web app at `https://app.t3.codes` also accepts an HTTPS pair link.

For a quick one-off token you can also run `t3 pair`, which mints a short-lived token and prints it as a QR code in the terminal. The `t3 auth pairing create` form above is the one to use when you want a long-lived token with a label.

## Daily management

```bash
systemctl --user status t3code.service   # Is it running?
tail -f ~/.t3/userdata/logs/boot-service.log   # Live logs
systemctl --user restart t3code.service  # After changing a provider or config
tailscale serve status                   # The served HTTPS mapping
t3 auth pairing create --help            # New tokens for other devices
```

### Updating T3 Code

Run `t3 update --yes` on the Pi to move to a newer release:

```bash
t3 update --yes
```

This downloads the new runtime into `~/.t3/runtime/versions/`, repoints the `~/.local/bin/t3` symlink, rewrites the service unit and state to point at it, and restarts `t3code.service` into the new version. The `--yes` matters over SSH: `t3 update` only asks the "restart now?" prompt when both stdin and stdout are TTYs, and some SSH sessions don't pass that check. Without `--yes` the new files land on disk but the service keeps running the old version, and you have to come back with `t3 service restart` to actually switch.

Check that the CLI and the service agree afterward:

```bash
t3 --version
t3 service status
```

`t3 service status` should report `installed · t3@<version>` with the same version and no `Next:` line. If it says `needs an update or repair`, run `t3 service install` to rewrite the unit for the version you are on.

Old versions stay in `~/.t3/runtime/versions/` after an update. Once the new one works, you can delete the old directories to free space.

### Moving to 0.1.0 (Orchestrator V2)

0.1.0 replaces the part of T3 Code that runs agents. There is no manual migration step, but check these:

- On first start, the server copies `~/.t3/userdata/state.sqlite` to `statev2.sqlite` and imports your threads into the copy. Messages come across. Old checkpoints, diffs, tool activity and live provider sessions do not. The old `state.sqlite` stays untouched, so you can delete it once you are sure you will not go back to 0.0.x.
- Update every client. The 2.0 mobile app, the desktop app and every other server you connect to must be on 0.1.0 or newer, or the connection is refused.
- The jump from 0.0.x is run by the old CLI, so it writes the service unit from the old template. Afterward run `t3 service status` from the new `t3`. If it says `needs an update or repair`, run `t3 service install`.
- Move OpenCode to 2.0.18 or newer (see step 3).
- Any script of your own that touches `state.sqlite` or matches the old `bin.mjs serve` process needs to point at `statev2.sqlite` and the `t3 serve` executable instead.

To stop sharing the HTTPS address entirely:

```bash
tailscale serve off
```

## Notes and gotchas

- The backend binds to `127.0.0.1`. Do not change it to `0.0.0.0`. Tailscale Serve is the intended path in, and keeping the backend local cuts down accidental LAN exposure.
- Run exactly one T3 server against a given T3 home. The failure mode when two run is not an error message, it is your devices getting logged out over and over: both servers want port 3773, the loser bounces to a random port and repoints Serve, and port-scoped session cookies stop matching.
- The mobile app connects over the tailnet, so the phone needs the Tailscale VPN on. The HTTPS address only resolves when the phone is on the tailnet or has the subnet routes it needs.
- Version mismatch shows up as a prompt in the app and under Settings, Connections. With the service installed the launcher handles updates itself and restarts into the new version.
- If your device disappears from app.t3.codes after an update, run `t3 --version` and `t3 service status` on the Pi. A CLI on a different version than the service usually means an old npm install of `t3` is earlier on the PATH (see step 2). If the status says `needs an update or repair`, follow its `Next:` line, then check again. It should report `installed · t3@<version>` with no `Next:` line.
- `t3 connect status` reports saved configuration only, not live reachability. A green-looking `t3 connect status` does not prove the device is online; check `t3 service status` and `systemctl --user is-active t3code.service` for the live state.
- The pairing URL contains a credential. Do not share it. It remains usable until it expires or you revoke it.

That is the whole setup. A service that survives reboots, with an HTTPS entry point and a pairing flow your phone already knows how to talk to.