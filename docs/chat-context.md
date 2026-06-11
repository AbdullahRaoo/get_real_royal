# Chat Context: Framer site edits and CTA locking

Last updated: 2026-06-10

This document consolidates the conversation, technical investigation, and code changes performed during the session to make user edits persist and force CTA navigation to the hosted app.

## 1. Goal
- Make user edits persistent on a Framer-exported static site where the Framer runtime rehydrates and overwrites static DOM edits.
- Hide Framer editor chrome (`#__framer-editorbar-container`).
- Lock page titles and paragraph copy so the Framer runtime does not revert them.
- Force all primary CTAs ("Try it now", "Create your profile", "Crea tu perfil") to navigate to `https://app.realroyal.co/` and prevent Framer runtime navigation interception (e.g., `#cta` or framer template links).

## 2. Workspace
- Root: D:\work\Websites\get.realroyal
- Files edited:
  - `index.html`
  - `contact/index.html`
  - `home-es/index.html`
- New file added: `docs/chat-context.md` (this file).

## 3. Key Findings
- Framer runtime registers handlers on elements with `[data-nested-link]` that intercept clicks/auxclick/keydown and call an internal navigation helper which can override anchor `href` behavior and navigate to `#cta` or Framer templates.
- Simply changing `href` attributes in source is not sufficient because runtime event handlers still intercept and route the click.

## 4. Defensive Strategy Implemented
1. Attribute locking
   - Scripts scan `document.querySelectorAll('a')` and set `href` to `https://app.realroyal.co/` for anchors matching target labels.
2. MutationObserver + timeouts
   - Observers and repeated `setTimeout` calls re-apply title, text, and `href` fixes after Framer rehydration.
3. Capture-phase click override (final fix)
   - Added `forceAppNavigation` handler attached with `{ capture: true }` to `click`, `auxclick` and `keydown` events.
   - Handler identifies anchors (via `closest('a')`) and, if the label matches known CTAs or the `href` looks like `#cta` or a framer link, prevents default, stops propagation, and executes `window.location.assign('https://app.realroyal.co/')`.
   - This ensures navigation is forced before Framer's own handlers run.

## 5. What was changed
- `index.html`:
  - Inserted script block with `applyText()` (forces paragraph text), `applyTitle()` (locks document title), `lockAppLinks()` (sets `href` on anchors), MutationObserver, repeated timeouts, and `forceAppNavigation` capturing handlers.
  - Replaced CTA `href` occurrences in source with `https://app.realroyal.co/`.
  - Added CSS to hide `#__framer-editorbar-container` earlier in session.
- `contact/index.html` and `home-es/index.html`:
  - Added similar guard scripts and capture-phase `forceAppNavigation` handlers tailored to localized CTA labels.

## 6. Why the capture-phase handler
- Framer binds click handlers to elements that can intercept navigation. A capturing listener runs before those handlers and can call `preventDefault()` and `stopImmediatePropagation()` to prevent Framer’s handlers from running, then perform the desired navigation directly.

## 7. How to validate locally
- Open the modified pages in a browser (hard refresh to avoid cached scripts):

```powershell
# from workspace root
Start-Process "file:///${PWD/\\/\/}/index.html"
# or open via file explorer
```

- Interact with the CTA buttons (`Try it now`, `Create your profile`, `Crea tu perfil`) and confirm they navigate to `https://app.realroyal.co/`.
- Check the page title and edited paragraph remain as set.

## 8. Next steps / Recommendations
- Long-term: fix the source Framer project and re-export with correct links and content, or self-host a trimmed Framer runtime to remove unexpected behavior.
- If further hardening desired: whitelist specific anchor `rel`/`target` attributes or add stricter label matching (e.g., case-insensitive normalization, ignore nested markup differences).
- Optionally run a quick grep across the repo for other Framer-provided links and duplicate `data-nested-link` usages to cover more CTA variants.

## 9. Contact / provenance
- Edits performed in-session on 2026-06-10. See commit history or local diffs for the exact patches applied.

---

If you want this content expanded into a developer README or an actionable PR (with tests and a small QA checklist), tell me where to place it and I will draft it.