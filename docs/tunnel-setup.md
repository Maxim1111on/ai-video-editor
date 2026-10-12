# edit.hypnotiicmedia.com — Cloudflare Tunnel setup

One-time setup, on Max's Mac, using the same Cloudflare account that already runs
`hypnotiicmedia.com` and `intel.hypnotiicmedia.com`.

## Install

```bash
brew install cloudflared
```

## Authenticate and create the tunnel (once)

```bash
cloudflared tunnel login                              # opens a browser, pick the hypnotiicmedia.com zone
cloudflared tunnel create edit-hypnotiic               # writes a credentials file under ~/.cloudflared/
cloudflared tunnel route dns edit-hypnotiic edit.hypnotiicmedia.com
```

## Config

Create `~/.cloudflared/config.yml`:

```yaml
tunnel: edit-hypnotiic
credentials-file: /Users/max/.cloudflared/<tunnel-id>.json
ingress:
  - hostname: edit.hypnotiicmedia.com
    service: http://localhost:47821
  - service: http_status:404
```

(`<tunnel-id>` is printed by `cloudflared tunnel create` — copy the exact filename it creates in
`~/.cloudflared/`.)

## Run it

Every time the editor's home page is open (alongside `home.py serve`):

```bash
cloudflared tunnel run edit-hypnotiic
```

Runs in the foreground; run it in the background the same way `home.py serve` and `home.py wait`
already run in the background (`run_in_background`). If the Mac or this process stops, the public
URL simply stops answering — same failure mode as the local server today.

## Verify

1. On a phone, **turn off Wi-Fi** (cellular only, so the request actually leaves Max's network) and
   load `https://edit.hypnotiicmedia.com/?t=<the token `home.py serve` printed>`.
2. Confirm the page loads and looks identical to `http://127.0.0.1:47821/?t=...` on the Mac.
3. Load `https://edit.hypnotiicmedia.com` with **no** `?t=` query string. Confirm it is refused
   (403), not served.
