# Spec: Floating Action Button — Hello World Test

## Problem Statement

This is a verification task: confirm that the end-to-end flow of adding a UI feature (frontend component → visible in browser) works correctly in this environment. The deliverable is a floating action button (FAB) rendered globally in the app; clicking it shows a "Hello World" toast or alert.

## Intended Behavior

- A circular button is fixed in the bottom-right corner of every authenticated page.
- Clicking the button displays the text **"Hello World"** via a toast notification (using Sonner, already wired in `main.tsx`).
- The button does not interfere with other fixed elements (PinnedTasksDock, dialogs).
- No backend changes are required.

---

## Concrete Changes Required

### 1. New Component

**File**: `frontend/src/components/FloatingHelloButton.tsx` _(new file)_

```tsx
// Renders a fixed circular button in the bottom-right corner.
// On click, fires a Sonner toast with "Hello World".
```

- Use the existing `Button` component from `frontend/src/components/ui/button.tsx`
  with `size="icon-lg"` and `variant="default"`.
- Use a HugeIcon (e.g., `Add01Icon` or `MessageAdd01Icon`) from `@hugeicons/react`
  as the button icon — the library is already installed.
- Wrap with a `Tooltip` (from `frontend/src/components/ui/tooltip.tsx`) so
  keyboard/screen-reader users see the label "Hello World".
- Position: `fixed bottom-6 right-6 z-50 rounded-full shadow-lg`.
  - `z-50` places it above `PinnedTasksDock` which uses `z-40`.
  - `bottom-6 right-6` avoids overlap with the dock (currently anchored at
    `bottom-0 right-0` with its own padding).

### 2. Mount the Component in the Global Layout

**File**: `frontend/src/components/layout/AppLayout.tsx` _(modify)_

- Import `FloatingHelloButton` and render it inside the `SidebarInset` element,
  after the existing `PinnedTasksDock` render site.
- No new state is needed — the toast is self-contained.

```diff
+ import { FloatingHelloButton } from "@/components/FloatingHelloButton";

  // inside the JSX return, after PinnedTasksDock:
+ <FloatingHelloButton />
```

### 3. Toast Call

Inside `FloatingHelloButton`, the click handler calls:

```ts
import { toast } from "sonner";
toast("Hello World");
```

Sonner's `<Toaster>` is already rendered in `main.tsx`, so no additional setup is
needed.

---

## No Backend / DocType Changes

This feature is pure frontend. No Frappe DocTypes, Python methods, or database
migrations are required.

---

## Edge Cases & Validation

| Concern | Resolution |
|---|---|
| FAB overlaps PinnedTasksDock | Use `z-50`; PinnedTasksDock uses `z-40`. Verify visually on pages where dock is visible. |
| FAB overlaps page scrollbars | `right-6` provides 24 px gap from viewport edge, sufficient for most OS scrollbar widths. |
| FAB hidden on mobile-sized viewports | No mobile-specific breakpoint needed; Tailwind `fixed` keeps it in view. If future mobile support requires hiding it, add `hidden sm:flex`. |
| Multiple rapid clicks | Sonner stacks toasts by default; acceptable for a test feature. No debounce needed. |
| Unauthenticated pages (login screen) | `FloatingHelloButton` is rendered inside `AppLayout`, which is only mounted for authenticated routes in `App.tsx`. It will not appear on the login page. |

---

## Verification Checklist

A reviewer can confirm the feature works by running through the following steps:

1. **Start the dev server**
   ```bash
   cd frontend && yarn dev
   ```
   or use the bench server at `pms.localhost:8000/hive`.

2. **Navigate to any authenticated page** (Dashboard, Projects, Tasks, Team).

3. **Locate the FAB** — a circular button should be visible in the bottom-right
   corner of the viewport, above any other fixed elements.

4. **Click the FAB** — a toast notification reading **"Hello World"** should
   appear (typically bottom-right or top-right, per Sonner defaults in `main.tsx`).

5. **Tooltip check** — hover over (or Tab to) the button; a tooltip labelled
   "Hello World" should appear.

6. **No overlap with PinnedTasksDock** — if pinned tasks are visible, confirm the
   FAB sits above the dock without obscuring the dock's controls.

7. **Login page check** — visit `/login`; the FAB must **not** appear there.

8. **No console errors** — browser DevTools console should be clean after clicking.
