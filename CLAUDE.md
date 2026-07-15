# Chess Library

A client-side, single-file chess PGN viewer/library for personal OTB (over-the-board)
games. No build step, no framework — one `index.html` with inline `<style>`/`<script>`.
Hosted on GitHub Pages at the custom domain `library.chessscenes.com` (migration complete,
see below). Ships a mobile-first card/bottom-sheet UI alongside the original desktop table,
a light/dark theme, and a set of editing tools (single + bulk game-metadata edit, per-move
comment editing, a native date picker) on top of the original read-only viewer.

## Development workflow (important — follow this, don't edit index.html directly)

The actual working copy is **`/Users/harman.singh/workspace/chess-analysis/library.html`**,
not this repo's `index.html`. Workflow for any change:

1. Edit `chess-analysis/library.html`.
2. Sanity check: `node -e '...new Function(script)...'` (extract the `<script>` block and
   confirm it parses).
3. For anything behavioral, write a quick Node test using a minimal DOM stub (see below)
   or a real Playwright screenshot/emulation for visual changes. This project has been
   tested this way all along — don't skip it just because it's a "small" change.
4. Copy to this repo's **`dev/index.html`** (staging path, same repo/domain, live at
   `https://library.chessscenes.com/dev/`) first: `cp chess-analysis/library.html
   chess-library/dev/index.html`, syntax-check the copy, commit, push. Let the user review
   on staging (including on their actual phone) before promoting.
5. Only after explicit approval, promote to production: `cp dev/index.html index.html`,
   syntax-check, commit, push. Run a full regression pass (desktop + real mobile emulation,
   not just a resized viewport — see the viewport-meta-tag lesson below) immediately before
   this push, since this is the file real users see.
6. GitHub Pages auto-deploys via `.github/workflows/pages.yml` (Actions-based deploy, not
   the legacy Jekyll pipeline). Push usually deploys in under a minute; check `gh run list
   --repo harman28/chess-library --limit 1` to confirm, but don't obsessively poll.

**Why the dev/ split exists**: the user explicitly rejected a separate repo for staging
("I don't like the idea of a separate repo at all"), so staging is a subpath of the same
repo/deployment instead. `dev/index.html` is allowed to be iterated on more freely;
`index.html` (root) is the one real users are on and needs the full regression pass before
every push. **This applies to visual/content decisions too, not just code risk** — a git
push to prod containing new, never-reviewed copy or design (e.g. a first-draft favicon, an
OG-image tagline nobody had seen yet) has been auto-blocked by the environment's own safety
classifier mid-session for exactly this reason. Stage new copy/visual work, show it to the
user, get explicit sign-off, *then* promote — even for things that feel "obviously fine."

Node test harness pattern used throughout: stub `document.getElementById`, `localStorage`,
`window` (needed for `window.addEventListener`/`matchMedia`/`scrollTo`), `history`,
`location` (needed since `STORAGE_KEY` and the API path both read `location.pathname`),
`navigator` (needed for `navigator.vibrate` used by the mobile long-press haptic), etc.,
concatenate test code onto the extracted `<script>` source, and `eval()` the whole thing as
one string (top-level `let`/`const` inside a bare `eval()` call don't leak out to separate
`eval()` calls in Node — build one combined script string, not multiple `eval()`s).

For anything mobile-specific, **use Playwright's real device emulation
(`context = await browser.newContext({ ...devices['iPhone 13'] })`), not just
`viewport: {width, height}`.** A plain resized viewport does not reproduce mobile browsers'
"virtual viewport" behavior — see the viewport meta tag bug below, which every
manually-resized-viewport test missed entirely. Playwright itself isn't installed as a
direct dependency in either project directory; `/Users/harman.singh/workspace/boardscan/node_modules`
has it — run test scripts with `NODE_PATH=/Users/harman.singh/workspace/boardscan/node_modules
node your_script.js`.

## Architecture / key concepts

- **All state is client-side.** `games` array + `playerName` + `lightModeEnabled` persisted
  to `localStorage` under `STORAGE_KEY`, gated by `SCHEMA_VERSION` — bump this constant
  whenever the *meaning* of a stored field changes, not just when a field is added/removed.
  This bit us once already: renaming `classifySpeed()`'s `"other"` return value to
  `"untimed"` without bumping the version meant every already-cached session had games with
  the stale value baked in, and the new "Untimed" filter matched zero of them for anyone
  with an existing session.
  `STORAGE_KEY` is also namespaced by path (`"chessLibraryData" + (location.pathname
  .startsWith("/dev/") ? "_dev" : "")`) — `localStorage` is scoped per-*origin*, not per-path,
  so staging and production were silently sharing the same browser storage key despite being
  served from different URLs, until this was namespaced.
- **Columns are fully dynamic.** `discoverColumns()` inspects whatever PGN header tags are
  actually present across loaded games and builds the column list from that.
  `columnState.order` + `columnState.visible` control display; users can show/hide/reorder
  via the Columns panel (floating icon over the table's top-right corner, drag-and-drop
  reordering via native HTML5 DnD events).
- **Sorting**: by PGN Date descending, then **Round descending as a tiebreaker**
  (`sortGamesByDate()` → `dateSortKey()` / `roundSortKey()`), so same-day informal games can
  be sequenced via the Round field. Re-applied on every load/add/restore/edit. `idx` is
  reassigned sequentially after each sort — not stable across sorts or edits, just a
  display/DOM-tracking number. Any edit that can change a game's Date or Round must re-sort
  and must not assume the previously-open detail view's `idx` is still valid afterward (see
  the game-edit modal's handling of this below).
- **Speed classification (`classifySpeed`) has a unit-ambiguity heuristic — read this
  before touching it.** Lichess/chess.com always store PGN `TimeControl` in seconds
  (`"1500+10"` = 25 min), and because their own UIs only offer whole-minute base times, that
  value is always an exact multiple of 60. Some OTB/manual PGN sources instead write the
  base time directly in minutes (`"90+30"` meaning 90 min, not 90 sec) — a different,
  equally valid convention, but if naively divided by 60 it turns a 90-minute classical game
  into "1.5 minutes" and misclassifies it as blitz. Fixed by checking `raw % 60`: if it's
  not an exact multiple of 60, treat `raw` as already being in minutes. This exactly
  explained a real bug report ("Blitz shows 53 games, I know of only 1") down to the exact
  count. Games with no parseable `TimeControl` at all (missing tag, `"-"`, `"?"`, etc.)
  classify as `"untimed"`, with its own filter chip alongside Blitz/Rapid/Classical.
- **Toolbar layout** uses `flex-direction: row-reverse` deliberately (not a media query)
  for structural (not override-based) responsive reordering on desktop widths. On mobile
  (`≤700px`) a separate, more aggressive layout takes over — see the mobile section below.
- **Selection mode is a single neutral state, not per-action sub-modes.** `selectionMode`
  is just a boolean. The *only* toolbar entry point is `#enterSelectionBtn` ("Edit"), which
  reveals `#bulkEditBtn` ("Edit", bulk-metadata edit), `#exportBtn`, `#deleteBtn`, and
  `#cancelSelectionBtn` all together — none of them lock you into a committed "export mode"
  or "delete mode" the way they used to. `#cancelSelectionBtn` exits with zero side effects
  (no download, no deletion) — this replaced an earlier design where the *only* way out of
  an accidentally-entered selection was to click Export or Delete again, which could trigger
  an unwanted download just to back out. Mobile long-press on a card and the mobile
  "select all" checkbox flow are additional entry points into the same shared state.
  **Shift-click range selection** works on both the desktop table (`lastCheckedRowIdx`) and
  mobile cards (`lastCheckedMobileIdx`): click one checkbox, shift-click another further up
  or down the *currently rendered/filtered* list, and everything between gets selected —
  scoped to DOM position in the current view, not the underlying game `idx`, so it does the
  right thing under an active filter or search. Both anchor variables reset on
  `exitSelectionMode()` so a fresh session doesn't range back to a stale anchor.
- **Game editing**: `openEditGameModal(g)` (single game, from the pencil icon on a desktop
  expanded row or the mobile sheet) vs `openBulkEditModal(indices)` (multiple selected
  games, from `#bulkEditBtn`) share one modal. Single mode shows White/Black and always
  applies every field (including clearing one intentionally). Bulk mode hides White/Black
  entirely (not meaningful across different games) and treats a **blank field as "leave
  unchanged"** for every other field — Result's blank state is an explicit "— Don't change
  —" dropdown option rather than empty-string-as-a-value, since empty string is itself a
  valid PGN result marker. Saving either mode rebuilds the affected game(s) via the existing
  `buildGameRecord()` so Result/TimeControl edits correctly reclassify `outcome`/`color`/
  `speed`, then re-sorts and refreshes only the currently-open detail view *in place*
  (`refreshOpenDetailViews`) rather than a full `render()`, so the row stays expanded / the
  mobile sheet stays open across an edit. The Date field is a native `<input type="date">`,
  converted to/from PGN's `YYYY.MM.DD` via `pgnDateToIso()`/`isoDateToPgn()` —
  `pgnDateToIso()` validates the date is a real, fully-specified calendar date via a
  round-trip through `Date.UTC` before accepting it, so PGN's partial/unknown dates (e.g.
  `"????.??.??"`) just leave the picker blank instead of crashing or showing garbage.
- **Per-move comment editing**: `tokenizeMovetext()` is the single source of truth for
  parsing PGN movetext into typed tokens (move number / comment / SAN move), used by both
  `formatMovesWithComments()` (rendering, tags each SAN span with `data-game-idx`/
  `data-move-idx` so it's clickable) and `getMoveCommentInfo()`/`applyCommentEdit()` (find
  whether a comment immediately follows a given move, and do the minimal string splice to
  insert/replace/remove it) — rendering and editing can never drift out of sync since they
  share one parser. Click/tap a move → a small modal (matching the game-edit modal's visual
  language) to add, edit, or delete that move's comment; Delete is hidden if there's no
  existing comment. Because desktop *always* renders every game's full move-list HTML into
  the DOM up front (just CSS-hidden until a row is expanded — see mobile section below),
  `refreshOpenDetailViews()` updates *both* the desktop `#pgnrow-<idx>` copy and the mobile
  sheet's copy after an edit, regardless of which one is actually visible.
- **Lichess embed** (click a row / tap a card → board + formatted PGN+comments): imports
  the game via Lichess's public `POST /api/import` (open CORS, no auth), then embeds
  `lichess.org/embed/game/<id>?theme=blue2&pieceSet=caliente&bg=<light|dark>` (the `bg` param
  follows the site's own light/dark theme setting), cropped via a fixed-height
  `overflow:hidden` wrapper to hide the move-list panel (pixel geometry measured directly
  per reference width — 360px for desktop, 260px for the mobile sheet — since Lichess has no
  query param for this and the layout doesn't scale perfectly linearly). **Known limitation,
  not a bug to "fix"**: Lichess's import strips custom PGN comments and replaces them with
  its own auto-generated notes, confirmed on both `/embed/game` and `/analysis`'s own
  Import-PGN feature — this is why the formatted-comments panel is the client-side source of
  truth for annotations, and why the **"Analyse" button intentionally still opens
  chess.com, not Lichess**.
  **`g.pgn` (and anything built from it, like `buildPgnForGames()`/export) must always be
  raw, unescaped PGN text** — it's sent to Lichess's import API and used for clipboard
  copy/export, never inserted into `innerHTML` (confirmed via grep before relying on this).
  It used to run PGN header values through `esc()` (HTML-entity escaping) when building
  `g.pgn`, which corrupted any header containing `&`, `<`, `>`, or `"` for every one of
  those uses. Don't re-introduce HTML-escaping anywhere in the `g.pgn`/export pipeline.
- **Accounts (username/password, replacing "Save My Library" as the primary flow)**:
  calls the `chess-library-api` backend's `/api/auth/*` and `/api/library/mine` endpoints
  (separate repo, see its own CLAUDE.md for the full endpoint list and auth design notes).
  A `#accountBtn` lives in `.page-header`, **outside `#app`/`#landing`**, so it's visible
  whether or not any games are currently loaded — it was originally nested inside the
  in-app toolbar (like the old Save My Library button was) but that meant a returning
  logged-in user with no local data yet had no way to log in, since the toolbar only
  renders once games are loaded. Don't renest it back inside `#app`.
  Signup/login share one modal (`#authModal`, same `.modal-overlay`/`.modal-card` pattern
  as Add Games/game-edit) toggled between modes via `authMode`/`updateAuthModalMode()`.
  **Signup pushes the current locally-loaded library up to the new account** (if any games
  are loaded); **login pulls the account's saved library down and replaces what's showing**
  — this asymmetry is deliberate: signup is "claim what I already have", login is "give me
  my account's data back". A 404 on login's pull (fresh account, nothing saved yet) is a
  no-op, not a reset — it deliberately does not clear locally-loaded data, to avoid
  destroying an anonymous session someone was just trying out.
  Session token lives in `localStorage` (`chessLibraryAuthToken` / `_dev` suffix, same
  path-based namespacing as `STORAGE_KEY`) and is sent as `Authorization: Bearer <token>`,
  not a cookie (cross-origin frontend/backend, see the API's CLAUDE.md for why). On load,
  `checkAuthOnLoad()` validates the stored token against `/api/auth/me` before trusting it.
  Every mutation still calls `saveToStorage()` exactly as before (instant local save is
  unchanged/preserved), which now also calls `scheduleAuthSync()` — a 1.5s-debounced
  `PUT /api/library/mine` that only fires when logged in, so the account copy stays in
  sync in the background without blocking the UI or hammering the API on every keystroke.
  The **old anonymous unlisted-link flow (`?lib=<id>`, `loadFromRemoteLibrary()`) still
  works and was deliberately kept** for backward compatibility with links already shared
  — it's just no longer the primary "save" action surfaced in the UI.
  **Google Sign-In is stubbed but not wired up**: the backend endpoint
  (`/api/auth/google`) and the frontend's hidden `#authGoogleWrap`/`#googleSignInBtn`
  placeholder exist, but there's no `GOOGLE_CLIENT_ID` yet (needs manual provisioning via
  Google Cloud Console — see the API repo's CLAUDE.md) and the frontend never loads
  Google's Identity Services script or calls `google.accounts.id.initialize()`, since
  there's no Client ID to configure it with. Don't half-wire this further until the
  Client ID exists; there's nothing useful to test without it.
- **Light/dark theme**: every color in the stylesheet is a CSS custom property, defined
  under `:root` (dark, the default) and re-defined under `:root[data-theme="light"]` — never
  hardcode a color directly in a new rule, add/reuse a token instead. The theme toggle
  (`#themeToggle` in Settings) sets `data-theme` on `<html>` via `applyTheme()`, persists
  `lightModeEnabled` alongside the rest of the saved session, and drives the Lichess embed's
  `bg` param too. A same-value-in-both-themes token still needs stating explicitly in both
  `:root` blocks for clarity (e.g. `--date-icon-filter`, which flips a native date-picker
  icon's CSS `filter` between `invert(1) brightness(1.3)` in dark mode and `none` in light,
  so the icon stays visible against either background).
- **Branding**: the favicon and the header logo lockup (`.logo-h1 .logo-dark` /
  `.logo-h1 .logo-light`, theme-toggled the same way as everything else) are inline SVG data
  URIs sourced from `logos/*.svg` in this repo (kept as tracked source files, even though
  their content is baked into the HTML as data URIs, not referenced live) — encode with
  `urllib.parse.quote(svg_text)`, never hand-escape. When toggling two theme-scoped elements
  by class, make sure the *toggle* rule is at least as specific as any general sizing rule
  touching the same elements (e.g. `.logo-h1 img{height:...}` will silently beat a
  lower-specificity `.logo-light{display:none}`, showing both variants stacked regardless of
  theme) — scope both to the same specificity, e.g. `.logo-h1 .logo-dark` /
  `.logo-h1 .logo-light`.
- **Social preview** (`og:image`/`twitter:image` etc. in `<head>`): points to `og-image.png`
  (also tracked in this repo, 1200×630, composed from the logo lockup on the site's own dark
  background) served at `https://library.chessscenes.com/og-image.png` — must be a real
  fetchable PNG at an absolute URL, not an SVG or data URI, since most social-platform
  crawlers won't render either for `og:image`. Since every URL into this site (root, or a
  saved `?lib=<id>` link) serves the exact same static HTML, one set of tags covers all of
  them automatically — no per-link dynamic generation needed or possible.
- **Demo data** (`DEMO_PGN` constant): four public-domain historical games, verified
  move-by-move against Wikipedia before shipping. If you ever add more demo content, verify
  against a real source first — the user does not trust generated chess content without
  independent checking. (One of these, the Opera Game, uses "Duke of Brunswick and Count
  Isouard" — spelled with "and", not "&" — deliberately, matching Wikipedia's own convention.)

## Mobile-first UI

Below `≤700px` (`@media (max-width:700px)`), the table (`.table-wrap`) is replaced by:

- **Card list** (`#mobileList`, `.mcard`): one card per game (date/event eyebrow, players
  with the user's own name bolded via a substring match, W/L dot + result + move count).
  Cards are `<div role="button" tabindex="0">`, not `<button>` — a real `<button>` cannot
  validly contain the selection checkbox (`.mcard-cb`). **Long-pressing a card (~500ms
  hold, tracked via `touchstart`/`touchmove`/`touchend` with a movement-tolerance check, not
  a native gesture API) enters selection mode and selects that card** — the trailing
  synthetic `click` that follows a long-press is suppressed via a `longPressFired` flag so it
  doesn't immediately toggle the same card back off.
- Desktop *always* renders every game's full move-list HTML into `#pgnrow-<idx>` up front
  during `render()`, regardless of whether that row is currently expanded — the `.open`
  class only controls visibility via CSS. This means `.move-san-editable` spans (and
  anything else scoped by game) exist in the DOM for *every* game simultaneously, both the
  hidden desktop copies and the currently-visible mobile sheet copy — don't assume a
  page-wide `document.querySelectorAll('.some-per-game-thing')` is scoped to "the one
  currently visible"; scope queries to `#pgnrow-<idx>` or `.sheet-moves` explicitly, or
  you'll silently grab a hidden desktop copy instead of the visible mobile one (bit a test
  script during comment-editing development).
- **Full-screen bottom sheet** (`#gameSheet` + `#sheetScrim`) on tapping a card: reuses the
  existing Lichess embed + `formatMovesWithComments` panel rather than a hand-rolled board.
  Layout is a **fixed, non-scrolling embed section on top + an independently scrolling move
  list below** (`.sheet-embed-sticky` / `.sheet-moves`, both children of a non-scrolling flex
  `.sheet-body`) — an earlier version had the embed scroll away with the move list, making it
  impossible to check the board while reading long annotations.
- Closing the sheet pushes/pops a `history.pushState` entry so the Android/mobile back
  gesture closes the sheet instead of leaving the page; Escape and the scrim also close it.

### Bugs found during real-device review (fixed, useful lessons — read before touching mobile CSS)

- **Missing `<meta name="viewport">` tag** — root cause the first time the user reported
  "staging looks identical to desktop" on a real phone. Without it, mobile browsers render
  at a fake ~980px "virtual viewport" and zoom out, so `@media (max-width:700px)` never
  matches on a real device even though every desktop-machine Playwright test with a
  manually-set narrow viewport "passed." **Check for this tag first on any future "mobile
  styles aren't applying" report.**
- **`.mcard` was missing `box-sizing:border-box`** — its `width:100%` + own padding/border
  defaulted to `content-box`, pushing the card wider than its container. Any element using
  `width:100%` alongside its own padding/border needs an explicit `box-sizing:border-box`.
- **Class name collision**: the mobile card's `<div class="players">` picked up an unrelated
  desktop rule, `.players{white-space:nowrap}` (meant for the *table's* Players *column*),
  purely because both share the literal class name. Reusing a generic class name across the
  desktop and mobile layouts is a real footgun — check for collisions.
- **Scroll-jumps-to-bottom on file upload**: hiding `#stepUpload` (containing the
  just-clicked, currently-*focused* `#continueBtn`) while revealing taller content beneath it
  is a real, reproducible browser quirk (confirmed under Chromium mobile emulation) — jumps
  scroll to the very bottom. Fixed with `document.activeElement.blur()` before hiding, and
  explicit `window.scrollTo(0,0)` at all landing→app transition points. Don't rely on the
  browser "doing the right thing" here — force it.
- **Dropdown panels anchored to their trigger button** (Settings, Columns) could overflow
  the viewport edge on narrow screens. Fixed via `positionPanelMobile(panel, trigger)`,
  called only under `window.matchMedia("(max-width:700px)")`, switching the panel to
  `position:fixed` with `top` computed from the trigger's `getBoundingClientRect()` and
  `left/right: 1rem` pinned to the viewport. (Save My Library and game-edit were converted to
  true centered modals instead, sidestepping this class of bug entirely.)

## Custom domain migration — complete

Fully live at `library.chessscenes.com` (migrated from `harman28.github.io/chess-library/`):
DNS resolves, dedicated Let's Encrypt cert issued and verified, `https_enforced: true`,
HTTP→HTTPS redirect confirmed. No outstanding work here. `chess-library-api`'s
`ALLOWED_ORIGIN` includes both the old and new origins (no urgency to drop the old one).

## Incidents worth remembering

**An unexpected background push to production.** Mid-session, a stray task-notification
referenced a background agent the current session had not spawned, claiming a feature had
been "pushed and deploying now." A real, unreviewed commit had landed directly on
`index.html`, bundling that feature together with in-progress work from *this* session —
almost certainly because both processes were editing the same underlying
`chess-analysis/library.html` concurrently, and the other process's "copy to prod and push"
swept up whatever was in the shared file at the time. First response was to revert the
entire commit — **itself a mistake**, since the bundled feature turned out to be something
the user had separately, legitimately asked for elsewhere. The fix was to isolate just the
unreviewed part, re-apply the wanted part alone, and ship them separately. **Lessons**: (1)
an unexpected prod push is worth reverting immediately as a safety default, but (2) don't
assume the whole change was unwanted just because it wasn't requested in *this*
conversation — confirm before treating a revert as final, and (3) be aware other concurrent
processes may be editing the same shared working file — diff the live site against your own
mental model before assuming you know what's on prod.

**The environment's own push-safety classifier caught a real lapse.** A `git push` that
bundled a brand-new, never-shown-to-the-user OG-image design and tagline copy straight to
`index.html` (production) was auto-blocked, correctly, before it executed — nothing was
actually pushed despite the commit/push commands having been issued. The right move (taken
after the block): show the user the actual rendered image and the exact proposed copy, get
explicit sign-off, *then* push — don't treat "this seems obviously fine and low-risk" as a
substitute for that, even for something as small as a tagline.

## Deployment safety

Real users are actively using the live site. Standing agreement: test locally first (syntax
+ functional + visual, using real mobile emulation for anything mobile-facing), stage in
`dev/` for anything substantial (a redesign *or* new copy/visual content), get explicit
approval, then promote to `index.html`. Don't push half-verified changes, and don't push
unreviewed content/design straight to prod even if it feels safe. Don't babysit GitHub Pages
deploys — a `gh run list` check is enough, no retry-looping.

## Things the user has explicitly pushed back on (don't repeat)

- Don't ship copy/content/visual-design decisions (button labels, demo content, a favicon,
  an OG-image tagline) without review first, even if the underlying code is well-tested or
  the change feels small — happened repeatedly across the project's history.
- Don't add a media-query "hack" when a structural CSS fix (e.g., flex order) is possible.
- Don't over-invest in verifying things the user can trivially check themselves.
- Don't assume a resized-viewport Playwright test is equivalent to a real mobile device — it
  misses virtual-viewport behavior entirely (see the viewport meta tag bug above).
- For anything with real stakes (e.g. a one-time bulk cleanup of the user's actual game
  data), don't rely on a single verification pass — independently re-derive and compare the
  specific things that must not change (move sequence, comment text) using a *different*
  code path than the one that did the editing, so a bug in the editor and a matching bug in
  its own self-check can't both be wrong the same way.
