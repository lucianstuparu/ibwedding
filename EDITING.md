# Editing the ibwedding PWA — fast path

How to change anything in the **Wedding** app (`ibwedding.stuparu.eu`) without
rediscovering everything. Read `README.md` (same repo) for hosting/TLS; this file is
the **modify → deploy → verify** loop. Pairs with the general `runbooks/create-pwa.md`
guide in the **`our.infra`** repo.

## ⭐ Itinerary text/data → edit `journey.json` (no decoding!)

**All itinerary content now lives in plain `journey.json`** (title, dates, route, days,
items — flights/stays/cars/drives/events, with times, seats, booking refs, maps links,
and the `start`/`end` `[Y,M,D,h,m]` datetimes that drive the now/next focus). To change a
time, seat, note, add a stop, etc.:

```bash
git clone https://github.com/lucianstuparu/ibwedding.git && cd ibwedding
# edit journey.json directly (normal JSON, readable diffs)
git commit -am "update itinerary" && git push
```

The app fetches `journey.json` at startup; the SW serves it **network-first**, so a data
edit appears on the next online load **without an SW bump** (CF/HTTP cache ~10 min — hard-
refresh for instant). It's also precached for offline. The copy embedded in the bundle
(`c2358d71`) is only an offline fallback for a first-ever visit; keep it roughly in sync if
you like, but editing `journey.json` is enough. **Only design/component changes need the
bundle-decoding flow below.**

## TL;DR (component / design changes only)

```bash
git clone https://github.com/lucianstuparu/ibwedding.git && cd ibwedding
# 1. inspect: decode the bundle, find the resource you need (see scripts below)
# 2. edit that one resource's decoded source
# 3. repack with repack.py (string-swaps just that resource's base64)
# 4. bump CACHE in sw.js (wedding-run-vN -> vN+1)
# 5. git commit && git push
# 6. wait for Pages build (~2-4 min, it queues), then verify (decode live + render)
```

## What this app actually is (critical)

`index.html` (~1.6 MB) is **not editable source** — it's a Claude **artifact bundle**:
one big `<script>` holding a JSON map of ~19 **gzip+base64** resources, plus a runtime
that decodes them to blob URLs and rewrites the document on boot.

Resource map (UUID keys; **content-addressed — they change if the app is regenerated**,
so find by content, not by hardcoded UUID). As of 2026-06-23:

| UUID prefix | what it is | edit it? |
|-------------|-----------|----------|
| `9812c350` | **JourneyApp JSX** — UI: `FlightCard`/`StayCard`/`InfoScreen`/`InfoBlock`/`BottomBar`/`renderItem` | YES (no SRI) |
| `c2358d71` | **`window.JOURNEY` data** — `contact`, `info.{verify,bags,car}`, `passengers`, day data | YES (no SRI) |
| `e72a9360` | `window.TripIcon` icon paths (plane, car, bed, heart, clock, pin, utensils, info, users, bag, key, alert, phone, map, check, chevL, chevR, display) | rarely |
| `0766c638`,`6296321c`,`7028b574`,`c2358d71`-libs | React / ReactDOM / Babel-standalone | NO (have SRI) |
| `fb05e85e`…(woff2) | embedded fonts | NO |

**Key facts that save time:**
- The two `text/babel` app scripts (`9812c350`, and the icon/data scripts) have **no SRI
  `integrity`** attribute — only the React/Babel libs do. So you can edit app source and
  re-pack **without** recomputing any hash.
- Compression is **gzip** (`gzip.decompress`/`gzip.compress` round-trips; verified).
- It's `text/babel` → JSX is transpiled in-browser. **A syntax error in your edit breaks
  the whole app** (blank screen). Always do the render check (below).
- The app **rebuilds `<head>` on boot and strips PWA `<link>`s** → the outer `index.html`
  has a re-injection script (manifest/icon links via interval+MutationObserver). Don't
  remove it or installability breaks (`no-manifest`). See `README.md` / create-pwa §1c-bis.

## UI structure cheat-sheet (`9812c350`)

- Two screens via `BottomBar`: the journey scroll and the **Info tab** (`InfoScreen`).
- `InfoScreen` renders `InfoBlock({icon,title,items,c})` — `items` is an array of strings,
  shown as an always-visible bulleted list. Existing blocks: `MUST VERIFY`, `BAGS`,
  `CAR PICKUP`, `PASSENGERS`, then the **UK ETA** block we added, then a "Printable
  overview" link.
- Card refs/phones/flight-nos are hidden until a card is tapped (journey screen pattern).
- Colors: `const d = TC.day` → `d.wed/.thu/.fri/.sun…` each `{main, soft}`.
- Example (how the UK ETA block was added, right after the `PASSENGERS` InfoBlock):
  ```jsx
  <InfoBlock icon="check" title="UK ETA" items={[
    "LUC  ·  2020-0000-3433-8252", "LUD  ·  2020-0000-3433-8722",
    "CA  ·  2020-0000-5516-5062",  "LI  ·  2020-0000-5516-4270",
  ]} c={d.thu} />
  ```

## Scripts (copy-paste; need Python 3 + Node ≥21 + Chrome)

### inspect.py — decode the bundle, find/print a resource's source
```python
import sys, re, base64, json, gzip
html = open(sys.argv[1], encoding="utf-8").read()                 # path to index.html
big  = max(re.findall(r"<script\b[^>]*>(.*?)</script>", html, re.S), key=len).strip()
data = json.loads(big)
needle = sys.argv[2] if len(sys.argv) > 2 else "InfoScreen"       # content to find
for k, v in data.items():
    if not v.get("compressed"):
        continue
    try: src = gzip.decompress(base64.b64decode(v["data"])).decode("utf-8", "replace")
    except Exception: continue
    if needle in src:
        print(f"=== {k[:8]} ({v['mime']}, {len(src)} chars) ===")
        i = src.find(needle); print(src[max(0,i-200):i+1200]); break
```
Run: `python inspect.py index.html "function InfoScreen"`

### repack.py — edit one resource and swap it back (surgical)
```python
import re, base64, json, gzip, io
PATH = "index.html"; PREFIX = "9812c350"        # resource to edit
html = io.open(PATH, encoding="utf-8").read()
big  = max(re.findall(r"<script\b[^>]*>(.*?)</script>", html, re.S), key=len).strip()
data = json.loads(big); key = next(k for k in data if k.startswith(PREFIX))
old  = data[key]["data"]
src  = gzip.decompress(base64.b64decode(old)).decode("utf-8")

# --- EDIT HERE: anchor must match exactly once ---
anchor = re.findall(r'<InfoBlock[^>]*title="PASSENGERS"[^>]*/>', src)
assert len(anchor) == 1, f"anchor matched {len(anchor)}"
INSERT = '\n      <InfoBlock icon="check" title="..." items={[ "..." ]} c={d.thu} />'
src2 = src.replace(anchor[0], anchor[0] + INSERT, 1)
assert src2 != src
# -------------------------------------------------

new = base64.b64encode(gzip.compress(src2.encode("utf-8"), mtime=0)).decode("ascii")
assert html.count(old) == 1
io.open(PATH, "w", encoding="utf-8", newline="").write(html.replace(old, new))
# self-check
d2 = json.loads(max(re.findall(r"<script\b[^>]*>(.*?)</script>", io.open(PATH,encoding='utf-8').read(), re.S), key=len))
gzip.decompress(base64.b64decode(d2[key]["data"]))  # raises if broken
print("OK")
```

### verify (Node, headless Chrome) — render + installability
```js
// node check.js https://ibwedding.stuparu.eu/   (needs Chrome at the path below)
const {spawn}=require('child_process'),os=require('os');
const CH='C:/Program Files/Google/Chrome/Application/chrome.exe',URL=process.argv[2]+'?x='+Date.now(),P=9377;
const c=spawn(CH,['--headless=new','--disable-gpu','--no-first-run',`--remote-debugging-port=${P}`,`--user-data-dir=${os.tmpdir()}/c${process.pid}`,URL],{stdio:'ignore'});
const s=ms=>new Promise(r=>setTimeout(r,ms));
(async()=>{let ws;for(let i=0;i<40;i++){try{const t=await(await fetch(`http://localhost:${P}/json`)).json();const p=t.find(x=>x.type==='page');if(p){ws=new WebSocket(p.webSocketDebuggerUrl);break;}}catch(e){}await s(500);}
let id=0,pend={};const send=(m,p={})=>new Promise(r=>{const i=++id;pend[i]=r;ws.send(JSON.stringify({id:i,method:m,params:p}));});
await new Promise(r=>ws.onopen=r);ws.onmessage=e=>{const d=JSON.parse(e.data);if(d.id&&pend[d.id]){pend[d.id](d);delete pend[d.id];}};
await send('Page.enable');await send('Runtime.enable');await s(10000);
const ev=async(x,a=false)=>(await send('Runtime.evaluate',{expression:x,returnByValue:true,awaitPromise:a})).result.result.value;
console.log('rendered:',await ev(`document.querySelector('#root')?.innerText.length>20`));
console.log('manifest:',(await send('Page.getAppManifest')).result.url||'(none)');
console.log('installable:',JSON.stringify((await send('Page.getInstallabilityErrors')).result.installabilityErrors));
console.log('nav→find text:',await ev(`(async()=>{const want='UK ETA';if(document.body.innerText.includes(want))return'ok';for(const b of document.querySelectorAll('button,a')){try{b.click()}catch(e){}await new Promise(r=>setTimeout(r,250));if(document.body.innerText.includes(want))return'ok-after-click'}return'NOT FOUND'})()`,true));
c.kill();process.exit(0)})().catch(e=>{console.error(e.message);process.exit(1)});
```
Expect: `rendered: true`, `manifest: https://…/manifest.webmanifest`, `installable: []`.

## Deploy + gotchas

1. **Bump `CACHE` in `sw.js`** every deploy (`wedding-run-vN` → `vN+1`) or returning
   clients keep the old precached bundle.
2. `git commit && git push origin main`.
3. **Pages build queues and lags** — poll:
   `gh api repos/lucianstuparu/ibwedding/pages/builds/latest --jq .status` until `built`
   (often 2-4 min; the `commit` field shown lags, trust the live fetch).
4. Verify against the **live** URL with a cache-buster (`?x=<ts>`). The HTML is CF
   `DYNAMIC` (served fresh); **`sw.js`/assets may sit in CF cache** and our DNS-scoped
   "Cloudflare API token" **cannot purge** (10000 auth error) — wait, or purge via the
   Cloudflare dashboard.
5. On device: pull-to-refresh / reopen once so the new service worker activates.

## Don't

- Don't edit the React/Babel/font resources (SRI-hashed → break) or remove the
  manifest-reinjection script in `index.html`.
- Don't hand-edit the giant base64 blob in a text editor — always go via `repack.py`.
- Don't skip the render check — a JSX typo blanks the whole app silently.
