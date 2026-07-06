# Chess Library

A client-side, single-file chess PGN viewer/library for personal OTB (over-the-board)
games. No build step, no framework — one `index.html` with inline `<style>`/`<script>`.
Hosted on GitHub Pages, currently migrating to a custom domain.

## Development workflow (important — follow this, don't edit index.html directly)

The actual working copy is **`/Users/harman.singh/workspace/chess-analysis/library.html`**,
not this repo's `index.html`. Workflow for any change:

1. Edit `chess-analysis/library.html`.
2. Sanity check: `node -e '...new Function(script)...'` (extract the `<script>` block and
   confirm it parses) — see prior session transcripts for the exact one-liner.
3. For anything behavioral, write a quick Node test using a minimal DOM stub (see below)
   or a real Playwright screenshot for visual changes. This project has been tested this
   way all along — don't skip it just because it's a "small" change.
4. Copy to this repo: `cp chess-analysis/library.html chess-library/index.html`.
5. Syntax-check the copy too, then `git add index.html && git commit && git push`.
6. GitHub Pages auto-deploys via `.github/workflows/pages.yml` (Actions-based deploy,
   not the legacy Jekyll pipeline — that was flaky, switched deliberately). Push usually
   deploys in under a minute; check `gh run list --repo harman28/chess-library --limit 1`
   if in doubt, but don't obsessively poll — the user has explicitly asked not to babysit
   deployments.

Node test harness pattern used throughout: stub `document.getElementById`, `localStorage`,
etc., concatenate test code onto the extracted `<script>` source, and `eval()` the whole
thing as one string (top-level `let`/`const` inside a bare `eval()` call don't leak out
to separate `eval()` calls in Node — this bit us repeatedly early on, so always build one
combined script string rather than multiple `eval()` calls).

## Architecture / key concepts

- **All state is client-side.** `games` array + `playerName` persisted to `localStorage`
  under `STORAGE_KEY`, gated by `SCHEMA_VERSION` — bump this constant whenever the shape
  of a game record changes, so stale cached sessions get discarded instead of silently
  causing bugs (this happened once already: added a field, forgot to bump, filters broke
  because old cached games lacked the new field).
- **Columns are fully dynamic.** `discoverColumns()` inspects whatever PGN header tags are
  actually present across loaded games and builds the column list from that (core columns:
  date/event/players/white/black/result/wl/moves, plus any custom tag as `hdr_<Name>`).
  `columnState.order` + `columnState.visible` control display; users can show/hide/reorder
  via the Columns panel (floating icon over the table's top-right corner).
- **Sorting is always by PGN Date, descending**, re-applied on every load/add/restore
  (`sortGamesByDate()`). `idx` is reassigned sequentially after each sort — it is *not* a
  stable id across sorts, it's just a display/DOM-tracking number. Row `#` column shows
  position within the *currently filtered* view, not the absolute idx.
- **Toolbar layout** uses `flex-direction: row-reverse` deliberately (not a media query)
  so that wrap order falls out structurally: buttons group is first in DOM (appears
  rightmost when unwrapped, topmost when wrapped), then search, then the filter-group —
  giving "buttons → search → filters" top-to-bottom stacking on narrow viewports while
  keeping the original side-by-side look on wide ones. Don't replace this with a
  `max-width` media-query override; that was tried and explicitly rejected as "flimsy."
- **Lichess embed** (click a row → board on the right, formatted PGN+comments on the
  left): imports the game via Lichess's public `POST /api/import` (open CORS, no auth),
  then embeds `lichess.org/embed/game/<id>?theme=blue2&pieceSet=caliente&bg=dark`, cropped
  via a fixed-height `overflow:hidden` wrapper to hide the move-list panel (measured pixel
  geometry at 360px width — Lichess has no query param for this). **Known limitation,
  confirmed by testing, not a bug to "fix"**: Lichess's import strips custom PGN comments
  and replaces them with its own auto-generated notes, and there is no query param or
  alternate route (checked both `/embed/game` and `/analysis`'s own Import-PGN feature)
  that preserves them or sets board orientation. This is why the formatted-comments panel
  exists client-side as the source of truth for annotations, and why the **"Analyse"
  button intentionally still opens chess.com, not Lichess** — chess.com's paste-to-analyze
  apparently preserves comments where both Lichess paths don't. Don't "fix" this to use
  Lichess without re-confirming that constraint still holds.
- **Save My Library**: calls the `chess-library-api` backend (separate repo/service, see
  its own CLAUDE.md) to store the current PGN under a random unlisted id, returns a
  `?lib=<id>` link. Loading that link fetches and loads automatically, then strips the
  query string via `history.replaceState`. No accounts — possession of the link is the
  only access control, same trust model as an unlisted Lichess game link.
- **Demo data** (`DEMO_PGN` constant): four public-domain historical games (Opera Game,
  Immortal Game, Evergreen Game, Fool's Mate), verified move-by-move against Wikipedia
  before shipping. If you ever add more demo content, verify against a real source first —
  the user explicitly does not trust generated chess content without independent checking.

## Custom domain migration (in progress as of this writing)

Moving from `harman28.github.io/chess-library/` to `library.chessscenes.com`:
- `CNAME` file committed to this repo's root (required for GitHub Pages custom domain).
- Namecheap DNS: CNAME record `library` → `harman28.github.io` (added by user, confirmed
  resolving via `dig`).
- GitHub Pages custom domain configured via `gh api repos/harman28/chess-library/pages -X
  PUT -f cname=library.chessscenes.com`.
- **Status at end of last session**: DNS resolves correctly, HTTP/HTTPS both return 200,
  but the HTTPS cert being served was still GitHub's generic `*.github.io` wildcard cert
  (confirmed via `openssl s_client` — SAN list didn't include `library.chessscenes.com`),
  meaning GitHub hadn't finished provisioning a dedicated Let's Encrypt cert yet.
  `https_enforced` was still `false`. **Before enabling "Enforce HTTPS"**, re-check the
  cert: `echo | openssl s_client -connect library.chessscenes.com:443 -servername
  library.chessscenes.com 2>/dev/null | openssl x509 -noout -ext subjectAltName` — it
  should list `library.chessscenes.com` specifically before enforcement is safe to turn on.
- The `chess-library-api` backend's `ALLOWED_ORIGIN` env var was updated (by the user, on
  Railway) to a comma-separated list including both the old and new origins during this
  transition — see that repo's CLAUDE.md.

## Deployment safety

Real users (the user's friends) are actively using the live site. Standing agreement:
test locally first (syntax + functional + visual as appropriate), then push — no separate
review/staging branch needed, but don't push half-verified changes. The user has also
explicitly said not to babysit/retry-loop GitHub Pages deployments if a push shows a
transient failure — it's usually a transient backend hiccup on GitHub's end, not our code.

## Things the user has explicitly pushed back on (don't repeat)

- Don't ship copy/content decisions (e.g., button labels, demo content) without review
  first — this happened twice: once with a "Copy & Analyse" button label change, once
  almost with demo game content. Ask or propose before pushing subjective/content changes,
  even if the underlying code is well-tested.
- Don't add a media-query "hack" when a structural CSS fix (e.g., flex order) is possible.
- Don't over-invest in verifying things the user can trivially check themselves (e.g., was
  told directly to stop scripting browser automation to verify a simple external site
  behavior — just reason it through and let the user do a 10-second manual check).
