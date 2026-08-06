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

Install the `t3` CLI globally so a backup service can use a stable path:

```bash
npm install -g --allow-scripts=node-pty,msgpackr-extract t3@latest
```

The `--allow-scripts` flag matters. Two native packages, `node-pty` and `msgpackr-extract`, build helper scripts at install time. Without the flag some npm setups block those scripts and the terminal features break.

Check it works:

```bash
t3 --version
```

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

Log out and back in, or restart the T3 Code service after doing this. This single grant lets the T3 Code server reconfigure Serve every time it starts, which keeps the HTTPS address working after a reboot.

## 5. Create the systemd service

T3 Code runs its own server, and a systemd unit keeps it alive and starts it on boot. The unit binds the backend to localhost only, then Tailscale Serve is the single public door.

Save this as `/home/<user>/t3-code.service`:

```ini
[Unit]
Description=T3 Code server
Wants=network-online.target tailscaled.service
After=network-online.target tailscaled.service

[Service]
Type=simple
User=<your-user>
WorkingDirectory=/home/<your-user>
Environment=PATH=/home/<your-user>/.local/bin:/usr/bin:/bin
ExecStart=/usr/bin/node /path/to/t3/dist/bin.mjs serve --host 127.0.0.1 --tailscale-serve
Restart=on-failure
RestartSec=5
KillSignal=SIGINT

[Install]
WantedBy=multi-user.target
```

Two things to adjust to your box:

- `--tailscale-serve` tells the server to expose itself over the tailnet via HTTPS.
- Point the ExecStart node binary at the real `node` path and the real t3 entry file. Under nvm this is something like `/home/<user>/.nvm/versions/node/v24/bin/node` and `/home/<user>/.nvm/.../lib/node_modules/t3/dist/bin.mjs`. If `t3` came from a plain global install, `npm root -g` finds the package dir.

Install and start it:

```bash
sudo cp /home/<user>/t3-code.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable t3-code
sudo systemctl start t3-code
```

Check it is up and listening:

```bash
sudo systemctl status t3-code
tailscale serve status
```

`tailscale serve status` should show an HTTPS address that proxies to `http://127.0.0.1:3773`.

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

## Daily management

```bash
sudo systemctl status t3-code        # Is it running?
journalctl -u t3-code -f             # Live logs
sudo systemctl restart t3-code       # After changing the unit or a provider
tailscale serve status               # The served HTTPS mapping
t3 auth pairing create --help        # New tokens for other devices
```

To stop sharing the HTTPS address entirely:

```bash
tailscale serve off
```

## Notes and gotchas

- The backend binds to `127.0.0.1`. Do not change it to `0.0.0.0`. Tailscale Serve is the intended path in, and keeping the backend local cuts down accidental LAN exposure.
- The mobile app connects over the tailnet, so the phone needs the Tailscale VPN on. The HTTPS address only resolves when the phone is on the tailnet or has the subnet routes it needs.
- Keep the unit file as a copy in your home directory. When you want to change flags, edit that file, copy it back to `/etc/systemd/system/`, and `daemon-reload` plus `restart`. Editing the copy in `/etc` gets overwritten on system updates.
- Version mismatch shows up as a prompt in the app and under Settings, Connections. The server can often update itself and reconnect; on a Pi a manual install update via one of the `t3` service commands is also fine.
- The pairing URL contains a credential. Do not share it. It remains usable until it expires or you revoke it.

That is the whole setup. A service that survives reboots, with an HTTPS entry point and a pairing flow your phone already knows how to talk to.