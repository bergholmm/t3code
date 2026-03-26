# Mobile Quick Wins Design

Three targeted fixes to make the T3 Code web app usable when accessed from a phone (iOS Safari, added to home screen, connected over Tailscale).

## 1. Fix iOS Home Screen Gray Screen

### Problem

Adding the web app to the iOS home screen and launching it shows a gray screen. The app works inside the browser but fails in standalone mode because there is no web app manifest and no iOS standalone meta tags.

### Root Cause

- No `manifest.webmanifest` file exists.
- No `apple-mobile-web-app-capable` meta tag in `index.html`.
- No `theme-color` meta tag.
- The HTML background color before CSS/JS loads is the browser default (white/gray), not the app's dark theme.

### Changes

**`apps/web/index.html`** — add meta tags inside `<head>`:

```html
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
<meta name="apple-mobile-web-app-title" content="T3 Code" />
<meta name="theme-color" content="#09090b" media="(prefers-color-scheme: dark)" />
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)" />
<link rel="manifest" href="/manifest.webmanifest" />
```

Add an inline `<style>` block before the bundle loads so the pre-render background matches the app theme:

```html
<style>
  html, body { background-color: #09090b; }
  @media (prefers-color-scheme: light) {
    html, body { background-color: #ffffff; }
  }
</style>
```

**`apps/web/public/manifest.webmanifest`** — new file:

```json
{
  "name": "T3 Code",
  "short_name": "T3 Code",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "background_color": "#09090b",
  "theme_color": "#09090b",
  "icons": [
    {
      "src": "/apple-touch-icon.png",
      "sizes": "180x180",
      "type": "image/png"
    },
    {
      "src": "/favicon-32x32.png",
      "sizes": "32x32",
      "type": "image/png"
    }
  ]
}
```

### Files Touched

- `apps/web/index.html` (edit)
- `apps/web/public/manifest.webmanifest` (new)

---

## 2. Loading Skeleton for Session List

### Problem

Over Tailscale the WebSocket handshake and initial snapshot fetch can take several seconds. During that time the sidebar shows nothing (empty space) and the thread content area returns `null`. The user sees "no chats," then threads pop in abruptly.

### Root Cause

- The Zustand store starts with `threads: []` and `threadsHydrated: false`.
- The sidebar renders the thread list directly from the store with no loading state.
- The thread route (`_chat.$threadId.tsx`) returns `null` while `threadsHydrated` is false.
- The root connecting screen (`__root.tsx`) shows plain text with no visual indicator of progress.

### Changes

**`apps/web/src/components/Sidebar.tsx`** — add skeleton rows:

When `threadsHydrated` is `false`, render 6 skeleton placeholder rows using the existing `Skeleton` component (`apps/web/src/components/ui/skeleton.tsx`). Each row mimics a thread entry: a skeleton line for the title and a shorter one for the subtitle. The skeletons use the standard animated shimmer.

When `threadsHydrated` is `true` and `threads.length === 0`, show the existing empty state. No change to that path.

**`apps/web/src/routes/_chat.$threadId.tsx`** — replace `return null`:

When `!threadsHydrated`, render a centered `Spinner` instead of returning nothing. Use the existing `Spinner` component from `apps/web/src/components/ui/spinner.tsx`.

**`apps/web/src/routes/__root.tsx`** — add spinner to connecting screen:

Add the `Spinner` component above the "Connecting to T3 Code server..." text so the loading state feels intentional.

### Files Touched

- `apps/web/src/components/Sidebar.tsx` (edit)
- `apps/web/src/routes/_chat.$threadId.tsx` (edit)
- `apps/web/src/routes/__root.tsx` (edit)

---

## 3. Auto-Close Sidebar on Mobile Thread Selection

### Problem

On mobile, tapping a thread in the sidebar navigates to it but the sidebar sheet stays open, covering the content. The user has to manually tap the backdrop or swipe to dismiss.

### Root Cause

`handleThreadClick()` in `Sidebar.tsx` calls `navigate({ to: "/$threadId", ... })` but does not close the mobile sheet. The `Sheet` component only auto-closes on backdrop click (`onOpenChange`), not on content interaction.

### Changes

**`apps/web/src/components/Sidebar.tsx`** — close sidebar after navigation:

After the `navigate()` call in `handleThreadClick()`, add:

```ts
if (isMobile) {
  setOpenMobile(false);
}
```

The `useSidebar()` hook (which exposes `isMobile` and `setOpenMobile`) is already available in the component tree via `SidebarProvider`. No new dependencies needed.

Audit the sidebar for other navigation actions that should also close the mobile sheet (e.g., "New Thread" button). If any of them call `navigate()` without closing the sidebar, apply the same `setOpenMobile(false)` pattern.

### Files Touched

- `apps/web/src/components/Sidebar.tsx` (edit)

---

## Out of Scope

The following were identified during exploration but are explicitly deferred:

- **Browser/push notifications** — requires service worker infrastructure; will be a separate spec.
- **Custom model sync across devices** — models are stored in per-browser localStorage; syncing needs a server-side settings store.
- **Full mobile UX redesign** — bottom navigation, gesture support, mobile-first layout are out of scope for this round.
- **Service worker / offline caching** — would improve actual load speed but is more effort than these quick wins warrant.
- **Additional icon sizes for the manifest** — the existing `apple-touch-icon.png` (180x180) and favicons cover the basics; a full icon set can be added later.

## Testing

- **iOS home screen**: Add to home screen on an iPhone, launch. Should show the dark-themed app immediately, not a gray screen.
- **Loading state**: Connect over a slow network (or add artificial delay). Sidebar should show skeleton rows, thread area should show spinner, connecting screen should show spinner.
- **Sidebar close**: On a phone-width viewport (or actual phone), open sidebar, tap a thread. Sidebar should dismiss and the thread should load underneath.
