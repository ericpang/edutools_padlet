# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Run / develop

Generate the config once (and again whenever `.env` changes), then serve the directory:

```
node scripts/generate-config.mjs
python -m http.server 8000 --bind 127.0.0.1      # any static server works
```

The app runs at `http://localhost:8000`. ES modules are loaded directly from `js/main.js`; Firebase comes from the `gstatic.com` CDN at version `12.12.1` — keep that version consistent across all `js/*.js` import URLs.

There is no bundler, no tests, no linter, no package manager. The only tooling is the dependency-free config script. Editing a file and reloading the page is the dev loop.

## Configuration

- `.env` (gitignored) holds the Firebase **web** config as `FIREBASE_*` variables. `.env.example` is the committed template.
- `scripts/generate-config.mjs` turns `.env` into `js/config.js` (gitignored, generated — never edit it by hand, it is overwritten). Real environment variables win over `.env`, which is how the GitHub Pages workflow supplies values from repository secrets. `js/firebase.js` imports `FIREBASE_CONFIG` from it.
- Adding a config key means updating **all four**: `.env.example`, the `REQUIRED`/`OPTIONAL` maps in `scripts/generate-config.mjs`, the `env:` block in `.github/workflows/deploy-pages.yml`, and the table in `README.md`.
- If `js/config.js` is missing the module graph fails to load and `index.html`'s `showStartupHelp()` shows a setup hint (a fresh clone needs the generator run).
- The dev server serves the whole folder, so `.env` must only ever contain Firebase web config (public by design) — never server-side secrets.
- The Firestore security rules live in `firestore.rules` (the single source of truth; the README links to it rather than copying it). They must stay in sync with the data model below, so update both together. Publish with `npx firebase-tools deploy --only firestore:rules --project <id>`.
- `firebase.json` deliberately lists **only** `firestore`. The Firebase project is shared with other apps that use Realtime Database (`database.rules.json`, if present, belongs to them and is not used here), so never add a `database` entry: a plain `firebase deploy` would overwrite their live rules.

## Architecture

Single-page collaborative board with multiple classrooms ("rooms"). The data model lives in Firestore; the UI is a thin DOM layer on top of real-time subscriptions.

**Module layout** (`js/`):
- `firebase.js` — single source of `auth`, `db`, `storage` singletons. Every other module imports from here, never from the SDK URLs directly. (`storage` is exported but nothing uses it; images and PDF pages live in Firestore.)
- `auth.js` — guest (anonymous + nickname) and Google sign-in, `onAuthChange`, `displayNameOf`.
- `rooms.js` — classrooms, memberships, sections, `isTrainer`.
- `posts.js`, `comments.js` — each owns one Firestore concern and exports `subscribeX(cb)` returning an unsubscribe function, plus mutation helpers. These modules never touch the DOM.
- `ui.js` — DOM helpers (`el`, `showModal`, `closeModal`, `toast`, `formatRelativeTime`). No app state.
- `admin.js` — `exportCSV(posts)`. Currently unused (the toolbar button was removed).
- `main.js` — the only module that imports from everything else. Owns the `state` object, wires DOM events, renders, and calls `subscribeX` to drive renders.

**Data model.** Everything except comments is a document in the `posts` collection, told apart by `type`: `room` (id = the 6-character room code), `membership` (`mem_{uid}_{roomId}`), `section`, `post`, `pdf-page` (`parentPostId`). The **trainer** of a room is the `authorId` of its `room` document. `subscribePosts` filters out every non-`post` type client-side (so no composite index is needed) — when you add a new `type`, add it to that filter. Comments are in `comments`, keyed by `postId`.

**Data flow.** `onAuthChange` → `handleAuthChange` → landing page, join prompt, or `enterRoom()`. `enterRoom` subscribes to the room doc, its posts and its sections; the posts/sections subscriptions re-attach themselves with exponential backoff on error. `teardownRoom()` unsubscribes everything. `state.posts` is the single source the board renders from; search and sort happen in-memory in `filterAndSort` — never re-query Firestore for these.

**`renderBoard()` rebuilds the whole board DOM on every snapshot**, so entry animations replay on every change. Keep them subtle, and don't put animations on state-dependent elements (e.g. a liked heart).

**Pin ordering.** `filterAndSort` always partitions pinned posts to the top before applying the user's sort. There is no UI to pin any more, but keep the partition if you add a sort option.

**Detail modal lifecycle.** `openDetail(p)` opens a comment subscription scoped to that post and uses a `MutationObserver` on `#modal-root` to detect when the modal closes (any path — Escape, overlay click, programmatic `closeModal()`) so the subscription is torn down. When `state.posts` updates and the detail modal is open for that post, `refreshDetailModal` re-renders the modal contents in place — keep this in sync if you add fields to the detail view.

**Media.** Images are compressed in the browser to JPEG data URLs and stored in the post document (`imageDataUrls`). PDFs are rendered with PDF.js (lazy-loaded from a CDN) and each page is stored as its own `pdf-page` document. Link previews read OpenGraph tags through public CORS proxies and are cached in `localStorage`.

**Likes** are an array of UIDs (`arrayUnion`/`arrayRemove`). The Firestore rules permit any signed-in user to update **only** the `likes` and `updatedAt` fields — when adding new fields, decide whether they belong in the same "anyone can mutate" path or the stricter author-only path, and update the rules accordingly.

## Theming (`styles.css`)

The stylesheet is a token-driven design system ("sticker book": bold outlines, hard offset shadows, bright fills).

- All colours are CSS variables in `:root`; the dark theme only overrides them in `.dark` (a class on `<html>`, toggled in `main.js`). Don't hardcode colours in components or JS — add a token and its `.dark` value.
- Bright fills (`--sun`, `--coral`, `--mint`, `--sky`, `--grape`, `--tangerine`, `--pink`, `--c1`…`--c7`) always carry `--ink` text, never `--foreground` (which flips light in dark mode). Neutral surfaces use `--edge` outlines and `--hard-shadow`. The palette was checked against WCAG AA; re-check if you change values.
- Board columns colour themselves with `.section-column:nth-of-type(7n+k)` setting `--col` / `--col-soft`.
- Specificity traps: the generic `.modal input[type=…]` / `.modal textarea` rules outrank a lone class, which is why the composer's borderless fields use `.modal .composer input.composer-subject`. `.btn-ghost` is defined after the header rules, hence `.btn-ghost.exit-room-btn`.
- Decorative emoji in `::before`/`::after` use `content: "x" / ""` (empty alt text) after a plain fallback declaration, so screen readers skip them.
- Every infinite animation must stay covered by the `prefers-reduced-motion` block at the end of the file.
- Background confetti is `assets/confetti-light.svg` / `-dark.svg` (plain SVG tiles).

## Conventions

- All Firebase SDK imports use the **modular** API at `https://www.gstatic.com/firebasejs/12.12.1/...`. Don't mix in compat or namespaced imports.
- Modules export named functions only — no default exports, no classes.
- DOM construction uses `el(tag, attrs, ...children)` from `ui.js`. Event handlers go in `attrs` as `onclick: fn` etc. Avoid building HTML strings (use `el` or `escapeHtml` if a string is unavoidable).
- User feedback is via `toast(msg, kind)` — never `alert()` except for destructive `confirm()` prompts.
- Firestore timestamps come back as `Timestamp` objects, not numbers. Use `tsMs()` in `main.js` or `formatRelativeTime()` in `ui.js`; don't do arithmetic on them directly.
- `localStorage` keys (`theme`, `padlet-recent-rooms`, `padlet-link-previews`) keep a historical `padlet-` prefix. Renaming them would reset users' saved lists.
