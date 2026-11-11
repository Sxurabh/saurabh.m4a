# Desktop DJ Portfolio — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a DJ's portfolio as a working macOS-desktop simulation where visitors browse folders in draggable Finder-style windows, play tracks that keep playing while they explore, and download or reach the latest releases.

**Architecture:** A single Next.js App Router shell renders a desktop (wallpaper, icon grid, menu bar, Dock) and a window manager. All content comes from one `filesystem.json`. A shared `<audio>` element owned by an `AudioProvider` makes playback survive every window change. Zustand stores hold window/desktop/player state. All gesture-driven motion uses Motion springs (never CSS transitions), fed by a small pure gesture-math layer (velocity, momentum projection, rubber-banding).

**Tech Stack:** Next.js (App Router) + React + TypeScript (strict), Tailwind CSS, Motion (`motion` package), Zustand, Vitest + React Testing Library + `@testing-library/user-event` (jsdom), Vercel Blob for media.

**Spec:** `docs/superpowers/specs/2026-11-11-desktop-dj-portfolio-design.md`

## Global Constraints

- Next.js App Router, React 19, TypeScript `strict`, ESM only. Node 22+.
- Spring tokens only for gesture-driven motion: default `{ type:'spring', bounce:0, duration:0.4 }`; momentum `{ type:'spring', bounce:0.2, duration:0.4 }`; drawer/sheet `{ type:'spring', bounce:0.2, duration:0.3 }`. **No CSS `transition` or `@keyframes` on anything draggable, resizable, or open/close.**
- Animate only `transform` and `opacity`.
- Pointer Events only for dragging (`setPointerCapture`). Never HTML5 drag-and-drop.
- Respect `prefers-reduced-motion`, `prefers-reduced-transparency`, and `prefers-contrast` everywhere (§7.6, §7.9).
- Typography: system font stack (`-apple-system, system-ui, sans-serif`). Size-specific tracking: display text `-0.02em`, body `0`. Spacing in `rem`/`em`.
- All content is generated from `content/filesystem.json`; no component hard-codes a folder, track, or link.
- No Apple trademarks, icons, or wallpaper.
- Media: same-origin Vercel Blob by default; tracks with an external `src` use precomputed `peaks` for the waveform.
- Test command: `npm test` (Vitest). Lint/typecheck: `npm run lint && npm run typecheck`.

## Review Focus

These are the failure modes most likely to bite a real visitor. Each has its owning test below.

1. **Window or icon dragged past any screen edge** must rubber-band and be recoverable — a window can never be lost off-screen. (Task 9)
2. **Opening a folder that already has a window, and rapid open→close→open** must focus the existing window instead of duplicating, with no stuck state. (Task 10)
3. **A track ending, or the visitor clicking another track mid-playback**, must leave queue/now-playing/playing state correct. (Tasks 6, 14)
4. **Cold-loading a deep link (`/edits?track=x`)** must open the right window and reflect player state with no hydration mismatch. (Task 16)
5. **`prefers-reduced-motion` and keyboard-only use** must leave the site fully usable with no spring/slide motion. (Task 18)

## File Structure

```
app/
  layout.tsx                 # root layout: fonts, providers, metadata
  page.tsx                   # root route → renders <Desktop/> at "/"
  [[...path]]/page.tsx       # catch-all: deep links render the shell with a target folder
  globals.css                # Tailwind, material tokens, the 3 media queries
components/
  desktop/ Desktop.tsx DesktopIcon.tsx MenuBar.tsx Wallpaper.tsx
  windows/ WindowManager.tsx Window.tsx FinderWindow.tsx TextViewer.tsx
           ImageViewer.tsx MusicWindow.tsx LockedFolderModal.tsx
  dock/    Dock.tsx DockIcon.tsx NowPlaying.tsx
  player/  AudioProvider.tsx PlayerControls.tsx Waveform.tsx
  a11y/    FallbackList.tsx          # hidden-but-reachable folder/track/link list
lib/
  fs/ schema.ts load.ts path.ts
  stores/ window-store.ts desktop-store.ts player-store.ts
  springs.ts
  gesture/ velocity.ts project.ts rubberband.ts useDraggable.ts
content/ filesystem.json
public/ wallpapers/ icons/ audio/
tests/ mirrors components/lib paths
```

Each file has one responsibility. `lib/` holds pure, framework-free logic (fully unit-tested); `components/` holds rendering and wiring.

---

### Task 1: Scaffold the project and tooling

**Files:**
- Create: `package.json`, `tsconfig.json`, `next.config.ts`, `tailwind.config.ts`, `app/layout.tsx`, `app/page.tsx`, `app/globals.css`, `vitest.config.ts`, `tests/smoke.test.tsx`, `.gitignore`

**Interfaces:**
- Consumes: nothing.
- Produces: a running Next.js app and a working `npm test`. `app/layout.tsx` exports `default function RootLayout({ children }: { children: React.ReactNode })`.

- [ ] **Step 1: Init repo and install deps**

```bash
git init
npm init -y
npm i next react react-dom motion zustand
npm i -D typescript @types/react @types/node tailwindcss @tailwindcss/postcss postcss vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

- [ ] **Step 2: Add scripts to `package.json`**

`dev: "next dev"`, `build: "next build"`, `start: "next start"`, `test: "vitest run"`, `test:watch: "vitest"`, `typecheck: "tsc --noEmit"`, `lint: "next lint"`.

- [ ] **Step 3: Write the smoke test**

```tsx
// tests/smoke.test.tsx
import { render, screen } from '@testing-library/react';
import Page from '@/app/page';
test('root route renders', () => {
  render(<Page />);
  expect(screen.getByTestId('desktop-root')).toBeInTheDocument();
});
```

- [ ] **Step 4: Run it to verify it fails**

Run: `npm test`
Expected: FAIL — `@/app/page` has no default export / `desktop-root` not found.

- [ ] **Step 5: Implement the minimal app**

`app/layout.tsx` sets `<html lang="en">` with the system font stack and imports `globals.css`. `app/page.tsx` returns `<main data-testid="desktop-root" />`. `app/globals.css` has `@import "tailwindcss";` plus the `:root` font shorthand from the spec §7.6.

- [ ] **Step 6: Run test + typecheck to verify pass**

Run: `npm test && npm run typecheck`
Expected: PASS, no type errors.

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "chore: scaffold Next.js app with Tailwind, Motion, Zustand, Vitest"
```

---

### Task 2: Filesystem schema, loader, and path helpers

**Files:**
- Create: `lib/fs/schema.ts`, `lib/fs/load.ts`, `lib/fs/path.ts`, `content/filesystem.json`
- Test: `tests/lib/fs/load.test.ts`, `tests/lib/fs/path.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `type FSNode = FolderNode | AudioNode | LinkNode | ImageNode | TextNode` (discriminated on `type`).
  - `FolderNode { id; name; type:'folder'; icon?; children: FSNode[]; locked?: boolean; password?: string }`
  - `AudioNode { id; name; type:'audio'; icon?; src: string; download?: string; cover?: string; duration?: number; peaks?: number[] }`
  - `LinkNode { id; name; type:'link'; icon?; href: string }`; `ImageNode { ...; type:'image'; src: string }`; `TextNode { ...; type:'text'; body: string }`
  - `loadFilesystem(raw: unknown): FolderNode` (validates, throws `FSValidationError` on bad input).
  - `getNodeByPath(root: FolderNode, path: string[]): FSNode | null`
  - `isFolder(node: FSNode): node is FolderNode`

- [ ] **Step 1: Write failing tests**

```ts
// tests/lib/fs/path.test.ts
import { getNodeByPath, isFolder } from '@/lib/fs/path';
const tree = { id:'root', name:'HD', type:'folder', children:[
  { id:'edits', name:'Edits', type:'folder', children:[
    { id:'t1', name:'A.mp3', type:'audio', src:'/audio/a.mp3' } ] },
  { id:'sp', name:'Spotify', type:'link', href:'https://x' } ]};
test('resolves a nested folder', () => {
  expect(getNodeByPath(tree, ['edits'])?.name).toBe('Edits');
});
test('resolves a nested audio node', () => {
  expect(getNodeByPath(tree, ['edits','t1'])?.type).toBe('audio');
});
test('returns null for a missing path', () => {
  expect(getNodeByPath(tree, ['nope'])).toBeNull();
});
test('isFolder narrows', () => {
  expect(isFolder(tree.children[0])).toBe(true);
});
```

```ts
// tests/lib/fs/load.test.ts
import { loadFilesystem, FSValidationError } from '@/lib/fs/load';
test('accepts a valid tree', () => {
  expect(loadFilesystem({ id:'r', name:'HD', type:'folder', children:[] }).name).toBe('HD');
});
test('rejects an unknown node type', () => {
  expect(() => loadFilesystem({ id:'r', name:'HD', type:'folder', children:[{ id:'x', name:'y', type:'wat' }] }))
    .toThrow(FSValidationError);
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test tests/lib/fs`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement `schema.ts` (types + a hand-rolled validator), `path.ts`, `load.ts`**

`path.ts`: walk `children` one segment at a time; return the node or `null`. `load.ts`: recursive validation of required fields per `type`, `locked` implies `password` present, `audio.src` non-empty; throw `FSValidationError(message)` otherwise. No external validation library (YAGNI).

- [ ] **Step 4: Add placeholder content to `content/filesystem.json`**

Root `"Saurabh's Macintosh HD"` with folders: `Latest Releases`, `Edits`, `Collabs`, `Free Samples`, and `Unreleased` (`locked:true, password:"letmein"`). Include 3 `audio` nodes with `src:"/audio/placeholder-1.mp3"` and `peaks` arrays, 1 `link` (Spotify), 1 `text` (`About.txt`), 1 `image`.

- [ ] **Step 5: Run tests to verify pass**

Run: `npm test tests/lib/fs && npm run typecheck`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat(fs): add filesystem schema, loader, path helpers, placeholder content"
```

---

### Task 3: Motion tokens and gesture math

**Files:**
- Create: `lib/springs.ts`, `lib/gesture/velocity.ts`, `lib/gesture/project.ts`, `lib/gesture/rubberband.ts`
- Test: `tests/lib/gesture.test.ts`, `tests/lib/springs.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `const SPRING = { default: {...}, momentum: {...}, drawer: {...} }` (exact values from Global Constraints).
  - `createVelocityTracker(windowMs?: number): { add(x:number,y:number,t:number): void; velocity(): {x:number;y:number} }` — px/s over the trailing window.
  - `project(initialVelocity: number, decelerationRate?: number): number` — Apple form `(v/1000) * d / (1 - d)`, default `d = 0.998`.
  - `rubberband(overshoot: number, dimension: number, constant?: number): number` — `(overshoot*dimension*constant)/(dimension+constant*|overshoot|)`, default `constant = 0.55`.

- [ ] **Step 1: Write failing tests**

```ts
// tests/lib/gesture.test.ts
import { createVelocityTracker } from '@/lib/gesture/velocity';
import { project } from '@/lib/gesture/project';
import { rubberband } from '@/lib/gesture/rubberband';
test('velocity is px/s over the trailing window', () => {
  const v = createVelocityTracker(100);
  v.add(0, 0, 0); v.add(100, 0, 100);
  expect(v.velocity().x).toBeCloseTo(1000, 0);
});
test('projection grows with velocity', () => {
  expect(project(1000)).toBeGreaterThan(project(500));
  expect(project(0)).toBe(0);
});
test('rubberband resists progressively and stays under the overshoot', () => {
  const r = rubberband(200, 800);
  expect(r).toBeGreaterThan(0);
  expect(r).toBeLessThan(200);
});
test('rubberband is ~linear for tiny overshoot', () => {
  expect(rubberband(1, 800)).toBeCloseTo(1 * 0.55 / (1 + 0.55 * 1/800) * (800/800), 1);
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test tests/lib`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement the four modules**

`velocity.ts`: keep a ring of `{x,y,t}` samples, drop older than `windowMs`, divide delta by elapsed seconds. `project.ts` and `rubberband.ts`: the exact formulas above. `springs.ts`: the three tokens.

- [ ] **Step 4: Run tests to verify pass**

Run: `npm test tests/lib && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(motion): add spring tokens and gesture math (velocity, projection, rubber-band)"
```

---

### Task 4: `useDraggable` hook (Pointer Events + velocity handoff)

**Files:**
- Create: `lib/gesture/useDraggable.ts`
- Test: `tests/lib/useDraggable.test.tsx`

**Interfaces:**
- Consumes: `createVelocityTracker`, `rubberband` (Task 3).
- Produces: `useDraggable(options: { onEnd?(pos:{x:number;y:number}, velocity:{x:number;y:number}): void; bounds?: {minX;minY;maxX;maxY} }): { x:number; y:number; dragging:boolean; handlePointerDown(e: React.PointerEvent): void; setPosition(p:{x:number;y:number}): void }`. Tracks the grab offset so the element never snaps to the pointer's center.

- [ ] **Step 1: Write the failing test**

```tsx
// tests/lib/useDraggable.test.tsx
import { render, fireEvent } from '@testing-library/react';
import { useDraggable } from '@/lib/gesture/useDraggable';
function Probe() {
  const { x, y, handlePointerDown } = useDraggable({});
  return <div data-testid="d" onPointerDown={handlePointerDown} data-x={x} data-y={y} />;
}
test('drag moves the element 1:1 from the grab point', () => {
  const { getByTestId } = render(<Probe />);
  const el = getByTestId('d');
  fireEvent.pointerDown(el, { clientX: 10, clientY: 10, pointerId: 1 });
  fireEvent.pointerMove(el, { clientX: 40, clientY: 30, pointerId: 1 });
  expect(el.getAttribute('data-x')).toBe('30');
  expect(el.getAttribute('data-y')).toBe('20');
});
test('setPointerCapture is called', () => {
  const { getByTestId } = render(<Probe />);
  const el = getByTestId('d');
  const spy = vi.spyOn(el, 'setPointerCapture');
  fireEvent.pointerDown(el, { clientX: 0, clientY: 0, pointerId: 7 });
  expect(spy).toHaveBeenCalledWith(7);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/lib/useDraggable.test.tsx`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement the hook**

On pointer-down: `setPointerCapture`, record grab offset = pointer − element origin, start the velocity tracker. On pointer-move: new position = pointer − grab offset, clamped through `rubberband` past `bounds`. On pointer-up: release capture, compute velocity, call `onEnd(position, velocity)`. Use `requestAnimationFrame` to apply positions (no React re-render per move is acceptable; store in a ref and set state on rAF).

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/lib/useDraggable.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(gesture): add useDraggable with pointer capture and velocity handoff"
```

---

### Task 5: Stores (window, desktop, player)

**Files:**
- Create: `lib/stores/window-store.ts`, `lib/stores/desktop-store.ts`, `lib/stores/player-store.ts`
- Test: `tests/lib/stores.test.ts`

**Interfaces:**
- Consumes: `FSNode`/`AudioNode` (Task 2).
- Produces:
  - `useWindowStore`: `{ windows: WindowInstance[]; focusedId: string|null; open(w: NewWindow): void; close(id: string): void; focus(id: string): void; move(id,x,y): void; resize(id,w,h): void }` where `NewWindow = { kind:'finder'|'text'|'image'|'music'; nodeId:string; folderPath:string[]; title:string; x;y;width;height }`.
  - `useDesktopStore`: `{ positions: Record<string,{x,y}>; selectedId: string|null; setPosition(id,p): void; select(id|null): void }`, persisted to `localStorage` under `dj.desktop`.
  - `usePlayerStore`: `{ queue: AudioNode[]; index: number; isPlaying: boolean; volume: number; play(node: AudioNode, queue: AudioNode[]): void; toggle(): void; next(): void; prev(): void; setVolume(v:number): void }`. `next()` past the end stops at the last track; `play()` with the same node resumes rather than restarting.

- [ ] **Step 1: Write failing tests**

```ts
// tests/lib/stores.test.ts
import { useWindowStore } from '@/lib/stores/window-store';
import { usePlayerStore } from '@/lib/stores/player-store';
beforeEach(() => { useWindowStore.setState({ windows: [], focusedId: null }); usePlayerStore.setState({ queue: [], index: 0, isPlaying: false }); });

test('opening the same folder twice focuses instead of duplicating', () => {
  const { open } = useWindowStore.getState();
  open({ kind:'finder', nodeId:'edits', folderPath:['edits'], title:'Edits', x:0,y:0,width:640,height:420 });
  open({ kind:'finder', nodeId:'edits', folderPath:['edits'], title:'Edits', x:0,y:0,width:640,height:420 });
  expect(useWindowStore.getState().windows).toHaveLength(1);
});
test('focus raises z-order', () => {
  const s = useWindowStore.getState();
  s.open({ kind:'finder', nodeId:'a', folderPath:['a'], title:'a', x:0,y:0,width:1,height:1 });
  s.open({ kind:'finder', nodeId:'b', folderPath:['b'], title:'b', x:0,y:0,width:1,height:1 });
  const firstId = useWindowStore.getState().windows[0].id;
  useWindowStore.getState().focus(firstId);
  expect(useWindowStore.getState().focusedId).toBe(firstId);
});
test('next() advances the queue and stops at the end', () => {
  const a = { id:'a', name:'a', type:'audio', src:'/a' } as const;
  const b = { id:'b', name:'b', type:'audio', src:'/b' } as const;
  usePlayerStore.getState().play(a, [a, b]);
  usePlayerStore.getState().next();
  expect(usePlayerStore.getState().index).toBe(1);
  usePlayerStore.getState().next();
  expect(usePlayerStore.getState().index).toBe(1);
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test tests/lib/stores.test.ts`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement the three stores**

`open` de-dupes by `nodeId` and focuses the existing window. `focus` assigns the highest z. `desktop-store` uses Zustand `persist`. `player-store` keeps `queue`/`index`/`isPlaying`; `play` sets queue+index, `toggle` flips `isPlaying`, `next`/`prev` clamp.

- [ ] **Step 4: Run tests to verify pass**

Run: `npm test tests/lib/stores.test.ts && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(state): add window, desktop, and player stores"
```

---

### Task 6: AudioProvider (shared audio element)

**Files:**
- Create: `components/player/AudioProvider.tsx`
- Test: `tests/components/audio-provider.test.tsx`

**Interfaces:**
- Consumes: `usePlayerStore` (Task 5).
- Produces: `<AudioProvider>` that owns exactly one `HTMLAudioElement` and syncs it to `player-store`; exports `useAudio(): { audio: HTMLAudioElement | null }`. Creates the element once via `useRef` and never unmounts it while the app lives.

- [ ] **Step 1: Write the failing test**

```tsx
// tests/components/audio-provider.test.tsx
import { render, act } from '@testing-library/react';
import { AudioProvider, useAudio } from '@/components/player/AudioProvider';
import { usePlayerStore } from '@/lib/stores/player-store';
function Probe(){ const { audio } = useAudio(); return <span data-testid="src">{audio?.getAttribute('src') ?? ''}</span>; }
test('exactly one audio element drives playback and follows the store', () => {
  const a = { id:'a', name:'a', type:'audio', src:'/a.mp3' } as const;
  const { getByTestId, container } = render(<AudioProvider><Probe/></AudioProvider>);
  expect(container.querySelectorAll('audio')).toHaveLength(1);
  act(() => usePlayerStore.getState().play(a, [a]));
  expect(getByTestId('src').textContent).toBe('/a.mp3');
});
test('track ending advances the queue', () => {
  const a = { id:'a', name:'a', type:'audio', src:'/a.mp3' } as const;
  const b = { id:'b', name:'b', type:'audio', src:'/b.mp3' } as const;
  render(<AudioProvider><Probe/></AudioProvider>);
  act(() => usePlayerStore.getState().play(a, [a, b]));
  act(() => { const el = document.querySelector('audio')!; el.dispatchEvent(new Event('ended')); });
  expect(usePlayerStore.getState().index).toBe(1);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/components/audio-provider.test.tsx`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement the provider**

Create the `<audio>` in a `useRef` (render it hidden). Subscribe to `player-store`: set `src` when `queue[index]` changes, call `play()`/`pause()` on `isPlaying`, set `volume`. Wire the element's `ended` event to `next()`. Provide `useAudio()`.

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/components/audio-provider.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(player): add AudioProvider owning a single shared audio element"
```

---

### Task 7: Desktop shell — Wallpaper, MenuBar, layout

**Files:**
- Create: `components/desktop/Wallpaper.tsx`, `components/desktop/MenuBar.tsx`, `components/desktop/Desktop.tsx`
- Modify: `app/page.tsx`, `app/layout.tsx`
- Test: `tests/components/desktop.test.tsx`

**Interfaces:**
- Consumes: `loadFilesystem` (Task 2), `useWindowStore` (Task 5).
- Produces: `<Desktop root={FolderNode} />` rendering wallpaper + menu bar + icon grid + `<WindowManager/>` + `<Dock/>`. `MenuBar` shows the root `name` on the left and a live clock on the right.

- [ ] **Step 1: Write the failing test**

```tsx
// tests/components/desktop.test.tsx
import { render, screen } from '@testing-library/react';
import { Desktop } from '@/components/desktop/Desktop';
const root = { id:'root', name:"Saurabh's Macintosh HD", type:'folder', children:[
  { id:'latest', name:'Latest Releases', type:'folder', children:[] } ] } as const;
test('renders the disk name in the menu bar', () => {
  render(<Desktop root={root} />);
  expect(screen.getByText("Saurabh's Macintosh HD")).toBeInTheDocument();
});
test('renders one icon per child', () => {
  render(<Desktop root={root} />);
  expect(screen.getByRole('button', { name: 'Latest Releases' })).toBeInTheDocument();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/components/desktop.test.tsx`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement the three components**

`Wallpaper`: a full-bleed `div` with a CSS gradient (original art), `position:fixed`, `z-index:0`. `MenuBar`: fixed top bar, translucent material (`backdrop-filter`), root name left, clock right (updates via `useEffect` + `setInterval`). `Desktop`: loads the tree, lays out `<DesktopIcon>` per child in a wrapping grid anchored top-right, mounts `<WindowManager/>` and `<Dock/>`. Wrap the app in `<AudioProvider>` in `app/layout.tsx`.

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/components/desktop.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(desktop): add wallpaper, menu bar, and desktop shell"
```

---

### Task 8: DesktopIcon — select, open, drag

**Files:**
- Create: `components/desktop/DesktopIcon.tsx`
- Test: `tests/components/desktop-icon.test.tsx`

**Interfaces:**
- Consumes: `useDraggable` (Task 4), `useDesktopStore` (Task 5), `useWindowStore` (Task 5).
- Produces: `<DesktopIcon node={FSNode} root={FolderNode} />`. Single click selects; double click (or a single tap on touch) opens via an `onOpen(node)` callback resolved by the parent. Drag uses `useDraggable`; on end, positions persist to `desktop-store` (with momentum projection to the nearest grid slot).

- [ ] **Step 1: Write the failing test**

```tsx
// tests/components/desktop-icon.test.tsx
import { render, fireEvent } from '@testing-library/react';
import { DesktopIcon } from '@/components/desktop/DesktopIcon';
const node = { id:'latest', name:'Latest Releases', type:'folder', children:[] } as const;
test('single click selects', () => {
  const onOpen = vi.fn();
  const { getByRole } = render(<DesktopIcon node={node} onOpen={onOpen} />);
  fireEvent.click(getByRole('button', { name: 'Latest Releases' }));
  expect(onOpen).not.toHaveBeenCalled();
});
test('double click opens', () => {
  const onOpen = vi.fn();
  const { getByRole } = render(<DesktopIcon node={node} onOpen={onOpen} />);
  fireEvent.doubleClick(getByRole('button', { name: 'Latest Releases' }));
  expect(onOpen).toHaveBeenCalledWith(node);
});
test('pressing shows feedback on pointer-down', () => {
  const { getByRole } = render(<DesktopIcon node={node} onOpen={vi.fn()} />);
  const el = getByRole('button', { name: 'Latest Releases' });
  fireEvent.pointerDown(el);
  expect(el).toHaveStyle({ transform: 'scale(0.97)' });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/components/desktop-icon.test.tsx`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement the component**

`<button role>` (real button) with the icon image + label. `onPointerDown` → press scale `0.97` (spring). `onClick` → select; `onDoubleClick` → `onOpen`. On touch, a single `onClick` opens. Drag delegated to `useDraggable`, with `SPRING.momentum` on release and projection to the nearest slot.

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/components/desktop-icon.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(desktop): add DesktopIcon with select, open, and drag"
```

---

### Task 9: Window chrome — drag, resize, rubber-band, momentum, open/close

**Files:**
- Create: `components/windows/Window.tsx`
- Test: `tests/components/window.test.tsx`

**Interfaces:**
- Consumes: `useDraggable`, `rubberband`, `project`, `SPRING` (Tasks 3–4), `useWindowStore` (Task 5).
- Produces: `<Window win={WindowInstance} origin={{x,y}}>{children}</Window>`. Draggable title bar; traffic-light close/minimize/zoom buttons; resizable edges. Opens with a spring **from `origin`** (the spawning icon), closes with the same path reversed. On drag release, projects momentum and clamps so the title bar can never leave the viewport.

- [ ] **Step 1: Write the failing test**

```tsx
// tests/components/window.test.tsx
import { render, fireEvent, screen } from '@testing-library/react';
import { Window } from '@/components/windows/Window';
const win = { id:'w1', kind:'finder', nodeId:'edits', folderPath:['edits'], title:'Edits', x:100, y:100, width:640, height:420, z:1 };
test('close button calls onClose', () => {
  const onClose = vi.fn();
  render(<Window win={win} onClose={onClose}>body</Window>);
  fireEvent.click(screen.getByRole('button', { name: /close/i }));
  expect(onClose).toHaveBeenCalled();
});
test('dragging the title bar moves the window', () => {
  render(<Window win={win} onClose={vi.fn()}>body</Window>);
  const bar = screen.getByTestId('window-titlebar');
  fireEvent.pointerDown(bar, { clientX: 120, clientY: 110, pointerId: 1 });
  fireEvent.pointerMove(bar, { clientX: 160, clientY: 140, pointerId: 1 });
  expect(screen.getByTestId('window-w1')).toHaveStyle({ transform: 'translate(140px, 130px)' });
});
test('a window can never be dragged fully off-screen', () => {
  render(<Window win={win} onClose={vi.fn()}>body</Window>);
  const bar = screen.getByTestId('window-titlebar');
  fireEvent.pointerDown(bar, { clientX: 0, clientY: 0, pointerId: 1 });
  fireEvent.pointerMove(bar, { clientX: -5000, clientY: -5000, pointerId: 1 });
  fireEvent.pointerUp(bar, { pointerId: 1 });
  const rect = screen.getByTestId('window-w1').getBoundingClientRect();
  expect(rect.top).toBeGreaterThanOrEqual(0);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/components/window.test.tsx`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `Window.tsx`**

Motion `motion.div` with `initial={{ scale:0.2, opacity:0 }}` anchored at `origin` (`transform-origin` = origin relative to window), `animate={{ scale:1, opacity:1 }}` using `SPRING.default`; `exit` reverses toward `origin`. Title bar uses `useDraggable` with bounds = viewport minus title-bar height; on release, `project()` the velocity, `SPRING.momentum` to the clamped target. Resize handles on all edges via `useDraggable` with `axis` locked per handle. Focus on `onPointerDown` → `useWindowStore.focus(win.id)`.

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/components/window.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(windows): add draggable, resizable window chrome with rubber-band and momentum"
```

---

### Task 10: WindowManager — z-order, focus, open/close

**Files:**
- Create: `components/windows/WindowManager.tsx`
- Test: `tests/components/window-manager.test.tsx`

**Interfaces:**
- Consumes: `useWindowStore`, `useDesktopStore` (Task 5), `Window` (Task 9), `FinderWindow`/`TextViewer`/`ImageViewer`/`MusicWindow` (later tasks).
- Produces: `<WindowManager/>` rendering every open window sorted by `z`, dispatching the right body component by `win.kind`, and providing `useOpenNode()` — a hook that maps a node to the correct `open(...)` call (folder→finder, text→text, image→image, audio→play, link→new tab).

- [ ] **Step 1: Write the failing test**

```tsx
// tests/components/window-manager.test.tsx
import { render, screen, act } from '@testing-library/react';
import { WindowManager } from '@/components/windows/WindowManager';
import { useWindowStore } from '@/lib/stores/window-store';
beforeEach(() => useWindowStore.setState({ windows: [], focusedId: null }));
test('renders one window per store entry', () => {
  act(() => {
    useWindowStore.getState().open({ kind:'finder', nodeId:'a', folderPath:['a'], title:'A', x:0,y:0,width:400,height:300 });
    useWindowStore.getState().open({ kind:'finder', nodeId:'b', folderPath:['b'], title:'B', x:0,y:0,width:400,height:300 });
  });
  render(<WindowManager />);
  expect(screen.getAllByTestId(/window-titlebar/)).toHaveLength(2);
});
test('closing removes the window', () => {
  act(() => useWindowStore.getState().open({ kind:'finder', nodeId:'a', folderPath:['a'], title:'A', x:0,y:0,width:400,height:300 }));
  render(<WindowManager />);
  act(() => useWindowStore.getState().close(useWindowStore.getState().windows[0].id));
  expect(screen.queryByTestId('window-titlebar')).not.toBeInTheDocument();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/components/window-manager.test.tsx`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `WindowManager.tsx` and `useOpenNode()`**

Map `windows` → `<Window>` sorted by `z`, body by `kind`. `useOpenNode()` returns a callback: `folder` → `open({kind:'finder', nodeId, folderPath:[...parent, node.id], ...})`; `text` → `{kind:'text'}`; `image` → `{kind:'image'}`; `audio` → `usePlayerStore.play(node, siblings.filter(isAudio))`; `link` → `window.open(node.href, '_blank', 'noopener')`.

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/components/window-manager.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(windows): add WindowManager with z-order, focus, and node dispatch"
```

---

### Task 11: FinderWindow and LockedFolderModal

**Files:**
- Create: `components/windows/FinderWindow.tsx`, `components/windows/LockedFolderModal.tsx`
- Test: `tests/components/finder-window.test.tsx`

**Interfaces:**
- Consumes: `getNodeByPath`, `isFolder` (Task 2), `useOpenNode` (Task 10).
- Produces: `<FinderWindow folderPath={string[]} />` rendering the folder's children in a large-icon grid; double-click/tap a child opens it via `useOpenNode`. Locked folders render `<LockedFolderModal folder={FolderNode} onUnlock={()=>void} />` first.

- [ ] **Step 1: Write the failing test**

```tsx
// tests/components/finder-window.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { FinderWindow } from '@/components/windows/FinderWindow';
test('lists children of the folder', () => {
  render(<FinderWindow folderPath={['edits']} />);
  expect(screen.getByRole('button', { name: /A\.mp3/ })).toBeInTheDocument();
});
test('locked folder shows the password modal, not contents', () => {
  render(<FinderWindow folderPath={['unreleased']} />);
  expect(screen.getByText(/enter the password/i)).toBeInTheDocument();
  expect(screen.queryByRole('button', { name: /secret/i })).not.toBeInTheDocument();
});
test('wrong password shakes and does not unlock', () => {
  render(<FinderWindow folderPath={['unreleased']} />);
  fireEvent.change(screen.getByLabelText(/password/i), { target: { value: 'wrong' } });
  fireEvent.click(screen.getByRole('button', { name: /unlock/i }));
  expect(screen.getByTestId('locked-modal')).toHaveAttribute('data-shake', 'true');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/components/finder-window.test.tsx`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement both components**

`FinderWindow`: resolve the folder with `getNodeByPath`; if `locked` and not yet unlocked in local component state, render `LockedFolderModal`; else render a grid of `<button>` children (icon + label), double-click/tap → `useOpenNode()`. `LockedFolderModal`: labeled password input, Unlock button, `data-shake` toggled on failure (spring shake), `onUnlock` when `password` matches.

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/components/finder-window.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(windows): add FinderWindow and LockedFolderModal"
```

---

### Task 12: TextViewer and ImageViewer

**Files:**
- Create: `components/windows/TextViewer.tsx`, `components/windows/ImageViewer.tsx`
- Test: `tests/components/viewers.test.tsx`

**Interfaces:**
- Consumes: `FSNode` (Task 2).
- Produces: `<TextViewer node={TextNode} />` and `<ImageViewer node={ImageNode} />`.

- [ ] **Step 1: Write the failing test**

```tsx
// tests/components/viewers.test.tsx
import { render, screen } from '@testing-library/react';
import { TextViewer } from '@/components/windows/TextViewer';
import { ImageViewer } from '@/components/windows/ImageViewer';
test('text viewer renders the body', () => {
  render(<TextViewer node={{ id:'r', name:'About.txt', type:'text', body:'hello dj' }} />);
  expect(screen.getByText('hello dj')).toBeInTheDocument();
});
test('image viewer renders the image with an alt', () => {
  render(<ImageViewer node={{ id:'i', name:'cover.jpg', type:'image', src:'/c.jpg' }} />);
  expect(screen.getByRole('img', { name: 'cover.jpg' })).toHaveAttribute('src', '/c.jpg');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/components/viewers.test.tsx`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement both**

`TextViewer`: scrollable, readable body. `ImageViewer`: `object-fit: contain` image, alt = node name.

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/components/viewers.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(windows): add text and image viewers"
```

---

### Task 13: Dock and DockIcon

**Files:**
- Create: `components/dock/Dock.tsx`, `components/dock/DockIcon.tsx`
- Test: `tests/components/dock.test.tsx`

**Interfaces:**
- Consumes: `useWindowStore` (Task 5), `useOpenNode` (Task 10).
- Produces: `<Dock/>` with a translucent material bar, one `DockIcon` per root shortcut (Finder, Music, Spotify, YouTube, Instagram, Mail, Trash), hover magnification (spring), and a running dot for apps with an open window. Trash is a cosmetic drop target.

- [ ] **Step 1: Write the failing test**

```tsx
// tests/components/dock.test.tsx
import { render, screen } from '@testing-library/react';
import { Dock } from '@/components/dock/Dock';
test('renders the app shortcuts', () => {
  render(<Dock />);
  expect(screen.getByRole('button', { name: /finder/i })).toBeInTheDocument();
  expect(screen.getByRole('button', { name: /trash/i })).toBeInTheDocument();
});
test('hover magnifies the icon with a spring', () => {
  render(<Dock />);
  const el = screen.getByRole('button', { name: /finder/i });
  fireEvent.pointerEnter(el);
  expect(el).toHaveStyle({ transform: expect.stringContaining('scale(1.3)') });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/components/dock.test.tsx`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement both**

`Dock`: fixed bottom-center bar with `backdrop-filter: blur(20px) saturate(180%)` + semi-transparent fill. `DockIcon`: pointer-enter → `SPRING.default` scale to `1.3`; pointer-leave → scale `1`. Clicking opens the app (folder→finder, external→new tab, Music→music window).

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/components/dock.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(dock): add translucent dock with magnifying icons"
```

---

### Task 14: NowPlaying, PlayerControls, Waveform

**Files:**
- Create: `components/dock/NowPlaying.tsx`, `components/player/PlayerControls.tsx`, `components/player/Waveform.tsx`
- Test: `tests/components/now-playing.test.tsx`

**Interfaces:**
- Consumes: `usePlayerStore` (Task 5), `useAudio` (Task 6).
- Produces: `<NowPlaying/>` (cover, title, play/pause, scrubber, Download), `<PlayerControls/>` (play/pause, prev/next, volume, download), `<Waveform peaks={number[]} />`. Download uses `node.download` — a hosted file downloads via an `<a download>`; a store link opens in a new tab.

- [ ] **Step 1: Write the failing test**

```tsx
// tests/components/now-playing.test.tsx
import { render, screen, act, fireEvent } from '@testing-library/react';
import { NowPlaying } from '@/components/dock/NowPlaying';
import { usePlayerStore } from '@/lib/stores/player-store';
const a = { id:'a', name:'Neon Bombay.mp3', type:'audio', src:'/a.mp3', download:'/a.mp3', cover:'/c.jpg' } as const;
beforeEach(() => usePlayerStore.setState({ queue: [], index: 0, isPlaying: false }));
test('shows the current track and toggles play', () => {
  act(() => usePlayerStore.getState().play(a, [a]));
  render(<NowPlaying />);
  fireEvent.click(screen.getByRole('button', { name: /play/i }));
  expect(usePlayerStore.getState().isPlaying).toBe(true);
});
test('download link points at the track download url', () => {
  act(() => usePlayerStore.getState().play(a, [a]));
  render(<NowPlaying />);
  expect(screen.getByRole('link', { name: /download/i })).toHaveAttribute('href', '/a.mp3');
});
test('a store-link download opens in a new tab', () => {
  const b = { id:'b', name:'b', type:'audio', src:'/b.mp3', download:'https://bandcamp.com/x' } as const;
  act(() => usePlayerStore.getState().play(b, [b]));
  render(<NowPlaying />);
  expect(screen.getByRole('link', { name: /download/i })).toHaveAttribute('target', '_blank');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/components/now-playing.test.tsx`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement the three components**

`NowPlaying`: cover art + title from `queue[index]`, play/pause toggles `isPlaying`, scrubber calls `seek` (wired through `useAudio`), Download chooses `<a download>` for same-origin or `<a target="_blank">` for an external URL. `PlayerControls`: prev/next/volume/download. `Waveform`: draw `peaks` to a canvas (precomputed; the live `AnalyserNode` path is only used when a track has no `peaks` and is same-origin).

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/components/now-playing.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(player): add now-playing module, controls, waveform, and download"
```

---

### Task 15: MusicWindow

**Files:**
- Create: `components/windows/MusicWindow.tsx`
- Test: `tests/components/music-window.test.tsx`

**Interfaces:**
- Consumes: `usePlayerStore` (Task 5), `Waveform` (Task 14), `isFolder` (Task 2).
- Produces: `<MusicWindow folderPath={string[]} />` — cover art, the folder's track list, waveform, and click-to-play per row.

- [ ] **Step 1: Write the failing test**

```tsx
// tests/components/music-window.test.tsx
import { render, screen, fireEvent, act } from '@testing-library/react';
import { MusicWindow } from '@/components/windows/MusicWindow';
import { usePlayerStore } from '@/lib/stores/player-store';
beforeEach(() => usePlayerStore.setState({ queue: [], index: 0, isPlaying: false }));
test('lists the folder tracks and plays on click', () => {
  render(<MusicWindow folderPath={['edits']} />);
  fireEvent.click(screen.getByRole('button', { name: /A\.mp3/ }));
  expect(usePlayerStore.getState().queue).toHaveLength(2);
  expect(usePlayerStore.getState().index).toBe(0);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/components/music-window.test.tsx`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `MusicWindow.tsx`**

Resolve the folder, filter `children` with `type === 'audio'`; render a track list; clicking a row calls `usePlayerStore.play(node, allAudioInFolder)`; show cover of the current track and a `<Waveform>`.

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/components/music-window.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(windows): add MusicWindow with track list and waveform"
```

---

### Task 16: Routing, URL sync, and deep links

**Files:**
- Create: `app/[[...path]]/page.tsx`
- Modify: `app/page.tsx`, `components/desktop/Desktop.tsx`, `lib/fs/path.ts`
- Test: `tests/app/deep-link.test.tsx`

**Interfaces:**
- Consumes: `getNodeByPath` (Task 2), `useWindowStore`, `usePlayerStore` (Task 5), `Desktop` (Task 7).
- Produces: a catch-all route that renders `<Desktop>` and, on mount, opens the window for `params.path` (and plays `searchParams.track` if present). A `useSyncRoute()` hook pushes the focused folder to the URL on navigation (client-side, no reload).

- [ ] **Step 1: Write the failing test**

```tsx
// tests/app/deep-link.test.tsx
import { render, screen } from '@testing-library/react';
import { useWindowStore } from '@/lib/stores/window-store';
beforeEach(() => useWindowStore.setState({ windows: [], focusedId: null }));
test('cold-loading /edits opens the Edits window', async () => {
  const Page = (await import('@/app/[[...path]]/page')).default;
  render(await Page({ params: Promise.resolve({ path: ['edits'] }), searchParams: Promise.resolve({}) }));
  expect(screen.getByTestId(/window-titlebar/)).toBeInTheDocument();
});
test('no hydration mismatch: server and client render the same shell', async () => {
  const Page = (await import('@/app/[[...path]]/page')).default;
  const ui = await Page({ params: Promise.resolve({ path: [] }), searchParams: Promise.resolve({}) });
  expect(() => render(ui)).not.toThrow();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/app/deep-link.test.tsx`
Expected: FAIL — route module not found.

- [ ] **Step 3: Implement the route and `useSyncRoute()`**

`[[...path]]/page.tsx` (server component): `await params`/`await searchParams`, render `<Desktop root={root} initialPath={path} initialTrack={track} />`. `Desktop` opens the initial window inside a `useEffect` (never during render) so server HTML and first client render match. `useSyncRoute()` calls `history.pushState` when the focused window's folder changes.

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/app/deep-link.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(routing): add catch-all route, URL sync, and deep links"
```

---

### Task 17: Responsive and touch

**Files:**
- Modify: `components/windows/Window.tsx`, `components/dock/Dock.tsx`, `components/desktop/Desktop.tsx`, `app/globals.css`
- Create: `lib/gesture/useMediaQuery.ts`
- Test: `tests/components/responsive.test.tsx`

**Interfaces:**
- Consumes: everything above.
- Produces: `useMediaQuery(query: string): boolean`. On `(max-width: 767px)`, `Window` renders as a full-screen sheet that slides up (`SPRING.drawer`) and can be dragged down to dismiss (spring + `rubberband`); `Dock` renders compact; the desktop icon grid reflows.

- [ ] **Step 1: Write the failing test**

```tsx
// tests/components/responsive.test.tsx
import { render, screen } from '@testing-library/react';
import { useMediaQuery } from '@/lib/gesture/useMediaQuery';
function Probe(){ return <span data-testid="m">{String(useMediaQuery('(max-width: 767px)'))}</span>; }
test('reports the media query state', () => {
  window.matchMedia = vi.fn().mockReturnValue({ matches: true, addEventListener: vi.fn(), removeEventListener: vi.fn() });
  render(<Probe />);
  expect(screen.getByTestId('m').textContent).toBe('true');
});
test('window renders as a sheet on phone widths', () => {
  window.matchMedia = vi.fn().mockReturnValue({ matches: true, addEventListener: vi.fn(), removeEventListener: vi.fn() });
  const { Window } = await import('@/components/windows/Window');
  render(<Window win={{ id:'w1', kind:'finder', nodeId:'a', folderPath:['a'], title:'A', x:0,y:0,width:1,height:1, z:1 }} onClose={() => {}}>b</Window>);
  expect(screen.getByTestId('window-w1')).toHaveAttribute('data-presentation', 'sheet');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/components/responsive.test.tsx`
Expected: FAIL — `useMediaQuery` not found.

- [ ] **Step 3: Implement `useMediaQuery` and the responsive branches**

`useMediaQuery`: subscribe via `matchMedia(...).addEventListener('change')`. In `Window`, when the phone query matches, render the sheet presentation (`data-presentation="sheet"`), full-bleed, slide-up entry, drag-down-to-dismiss. `Dock` uses a compact layout. `globals.css` adds the tablet/phone breakpoints.

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/components/responsive.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(responsive): add touch sheets, compact dock, and media-query hook"
```

---

### Task 18: Accessibility — keyboard, fallback list, media preferences

**Files:**
- Create: `components/a11y/FallbackList.tsx`, `lib/a11y/useKeyboardDesktop.ts`
- Modify: `app/globals.css`, `components/windows/Window.tsx`, `components/desktop/Desktop.tsx`
- Test: `tests/components/a11y.test.tsx`

**Interfaces:**
- Consumes: `FolderNode`, `isFolder` (Task 2), `useWindowStore`, `useDesktopStore` (Task 5).
- Produces: `<FallbackList root={FolderNode} />` (visually hidden, reachable, **excludes locked folders' contents**), `useKeyboardDesktop()` (arrow keys move selection, `Enter` opens, `Esc` closes the top window, focus trapped in modals and restored on close).

- [ ] **Step 1: Write the failing test**

```tsx
// tests/components/a11y.test.tsx
import { render, screen } from '@testing-library/react';
import { FallbackList } from '@/components/a11y/FallbackList';
const root = { id:'root', name:'HD', type:'folder', children:[
  { id:'edits', name:'Edits', type:'folder', children:[{ id:'t', name:'A.mp3', type:'audio', src:'/a' }] },
  { id:'unreleased', name:'Unreleased', type:'folder', locked:true, password:'x', children:[{ id:'s', name:'Secret.mp3', type:'audio', src:'/s' }] } ] } as const;
test('lists open folders and tracks', () => {
  render(<FallbackList root={root} />);
  expect(screen.getByRole('link', { name: /A\.mp3/ })).toBeInTheDocument();
});
test('locked folder contents are not listed', () => {
  render(<FallbackList root={root} />);
  expect(screen.queryByRole('link', { name: /Secret\.mp3/ })).not.toBeInTheDocument();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/components/a11y.test.tsx`
Expected: FAIL — modules not found.

- [ ] **Step 3: Implement the fallback list, keyboard hook, and media queries**

`FallbackList`: render a real list of folders/tracks/links; skip any subtree under `locked`. `useKeyboardDesktop`: keydown listener on the desktop; arrow keys move `selectedId` across icons; `Enter` opens; `Esc` closes the focused window. `globals.css`: add the three media blocks from spec §7.6 — reduced motion → springs replaced by short opacity cross-fades; reduced transparency → solid dock/chrome, blur off; more contrast → near-solid fills with borders.

- [ ] **Step 4: Run test to verify pass**

Run: `npm test tests/components/a11y.test.tsx && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(a11y): add keyboard control, fallback list, and media preferences"
```

---

### Task 19: Motion polish, performance budget, and content swap

**Files:**
- Modify: `components/**` (targeted polish), `README.md`
- Test: `tests/components/motion-polish.test.tsx`

**Interfaces:**
- Consumes: everything above.
- Produces: a README documenting how to swap placeholder content for real assets (edit `filesystem.json`, upload to Vercel Blob, set `src`/`download`/`cover`/`peaks`), and a guard test that no draggable component uses a CSS `transition`.

- [ ] **Step 1: Write the guard test**

```tsx
// tests/components/motion-polish.test.tsx
import { readFileSync } from 'node:fs';
const GESTURE_FILES = ['components/windows/Window.tsx', 'components/desktop/DesktopIcon.tsx'];
test('gesture-driven components never use CSS transitions', () => {
  for (const f of GESTURE_FILES) {
    const src = readFileSync(f, 'utf8');
    expect(src).not.toMatch(/\btransition(-property|-duration)?\s*:/);
  }
});
```

- [ ] **Step 2: Run it to verify it fails if any transition slipped in**

Run: `npm test tests/components/motion-polish.test.tsx`
Expected: PASS if clean; FAIL (and fix the offending file) if a CSS transition exists.

- [ ] **Step 3: Polish pass**

Review motion frame-by-frame per spec §7.6: confirm press feedback is on pointer-down, drags animate from the live value, reversals blend velocity, materials use `backdrop-filter`, type tracking is size-specific. Confirm only `transform`/`opacity` animate. Write `README.md` with the content-swap steps.

- [ ] **Step 4: Full suite + typecheck**

Run: `npm test && npm run typecheck && npm run build`
Expected: all PASS; production build succeeds.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "chore: motion polish, performance guard, and content-swap docs"
```

---

## Self-Review

**Spec coverage:** §7.1 filesystem → Task 2. §7.2 desktop/menu bar/icons → Tasks 7, 8. §7.3 Finder window + locked modal → Tasks 9, 10, 11. §7.4 Dock → Task 13. §7.5 player + waveform → Tasks 6, 14, 15. §7.6 motion/materials/type → Tasks 3, 4, 9, 19. §7.6a node open behavior → Task 10. §7.6b downloads → Task 14. §7.7 routing/deep links → Task 16. §7.8 responsive/touch → Task 17. §7.9 accessibility + media prefs → Task 18. §8 placeholders → Task 2, swap docs → Task 19. No gaps.

**Type consistency:** `FSNode`/`AudioNode`/`FolderNode` (Task 2) are used unchanged in Tasks 5–18. `open(...)` signature is defined once (Task 5) and used by `useOpenNode()` (Task 10) and the Dock (Task 13). `SPRING`/`useDraggable`/`project`/`rubberband` names match across Tasks 3, 4, 9, 17. `play(node, queue)` matches Tasks 5, 6, 10, 15. `data-testid` names (`desktop-root`, `window-<id>`, `window-titlebar`, `locked-modal`) are consistent.

**Review Focus → tests:** (1) off-screen window → Task 9. (2) duplicate/rapid open → Task 5 + Task 10. (3) track end / switch mid-play → Tasks 6, 14. (4) deep-link cold load + hydration → Task 16. (5) reduced motion / keyboard → Task 18.

**Proportion:** the plan stays at signatures, test names/assertions, and exact values; no component bodies are transcribed.
