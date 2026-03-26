# Mobile Quick Wins Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix three mobile UX issues — iOS home screen gray screen, missing loading skeletons, and sidebar not closing on thread selection.

**Architecture:** All changes are in `apps/web`. Task 1 adds PWA meta tags and a web manifest file. Task 2 adds loading indicators to three components using existing `Skeleton` and `Spinner` UI primitives. Task 3 adds a one-line mobile sidebar close call after navigation.

**Tech Stack:** React, Tailwind CSS, existing `Skeleton`/`Spinner`/`SidebarMenuSkeleton` UI components, Web App Manifest spec.

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `apps/web/index.html` | Modify | Add PWA meta tags, manifest link, inline background style |
| `apps/web/public/manifest.webmanifest` | Create | Web app manifest for standalone iOS/Android install |
| `apps/web/src/routes/__root.tsx` | Modify | Add Spinner to "Connecting" screen |
| `apps/web/src/components/Sidebar.tsx` | Modify | Add skeleton rows when `!threadsHydrated`; close mobile sidebar on navigate |
| `apps/web/src/routes/_chat.$threadId.tsx` | Modify | Show Spinner instead of `null` while hydrating |

---

### Task 1: PWA Manifest and iOS Meta Tags

**Files:**
- Modify: `apps/web/index.html`
- Create: `apps/web/public/manifest.webmanifest`

- [ ] **Step 1: Create the web app manifest**

Create `apps/web/public/manifest.webmanifest`:

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

- [ ] **Step 2: Add meta tags and inline styles to index.html**

In `apps/web/index.html`, add the following inside `<head>`, after the existing `<link rel="apple-touch-icon" ...>` line (line 7):

```html
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
    <meta name="apple-mobile-web-app-title" content="T3 Code" />
    <meta name="theme-color" content="#09090b" media="(prefers-color-scheme: dark)" />
    <meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)" />
    <link rel="manifest" href="/manifest.webmanifest" />
```

Also add an inline `<style>` block right before `</head>` so the background color renders before any JS loads:

```html
    <style>
      html, body { background-color: #09090b; }
      @media (prefers-color-scheme: light) {
        html, body { background-color: #ffffff; }
      }
    </style>
```

The full `<head>` should now look like:

```html
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="icon" href="/favicon.ico" sizes="48x48" />
    <link rel="apple-touch-icon" href="/apple-touch-icon.png" />
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
    <meta name="apple-mobile-web-app-title" content="T3 Code" />
    <meta name="theme-color" content="#09090b" media="(prefers-color-scheme: dark)" />
    <meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)" />
    <link rel="manifest" href="/manifest.webmanifest" />
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
    <link
      href="https://fonts.googleapis.com/css2?family=DM+Sans:ital,opsz,wght@0,9..40,300..800;1,9..40,300..800&display=swap"
      rel="stylesheet"
    />
    <title>T3 Code (Alpha)</title>
    <style>
      html, body { background-color: #09090b; }
      @media (prefers-color-scheme: light) {
        html, body { background-color: #ffffff; }
      }
    </style>
  </head>
```

- [ ] **Step 3: Verify the build passes**

Run: `bun fmt && bun lint && bun typecheck`

Expected: All three pass with no errors.

- [ ] **Step 4: Commit**

```bash
git add apps/web/index.html apps/web/public/manifest.webmanifest
git commit -m "feat(web): add PWA manifest and iOS standalone meta tags

Fixes gray screen when launching from iOS home screen by adding
apple-mobile-web-app-capable, theme-color, and a web app manifest.
Inline background style prevents white flash before CSS loads."
```

---

### Task 2: Loading Skeletons and Spinners

**Files:**
- Modify: `apps/web/src/routes/__root.tsx`
- Modify: `apps/web/src/components/Sidebar.tsx`
- Modify: `apps/web/src/routes/_chat.$threadId.tsx`

- [ ] **Step 1: Add Spinner to the "Connecting" screen in `__root.tsx`**

In `apps/web/src/routes/__root.tsx`, add the Spinner import at the top:

```ts
import { Spinner } from "../components/ui/spinner";
```

Then update the `RootRouteView` connecting screen (lines 41-47). Replace:

```tsx
        <div className="flex flex-1 items-center justify-center">
          <p className="text-sm text-muted-foreground">
            Connecting to {APP_DISPLAY_NAME} server...
          </p>
        </div>
```

With:

```tsx
        <div className="flex flex-1 flex-col items-center justify-center gap-3">
          <Spinner className="size-5 text-muted-foreground" />
          <p className="text-sm text-muted-foreground">
            Connecting to {APP_DISPLAY_NAME} server...
          </p>
        </div>
```

- [ ] **Step 2: Add skeleton rows to `Sidebar.tsx` when threads are not hydrated**

In `apps/web/src/components/Sidebar.tsx`, add the following imports. `Skeleton` is already available in the UI kit. `SidebarMenuSkeleton` is already exported from `./ui/sidebar` but not currently imported in this file. Add it to the existing sidebar import block (line 75-89):

Add `SidebarMenuSkeleton` to the import from `"./ui/sidebar"`:

```ts
import {
  SidebarContent,
  SidebarFooter,
  SidebarGroup,
  SidebarHeader,
  SidebarMenuAction,
  SidebarMenu,
  SidebarMenuButton,
  SidebarMenuItem,
  SidebarMenuSkeleton,
  SidebarMenuSub,
  SidebarMenuSubButton,
  SidebarMenuSubItem,
  SidebarSeparator,
  SidebarTrigger,
} from "./ui/sidebar";
```

Also read `threadsHydrated` from the store. In the `Sidebar` component body (around line 366, near the existing `useStore` calls), add:

```ts
const threadsHydrated = useStore((store) => store.threadsHydrated);
```

Then, inside the `<SidebarContent>` area, wrap the project list section (the `<SidebarGroup className="px-2 py-2">` block starting at line 1684) so that when `!threadsHydrated`, skeleton rows render instead.

Find this block (line 1684):

```tsx
        <SidebarGroup className="px-2 py-2">
          <div className="mb-1 flex items-center justify-between px-2">
            <span className="text-[10px] font-medium uppercase tracking-wider text-muted-foreground/60">
              Projects
            </span>
```

Immediately before this `<SidebarGroup>`, add:

```tsx
        {!threadsHydrated ? (
          <SidebarGroup className="px-2 py-2">
            <div className="mb-1 px-2">
              <span className="text-[10px] font-medium uppercase tracking-wider text-muted-foreground/60">
                Projects
              </span>
            </div>
            <SidebarMenu>
              {Array.from({ length: 6 }, (_, i) => (
                <SidebarMenuItem key={i}>
                  <SidebarMenuSkeleton showIcon />
                </SidebarMenuItem>
              ))}
            </SidebarMenu>
          </SidebarGroup>
        ) : (
```

Then after the closing of the existing `<SidebarGroup>` (the one with the "No projects yet" empty state, ending at line 1827 `</SidebarGroup>`), add the closing bracket:

```tsx
        )}
```

This wraps the entire project list section in a conditional: skeletons when loading, real content when hydrated.

- [ ] **Step 3: Add Spinner to `_chat.$threadId.tsx` hydration guard**

In `apps/web/src/routes/_chat.$threadId.tsx`, add the Spinner import at the top:

```ts
import { Spinner } from "../components/ui/spinner";
```

Then replace the null return at line 215-217. Change:

```tsx
  if (!threadsHydrated || !routeThreadExists) {
    return null;
  }
```

To:

```tsx
  if (!threadsHydrated) {
    return (
      <div className="flex h-full items-center justify-center">
        <Spinner className="size-5 text-muted-foreground" />
      </div>
    );
  }

  if (!routeThreadExists) {
    return null;
  }
```

This shows a spinner while hydrating but still returns null (and redirects via the useEffect above) when the thread doesn't exist.

- [ ] **Step 4: Verify the build passes**

Run: `bun fmt && bun lint && bun typecheck`

Expected: All three pass with no errors.

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/routes/__root.tsx apps/web/src/components/Sidebar.tsx apps/web/src/routes/_chat.$threadId.tsx
git commit -m "feat(web): add loading skeletons and spinners during hydration

Show skeleton rows in the sidebar while threads are being fetched,
a spinner on the connecting screen, and a spinner in the thread
content area instead of rendering nothing."
```

---

### Task 3: Auto-Close Mobile Sidebar on Navigation

**Files:**
- Modify: `apps/web/src/components/Sidebar.tsx`

- [ ] **Step 1: Import `useSidebar` and wire up mobile close**

In `apps/web/src/components/Sidebar.tsx`, add `useSidebar` to the import from `"./ui/sidebar"` (same import block updated in Task 2):

```ts
import {
  SidebarContent,
  SidebarFooter,
  SidebarGroup,
  SidebarHeader,
  SidebarMenuAction,
  SidebarMenu,
  SidebarMenuButton,
  SidebarMenuItem,
  SidebarMenuSkeleton,
  SidebarMenuSub,
  SidebarMenuSubButton,
  SidebarMenuSubItem,
  SidebarSeparator,
  SidebarTrigger,
  useSidebar,
} from "./ui/sidebar";
```

At the top of the `Sidebar` component body (near the other hooks, around line 382), add:

```ts
const { isMobile, setOpenMobile } = useSidebar();
```

- [ ] **Step 2: Close sidebar after thread navigation**

In `handleThreadClick` (line 938-974), after the `navigate()` call (line 961-964), add the mobile close. The plain-click branch becomes:

```ts
      // Plain click — clear selection, set anchor for future shift-clicks, and navigate
      if (selectedThreadIds.size > 0) {
        clearSelection();
      }
      setSelectionAnchor(threadId);
      void navigate({
        to: "/$threadId",
        params: { threadId },
      });
      if (isMobile) {
        setOpenMobile(false);
      }
```

Also add `isMobile` and `setOpenMobile` to the `useCallback` dependency array:

```ts
    [
      clearSelection,
      isMobile,
      navigate,
      rangeSelectTo,
      selectedThreadIds.size,
      setOpenMobile,
      setSelectionAnchor,
      toggleThreadSelection,
    ],
```

- [ ] **Step 3: Close sidebar after "New Thread" navigation**

The "New Thread" button (around line 1352-1359) calls `handleNewThread()` which internally calls `navigate()`. Since `handleNewThread` is from a shared hook, the cleanest fix is to close the sidebar after the `handleNewThread` call resolves.

Update the onClick handler at line 1352-1359 from:

```tsx
                  onClick={(event) => {
                    event.preventDefault();
                    event.stopPropagation();
                    void handleNewThread(project.id, {
                      envMode: resolveSidebarNewThreadEnvMode({
                        defaultEnvMode: appSettings.defaultThreadEnvMode,
                      }),
                    });
                  }}
```

To:

```tsx
                  onClick={(event) => {
                    event.preventDefault();
                    event.stopPropagation();
                    void handleNewThread(project.id, {
                      envMode: resolveSidebarNewThreadEnvMode({
                        defaultEnvMode: appSettings.defaultThreadEnvMode,
                      }),
                    }).then(() => {
                      if (isMobile) {
                        setOpenMobile(false);
                      }
                    });
                  }}
```

- [ ] **Step 4: Close sidebar after Settings navigation**

The Settings button (line 1847) also navigates without closing the mobile sidebar. Update from:

```tsx
                onClick={() => void navigate({ to: "/settings" })}
```

To:

```tsx
                onClick={() => {
                  void navigate({ to: "/settings" });
                  if (isMobile) {
                    setOpenMobile(false);
                  }
                }}
```

- [ ] **Step 5: Verify the build passes**

Run: `bun fmt && bun lint && bun typecheck`

Expected: All three pass with no errors.

- [ ] **Step 6: Commit**

```bash
git add apps/web/src/components/Sidebar.tsx
git commit -m "fix(web): auto-close mobile sidebar on navigation

Close the sidebar sheet when tapping a thread, creating a new thread,
or opening settings on mobile. Previously the sheet stayed open and
covered the content."
```

---

### Task 4: Final Verification

- [ ] **Step 1: Run the full check suite**

Run: `bun fmt && bun lint && bun typecheck`

Expected: All three pass with no errors.

- [ ] **Step 2: Run tests**

Run: `bun run test`

Expected: All existing tests pass.

- [ ] **Step 3: Manual smoke test checklist**

Verify the following by running the dev server (`bun dev` in `apps/web`):

1. Open the app in a desktop browser — everything works as before.
2. Open browser dev tools, toggle device emulation to an iPhone viewport:
   - Sidebar should open as a Sheet overlay.
   - Click a thread — sidebar should close and thread content should load.
   - Click "New Thread" — sidebar should close.
   - Click "Settings" — sidebar should close.
3. Hard-refresh the page — should see a dark background immediately (no white/gray flash), then a spinner on the connecting screen, then skeleton rows in the sidebar, then real threads.
4. Navigate directly to a thread URL before hydration completes — should see a centered spinner instead of a blank page.
