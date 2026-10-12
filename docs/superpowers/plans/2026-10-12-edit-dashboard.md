# edit.hypnotiicmedia.com (Phase 1+2) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Put a branded, public URL (`edit.hypnotiicmedia.com`) in front of the existing local
ai-editor home page, restyled in the Hypnotiic dashboard's real look, with a persistent chat input
added to the Home tab that queues free-text instructions the same way every other action on the
page already does.

**Architecture:** No new backend, no new networking inside `home.py` — a Cloudflare Tunnel process
on Max's Mac forwards the public hostname to the existing `127.0.0.1:47821` server unchanged.
`home.html` gets a CSS-token swap to the dashboard's palette/font/radii (no DOM restructuring). One
new job kind (`chat`) is added to the existing job-queue mechanism (`JOBS` tuple, `job_line()`
formatter, the `/job` POST handler) exactly the way `style-change` already works, plus one new
textarea+button in the Home view that posts to it. **Interpreting what a chat message means is not
code** — it's Claude (this session, or whoever is running `home.py wait` next) reading the `JOB chat
text=...` line and routing it through the decision logic already used for every other instruction
this session, documented as a contract in `home.md` rather than implemented as a classifier.

**Tech Stack:** Python 3 stdlib (`http.server`, no framework), hand-rolled HTML/CSS/vanilla JS (no
build step, no bundler, no JS test framework), Cloudflare Tunnel (`cloudflared`, free).

**Spec:** `docs/superpowers/specs/2026-10-12-edit-dashboard-design.md`

## Global Constraints

- No cloud backend. All editing work stays on Max's Mac via the existing `home.py` server.
- No multi-user support. One fixed tunnel to one Mac. No accounts, no pairing, no new auth system.
- No changes to the editing pipeline (cut/style-edit/render). This plan only touches the home page's
  transport, look, and one new input.
- Access control stays the existing long secret token in the URL. No login screen.
- A chat message is truncated to 400 characters, same limit as `style-change`'s `text` field
  (`home.py:1036`).
- This repo (`github.com/Maxim1111on/ai-video-editor`) is **public**. Never commit a key, token, or
  credential in any task below. `.gitignore` already blocks `.env`, `.env.*`, `*.pem`, `.cloudflared/`.

## Review Focus

- **Empty or whitespace-only chat message.** A reasonable person expects "Send" to do nothing (or
  show an error), not queue a blank instruction. Mirrored from `style-change`'s existing "say what
  to change" 400 response — tested in Task 2.
- **A chat message over 400 characters.** Should silently truncate, not error or crash the queue —
  tested in Task 2 (mirrors `style-change`'s existing `[:400]` behavior).
- **Several chat messages sent before the first is picked up.** `cmd_wait` takes one job at a time;
  a reasonable person expects their second message to wait its turn, not get lost or overwrite the
  first. Tested in Task 2 (two `chat` jobs queued, two sequential `cmd_wait` calls each return one).
- **The public URL without the token.** A reasonable person (or a search engine, or a stray link)
  hitting `edit.hypnotiicmedia.com` with no `?t=` must get refused, not served. Already covered by
  the existing generic 403 check in `demo()` (`home.py` — any path without `?t=` 403s); Task 1's
  manual verification explicitly re-checks this over the real public tunnel, not just localhost,
  since a tunnel misconfiguration could theoretically bypass it.
- **Dark/light mode toggle after the restyle.** A reasonable person expects the existing header
  switch (`toggleTheme()`) to keep working and both themes to use the new palette, not strand the
  page half-restyled in one mode. Checked explicitly in Task 3's manual verification.

---

## Task 1: Cloudflare Tunnel front door

**Files:**
- Create: `docs/tunnel-setup.md`
- Modify: `plugins/ai-editor/skills/start/references/home.md:14-29` ("Open it" section)

**Interfaces:**
- Consumes: the existing `home.py serve` command (`skills/start/references/home.md:14-29`), unchanged.
- Produces: a documented, repeatable runbook later tasks and future sessions can follow; no code
  interface (this task ships no application code).

- [ ] **Step 1: Write the runbook**

Create `docs/tunnel-setup.md`:

```markdown
# edit.hypnotiicmedia.com — Cloudflare Tunnel setup

One-time setup, on Max's Mac, using the same Cloudflare account that already runs
`hypnotiicmedia.com` and `intel.hypnotiicmedia.com`.

## Install

\`\`\`bash
brew install cloudflared
\`\`\`

## Authenticate and create the tunnel (once)

\`\`\`bash
cloudflared tunnel login                              # opens a browser, pick the hypnotiicmedia.com zone
cloudflared tunnel create edit-hypnotiic               # writes a credentials file under ~/.cloudflared/
cloudflared tunnel route dns edit-hypnotiic edit.hypnotiicmedia.com
\`\`\`

## Config

Create `~/.cloudflared/config.yml`:

\`\`\`yaml
tunnel: edit-hypnotiic
credentials-file: /Users/max/.cloudflared/<tunnel-id>.json
ingress:
  - hostname: edit.hypnotiicmedia.com
    service: http://localhost:47821
  - service: http_status:404
\`\`\`

(`<tunnel-id>` is printed by `cloudflared tunnel create` — copy the exact filename it creates in
`~/.cloudflared/`.)

## Run it

Every time the editor's home page is open (alongside `home.py serve`):

\`\`\`bash
cloudflared tunnel run edit-hypnotiic
\`\`\`

Runs in the foreground; run it in the background the same way `home.py serve` and `home.py wait`
already run in the background (`run_in_background`). If the Mac or this process stops, the public
URL simply stops answering — same failure mode as the local server today.

## Verify

1. On a phone, **turn off Wi-Fi** (cellular only, so the request actually leaves Max's network) and
   load `https://edit.hypnotiicmedia.com/?t=<the token `home.py serve` printed>`.
2. Confirm the page loads and looks identical to `http://127.0.0.1:47821/?t=...` on the Mac.
3. Load `https://edit.hypnotiicmedia.com` with **no** `?t=` query string. Confirm it is refused
   (403), not served.
\`\`\`
```

- [ ] **Step 2: Add the tunnel as a documented optional step in `home.md`**

In `plugins/ai-editor/skills/start/references/home.md`, under the existing `## Open it` section
(around line 14-29), add one paragraph after the existing `python3 "$H" serve` / `python3 "$H" wait`
code block:

```markdown
For a branded public URL instead of `127.0.0.1`, also start `cloudflared tunnel run <name>` in the
background alongside `serve` and `wait` — one-time setup in `docs/tunnel-setup.md`. The page and
job queue behave identically either way; the tunnel only changes how the browser reaches them.
```

- [ ] **Step 3: Run the runbook for real and verify**

Follow `docs/tunnel-setup.md` end to end on Max's Mac. Complete when all three checks in its
"Verify" section pass.

- [ ] **Step 4: Commit**

```bash
git add docs/tunnel-setup.md plugins/ai-editor/skills/start/references/home.md
git commit -m "docs: add Cloudflare Tunnel runbook for edit.hypnotiicmedia.com"
```

---

## Task 2: `chat` job kind (backend)

**Files:**
- Modify: `plugins/ai-editor/lib/ai_editor/home.py:74` (`JOBS` tuple)
- Modify: `plugins/ai-editor/lib/ai_editor/home.py:633-646` (`job_line()`)
- Modify: `plugins/ai-editor/lib/ai_editor/home.py:1008-1047` (`job()` handler in `make_handler`)
- Modify: `plugins/ai-editor/lib/ai_editor/home.py:1267-1550` (`demo()` — add the test)
- Modify: `plugins/ai-editor/skills/start/references/home.md` (document the `JOB chat` contract)

**Interfaces:**
- Consumes: `queue(d, kind, args)` (`home.py:625`), `cmd_wait()` (`home.py:649`), `cmd_status(a)`
  (`home.py:678`) — all unchanged, already generic over job kind.
- Produces: `POST /job {"type": "chat", "text": "<words>"}` → 200 `{"id": "chat-N"}` on non-empty
  text, 400 `{"error": "say what you want"}` on empty/whitespace-only text. `home.py wait` emits
  `JOB chat text=<words> root=<dir>` for a queued chat job. `home.py status chat --step "..." ...`
  works (now a valid `--job` choice).

- [ ] **Step 1: Write the failing test**

In `plugins/ai-editor/lib/ai_editor/home.py`, inside `demo()`, immediately after the existing
`style-change` block (after line 1548, `assert get("/style?id=mine")["job"]["result"]["changed"] ==
["No sound effects"]`), add:

```python
    # The persistent chat bar: free text queues as its own job kind, same mechanics as style-change
    post("/job", {"type": "chat", "text": "  make the captions a bit bigger  "})
    assert out(cmd_wait, 0.01) == f"JOB chat text=make the captions a bit bigger root={root.resolve()}"
    out(cmd_status, argparse.Namespace(job="chat", step="Claude: checking your style", detail=None,
                                        progress=None, result=None, done=False, failed=None))
    assert get("/state")["jobs"][-1]["steps"][-1]["text"] == "Claude: checking your style"
    # two queued before either is taken: wait hands them out one at a time, in order
    post("/job", {"type": "chat", "text": "first"})
    post("/job", {"type": "chat", "text": "second"})
    assert "text=first" in out(cmd_wait, 0.01)
    assert "text=second" in out(cmd_wait, 0.01)
    # over 400 chars: truncated, not rejected
    post("/job", {"type": "chat", "text": "x" * 500})
    assert len(get("/state")["jobs"][-1]["args"]["text"]) == 400
    # empty or whitespace-only: rejected before it ever reaches the queue
    try:
        post("/job", {"type": "chat", "text": "   "})
        raise AssertionError("empty chat text should be rejected")
    except urllib.error.HTTPError as e:
        assert e.code == 400, e.code
```

- [ ] **Step 2: Run the demo self-check and confirm it fails**

Run: `python3 plugins/ai-editor/lib/ai_editor/home.py demo`
Expected: fails on the first new assertion — `post("/job", {"type": "chat", ...})` raises
`HTTPError: 400` ("unknown job"), because `job()` has no `chat` branch yet.

- [ ] **Step 3: Add `chat` to the `JOBS` tuple**

In `home.py:74`:

```python
JOBS = ("teardown", "skill", "video", "modal", "setup", "style-change", "update", "chat")
```

- [ ] **Step 4: Add the `chat` branch to `job_line()`**

In `home.py`, inside `job_line(j, root)` (currently lines 633-646), add before the final
`return f"JOB {j['type']}"` fallback:

```python
    if j["type"] == "chat":
        return f"JOB chat text={a['text']} root={root}"
```

- [ ] **Step 5: Add the `chat` branch to the `/job` handler**

In `home.py`, inside `def job(self, b):` (currently lines 1008-1047), add a new `elif` branch after
the existing `style-change` branch (after line 1043, before the final `else: return self.send(400,
{"error": "unknown job"})`):

```python
            elif kind == "chat":   # the persistent chat bar: free text, Claude decides what it means
                text = re.sub(r"\s+", " ", str(b.get("text") or "")).strip()[:400]
                if not text:
                    return self.send(400, {"error": "say what you want"})
                args = {"text": text}
```

- [ ] **Step 6: Run the demo self-check and confirm it passes**

Run: `python3 plugins/ai-editor/lib/ai_editor/home.py demo`
Expected: no assertion errors, process exits 0.

- [ ] **Step 7: Document the `JOB chat` contract**

In `plugins/ai-editor/skills/start/references/home.md`, add a new subsection under `## The jobs`,
after the existing `### JOB modal` section and before `### JOB cancel teardown`:

```markdown
### `JOB chat text=<the user's words> root=<dir>`

The user typed a free instruction into the Home tab's persistent chat box — not scoped to one
style or one video. Route the words the same way you already route anything said directly in this
session:

- A link, handle, or "study @x" → the creator-teardown quick-mode path (as if it arrived as a
  `creator` item from `links.py route`).
- "edit my video" / a dropped or named video → the `start` skill's pipeline (same as a `JOB video`).
- A style correction ("captions bigger", "fewer zooms") → the same `taste.py add` /
  `profile.py set` path the `style-change` job already uses, scoped to `mine` unless a style was
  named.
- A settings/profile change ("switch to 16:9", "use my brand colors") → `profile.py set ...`
  directly.
- Anything not actionable (a question) → answer it directly with `status chat --step "<answer>"
  --done`; no other side effect.

Report progress the same way every other job does:

\`\`\`bash
python3 "$H" status chat --step "..." [--detail "..."] [--progress 0.4]
python3 "$H" status chat --result '<json>' --done
\`\`\`
```

- [ ] **Step 8: Commit**

```bash
git add plugins/ai-editor/lib/ai_editor/home.py plugins/ai-editor/skills/start/references/home.md
git commit -m "feat(home): add chat job kind, mirroring style-change's queue/status mechanics"
```

---

## Task 3: Restyle `home.html` to the Hypnotiic dashboard's tokens

**Files:**
- Modify: `plugins/ai-editor/lib/ai_editor/home.html:1-19` (root CSS tokens, font)
- Modify: `plugins/ai-editor/lib/ai_editor/home.html` (every selector listed in Step 3 below)

**Interfaces:**
- Consumes: nothing new — this is a value-only edit of existing CSS custom properties.
- Produces: `--accent` and `--accent-hover` custom properties, newly available to Task 4's chat UI
  and to any future styling. All existing selectors keep their names; only values change.

- [ ] **Step 1: Add the Inter font**

In `home.html`, in `<head>` (after the existing `<title>` tag, line 6), add — mirroring
`/Users/max/Desktop/hypnotiic-dashboard/index.html` exactly:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
```

- [ ] **Step 2: Swap the root token values**

Replace lines 10-16 of `home.html`:

```css
:root{--bg:#FAFAF9;--panel:#FFFFFF;--ink:#0C0A09;--t2:#57534E;--t3:#78716C;--t4:#A8A29E;--line:#E7E5E4;--soft:#F5F5F4;--ring:#D6D3D1;
  --onink:#FFFFFF;--mono:"JetBrains Mono",ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;color-scheme:light}
@media (prefers-color-scheme:dark){:root:not([data-theme=light]){--bg:#100E0E;--panel:#161413;--ink:#F2F0EB;--t2:rgba(242,240,235,.80);--t3:rgba(242,240,235,.58);--t4:rgba(242,240,235,.42);
  --line:rgba(255,255,255,.10);--soft:rgba(255,255,255,.06);--ring:rgba(255,255,255,.22);--onink:#100E0E;color-scheme:dark}}
:root[data-theme=dark]{--bg:#100E0E;--panel:#161413;--ink:#F2F0EB;--t2:rgba(242,240,235,.80);--t3:rgba(242,240,235,.58);--t4:rgba(242,240,235,.42);
  --line:rgba(255,255,255,.10);--soft:rgba(255,255,255,.06);--ring:rgba(255,255,255,.22);--onink:#100E0E;color-scheme:dark}
```

with the dashboard's real tokens, keeping the same variable names (so every other selector in the
file needs no change) but Hypnotiic's values, plus the two new accent tokens:

```css
:root{--bg:#FAFAFA;--panel:#FFFFFF;--ink:#111827;--t2:#6B7280;--t3:#6B7280;--t4:#9CA3AF;--line:#E5E7EB;--soft:#F9FAFB;--ring:#6C63FF;
  --onink:#FFFFFF;--accent:#6C63FF;--accent-hover:#5B52E8;--mono:"JetBrains Mono",ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;color-scheme:light}
@media (prefers-color-scheme:dark){:root:not([data-theme=light]){--bg:#0F0F0F;--panel:#1A1A1A;--ink:#F5F5F5;--t2:#888888;--t3:#888888;--t4:#6B6B6B;
  --line:#2A2A2A;--soft:#161616;--ring:#6C63FF;--onink:#F5F5F5;--accent:#6C63FF;--accent-hover:#5B52E8;color-scheme:dark}}
:root[data-theme=dark]{--bg:#0F0F0F;--panel:#1A1A1A;--ink:#F5F5F5;--t2:#888888;--t3:#888888;--t4:#6B6B6B;
  --line:#2A2A2A;--soft:#161616;--ring:#6C63FF;--onink:#F5F5F5;--accent:#6C63FF;--accent-hover:#5B52E8;color-scheme:dark}
```

- [ ] **Step 3: Switch the body font and apply radii/accent to the selectors that already exist**

In `home.html`, line 19, replace:

```css
body{background:var(--bg);color:var(--ink);font:14px/20px system-ui,-apple-system,"Segoe UI",sans-serif;-webkit-font-smoothing:antialiased;overflow-wrap:anywhere}
```

with:

```css
body{background:var(--bg);color:var(--ink);font:14px/20px 'Inter',system-ui,-apple-system,"Segoe UI",sans-serif;-webkit-font-smoothing:antialiased;overflow-wrap:anywhere}
```

Then, in the same file, update these five existing declarations (unchanged selectors, new values
only — grep each exact string first since line numbers shift as earlier edits land):

| Selector (search for this exact text) | Change |
|---|---|
| `.panel{background:var(--panel);border:1px solid var(--line);border-radius:20px;` | `border-radius:20px` → `border-radius:12px` |
| `.btn{white-space:nowrap;height:42px;padding:0 22px;border-radius:12px;` | `border-radius:12px` → `border-radius:8px` |
| `.btn.dark{background:var(--ink);color:var(--onink)}` | `background:var(--ink)` → `background:var(--accent)` |
| `.small.dark{background:var(--ink);border-color:var(--ink);color:var(--onink);font-weight:600}` | both `var(--ink)` → `var(--accent)` |
| `.input:focus{outline:none;border:1.5px solid var(--ink)}` | `var(--ink)` → `var(--accent)` |
| `.big-input{display:flex;align-items:center;gap:10px;height:72px;border:1.5px solid var(--ink);border-radius:14px;` | `border:1.5px solid var(--ink)` → `border:1.5px solid var(--accent)`; `border-radius:14px` → `border-radius:12px` |

Leave every other selector untouched — they reference the same variable names, which now resolve
to the new palette automatically.

- [ ] **Step 4: Run the demo self-check (regression only — it does not see CSS)**

Run: `python3 plugins/ai-editor/lib/ai_editor/home.py demo`
Expected: still exits 0. This only proves the HTML/JS wasn't broken while editing; it says nothing
about how the page looks.

- [ ] **Step 5: Manual visual verification**

```bash
python3 plugins/ai-editor/lib/ai_editor/home.py serve
```

Open the printed URL. Confirm:
- Accent purple (`#6C63FF`) appears on primary buttons and focused inputs.
- Dark mode (toggle the header switch) shows near-black `#0F0F0F`/`#1A1A1A`, not the old
  near-black-but-different `#100E0E`/`#161413`.
- Light mode shows `#FAFAFA`/`#FFFFFF`.
- Font is visibly Inter, not the system font (check numerals/letterforms against
  `hypnotiic-dashboard`'s own running dev server side by side).
- Every existing page (Home, New video, Your videos, My style, Settings) still works exactly as
  before — click through all five.

- [ ] **Step 6: Commit**

```bash
git add plugins/ai-editor/lib/ai_editor/home.html
git commit -m "style(home): restyle to Hypnotiic dashboard tokens (accent, dark palette, Inter, radii)"
```

---

## Task 4: Persistent chat panel on the Home tab

**Files:**
- Modify: `plugins/ai-editor/lib/ai_editor/home.html` (`vHome()`, currently lines 994-1003)
- Modify: `plugins/ai-editor/lib/ai_editor/home.html` (new `chatSend()` function, placed beside
  `tellChange()`, currently line 1069)

**Interfaces:**
- Consumes: `queue(b)` (existing JS helper, posts to `/job`), the `chat` job kind from Task 2
  (`POST /job {"type": "chat", "text": "..."}`), `latest(t)` (existing JS helper, `latest('chat')`
  reads the newest chat job from state).
- Produces: nothing new for other code to consume — this is the leaf UI for the feature.

- [ ] **Step 1: Add `chatSend()`, mirroring `tellChange()` exactly**

In `home.html`, immediately after the existing `tellChange()` function (currently line 1069):

```js
async function chatSend(){const t=($('#chatbox')||{}).value||'';if(!t.trim())return;const r=await queue({type:'chat',text:t.trim()});if(r._ok)$('#chatbox').value=''}
```

- [ ] **Step 2: Add the chat panel markup to `vHome()`**

In `home.html`, inside `vHome()` (currently lines 994-1003), insert a new panel immediately after
the `<h1>Welcome back</h1>` block and before the `New video` panel (i.e., right after line 997's
closing `</div>`, before line 998's `<div class="panel" style="padding:24px 28px;gap:22px">`):

```js
function vHome(){
  const vids=L.videos,ready=vids.filter(v=>v.status==='ready').length,making=vids.filter(v=>v.status==='making'||v.status==='cutting').length;
  const rec=(R?R.videos:[]).slice(0,4);
  const cj=latest('chat'),cjLine=!cj?'':cj.state==='queued'?(S.listening?'Claude is picking this up.':`Waiting for Claude. Type ${CMD} in Terminal.`):cj.state==='taken'?`Claude: ${esc((cj.steps.at(-1)||{}).text||'working on it')}`:cj.state==='done'?esc((cj.result||{}).step||'Done.'):cj.state==='failed'?esc(cj.error):'';
  return `<div class="col grow" style="min-width:0;padding:12px 14px 0;gap:28px"><div><h1>Welcome back</h1>${ready||making?`<p class="lede" style="margin-top:10px">${[ready?`${ready} ready for you to watch.`:'',making?`${making} still being made.`:''].filter(Boolean).join(' ')}</p>`:''}</div>
    <div class="panel" style="padding:20px 24px;gap:12px"><div class="h">Ask for anything</div>
      <div class="row" style="align-items:stretch"><textarea id="chatbox" class="input" style="height:64px;padding:10px 12px;font:14px/20px 'Inter',system-ui,sans-serif;resize:vertical" placeholder="edit my video like @sam.gaudet, make captions bigger, study @handle..." onkeydown="if(event.key==='Enter'&&!event.shiftKey){event.preventDefault();chatSend()}"></textarea><button class="btn dark" style="height:auto" onclick="chatSend()">Send</button></div>
      ${cjLine?`<div class="muted" style="color:var(--ink)">${cjLine}</div>`:''}</div>
    <div class="panel" style="padding:24px 28px;gap:22px"><div class="between" style="align-items:flex-start;flex-wrap:wrap"><div><div style="font-size:22px;font-weight:600">New video</div></div><button class="btn dark" style="height:52px;padding:0 40px;font-size:16px" onclick="rgo('new')">+ New video</button></div>
      <div class="row recrow" style="gap:12px;align-items:flex-end"><div class="droptile" id="droptile" style="width:150px;height:150px" onclick="rgo('new')">${UP}<div style="font-weight:500;font-size:13px">Drop files here</div><div class="muted" style="font-size:12px">or choose files</div></div>${rec.map(v=>`<button class="clip" style="width:84px" onclick="nv.sel=[${esc(JSON.stringify(v.path))}];rgo('new')"><div class="pic" style="width:84px;height:150px;background-image:url('${esc(v.thumb)}')"></div></button>`).join('')}${rec[0]?`<div class="reclabel" style="padding-left:12px;min-width:140px"><div style="font-size:15px">${esc(rec[0].name)}</div><div class="muted" style="margin-top:4px;font-size:14px">${esc(rec[0].where)} · ${ago(rec[0].mtime)}</div></div>`
        :[0,1,2].map(i=>`<div class="clip ex" style="width:84px"><div class="pic" style="width:84px;height:150px;background-image:url('${DEMO(i)}')"></div></div>`).join('')+`<div class="reclabel muted" style="padding-left:12px;min-width:140px;font-size:14px;line-height:20px">Example clips. Your newest clips show here.</div>`}</div></div>
    <div><div class="between" style="padding-bottom:12px"><span class="h" style="font-size:17px">Your videos</span>${vids.length>3?`<a href="#" onclick="rgo('videos');return false">See all ${vids.length}</a>`:''}</div>${vids.slice(0,3).map(vRow).join('')||EXROW}</div>
  </div>${styleSide()}`;
}
```

(This replaces the whole function body — same closing structure as before, with the new chat panel
inserted and `cj`/`cjLine` computed at the top, mirroring the `jobLine` pattern already used in
`vStylePage()`.)

- [ ] **Step 3: Run the demo self-check (regression only)**

Run: `python3 plugins/ai-editor/lib/ai_editor/home.py demo`
Expected: still exits 0 — `demo()` doesn't execute browser JS, so this only confirms the Python
side and the static HTML payload are intact.

- [ ] **Step 4: Manual end-to-end verification**

```bash
python3 plugins/ai-editor/lib/ai_editor/home.py serve
python3 plugins/ai-editor/lib/ai_editor/home.py wait
```

(second command in another terminal, or backgrounded). In the browser:
1. On the Home tab, type "make the captions a bit bigger" into the new "Ask for anything" box,
   press Send (or Enter).
2. Confirm the box clears and a status line appears under it.
3. Confirm the terminal running `home.py wait` printed `JOB chat text=make the captions a bit
   bigger root=...`.
4. Manually run `python3 plugins/ai-editor/lib/ai_editor/home.py status chat --step "Claude: on
   it" --detail "checking your style"` and confirm the status line in the browser updates without a
   page reload (the page already polls `/state`).
5. Press Enter with the box empty — confirm nothing is sent (no network request, check the
   browser's dev tools Network tab).
6. Type over 400 characters, send, and confirm (via `/state` in a second browser tab, or
   `curl localhost:47821/state?t=<token>`) the stored `text` is truncated to 400 characters.

- [ ] **Step 5: Commit**

```bash
git add plugins/ai-editor/lib/ai_editor/home.html
git commit -m "feat(home): add persistent chat panel to the Home tab"
```

---

## After this plan

Phases 3-5 from the spec (project page, timeline editor, timestamp-referenced style matching) are
explicitly out of scope here and get their own brainstorm → spec → plan cycle when Max is ready to
start them.
