# Spec: Agent Live Showcase Page (`/agent-live`)

**Task:** TASK-00052  
**Branch:** `agent/task-TASK-00052`

---

## Problem Statement

Hive needs a public-facing, guest-accessible showcase page at `/agent-live` to demonstrate the agentic development platform to stakeholders. The page must be visually polished (dark theme, animated pipeline), fully self-contained (no external CDN, no React app dependency), and require zero authentication. It is a static marketing/demo page, not an application feature.

## Intended Behavior

- A visitor who navigates to `https://<site>/agent-live` (not logged in) sees a dark-themed landing page.
- The page renders immediately with no login redirect.
- The centerpiece is an animated horizontal pipeline: **Issue → Dev Box → Spec → Implement → Pull Request**. Stages light up in sequence; a glowing "progress dot" travels from left to right on repeat.
- Below the pipeline, four feature cards describe key capabilities with subtle hover effects.
- A footer credits the page as "Built autonomously by a Hive agent."
- The page is fully self-contained: all CSS and JS are inline — no external CDN, no `/assets/` references that could break, no dependency on the React frontend bundle.

---

## Concrete Changes Required

### 1. New Frappe www page: `bwh_hive/www/agent-live.html`

This is a **single HTML file** (no paired `.py` needed) because the page is entirely static. Frappe serves any `.html` file in `www/` as a guest-accessible route automatically.

**File path:** `bwh_hive/www/agent-live.html`

Structure:

```html
{% extends "templates/web.html" %}

{% block title %}Hive – AI agents that ship your features{% endblock %}

{% block head_include %}
<style>
  /* All page styles inline here */
  /* Dark theme variables, pipeline animation, card hover effects */
</style>
{% endblock %}

{% block page_content %}
  <!-- Hero section: product name + tagline -->
  <!-- Animated pipeline -->
  <!-- Feature cards grid -->
  <!-- Footer -->
{% endblock %}

{% block script %}
<script>
  /* Pipeline stage sequencer — pure vanilla JS, no deps */
</script>
{% endblock %}
```

Key design implementation details:

**Dark theme:** `background: #0d0f14`, accent gradient `linear-gradient(135deg, #7c3aed, #2563eb)`, text `#e2e8f0`.

**Pipeline animation (CSS + JS):**
- Five `<div class="stage">` elements in a flex row, connected by `<div class="connector">` lines.
- Each stage has a label below and a circular icon node.
- A `<div class="travel-dot">` absolutely positioned on the connector track, animated via `@keyframes travelDot` (CSS `left: 0% → 100%`).
- JS cycles an `.active` class across stages every 1.2 s with `setInterval`, restarting after stage 5.

**Feature cards:**
```
| Firecracker microVM in ~20s | Claude writes the spec |
| Human approves              | Auto-opens a PR        |
```
Cards use `background: #1a1d27`, border `1px solid #2d3148`, `border-radius: 12px`, `padding: 24px`. On `:hover`: `transform: translateY(-4px)` + `box-shadow: 0 8px 32px rgba(124,58,237,0.3)`, transition `0.25s ease`.

**Footer:** `<footer>Built autonomously by a Hive agent.</footer>` — small, centered, muted color `#64748b`.

### 2. No `.py` context file needed

Because the page is entirely static (no server-side data), Frappe will serve `agent-live.html` directly. A paired `agent-live.py` is only required if server-side context variables are needed. This page requires none.

### 3. No `hooks.py` changes needed

Frappe auto-discovers `www/*.html` files and routes them to `/<filename>`. The route `/agent-live` will work without any addition to `website_route_rules` in `bwh_hive/hooks.py`.

### 4. No DocType, migration, or API changes

This is a pure presentation page. No new DocTypes, no database columns, no API endpoints, and no `bench migrate` step are required.

---

## Guest Access Verification

Frappe grants guest access to `www/` pages by default. The key controls are:

- `no_cache` flag: not needed for a static `.html` page (Frappe handles caching headers).
- Login required: NOT set (the page does not use `{% if frappe.session.user == "Guest" %}` guards).
- Frappe's `website_settings` does not gate individual `www/` pages unless `login_required` is set in a Python context file. Since we have no `.py` file, the default is guest-accessible.

If the site has globally enabled "Login Required" in Website Settings, an admin would need to add `/agent-live` to the allowed guest pages list — document this in a comment at the top of the HTML file.

---

## Edge Cases and Constraints

| Scenario | Handling |
|---|---|
| Site has global "Login Required" enabled | Page will redirect to login. Mitigation: add `agent-live` to Website Settings > Allow Guest Access, or add a paired `agent-live.py` with `login_required = False`. Document in file header. |
| CSS animation not supported (old browser) | The pipeline still renders as a static row of stages — no layout breakage. Animation is a progressive enhancement. |
| Narrow viewport (mobile) | Pipeline flex row wraps or overflows. Mitigation: add `overflow-x: auto` on the pipeline container and a `@media (max-width: 640px)` rule that stacks stages vertically. |
| Dark OS theme / forced colors | Page explicitly sets `background` and `color` on `:root`, so it is unaffected by system theme. |
| `prefers-reduced-motion` | Wrap animation keyframes and `setInterval` in a `matchMedia('(prefers-reduced-motion: reduce)')` check and skip animation if true. |
| File naming: `agent-live.html` vs `agent_live.html` | Frappe maps filenames directly to URL paths. A hyphenated filename (`agent-live.html`) produces the route `/agent-live`. Underscore would produce `/agent_live`. Use the hyphenated name per the task spec. |
| Frappe version compatibility | `{% extends "templates/web.html" %}` and `{% block page_content %}` are standard across Frappe v14+. No version-specific concern. |

---

## File Summary

| File | Action | Notes |
|---|---|---|
| `bwh_hive/www/agent-live.html` | **Create** | Self-contained HTML page, all CSS/JS inline |
| `bwh_hive/hooks.py` | No change | Auto-routing handles `/agent-live` |
| Any DocType JSON | No change | No data model changes |
| Database / migrations | No change | Pure frontend page |

---

## Verification Checklist

A reviewer can validate the implementation by running the following steps:

1. **File exists:**
   ```
   ls bwh_hive/www/agent-live.html
   ```
   Expected: file present.

2. **No external CDN references:**
   ```
   grep -E 'https?://' bwh_hive/www/agent-live.html
   ```
   Expected: zero matches (or only comments — no live `src=` / `href=` pointing to CDNs).

3. **Guest access without login:**
   Open `http://pms.localhost:8000/agent-live` in a private/incognito browser window (not logged in to Frappe).
   Expected: page loads fully, no redirect to `/login`.

4. **Dark theme renders correctly:**
   Background is dark (`#0d0f14` or similar), text is light. No bright-white flash on load.

5. **Pipeline animation plays:**
   Wait 6 seconds and observe all five stages — **Issue, Dev Box, Spec, Implement, Pull Request** — lighting up in sequence. The cycle restarts after the last stage.

6. **Feature cards hover effect:**
   Hover over each of the four cards and confirm the card lifts (`translateY`) with a purple glow shadow.

7. **Footer text present:**
   ```
   grep "Built autonomously" bwh_hive/www/agent-live.html
   ```
   Expected: one match.

8. **Mobile/narrow viewport:**
   Resize browser to ≤ 640 px width. Pipeline container should be scrollable or stack vertically — no horizontal page overflow breaking the layout.

9. **Reduced-motion respected:**
   In DevTools, enable "Emulate CSS prefers-reduced-motion: reduce". Pipeline should display as a static state (no traveling dot or stage cycling).

10. **No Python errors in Frappe log:**
    After visiting the page, check `frappe.log` or the bench console for any 500 errors.
    Expected: clean — no tracebacks.
