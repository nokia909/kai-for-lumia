# Building Web Apps for the Lumia 1020 (IE11 / Windows Phone 8.1)

A durable guide, written from what actually worked building **Kai for Lumia** — a
167-episode video player that runs on the Lumia 1020's Internet Explorer 11,
tested on the device on Nov 22, 2026. Follow this recipe and the app works
first time; deviate from it and IE11 fails silently.

## 0. The recipe in one paragraph

One single `.html` file, ES5-only JavaScript, pre-modern CSS, plain
`<video>` / `<audio>` tags, H.264 MP4 and MP3. Serve the file at a plain
direct URL (GitHub Pages) — never through a wrapper that needs modern JS.
Keep state in `localStorage`, guarded with try/catch. That's it. Everything
below is the detail behind those five sentences.

## 1. What was proven — and what failed first

**Main Kai does not work on the Lumia.** It uses `const`/`let`, arrow
functions, template literals, `fetch`, Promises, CSS variables, flexbox gap,
`aspect-ratio`, `clamp()`, plus the Cast SDK, a preloader, and stall
watchdogs. IE11 stops at the first modern-JS feature and the script never
runs. The MP4s themselves were never the problem — IE11 plays H.264 fine.

**Kai for Lumia works.** A separate, stripped-down build: ES5-only script,
float/block CSS, plain `<video>` element, direct MP4s from Archive.org.
Same 167 episodes, same titles, same 7 sagas (Saiyan through Evil Buu),
Original / Faulconer toggle, Prev/Next, saga jumps, an "Open the MP4
directly" fallback link, and the "Episode N didn't play / Try Again" error
panel. No Chromecast, preloader, stall watchdog, or offline support. File
size: ~48 KB with resume.

**The muse.ai share link failed — and the cause was not the app.** The
Lumia loaded the share page's header ("Kai For Lumia", "Content is user
generated and unverified", Log in / Sign up) with a blank white frame below.
The app itself is verified IE11-ready and would paint even with JavaScript
fully dead. The share shell fills its frame with client-side modern JS
(arrow functions, `??=`, WeakMap, blob-URL rewriting) that IE11 cannot
parse. No edit to the app can fix the shell. The fix was delivery, not
code: the same file served at a plain direct URL.

**The direct file worked on the first try.** Hosted as a single HTML file
at a direct link, shortened with is.gd for typing on the phone, episode 1
played on the Lumia on Nov 22, 2026. That is the proof this recipe works.

## 2. JavaScript: ES5 only, no exceptions

IE11 is a hard wall, not a styling issue. One forbidden feature anywhere
in the file and the whole script dies — no partial execution, no error
message on the phone.

**Banned:**

- `const`, `let` — `var` only, function scope
- Arrow functions — `function () {}` only
- Template literals — string concatenation with `+` only
- `fetch`, `Promise`, `async`/`await` — `XMLHttpRequest` for network
- Destructuring, default parameters, rest/spread, classes, modules
- `classList.toggle` with a second argument — use `className` string
  assignment instead (`el.className = 'sel'` / `el.className = ''`).
  Plain `classList` exists in IE11, but the two-argument toggle does not.
- `Array.prototype.includes`, `find`, `findIndex`, `Object.assign`,
  `String.prototype.startsWith/endsWith` — use `indexOf`, loops, manual
  checks
- `console.log` unguarded — `console` is undefined until dev tools open,
  and there are no dev tools on the phone. Either strip console calls or
  wrap them: `if (window.console && console.log) { console.log(...); }`

**Allowed and proven:**

- `var`, function declarations, function expressions, IIFEs
- `onclick` handlers (the proven pattern in Kai for Lumia). IE11 also has
  `addEventListener`, but the build didn't rely on it — onclick is the
  tested path.
- `setTimeout`, `setInterval`, `Date`, `Math`
- `JSON.parse` / `JSON.stringify` (IE11 has them)
- `localStorage` (see §5)
- `encodeURIComponent` — note: IE11's `encodeURIComponent` leaves
  `'`, `!`, `(`, `)`, `*` unescaped; Kai for Lumia re-encodes them
  manually for Archive.org paths (see the `encodePath` helper in §5)
- `video.play()` inside try/catch — IE11's `play()` can throw; never call
  it bare

**Audit before shipping.** Grep the finished file for the banned list. Kai
for Lumia's audit passed with zero occurrences of: `const`/`let`, arrow
functions, template literals, `fetch`, `Promise`, `classList`, CSS `var()`,
`aspect-ratio`, `clamp()`, `dvh`, `flex`, `gap`. Run the same check on
every new app:

```
grep -nE 'const |let |=>|`|fetch\(|Promise|classList|var\(|aspect-ratio|clamp\(|dvh|display:\s*flex|gap:' app.html
```

Any hit is a bug. (Watch for false positives: the word "flex" inside a
comment, backticks inside a string. Read each hit.)

## 3. CSS: pre-modern layout only

**Banned:** CSS variables (`var(--x)`), flexbox, grid, `gap`,
`aspect-ratio`, `clamp()`, `min()`/`max()`, `dvh`/`svh`/`lvh`,
`position: sticky`, `backdrop-filter`.

**The proven layout:** floats, `display: block` / `inline-block`, fixed
pixel widths, percentage widths, `max-width` centering. Kai for Lumia's
stage is just:

```css
.wrap { max-width: 720px; margin: 0 auto; padding: 14px 12px 32px 12px; }
.stage video { display: block; width: 100%; }
```

No aspect-ratio hack needed — the video element sizes itself from the
stream. For a fixed-ratio box (e.g. a 16:9 placeholder before the video
loads), use the padding-top trick: a wrapper with `padding-top: 56.25%`
and an absolutely positioned child.

**Viewport.** Always include both of these — the second is IE11-specific:

```html
<meta name="viewport" content="width=device-width, initial-scale=1" />
```

```css
@-ms-viewport { width: device-width; }
```

**Paint without JavaScript.** Put the real UI in the initial static HTML,
not generated by script. Kai for Lumia's stage, title, toggles, and error
box are all in the markup, so the dark UI paints in IE11 even if every
script fails. Lists can be built by script (Kai builds its 167 episode
buttons in JS), but the frame around them must exist in HTML. This is
also how you diagnose failures on the phone: if the static UI shows and
the list doesn't, the script died; if nothing shows, the file never
loaded.

**Fonts.** Use the system stack. Windows Phone's UI font is Segoe UI;
declare `font-family: "Segoe UI", Tahoma, sans-serif;` and stop. No web
fonts — WOFF technically works in IE11, but every extra request is weight
on old Wi-Fi for no job.

**Touch targets.** The Lumia is a 4.5" phone. Buttons and list rows want
at least ~44px height. Kai's episode rows are full-width buttons — big,
tappable, no precision needed.

## 4. Media: what the phone can actually play

- **Video:** H.264 MP4, AAC-LC audio. The Kai episodes are H.264 Main
  profile, 640×480, faststart — proven to play. Keep to that envelope.
  No HLS, no DASH, no MSE, no DRM, no WebM.
- **Audio:** MP3. Plain `<audio>` tag, same rules as video.
- **Images:** JPEG, PNG, GIF. No WebP, no AVIF. (DNGs do not display in
  a browser — galleries use the JPEGs.)
- **Fallback host pattern.** Kai for Lumia streams from a primary
  Archive.org host and retries once from `archive.org/download/` on
  `video.onerror`. That one retry is the whole error strategy, and it
  worked. A mid-stream stall watchdog was considered and rejected: on
  the Lumia it would compete for bandwidth with the stream and risk false
  reloads. If a stream fails, show the error panel with Try Again and
  let the user decide.

```js
video.onerror = function () {
  if (!video.src) return;
  if (!triedFallback) {
    triedFallback = true;
    video.src = fallbackUrl(current);
    try { video.play(); } catch (e) {}
  } else {
    showError(); /* "Episode N didn't play." + Try Again */
  }
};
```

## 5. State: localStorage, guarded

IE11 supports `localStorage`, and it is the only persistence you get —
no service workers, no Cache API, no IndexedDB worth trusting (IE11 has
it, but it is buggy; don't build on it). Rules:

- **Test availability once, at load.** Storage can be unavailable or
  throw (full, disabled, private mode). Every read and write sits inside
  try/catch.
- **Strings only.** `JSON.stringify` on save, `JSON.parse` in try/catch
  on load. Validate what comes back — a corrupt value must return null,
  never crash the app.
- **Per-origin, ~5–10 MB.** Keep saved state small. A resume record is
  ~60 bytes; a notebook with a hundred notes is fine; a photo library
  index is not.
- **Keys are namespaced per app** (`kaiLumiaResumeV1`), so two apps never
  collide.

### Reference implementation: Kai for Lumia resume

This is the real code from Kai for Lumia, condensed only where the
episode logic is specific to that app. Copy the shape for any app that
needs "remember where I was."

```js
var RESUME_KEY = 'kaiLumiaResumeV1';
var pendingSeek = null;
var seekApplied = true;
var lastSaveAt = 0;

function storageAvailable() {
  try {
    if (typeof localStorage === 'undefined' || !localStorage) return false;
    var k = '__test__';
    localStorage.setItem(k, '1');
    localStorage.removeItem(k);
    return true;
  } catch (e) { return false; }
}
var canStore = storageAvailable();

function loadResume() {
  if (!canStore) return null;
  try {
    var raw = localStorage.getItem(RESUME_KEY);
    if (!raw) return null;
    var data = null;
    try { data = JSON.parse(raw); } catch (e2) { return null; }
    if (!data) return null;
    var ep = parseInt(data.ep, 10);
    if (isNaN(ep) || ep < 1 || ep > TOTAL) return null;
    var t = parseFloat(data.t);
    if (isNaN(t) || t < 0) t = 0;
    var dur = parseFloat(data.dur);
    if (isNaN(dur) || dur < 0) dur = 0;
    var ed = (data.ed === 'faulconer') ? 'faulconer' : 'original';
    return { ep: ep, t: t, dur: dur, ed: ed };
  } catch (e) { return null; }
}

function saveResume() {
  if (!canStore) return;
  /* Do not overwrite a pending restore with the pre-seek position. */
  if (pendingSeek !== null && !seekApplied) {
    try {
      if (video.currentTime < pendingSeek - 5) return;
    } catch (e) {}
  }
  try {
    var t = 0, dur = 0;
    try { t = video.currentTime || 0; } catch (e1) {}
    try { dur = video.duration || 0; } catch (e2) {}
    if (isNaN(t) || t < 0) t = 0;
    if (isNaN(dur) || dur < 0) dur = 0;
    var payload = '{"ep":' + current + ',"ed":"' + edition +
      '","t":' + t + ',"dur":' + dur + '}';
    localStorage.setItem(RESUME_KEY, payload);
  } catch (e) {}
}
```

The save points — this is the part people get wrong. Saving only on
page close loses everything if IE11 is killed; saving on every
`timeupdate` hammers storage. Kai for Lumia does both, throttled:

```js
video.ontimeupdate = function () {
  var nowMs = 0;
  try { nowMs = new Date().getTime(); } catch (e) {}
  if (nowMs - lastSaveAt >= 5000) {   /* every 5 seconds while playing */
    lastSaveAt = nowMs;
    saveResume();
  }
};
video.onpause = function () { saveResume(); };
video.onplay = function () { saveResume(); };
video.onseeked = function () { saveResume(); };

if (window.addEventListener) {
  window.addEventListener('beforeunload', function () { saveResume(); }, false);
  window.addEventListener('pagehide', function () { saveResume(); }, false);
} else if (window.attachEvent) {
  window.attachEvent('onbeforeunload', function () { saveResume(); });
}
```

The seek must wait for the video to be ready — setting `currentTime`
before metadata loads is silently ignored in IE11:

```js
function applyPendingSeek() {
  if (pendingSeek === null || seekApplied) return;
  try {
    if (video.readyState >= 1) {
      var target = pendingSeek;
      var dur = 0;
      try { dur = video.duration || 0; } catch (e) {}
      if (dur > 0 && target > dur - 5) target = 0;
      if (target > 0) {
        try { video.currentTime = target; } catch (e2) {}
      }
      seekApplied = true;
      pendingSeek = null;
    }
  } catch (e) {}
}
video.onloadedmetadata = function () { applyPendingSeek(); };
video.oncanplay = function () { applyPendingSeek(); };
```

And the restore, on load — including the near-end rule (left within the
last 15 seconds of an episode? start the next one instead) and the
edition being remembered across restores and switches:

```js
(function () {
  var saved = loadResume();
  if (saved) {
    var ep = saved.ep;
    var t = saved.t;
    var dur = saved.dur;
    if (dur > 0 && t > 0 && t >= dur - 15) {
      if (ep < TOTAL) { ep = ep + 1; }
      t = 0;
    }
    if (saved.ed !== edition) { edition = saved.ed; /* update toggle UI */ }
    if (ep !== 1 || t > 5 || edition !== 'original') {
      if (t > 5) { selectEpisode(ep, false, t); showResumeNote(ep, t); }
      else { selectEpisode(ep, false); }
      return;
    }
  }
  markSelected();
})();
```

The resume note is a small line under the player — "Resumed Episode N
at M:SS — remembered on this device." — with a **Start over** button
that seeks to 0:00 and overwrites the saved position. Always give the
user a way out of a bad restore.

**What the resume deliberately does not do:** preload the next episode
(would compete for bandwidth with the one playing, on a slow stream and
old Wi-Fi — likely more stalls, not fewer), or run a stall watchdog
(the fallback-host retry on error already covers failures; a watchdog
is real work and real risk of false reloads).

## 6. Hosting: serve it direct, or it doesn't exist

The Lumia must open a plain `.html` URL top-level in IE11. Anything
that wraps the page in a JavaScript loader fails — proven by the muse.ai
share shell (§1).

**Permanent, free: GitHub Pages.** The recommended host. Free, no
expiry, serves a plain HTML file directly with the right content type.

1. Create a GitHub account (free, no payment) and a public repo,
   e.g. `lumia-apps`.
2. Upload the single HTML file, e.g. `kai-for-lumia.html`.
3. Repo Settings → Pages → deploy from the main branch.
4. The app lives at `https://<username>.github.io/lumia-apps/kai-for-lumia.html`.
5. Shorten it with is.gd (no login) so it's typable on the phone:
   `https://is.gd/<short>`.

**Temporary, no signup:** ship.page — upload the HTML, get a link, lasts
7 days, then it's gone. Netlify Drop — a link with no account is deleted
within 1 hour; claiming it needs a free account. Muse's built-in file
storage also expires (Kai for Lumia's test link expired Nov 24, 2026,
10:08 p.m. UTC). Use these only to test.

**Check before trusting a host:** open the direct link on the Lumia. If
IE11 renders the page, the host works. If it prompts to download the
file instead, the host is serving the wrong content type — find another
host. HTTPS only; IE11 on Windows Phone 8.1 speaks TLS 1.2, and
GitHub Pages is fine.

**One file per app, one URL per file.** Don't build a multi-page app
with relative links and a router — every app is one self-contained file.
New app, new file, new short link. The Lumia's home screen (or a
bookmarks page, which is itself one of these apps) holds the links.

## 7. What works, what doesn't

**Works (proven or directly implied by the recipe):**

- Video and audio playback (H.264 MP4, MP3)
- Lists, buttons, taps, forms, text input
- localStorage state (resume, settings, notes)
- Images (JPEG/PNG/GIF) in a gallery
- Timers via `setInterval`
- `XMLHttpRequest` to CORS-enabled endpoints (Archive.org direct file
  links work; a JSON API that sends `Access-Control-Allow-Origin: *`
  works too)
- Anchor navigation and hash-based "screens" (`#notes`, `#settings`)
  with `location.hash` — the cheap way to have multiple views in one
  file

**Doesn't work — don't build on it:**

- Service workers, offline cache, AppCache (no offline; the app needs
  network every load, or the file must be opened locally)
- Chromecast / Cast SDK
- Login, OAuth, or anything that opens a popup and expects modern JS
  to complete the flow
- DRM, HLS, DASH, MSE
- `fetch`, Promises, or any library written after ~2015 (jQuery 1.x /
  2.x is the last IE11-safe line; in practice, you don't need a library
  at all for these apps — Kai for Lumia uses none)
- Anything that depends on a live API that only speaks modern JS

**Unproven — test before building on it:**

- WebSockets (IE11 has them; never tested on the Lumia)
- Geolocation (IE11 has it; the phone's location stack is old)
- `XMLHttpRequest` to an API that doesn't send CORS headers (IE11
  falls back to XDomainRequest for cross-origin, which is more
  limited — same-origin or CORS-enabled only, in practice)

## 8. Worked examples

Each of these is one ES5 single file, following §2–§6. The player is
the reference; the rest are the same pattern with different data.

### A second show player (same as Kai for Lumia)

Copy the Kai for Lumia file, swap the episode list and hosts. The
structure is: a `TITLES` array (or object), a host URL, a fallback
host URL, a `<video>` element, Prev/Next, a list built by script, the
error panel, and the resume code from §5. The episodes are direct MP4
links from Archive.org. Candidates already surveyed: Fairy Tail,
Star Trek: Voyager, BBC nature series — any complete Archive.org series
with direct MP4s.

### A photo gallery (the Lumia showing its own photos)

A grid of thumbnails (small JPEGs), tap to view full-size. Layout:
floated `inline-block` thumbnails, `width: 31%` with margins. The
full view is one `<img>` with a back button. Keep the index small —
thumbnails link to files, they aren't embedded. DNGs can't display;
gallery uses the JPEGs, DNGs stay on the PC for editing. A "favorites"
toggle can be stored in localStorage (one key, a JSON array of
filenames).

### A notebook (Moleskine in the browser)

A `<textarea>`, a Save button, and a list of saved notes. Storage: one
localStorage key holding a JSON array of `{title, body, date}`. No
account, no sync — the notes live on the device, which is the point.
Keep it plain: new note, open note, delete note. That is the whole app.

```js
var NOTES_KEY = 'lumiaNotesV1';
function loadNotes() {
  try {
    var raw = localStorage.getItem(NOTES_KEY);
    if (!raw) return [];
    var notes = JSON.parse(raw);
    if (notes instanceof Array) return notes;
    return [];
  } catch (e) { return []; }
}
function saveNotes(notes) {
  try { localStorage.setItem(NOTES_KEY, JSON.stringify(notes)); }
  catch (e) { /* storage full or unavailable: keep the note in memory only */ }
}
```

### A music player

MP3s already play locally on the phone, but a single-file player gives
one list, big touch targets, and the 1020's speaker. Same shape as the
video player with `<audio>` instead of `<video>`: a track list, Prev/
Next, and the resume pattern from §5 (remember track + position). Point
it at MP3 files on a host, or at files the Lumia can reach over the
local network. No streaming service will work — no login, no DRM.

### A timer

The simplest app in the set, and a good first build to prove the
pipeline: an input for minutes, a Start button, a big countdown via
`setInterval`, a beep via `<audio>` at zero. No state needed, or save
the last-used duration in localStorage. If this works on the Lumia, the
hosting and the ES5 rules are proven for everything else.

### Simple tools

Unit converter, checklist, tip calculator — anything that is a form plus
a little arithmetic. The rule for all of them: all logic in one
`<script>`, all state in localStorage if it needs to persist, nothing
fetched at runtime.

## 9. Testing checklist (on the Lumia, every time)

1. Open the direct link. Page paints immediately, including the static
   UI — if it's blank, the host or the file is wrong, not the code.
2. Tap every button. Nothing should do nothing; a dead tap usually
   means the script died above it — check the banned list (§2).
3. Play the media. Video and audio both, start to finish of at least
   one item.
4. Close IE11 mid-episode (or mid-note, mid-track). Reopen the link.
   State restores — that's the §5 test.
5. Test the error path: give it a bad URL once and confirm the error
   panel and Try Again appear.
6. Check the file size. Kai for Lumia is ~48 KB. Descriptions alone
   would have added ~130 KB — weight is a real cost on old Wi-Fi, so
   every kilobyte has to earn its place.
7. Type the short link from scratch. If it's hard to type on the phone,
   shorten it again.

## 10. The ask pattern

When you want a new Lumia app, ask for it the same way: **"ES5-only,
single file, for the Lumia."** Build it that way from the start instead
of building modern and back-porting — back-porting is where the bugs
hide, because every modern feature has to be found and removed by hand.

---

*Proven on: Lumia 1020 (NOKIA 909, AT&T), Windows Phone 8.1 Update,
OS 8.10.14226.359 (Denim), IE11. Reference app: Kai for Lumia,
tested Nov 22, 2026. Main Kai untouched.*
