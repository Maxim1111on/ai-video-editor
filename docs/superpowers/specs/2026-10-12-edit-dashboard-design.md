# edit.hypnotiicmedia.com — branded tunnel + chat (Phase 1+2)

## Goal

Max wants his own branded front door to the ai-editor home page, reachable at
`edit.hypnotiicmedia.com`, styled like his existing Hypnotiic dashboard app, with a persistent
chat box (like the Claude app) for typing instructions instead of clicking through pages.

This spec covers the first two of five phases agreed with Max (see conversation log, 2026-10-12):

1. **Plumbing** — a real, branded URL in front of the existing local tool.
2. **Chat panel** — a persistent free-text input that queues instructions the same way every other
   action on the page already does.

Phases 3-5 (project page, timeline editor, timestamp-referenced style matching) are out of scope
for this spec and get their own brainstorm/spec when started.

## Non-goals (explicit, to stop scope creep)

- **No cloud backend.** Max does not want to pay for hosting compute. All editing (transcription,
  cutting, rendering, capturing) keeps running locally on his Mac through this same Claude Code
  session, exactly as it does today via `home.py`.
- **No multi-user support.** One fixed tunnel to Max's own Mac. No accounts, no pairing flow, no
  per-user session isolation. (Confirmed with Max: "Just me for now.")
- **No new editing capability.** This is a re-skin plus a new input method. Every existing flow
  (New video, Your videos, My style, Settings, teardown) keeps working exactly as it does on
  `127.0.0.1` today — nothing about the cut/style-edit/render pipeline changes.
- **No login screen in this phase.** Access control stays the existing long secret token in the
  URL (`home.py`'s existing token-per-install scheme). Max was told explicitly this is adequate
  for "just me" but not a real auth system; he can ask for a password gate later.

## Architecture

```
Browser (anywhere) --https--> Cloudflare edge --tunnel--> cloudflared (Max's Mac)
                                                                |
                                                                v
                                                     home.py serve, 127.0.0.1:47821
                                                     (UNCHANGED — same job queue,
                                                      same `wait`, same status reporting
                                                      this session already uses)
```

- `cloudflared` is a small background process on Max's Mac. A **named tunnel** (not a quick/ephemeral
  one) bound to the `hypnotiicmedia.com` zone, with a DNS CNAME for `edit.hypnotiicmedia.com`
  pointing at the tunnel, so the URL is stable and matches his other Hypnotiic subdomains
  (`intel.hypnotiicmedia.com`).
- `home.py` itself needs **no networking changes** — it already listens on `127.0.0.1:47821`;
  `cloudflared` forwards the public hostname to that local address. If the Mac or this session is
  off, the site is simply unreachable — same failure mode as today's local server, just at a nicer
  address.
- Setup is one-time: `brew install cloudflared`, `cloudflared tunnel login` (OAuth to Max's
  Cloudflare account — the same account already running `hypnotiicmedia.com` and
  `intel.hypnotiicmedia.com`), `cloudflared tunnel create edit-hypnotiic`,
  `cloudflared tunnel route dns edit-hypnotiic edit.hypnotiicmedia.com`, then a tunnel config file
  pointing `edit.hypnotiicmedia.com` → `http://localhost:47821`. Running the tunnel is a second
  background process started alongside `home.py serve` (documented in the plugin's own
  `references/home.md` "Open it" step, extended to also start `cloudflared`).

## Visual layer

`plugins/ai-editor/lib/ai_editor/home.html` gets restyled to match the **real** design tokens read
from Max's dashboard repo (`/Users/max/Desktop/hypnotiic-dashboard/tailwind.config.js`), not the
generic "dark editorial" rules in his global frontend notes, since he explicitly pointed at this
app as the model:

| Token | Value |
|---|---|
| Accent | `#6C63FF` (hover `#5B52E8`) |
| Canvas (dark) | `#0F0F0F` |
| Surface (dark) | `#1A1A1A` |
| Hairline (dark) | `#2A2A2A` |
| Ink (dark) | `#F5F5F5`, soft `#888888` |
| Row alt (dark) | `#161616` |
| Font | Inter |
| Card radius | 12px (`card`), 8px (`ctl`), 6px (`pill`) |
| Success/warning/error | `#22C55E` / `#F59E0B` / `#EF4444` |
| Dark mode strategy | class-based (`dark:` variants) |

This is a CSS/markup pass over the existing `home.html` — same DOM structure, same JS logic
(`fetch`/`post` calls, job polling), new color variables, font, radii, and the class-based
light/dark toggle matching the dashboard's existing pattern. No page's functionality changes.

## Chat layer

A new persistent input on the Home tab, styled like the existing "Tell it what to change" textarea
(`home.html` line ~827) but always visible, not scoped to one style page.

**Frontend (`home.html`):** one `<textarea>` + send button, posting to the existing `/job` endpoint
with a new job type:

```json
{"type": "chat", "text": "<Max's words>"}
```

Reuses the existing `post('/job', b)` / `queue(b)` JS helpers already in the file — no new endpoint
needed, just a new case in the job-formatting switch.

**Backend (`home.py`):** one new branch in the job-line formatter (alongside `teardown`, `skill`,
`video`, `style-change`, `modal`) that emits:

```
JOB chat text=<Max's words> root=<DIR>
```

**Handling (Claude, this session, via `home.py wait`):** when a `JOB chat` line arrives, route the
free text through the same decision logic already used manually throughout this session:

- A link, handle, or "study @x" → creator-teardown quick mode (the existing `JOB teardown` path).
- "edit my video" / a video reference → the existing `JOB video` → start skill pipeline.
- A style correction ("captions bigger", "fewer zooms") → the existing style-change logic
  (`taste.py add` / `profile.py set`), same as the style page's textarea already does.
- A settings/profile change ("switch to 16:9", "use my brand colors") → `profile.py set ...`
  directly.
- Anything not actionable (a question) → answered directly in the status line, no job side-effect.

Status streams back the same way the style page's `jobLine` already renders ("Claude: cutting out
the mistakes" / "Changed: ..." etc.) — reusing `home.py status chat --step ...` /
`--result ... --done`, the same pattern every other job type already uses.

## Security note (told to Max, repeated here for the record)

The existing token-in-URL scheme is the only access control in this phase. Adequate for solo
personal use since the token is long and random, but anyone who obtains the link can reach the
page. No password/login screen in this phase — flagged to Max as something to add later if wanted.

## Testing

- Load `https://edit.hypnotiicmedia.com/?t=<token>` from a phone on cellular data (not Max's Wi-Fi)
  — confirms the public path actually works, not just localhost.
- Confirm every existing page (Home, New video, Your videos, My style, Settings) behaves exactly as
  it does on `127.0.0.1:47821` today — this is a re-skin, regressions here are the main risk.
- Confirm dark/light mode toggle matches the dashboard's own behavior.
- Type a real instruction into the new chat box ("make captions a bit bigger") end to end: job
  queues, `home.py wait` emits the `JOB chat` line, Claude routes it to the style-change path,
  status appears under the input, the change actually lands in `taste.json`.
- Type a non-actionable message ("what creators do I have set up?") and confirm it's answered in
  the status line without side effects.
- Restart the Mac / stop `cloudflared` and confirm the public URL fails cleanly (no confusing
  partial state) — expected behavior for this phase, not a bug to fix.

## Risks / open questions

- **cloudflared as a long-running background process**: needs to restart if Max's Mac reboots. Out
  of scope to solve with a LaunchAgent in this phase (quick fix if it becomes annoying); for now,
  Max starts it the same way he starts `home.py serve` today.
- **Public GitHub fork**: `github.com/Maxim1111on/ai-video-editor` is public (matches the
  upstream open-source project's own visibility). Confirmed with Max this is acceptable; no secrets
  ever enter this repo (verified: no API keys saved anywhere on this machine as of 2026-10-12).
- **Re-skinning risk**: `home.html` is ~900 lines of hand-rolled HTML/CSS/JS with no build step.
  The restyle must be done as a token/CSS swap, not a rewrite, to avoid silently breaking existing
  job-polling logic.
