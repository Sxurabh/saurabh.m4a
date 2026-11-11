# Desktop DJ Portfolio — Design

**Date:** 2026-11-11
**Status:** Draft for approval
**Chosen approach:** A — Desktop shell with draggable windows (single page)

---

## 1. Intent and why

A DJ's website that *is* a macOS desktop. Visitors land on a desktop, browse folders of
the DJ's work, play tracks, grab downloads, and see the latest releases — and they can
leave with a track still playing.

The metaphor is the point. A normal artist page lists links; this one lets someone
*explore* the DJ's hard disk the way you'd poke around a friend's laptop: open the
"Latest Releases" folder, double-click a track, watch a Finder window spring open, hear
it play while you dig into "Unreleased" in another window. It should feel like a small
piece of software, not a landing page.

Why now: the reference site (karankanchan.com) proves the format lands for music
audiences, but it is a Framer page that navigates away on every folder and never plays
audio itself. The opening is to build the same feeling *properly* — persistent, playful,
and actually playable.

**Success looks like:** a visitor double-clicks a track within ~5 seconds of landing,
hears it play, keeps browsing with the music uninterrupted, and downloads something or
hits the latest release before they leave.

---

## 2. Goals

- **Desktop metaphor, done for real.** Icons on a desktop, draggable Finder-style
  windows, a Dock, a menu bar. Multiple windows open at once.
- **Music first.** Clicking a track plays it immediately; playback survives opening,
  moving, and closing windows. A persistent player lives in the Dock.
- **Browsing is the journey.** Folders nest (Latest Releases, Edits, Unreleased, Collabs,
  Samples). Deep enough to explore, shallow enough that nothing is buried.
- **Download and releases are obvious.** Every track can be downloaded; "Latest Releases"
  is a first-class destination, not an easter egg.
- **It feels like Apple made it.** Motion, materials, and type follow the Apple design
  principles in §8 *strictly* — see that section for the concrete values.
- **Works on a phone too.** Same metaphor, adapted: tap to open, windows become sheets.

## 3. Non-goals

- Not a real OS. No file creation/editing, no terminal, no real filesystem.
- Not a CMS or admin panel. Content lives in a hand-edited JSON file.
- No user accounts or server-side auth. Locked folders use a client-side password gate
  (a playful door, not security).
- Not an installable native/desktop app.

## 4. Anti-goals

- **The metaphor must never hide the music.** If a visitor can't find something to play
  in a few seconds, the design has failed — gimmick over substance.
- **Not slow.** No huge upfront audio or image payload; nothing blocks first paint.
- **Not inaccessible.** No keyboard trap, no motion that can't be reduced, no
  interaction that only works with a mouse.
- **Not a pixel-for-pixel macOS clone.** It should feel native to Apple's *craft*
  without shipping Apple's trademarks, icons, or wallpaper.

## 5. Constraints

- Reference site is desktop-first on a fixed canvas; ours must be responsive (desktop,
  tablet, phone).
- Audio files may be large; assume they are streamed from external storage, not bundled.
- Must run on Vercel with no server to maintain.
- Greenfield repo: `/Users/berz/GitHub Projects/saurabh.m4a` (currently empty, not yet git).

---

## 6. Approaches considered

| # | Approach | Trade-off | Verdict |
|---|----------|-----------|---------|
| **A** | **Desktop shell with draggable windows (single page).** One persistent desktop; double-click opens draggable Finder windows; Dock player keeps audio alive; folders sync to the URL. | Most engineering (window manager, focus, z-order), but the strongest metaphor and the only one where music never stops. | **Chosen** |
| B | Faithful to the reference: one URL per folder, full navigation. | Simplest and fastest to ship; but every folder is a page swap, audio is awkward to keep alive, feels less like a desktop. | Fallback |
| C | Hybrid: window chrome + persistent player, but each folder a real route underneath. | Deep links for free, but the most moving parts to keep in sync. | Rejected |

**Why A:** the goal "anyone can leave with a track playing" only feels right if music
survives while the visitor digs through folders. A persistent desktop with a Dock player
gives that for free; route-per-folder fights it. We keep A's shareability by syncing the
open folder to the URL (see §7.7).

**Tech stack (agreed):**

- **Next.js (App Router) + React + TypeScript** on Vercel.
- **Motion** (Framer Motion) for springs and gestures.
- **Pointer Events** for all dragging (never HTML5 drag-and-drop — it can't do 1:1).
- **HTML5 `<audio>`** (single shared instance) for playback; Web Audio `AnalyserNode`
  for a live waveform where used.
- **Zustand** for the window manager + player state (light, no boilerplate).
- **Tailwind CSS** with CSS custom properties for material tokens; Motion for behavior.
- **`localStorage`** for icon positions, window layout, and volume.
- **Audio & media hosting:** **Vercel Blob** (same-origin) as the default, with external
  links (e.g. Google Drive) allowed for very large files. Covers/images also on Blob.
- **Content:** one editable `filesystem.json` drives the entire desktop.

---

## 7. The design

### 7.1 The virtual filesystem (content model)

Everything on screen is generated from `filesystem.json` — a tree of typed nodes. No
component hard-codes a folder or track.

```jsonc
{
  "name": "Saurabh's Macintosh HD",     // placeholder DJ name
  "children": [
    { "id": "latest",  "type": "folder", "name": "Latest Releases", "icon": "folder",
      "children": [ /* audio + links */ ] },
    { "id": "edits",   "type": "folder", "name": "Edits",  "children": [ /* … */ ] },
    { "id": "collabs", "type": "folder", "name": "Collabs", "children": [ /* … */ ] },
    { "id": "unreleased", "type": "folder", "name": "Unreleased", "locked": true,
      "password": "letmein", "children": [ /* … */ ] },
    { "id": "samples", "type": "folder", "name": "Free Samples", "children": [ /* … */ ] },
    { "id": "t1", "type": "audio", "name": "Neon Bombay.mp3", "src": "/audio/…",
      "download": "https://…", "cover": "/covers/…", "duration": 214 },
    { "id": "spotify", "type": "link", "name": "Spotify",
      "href": "https://open.spotify.com/…", "icon": "app:spotify" },
    { "id": "readme", "type": "text", "name": "About.txt", "body": "…" }
  ]
}
```

Node types: `folder`, `audio`, `link`, `image`, `text`. A node may declare a custom
`icon` (a folder, a doc, or a named app icon) and, for `audio`, a `download` URL.

### 7.2 Desktop

```
┌───────────────────────────────────────────────────────────────┐
│   Saurabh            File  Edit  View  Go        🔊  🔋  Tue 9:41│  ← menu bar
│                                                               │
│                                                   ┌─────────┐ │
│                                                   │ Latest  │ │  ← icons, top-right,
│                                                   │Releases │ │    like macOS
│                                                   └─────────┘ │
│        ┌─────────┐   ┌─────────┐   ┌─────────┐    ┌─────────┐ │
│        │  Edits  │   │ Collabs │   │Unreleased│   │ Samples │ │
│        └─────────┘   └─────────┘   └─────────┘    └─────────┘ │
│                                                               │
│                                                               │
│                        ╭──────────────────╮                   │
│                        │ ▶  Neon Bombay   │                   │  ← Dock (translucent)
│                        │ ▓▓▓▓▓░░░ 1:02    │                   │    + now-playing
│                        ╰──────────────────╯                   │
│              [Finder][Music][Spotify][YT][IG][Mail]           │
└───────────────────────────────────────────────────────────────┘
```

- **Background:** a calm, custom wallpaper (a blurred dark gradient or the DJ's own
  artwork). Never busy enough to fight the icons.
- **Icons:** image + label underneath, arranged top-right by default. **Freely
  draggable**; positions persist in `localStorage`. Single-click selects (rounded
  highlight), double-click opens; **tap opens on touch**.
- **Menu bar:** DJ name (left), a few cosmetic menus (File/Edit/View/Go), and a live
  clock + status glyphs (right). Menus may open simple dropdowns; they are not required
  to do anything beyond that.

### 7.3 Finder window

- **Chrome:** draggable title bar; traffic-light buttons (close / minimize / zoom) top-left;
  folder name centered; back/forward chevrons.
- **Body:** a large-icon grid (default) of the folder's children. Optional list view.
- **Behavior:** resizable from edges/corners; **multiple windows** at once; click a window
  to bring it to front (z-order); close animates back toward the icon it came from.
- **Locked folders** open a small modal: title, "Enter the password to continue", input,
  shake on failure. This is a door, not security.

### 7.4 Dock

- Bottom-center, **translucent material** (`backdrop-filter: blur(20px) saturate(180%)`
  over a semi-transparent fill).
- Holds app shortcuts (Finder, Music, Spotify, YouTube, Instagram, Mail, Trash) and the
  **now-playing module**.
- **Magnification on hover** (icons grow toward the cursor), spring-based.
- A small dot under any app that has an open window; Trash is a drop target for
  discarded icons (cosmetic).

### 7.5 Player

- One shared `<audio>` element, created once at the app root, so playback is untouched by
  window changes.
- **Controls:** play/pause, seek/scrub, volume, next/prev within the folder's track queue,
  and **Download**.
- **Now-playing module** in the Dock: cover art, title, a scrubber, play/pause.
- **Music window** (opened from the Dock): track list for the current folder, cover art,
  and a waveform. **Decided:** audio is hosted on **Vercel Blob** (same-origin), so the
  live Web Audio `AnalyserNode` waveform works and downloads are real. For any track that
  is instead linked to an external host (e.g. a large file kept on Google Drive), the
  analyser would read silence, so that track **falls back to precomputed peaks**. Peaks are
  generated at build time for anything we host.
- Clicking any `audio` icon plays it and sets the queue to that folder's tracks.

### 7.6a Opening each node type (Call)

- **`folder`** → opens a Finder window in place (no navigation).
- **`audio`** → plays immediately and sets the folder's track queue.
- **`link`** and **Dock apps** (Spotify, Apple Music, YouTube, Instagram, Mail) → open in a
  **new browser tab** (never an in-page iframe, which would break out of the metaphor and
  run into X-Frame-Options/CSP walls). A short spring animation plays as the window "hands
  off".
- **`text`** → opens a small, resizable **TextEdit-style window** (a document viewer).
- **`image`** → opens a **Preview-style window** with the image fit to the window.

### 7.6b Download semantics (Call)

- `download` is **any URL**. If it points at a file we host, the control downloads it
  (real `Content-Disposition: attachment`). If it points at a store page (Bandcamp,
  Beatport, Drive), it opens in a new tab. The schema supports both, so the choice is
  per-track content, not per-build.
- Tracks inside a **locked folder** are not downloadable and not listed in the
  screen-reader/SEO fallback until the folder is unlocked. The lock is a door for
  presentation purposes only; it is not security.

### 7.6 Interaction & motion spec — Apple design skill, applied strictly

This is the part that makes it feel Apple-made. Values are non-negotiable defaults.

| Principle (skill §) | How we apply it |
|---|---|
| **Response, kill latency** (§1) | Icons and buttons highlight on **pointer-down**, not on click (`scale(0.97)`, ~100ms). Windows start opening on the double-click frame. No artificial delays, no debounce on the input path. |
| **Direct manipulation, 1:1** (§2) | Icon and window drags use **Pointer Events + `setPointerCapture`**, tracking the pointer 1:1 and **respecting the grab offset** (never snap-to-center on grab). |
| **Interruptibility** (§3) | Every window/icon animation is interruptible. Animations always start from the **live presentation value**, never the logical target. A window grabbed mid-open follows the pointer immediately. **No CSS transitions or `@keyframes` on anything gesture-driven** — springs only, so reversal blends velocity instead of hitting a "brick wall". 2D drags decompose into independent X and Y springs. |
| **Springs** (§4) | Default **critically damped: damping `1.0`, response `0.3–0.4`** (no overshoot) for opening windows, menus, sheets. **Bounce only after momentum** — `damping ~0.8` for a flicked/ thrown icon or a drag-release snap. |
| **Velocity handoff** (§5) | On drag release, the spring continues at the pointer's **release velocity** (Motion `velocity` option) so there's no seam between dragging and animating. |
| **Momentum projection** (§6) | A flicked icon/window is thrown to its resting point via Apple's projection `(v/1000)·d/(1−d)`, `d ≈ 0.998`, then snaps to the nearest slot — never snap from the release point. |
| **Spatial consistency** (§7) | A window **opens from and closes to** the icon that spawned it (`transform-origin` at the icon). Enter and exit travel the same path; reversible transitions mirror their easing. |
| **Hint in the direction** (§8) | Window-open scale/translate telegraphs the folder; Dock magnification points at the cursor. |
| **Rubber-banding** (§9) | Dragging a window past a screen edge resists progressively (`rubberband()` from the skill), then eases back — soft boundaries, never a hard stop. |
| **Gesture details** (§10) | ~10px hysteresis before a drag commits; tap highlights down / commits up; cancel-by-dragging-away. All gesture candidates detected in parallel from the first move. |
| **Frame smoothness** (§11) | Animate only `transform`/`opacity`; `will-change` on imminently-moving surfaces; `requestAnimationFrame`-driven. |
| **Materials & depth** (§12) | Dock, menu bar, and window chrome are translucent `backdrop-filter` layers with content scrolling under. **Material weight encodes hierarchy**; never stack a light translucent surface on a light one. Bigger surfaces get stronger blur + deeper shadow. Scroll-edge fade masks instead of hard 1px dividers. Glass surfaces **materialize** (animate blur + scale together), not plain opacity fades. |
| **Multimodal feedback** (§13) | Feedback fires on the causal event, on the **same frame**: a soft click on window-open/track-start, optional subtle sound on snap. Used sparingly. |
| **Reduced motion & accessibility** (§14) | `prefers-reduced-motion: reduce` → springs become short **opacity cross-fades**, no overshoot/parallax. `prefers-reduced-transparency: reduce` → dock/chrome go solid, blur off. `prefers-contrast: more` → near-solid fills with defined borders. No full-viewport moving backgrounds, no ~0.2 Hz loops, eased theme changes. |
| **Typography** (§15) | **System font first** (`-apple-system, system-ui`) — it already ships optical sizing and tracking. Tracking is **size-specific**: negative on large display text (`-0.02em`), near `0` on body. Leading tight on headings, looser on body. Layout spacing in `rem`/`em` so Dynamic Type scaling doesn't break it. |
| **Design foundations** (§16) | **Purpose** (every icon earns its place), **Agency** (undo/close, confirmation only for the truly destructive), **Familiarity** (macOS metaphors honored — close is top-left, Trash deletes), **Flexibility** (mouse, touch, keyboard; desktop and phone), **Simplicity** (few top-level folders; advanced depth one level down), **Craft** (every spacing/timing value deliberate), **Delight** (the result of the rest, not confetti). |

### 7.7 Routing & shareability

Single page in feel, but **real routes underneath**. A catch-all Next.js route renders the
desktop shell so that deep links (`/latest`, `/edits`, `/collabs`) **survive a cold load**
and are shareable — refresh works, and someone else opening your link lands straight in the
right folder with its window materializing on mount. Navigating between folders uses client
navigation (no reload), so the desktop and the player persist. A `?track=<id>` param
deep-links to a track and its queue.

**Call:** because the SEO / screen-reader fallback in §7.9 lives behind these routes, the
shell is server-rendered with the fallback list in the initial HTML.

### 7.8 Responsive & touch

- **Desktop (≥1024px):** full metaphor as above.
- **Tablet:** same, tighter icon grid, Dock shrinks.
- **Phone (<768px):** tap opens; **windows become full-screen sheets** that slide up and
  can be dragged down to dismiss (spring + rubber-band). Dock becomes a compact bar. Drag
  still works via touch. Icon layout switches to a comfortable grid.

### 7.9 Accessibility

- **Keyboard:** arrow keys move the desktop selection, `Enter` opens, `Esc` closes the top
  window, `Tab` cycles focus within a window; focus is trapped in modals and restored on
  close.
- **Semantics:** icons are buttons with accessible names; windows use `role="dialog"` with
  labels; the player exposes proper controls.
- **Screen-reader fallback:** a hidden-but-reachable plain list of folders, tracks, and
  links (also useful for SEO), so nothing is locked behind the metaphor. **Locked folders'
  contents are excluded** from this list until unlocked (§7.6b).
- **Motion/contrast/transparency:** all three preferences honored per §7.6.

---

## 8. What's left to the builder

- Exact content: folder names, tracks, covers, download URLs, and the locked-folder
  password(s) — supplied via `filesystem.json`.
- **Decided: we build with placeholder content.** Placeholder folders, a few short
  royalty-free/silent audio clips, and generated cover art let us verify playback, queue,
  download, and waveform end-to-end. Real assets are swapped into `filesystem.json` (and
  uploaded to Vercel Blob) later — no code changes needed.
- The wallpaper and the app icon set (must be original; no Apple trademarks).
- Whether the menu-bar menus open real dropdowns or stay cosmetic.
- Whether to implement window minimize ("genie") or keep it simple (zoom + close only).
- Optional ambient sound design for opens/snaps.
- The DJ's real name and branding, to replace the "Saurabh" placeholder.

---

## 9. Build order (preview — full plan to follow)

1. Scaffold Next.js + TypeScript + Tailwind + Motion on Vercel; repo init.
2. `filesystem.json` schema + loader + typed node model.
3. Desktop shell: wallpaper, menu bar, icon grid, select/drag (Pointer Events + springs).
4. Window manager: open/close/focus/z-order, drag, resize, rubber-band, momentum.
5. Finder content: icon grid, locked-folder modal.
6. Dock: material, magnification, running dots.
7. Player: shared audio, now-playing module, Music window, download, queue.
8. Routing/URL sync + deep links.
9. Responsive/touch sheets.
10. Accessibility pass (keyboard, SR fallback, the three media preferences).
11. Motion polish pass reviewed frame-by-frame; performance budget check.
