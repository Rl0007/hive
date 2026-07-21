# Spec: Add FAQ Section to README.md

## Problem Statement

The project README (`README.md`) currently covers installation, contributing, CI, and license but provides no quick-reference answers to the most common first-contact questions: what the project is, how to run it locally for development, and how to contribute code. A short FAQ section at the end of the file removes that friction for new contributors and evaluators.

## Intended Behavior

After the change, `README.md` ends with a `### FAQ` section containing three Q&A pairs rendered as collapsible `<details>` blocks (GitHub renders these natively) or plain bold-question / paragraph-answer pairs. Content must be accurate against the actual repo layout and existing documentation already in the file.

---

## Concrete Changes Required

### File: `README.md`

**Only change:** append a `### FAQ` section after the existing `### License` block (line 40).

No other files are touched. No DocTypes, API methods, UI components, or database migrations are involved — this is a documentation-only change.

#### Questions and answers to include

| # | Question | Answer source in repo |
|---|----------|-----------------------|
| 1 | **What is BWH Hive?** | Derive from existing README header ("Modern Project Management Software") and the app name `bwh_hive`. |
| 2 | **How do I run it locally?** | Combine the bench installation steps already in `### Installation` with the frontend dev-server command `yarn dev` documented in `CLAUDE.md` (`frontend/` runs Vite on `localhost:8080`; Frappe backend on `pms.localhost:8000`). |
| 3 | **How do I contribute?** | Summarise the existing `### Contributing` section: fork → branch → `pre-commit install` → PR targeting `develop`. |

#### Exact location to append

```
README.md line 41 (after "agpl-3.0"):

### FAQ
...
```

---

## Edge Cases and Validation

- **No duplicate content:** The FAQ must summarise, not copy verbatim, the Installation and Contributing sections to avoid drift if those sections are later updated.
- **Markdown rendering:** Test that the section renders correctly on GitHub (bold questions or `<details>` elements). Avoid raw HTML beyond `<details>`/`<summary>` if used; both are supported by GitHub Markdown.
- **Line endings:** The file currently uses Unix line endings (`\n`). The appended block must not introduce CRLF.
- **Pre-commit:** The `prettier` hook is configured for this repo (`.pre-commit-config.yaml`). Run `pre-commit run --all-files` after the edit to confirm no formatting violation is flagged on the Markdown file (prettier may or may not lint `.md`; verify it passes clean).

## Migration Concerns

None. This is a pure documentation change with no schema, data, or dependency impact.

---

## Verification Checklist

A reviewer can confirm the change is correct by running the following steps:

1. **File exists and is modified**
   ```bash
   grep -n "### FAQ" README.md
   # Expected: one match near the end of the file
   ```

2. **All three questions are present**
   ```bash
   grep -c "What is BWH Hive\|run it locally\|contribute" README.md
   # Expected: 3
   ```

3. **Section is at the end** — `### FAQ` must be the last `###` heading:
   ```bash
   grep -n "^### " README.md | tail -1
   # Expected line should contain "FAQ"
   ```

4. **No line-ending issues**
   ```bash
   file README.md
   # Expected: "ASCII text" or "UTF-8 Unicode text" (no "CRLF")
   ```

5. **Pre-commit passes**
   ```bash
   cd /workspace/repo && pre-commit run --all-files
   # Expected: all hooks pass (exit 0)
   ```

6. **Visual check** — open `README.md` on GitHub (or in a Markdown preview) and confirm:
   - The FAQ heading is visible.
   - Each question is clearly distinguished from its answer.
   - No broken Markdown (unclosed backticks, stray `#` symbols, etc.).
