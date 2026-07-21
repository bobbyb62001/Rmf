# Integrating the RMF Module into grc-toolkit.html

This walks through folding `rmf-module.html` into your existing single-file
app as a 13th tab, alongside Evidence Collector, POA&M Tracker, etc.

## 1. Add the tab button

Find where your existing 12 tab buttons are defined (likely a row of
`<button>` elements with `onclick` handlers or a JS array driving them) and
add a 13th:

```html
<button class="tab-btn" data-tab="rmf">RMF</button>
```

Match whatever class/attribute pattern your other 12 already use — don't
introduce a second tab-switching mechanism. If your app switches tabs via a
JS function like `showTab('evidenceCollector')`, call the same function with
a new tab id, e.g. `showTab('rmf')`, rather than using the `tab-tracker` /
`tab-study` / `tab-certs` / `tab-nist` switcher built into the standalone file.

## 2. Add the panel

Copy the four `<div class="panel" id="panel-...">` blocks (tracker, study,
certs, nist) from `rmf-module.html` into a single wrapping panel:

```html
<div id="panel-rmf" class="app-panel"> <!-- match your app's panel class -->
  <!-- paste the RMF module's internal 4-tab sub-nav + all 4 panel divs here -->
</div>
```

Keeping the module's own internal sub-tabs (Tracker/Study/Certs/NIST Library)
nested inside your single outer "RMF" tab avoids fighting your app's existing
tab system — you get a tab-within-a-tab, which is a normal pattern.

## 3. Namespace the CSS

This is the step most likely to bite you. The module's CSS uses generic
class names (`.card`, `.detail-card`, `.task`, `.choice`, `.progress-fill`,
etc.) that are likely to collide with your existing 12 tabs' styles.

Prefix every selector in the module's `<style>` block with `.rmf-module`:

```css
/* before */
.card{background:var(--paper);...}

/* after */
.rmf-module .card{background:var(--paper);...}
```

Then wrap the pasted panel markup:

```html
<div id="panel-rmf" class="app-panel rmf-module">
  ...
</div>
```

A quick way to do the prefixing in bulk: copy the `<style>` block into a
scratch file and run a find/replace that prefixes every rule starting with
`.` (skip `@media`, `:root`, and `*` selectors — leave those as-is, or
better, drop `:root` entirely and inline the CSS variables your existing app
already defines if the names collide, e.g. if you already have `--ink` or
`--navy` defined elsewhere).

## 4. Namespace the storage keys

The module uses three `window.storage` keys:

- `rmf-tracker-checks`
- `rmf-quiz-best`
- `rmf-cert-checks`

These are already namespaced with an `rmf-` prefix, so they're unlikely to
collide with your other tabs' storage (Evidence Collector, POA&M Tracker,
etc.) — but double check none of your existing tabs already use a key
starting with `rmf-`.

## 5. Fonts

The module pulls Source Serif 4 / IBM Plex Sans / IBM Plex Mono from Google
Fonts via a `<link>` tag in `<head>`. If your app already loads its own
fonts, either:
- merge the `<link>` tags (fine to load multiple font families), or
- swap the module's `font-family` declarations to match your existing app's
  fonts, so the RMF tab doesn't look visually disconnected from the other 12.

## 6. Test in isolation first

Before merging, open `rmf-module.html` standalone and click through all four
sub-tabs, check a few tracker boxes, run a quiz question, and reload the page
to confirm `window.storage` persistence works in your environment. That
isolates "did the merge break something" from "did the module ever work."

## 7. Merge order

Given your dev setup (Cursor + Git, private repo), the safest path:

1. Create a branch (`feature/rmf-module`)
2. Paste in the tab button, panel, CSS (prefixed), and JS in that order
3. Run it locally, click through all 4 sub-tabs
4. Check the browser console for any duplicate `id` warnings — the module
   uses plain `id` attributes (`taskList`, `studyArea`, `scoreNum`, etc.)
   which must be unique across your *entire* page, not just within the panel
5. Commit, open a PR against your normal branch even if it's just you
   reviewing — keeps `docs/architecture.md` accurate if you log it there

## 8. Link the NIST Library to your existing Control Lookup tab

Each row in the 800-53 family table (NIST Library tab) is now clickable and
calls `window.grcJumpToControlFamily(code)` if that function exists — e.g.
clicking "AC — Access Control" calls `grcJumpToControlFamily('AC')`. Until
you define it, clicking just logs a console message and does nothing, so
it's safe to merge before this step is done.

To wire it up, add a function to your app's global scope (outside the RMF
module's own script, in whatever file drives your Control Lookup / SCF
Explorer tab) that does two things: switch to that tab, then apply the
family code as a filter. Adapt the specifics to your actual element IDs and
tab-switch function — this is a sketch, not exact code:

```javascript
window.grcJumpToControlFamily = function(code) {
  // 1. switch to your existing Control Lookup tab
  showTab('controlLookup'); // use your app's real tab-switch function/id

  // 2. apply the family code as a filter
  const searchInput = document.getElementById('controlSearchInput'); // real id
  if (searchInput) {
    searchInput.value = code;
    searchInput.dispatchEvent(new Event('input')); // triggers existing filter logic
  }
};
```

If Control Lookup filters via a dropdown or chip-select instead of a text
input, swap step 2 for whatever sets that filter — the key parts are: (a)
switch tabs first, (b) then apply the filter, in that order, since some tab
switches reset visible state.

Once that function exists anywhere in the page's global scope, the NIST
Library rows will jump straight into a filtered Control Lookup view with no
further changes needed on the RMF module side.

