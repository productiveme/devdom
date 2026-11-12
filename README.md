# devdom

Hotel/chalet-style local domains for any project folder:

```sh
cd ~/Projects/my-app
devd hook 3000        # → https://my-app.dev.localhost
devd unhook           # remove it again
```

One shared Caddy reverse proxy + one dnsmasq wildcard. No config per app, real
trusted HTTPS (internal Caddy CA via Caddy's automatic local TLS), and hooks
survive reboots because each hook is just a file that Caddy imports.

Works with any port — Node, Vite, Go, Docker-published ports, whatever is
listening on `localhost:<port>`.

## Install

```sh
npm install -g devdom
```

Requires macOS (or Linux) with [Caddy](https://caddyserver.com) and
[dnsmasq](https://www.thekelleys.org.uk/dnsmasq/doc.html) installed:

```sh
brew install caddy dnsmasq
```

## One-time setup

dnsmasq needs one wildcard so that every `*.dev.localhost` (and any other
`*.localhost`) resolves to 127.0.0.1. `devd install` checks all of this and
prints the exact fixes; the manual steps are:

**1. Configure dnsmasq** — merge `devdom`'s bundled template
(`$(npm root -g)/devdom/templates/dnsmasq.conf`) into
`/opt/homebrew/etc/dnsmasq.conf` (Apple Silicon) or `/usr/local/etc/dnsmasq.conf`
(Intel):

```
port=5754
address=/localhost/127.0.0.1
no-resolv
server=1.1.1.1
```

then `brew services restart dnsmasq`.

**2. Route `*.localhost` queries to dnsmasq**:

```sh
sudo mkdir -p /etc/resolver
printf 'nameserver 127.0.0.1\nport 5754\n' | sudo tee /etc/resolver/localhost
```

**3. Verify and start caddy**:

```sh
devd install   # prints ✓/✗ per requirement
devd start     # sudo caddy start --config ~/.devd/Caddyfile
```

On first start Caddy generates an internal CA and adds it to your system trust
store — that's the one point it needs `sudo`, and browsers will show a trusted
padlock on every `*.dev.localhost` from then on.

## Usage

```sh
devd hook 3000                     # <folder>.dev.localhost → localhost:3000
devd hook 3001 api.localhost      # explicit full domain passes through
PORT=3000 devd hook               # port from $PORT
devd ls                           # list hooks
devd unhook                       # in the same folder, or:
devd unhook api.localhost
devd status                       # resolver + caddy checks
devd start                        # start the shared caddy (needs sudo once)
```

Hooks live as files in `~/.devd/routes/<domain>.caddy` — the *files are the
registry*, so `devd ls` works after reboots and nothing needs bookkeeping.

### Project-owned domains

If a project has its own `*.localhost` names (e.g. an internal API host), ship
the route snippet in the repo and symlink it in — the hook travels with the
code:

```sh
# repo: caddy/local.caddy → myapi.internal.localhost { reverse_proxy localhost:9000 }
ln -sf "$PWD/caddy/local.caddy" ~/.devd/routes/myapi.internal.localhost.caddy
```

## How it works

- **dnsmasq** answers `*.localhost` with 127.0.0.1 via a single suffix rule —
  no per-app DNS entries ever. macOS sends those queries to it through
  `/etc/resolver/localhost` on port 5754 (unprivileged, no root port binding).
- **Caddy** imports every `~/.devd/routes/*.caddy` file and reloads
  through its admin API (localhost:2019), so hook/unhook is instant and never
  needs `sudo`. Site blocks on `*.localhost` automatically get
  Caddy-issued internal certificates.

## Notes & gotchas

- `.dev.localhost` stays inside the `.localhost` special-use TLD (RFC 6761):
  browsers already resolve `*.localhost` natively, dnsmasq just makes `curl`,
  CI scripts, etc. resolve it too — and nothing inside `.localhost` can ever
  collide with a real public domain (avoid bare `.dev`, which is HSTS-preloaded).
- **Docker containers** don't read `/etc/resolver`. For an app *inside*
  Docker that calls `api.myproject.localhost`, add
  `extra_hosts: ["api.myproject.localhost:host-gateway"]` to its compose;
  Caddy listens on all interfaces and routes by SNI.
- Windows: npm installs fine but the bash CLI won't run (os fields guard this).

## License

[MIT](./LICENSE)
