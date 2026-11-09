## Claude Code through a proxy

Claude Code always goes through an upstream proxy, with no direct fallback. Firefox sends only Claude sites through the same proxy. A separate Firefox profile, `proxy`, sends all its traffic through it.

```
Claude Code ──────────────────────────┐
Firefox (Claude sites only, via PAC) ─┼─> Privoxy 127.0.0.1:8120 ─> SOCKS5 upstream (user/pass) ─> internet
Firefox profile "proxy" (all sites) ──┘

OpenSnitch: the Claude Code process may only connect to loopback (i.e. Privoxy)
```

Placeholders used below: `PROXY_USER`, `PROXY_PASS`, `PROXY_HOST`, `PROXY_SOCKS_PORT`.
Credentials live in exactly one place — `~/.config/privoxy/config` (mode 600).

Why Privoxy instead of pointing apps at the proxy directly:

* Claude Code supports HTTP(S) proxies, but not SOCKS.
* One place to swap the upstream for every client.
* Credentials stay out of `HTTPS_PROXY`, which every child process of Claude Code would inherit.

> Inside Claude Code, `! sudo ...` fails (no TTY for the password). Use `! pkexec ...` instead — it asks for the password in a GUI dialog.

### 1. Privoxy

```sh
sudo dnf install privoxy
```

The packaged system service (`privoxy.service`, port 8118) stays disabled — a separate user instance is used instead, so changing the proxy never needs root.

`~/.config/privoxy/config`:

```
confdir /home/evgn/.config/privoxy
templdir /etc/privoxy/templates
listen-address 127.0.0.1:8120

# Upstream: SOCKS5 with auth; DNS is resolved by the proxy
forward-socks5 / PROXY_USER:PROXY_PASS@PROXY_HOST:PROXY_SOCKS_PORT .

# Forward only, no filtering
toggle 0
enable-remote-toggle 0
enable-edit-actions 0
enable-remote-http-toggle 0

# Long streaming API responses
socket-timeout 600
keep-alive-timeout 60

# A whole browser profile goes through here; the default 128 runs out
max-client-connections 512

# Log startup, warnings and errors to the journal
debug 4096
debug 8192
```

```sh
chmod 600 ~/.config/privoxy/config
```

`~/.config/systemd/user/privoxy-claude.service`:

```ini
[Unit]
Description=Privoxy 127.0.0.1:8120 -> upstream proxy (for Claude Code etc.)
After=network-online.target

[Service]
ExecStart=/usr/sbin/privoxy --no-daemon %h/.config/privoxy/config
Restart=always
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=default.target
```

`templdir` points at the packaged error pages, so a failed request shows Privoxy's error page instead of an empty response. `LimitNOFILE` is needed for `max-client-connections 512`: each connection takes two file descriptors (client and upstream), and the default soft limit is 1024. When the limit is hit, Privoxy rejects everyone, Claude Code included. The journal then shows `Maximum number of connections reached`.

```sh
systemctl --user daemon-reload
systemctl --user enable --now privoxy-claude
curl -x http://127.0.0.1:8120 https://ipinfo.io/json   # must show the proxy's IP and country
```

### 2. Claude Code

Only switch Claude Code over **after** Privoxy is verified — a running session picks up `settings.json` changes live and loses connectivity if the proxy is down.

Add to every profile in use — `~/.claude/settings.json` and the one under `CLAUDE_CONFIG_DIR` (here `~/.claude-work/settings.json`):

```json
"env": {
  "HTTPS_PROXY": "http://127.0.0.1:8120",
  "HTTP_PROXY": "http://127.0.0.1:8120",
  "NO_PROXY": "localhost,127.0.0.1,::1,.local"
},
```

Commands Claude Code runs in Bash inherit these variables, so `git` over HTTPS, `curl`, `npm`, `composer` etc. also go through the proxy (except `localhost` and `*.local`).

### 3. OpenSnitch: block direct connections from Claude Code

```sh
sudo dnf install opensnitch
```

Changes in `/etc/opensnitchd/default-config.json`:

* `"DefaultAction": "allow"`, `"InterceptUnknown": false` — package defaults; other apps are not affected and nothing prompts.
* `"ProcMonitorMethod": "proc"` — the Fedora package ships no eBPF modules.
* `"Internal": {"FlushConnsOnStart": false}` — otherwise starting the daemon drops every existing connection (VPN included).

Rules in `/etc/opensnitchd/rules/`. Rules are checked in name order, and a matching rule with `precedence`, `deny` or `reject` stops the check. So the `000-` allow rules are checked first, and `001-` rejects everything else:

| File | Effect |
|---|---|
| `000-claude-allow-loopback.json` | Claude Code → `127.0.0.0/8` allowed |
| `000-claude-allow-loopback6.json` | Claude Code → `::1/128` allowed |
| `001-claude-reject-direct.json` | Claude Code → anything else rejected |

`000-claude-allow-loopback.json` (the `loopback6` one is identical with `::1/128`):

```json
{
  "name": "000-claude-allow-loopback",
  "description": "Claude Code: only via local Privoxy (kill switch)",
  "enabled": true,
  "precedence": true,
  "nolog": false,
  "action": "allow",
  "duration": "always",
  "operator": {
    "type": "list",
    "operand": "list",
    "sensitive": false,
    "data": "[{\"type\": \"regexp\", \"operand\": \"process.path\", \"sensitive\": false, \"data\": \"^/home/evgn/\\\\.local/share/claude/versions/\", \"list\": null}, {\"type\": \"network\", \"operand\": \"dest.network\", \"sensitive\": false, \"data\": \"127.0.0.0/8\", \"list\": null}]",
    "list": [
      {"type": "regexp", "operand": "process.path", "sensitive": false, "data": "^/home/evgn/\\.local/share/claude/versions/", "list": null},
      {"type": "network", "operand": "dest.network", "sensitive": false, "data": "127.0.0.0/8", "list": null}
    ]
  }
}
```

`001-claude-reject-direct.json`:

```json
{
  "name": "001-claude-reject-direct",
  "description": "Claude Code: only via local Privoxy (kill switch)",
  "enabled": true,
  "precedence": true,
  "nolog": false,
  "action": "reject",
  "duration": "always",
  "operator": {"type": "regexp", "operand": "process.path", "sensitive": false, "data": "^/home/evgn/\\.local/share/claude/versions/", "list": null}
}
```

```sh
sudo systemctl enable --now opensnitchd   # rules are hot-reloaded on change
```

Notes:

* The path regex matches the native Claude Code install, auto-updates included (`readlink /proc/<pid>/exe` shows the real path).
* A broken allow rule cuts off the running Claude Code session. Before applying the rules to Claude, test the same rule structure on `^/usr/bin/curl$`: `curl -x http://127.0.0.1:8120 ...` must work, a direct `curl` must fail.
* Emergency stop: `sudo systemctl stop opensnitchd` (from Claude Code: `! pkexec systemctl stop opensnitchd`).
* `QueueBypass` is `true`: if the daemon stops, traffic passes unchecked. Claude Code still uses the proxy through its settings.
* Commands Claude Code runs in Bash are separate processes and are not restricted by these rules.
* `opensnitch-ui` comes with the package but is not autostarted. Once launched, it starts prompting for every app.

### 4. Firefox: only Claude sites through the proxy

`~/.config/privoxy/claude.pac`:

```js
var CLAUDE_DOMAINS = [
  "claude.ai",
  "claude.com",
  "anthropic.com",
  "claudeusercontent.com"   // artifacts on claude.ai are served from here
];

function FindProxyForURL(url, host) {
  host = host.toLowerCase();
  for (var i = 0; i < CLAUDE_DOMAINS.length; i++) {
    var d = CLAUDE_DOMAINS[i];
    if (host === d || dnsDomainIs(host, "." + d)) {
      return "PROXY 127.0.0.1:8120";   // no "; DIRECT" — no fallback
    }
  }
  return "DIRECT";
}
```

`user.js` in the Firefox profile directory (`about:profiles` → Root Directory, e.g. `~/.config/mozilla/firefox/<id>.default-release*/`):

```js
user_pref("network.proxy.type", 2);
user_pref("network.proxy.autoconfig_url", "file:///home/evgn/.config/privoxy/claude.pac");
// Don't go direct if the proxy is down
user_pref("network.proxy.failover_direct", false);
// WebRTC on proxied pages only through the proxy (no direct UDP)
user_pref("media.peerconnection.ice.proxy_only_if_behind_proxy", true);
```

Fully quit Firefox (`pgrep firefox` prints nothing) and start it again. Without a restart, the PAC can be reloaded in Settings → Network Settings → Reload.

Check: https://claude.ai/cdn-cgi/trace must show the proxy's IP. https://www.cloudflare.com/cdn-cgi/trace must show the usual one.

Caveats:

* A syntax error in the PAC sends everything direct — Claude sites included.
* A new profile ("Refresh Firefox", Flatpak or Mozilla's own build) has no `user.js`, and Claude sites silently go direct.
* While `user.js` exists, manual proxy changes in Firefox settings are reverted on restart.
* Chrome is not configured.

### 5. Firefox: a separate profile with all traffic through the proxy

For the occasional case when other sites, not just Claude, should go through the proxy. A separate profile rather than a toggle in the main one: the main profile's Claude-only setup stays untouched, the windows can't be confused, and cookies and logins don't mix between the real IP and the proxy IP.

```sh
firefox -CreateProfile proxy   # creates ~/.config/mozilla/firefox/<id>.proxy; the default profile doesn't change
```

`user.js` in the new profile directory:

```js
user_pref("network.proxy.type", 1);
user_pref("network.proxy.http", "127.0.0.1");
user_pref("network.proxy.http_port", 8120);
user_pref("network.proxy.ssl", "127.0.0.1");
user_pref("network.proxy.ssl_port", 8120);
// Don't go direct if the proxy is down
user_pref("network.proxy.failover_direct", false);
// WebRTC only through the proxy (no direct UDP)
user_pref("media.peerconnection.ice.proxy_only_if_behind_proxy", true);
// No DNS prefetch: the proxy resolves names, the local DNS doesn't see them
user_pref("network.dns.disablePrefetch", true);
```

Optional, so its windows stand out: before the first start, create `prefs.js` in the profile directory with a different theme. Unlike `user.js`, it can be changed later in the UI.

```js
user_pref("extensions.activeThemeID", "firefox-alpenglow@mozilla.org");
```

Launcher, [`firefox-proxy.desktop`](firefox-proxy.desktop):

```sh
curl -o ~/.local/share/applications/firefox-proxy.desktop https://raw.githubusercontent.com/evgenygorchakov/env/main/firefox-proxy.desktop
```

`MOZ_APP_REMOTINGNAME=firefox-proxy` in its `Exec` gives the profile its own Wayland app_id. The profile gets its own dock icon, and links opened from other apps still go to the main profile. The launcher has no `MimeType`, so it never becomes a link handler.

Check: in "Firefox (прокси)", https://ipinfo.io/json must show the proxy's IP. In the main Firefox it must show the usual one.

Notes:

* Bookmarks, logins and extensions are separate per profile.
* If Privoxy is down, pages in this profile don't open (no direct fallback).
* Some Russian sites drop part of the connections from the proxy's foreign IP. For example, https://ip.platipomiru.com/ (Yandex Cloud) times out about every other attempt, even straight through the upstream SOCKS without Privoxy. Open such sites in the main profile.

### Change the proxy

Edit the `forward-socks5` line in `~/.config/privoxy/config`, then:

```sh
systemctl --user restart privoxy-claude
```

No client needs a restart.

### Troubleshooting

```sh
systemctl --user status privoxy-claude
journalctl --user -u privoxy-claude -n 30
ss -ltnp | grep 8120
curl -x http://127.0.0.1:8120 https://ipinfo.io/json
systemctl status opensnitchd
sudo tail -n 50 /var/log/opensnitchd.log
```

### Rollback

1. Remove the `"env"` block from the Claude Code `settings.json` files.
2. Privoxy:
   ```sh
   systemctl --user disable --now privoxy-claude
   rm ~/.config/systemd/user/privoxy-claude.service ~/.config/privoxy/config
   systemctl --user daemon-reload
   sudo dnf remove privoxy
   ```
3. Firefox: delete `user.js` from the profile and `~/.config/privoxy/claude.pac`. In Settings → Network Settings, pick "No proxy", then restart Firefox.
4. Firefox profile `proxy`: `rm ~/.local/share/applications/firefox-proxy.desktop`, then remove the profile in `about:profiles` (Remove → Delete Files).
5. OpenSnitch:
   ```sh
   sudo rm /etc/opensnitchd/rules/00?-claude-*.json
   sudo systemctl disable --now opensnitchd
   sudo dnf remove opensnitch
   ```
