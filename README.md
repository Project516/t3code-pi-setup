# Run T3 Code on a Raspberry Pi and reach it from your phone

T3 Code is a control panel for coding agents. It wraps CLIs like OpenCode, Claude Code, Codex, Cursor, and Grok Build, and lets you start and steer sessions from a web app, a desktop app, or a mobile app. This guide runs the T3 Code server on a Raspberry Pi and makes it reachable from outside your home through Tailscale, so you can control agents from your phone wherever you are.

You do all the heavy lifting on the Pi. The phone is just a client. Projects, files, git state, terminals, and provider logins live on the Pi.

## What you need

- A Raspberry Pi running Linux and systemd (the background service step relies on it).
- Node.js 22.16 or newer, 23.11 or newer, or 24.10 or newer.
- Tailscale, with the Pi in the same tailnet as the phone.
- At least one provider CLI installed and authenticated on the Pi. OpenCode is a good starting point.
- The T3 Code mobile app on your phone.

## 1. Install Node and Tailscale

If Node and Tailscale are already installed, skip this part. If not, install both before anything else.

Node version managers like nvm work fine. Whatever you use, make sure `node` resolves to a version in the range above when a non-interactive service starts it. The background service does not always pick up your shell profile, so a default alias or a version already on the system PATH is safer than one that only loads when you open a terminal.

For Tailscale, install the package for your Pi and join the tailnet:

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

Install the `t3` CLI globally. This is the bootstrap: it gives you the commands that install the background service, mint pairing tokens, and manage updates.

```bash
npm install -g --allow-scripts=node-pty,msgpackr-extract t3@latest
```

The `--allow-scripts` flag matters. Two native packages, `node-pty` and `msgpackr-extract`, build helper scripts at install time. Without the flag some npm setups block those scripts and the terminal features break.

Check it works:

```bash
t3 --version
```

The global install is only the bootstrap. The server itself runs from a managed copy under `~/.t3/runtime/versions/` that the service keeps up to date, so updating later does not depend on repeating this npm command.

## 3. Add a provider

T3 Code does not ship provider CLIs. You install each one and authenticate it. For OpenCode:

```bash
opencode auth login
```

For Claude Code you install that separately and run:

```bash
claude auth login
```

Run these logins on the Pi, not on the phone. Each provider binary must sit on the PATH that the systemd service uses, or you set the binary path explicitly in the T3 Code settings.

A provider that is not authenticated still shows in the app; it just fails when you start a session. You can add providers later without restarting anything.

## 4. Allow the user to configure Tailscale Serve

The `tailscale serve` command normally needs root. You grant your user the right to manage Serve without sudo once:

```bash
sudo tailscale set --operator=$USER
```

Log out and back in after doing this. With the grant you can set the Serve mapping yourself, which is all this setup needs: one static rule that keeps working across reboots and server updates.

## 5. Install the background service

T3 Code ships a service installer. It creates a systemd user unit that runs a small launcher, and the launcher starts and self-updates the actual server under `~/.t3/runtime/versions/`. The server binds to localhost only on port 3773, then Tailscale Serve is the single public door.

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

On my Pi I also run a small watchdog that kills any T3 server process outside the service's cgroup and restarts the service if its runtime state goes stale. It is what catches a second server before it can cause damage. The script lives at `~/.t3/bin/stray-guard.sh` with a matching `t3-stray-guard.service` and `t3-stray-guard.timer` user units checking every five minutes. It is optional; the single-server rule above is the part that actually matters.

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

The service updates itself. New server versions land under `~/.t3/runtime/versions/` and the launcher restarts into them, so there is no separate update step on the Pi.

To stop sharing the HTTPS address entirely:

```bash
tailscale serve off
```

## Notes and gotchas

- The backend binds to `127.0.0.1`. Do not change it to `0.0.0.0`. Tailscale Serve is the intended path in, and keeping the backend local cuts down accidental LAN exposure.
- Run exactly one T3 server against a given T3 home. The failure mode when two run is not an error message, it is your devices getting logged out over and over: both servers want port 3773, the loser bounces to a random port and repoints Serve, and port-scoped session cookies stop matching.
- The mobile app connects over the tailnet, so the phone needs the Tailscale VPN on. The HTTPS address only resolves when the phone is on the tailnet or has the subnet routes it needs.
- Version mismatch shows up as a prompt in the app and under Settings, Connections. With the service installed the launcher handles updates itself and restarts into the new version.
- The pairing URL contains a credential. Do not share it. It remains usable until it expires or you revoke it.

That is the whole setup. A service that survives reboots, with an HTTPS entry point and a pairing flow your phone already knows how to talk to.