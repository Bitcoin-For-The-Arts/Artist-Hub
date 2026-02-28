## Bitcoin for the Arts — Artist Hub

The Artist Hub is a **SvelteKit + TypeScript** single-page app, deployed at **`/artist-hub`** on the main site (see [bitcoinforthearts.org/artist-hub](https://bitcoinforthearts.org/artist-hub)). It’s **Bitcoin-native** and **Nostr-hosted**:

- **Auth**: NIP-07 browser signers (Alby / nos2x) — no email/password
- **Profiles**: `kind:0` metadata (+ optional `skills/hashtags/portfolio` extensions)
- **Listings**:
  - **NIP-15** marketplace: stalls (`kind:30017`) + products/services (`kind:30018`)
  - **NIP-99** classifieds: one-offs/collab requests (`kind:30402`)
- **Messaging**: encrypted DMs via **NIP-04** (`kind:4`)
- **Payments**: WebLN invoices + **NIP-57** zaps (when receiver supports zaps)
- **Curation**: “Featured Artists” via **NIP-51** curated sets (`kind:30004`)
- **Follow packs**: following.space-style packs (`kind:39089`) + pack tools
- **Pulse**: live posts feed (notes + zaps) with media previews
- **Notifications**: in-app notifications feed
- **Live**: NIP-28 public channels (kinds `40/41/42`)
- **Streams**: NIP-53 streams browsing
- **Virtual Studios**: BFTA extension `kind:30050` (parameterized replaceable + `d` tag)
- **Zap Challenges**: BFTA extension `kind:30051` (+ leaderboard derived from zap receipts)
- **Events**: NIP-99 event listings tagged with `#event` (plus optional `start/end/location/url` tags)

All user-generated content is published as signed Nostr events to public relays. There is **no backend API** and **no server-side storage** required to run the app.

**Policy**: Bitcoin for the Arts is **Bitcoin-only**. This project intentionally does **not** support ordinals or NFTs.

## Routes (relative to `BASE_PATH`)

By default, the app is served under `/artist-hub` (configurable via `BASE_PATH`; see below). Common routes:

- `/`: hub home
- `/discover`: search + category/tag filters across relays
- `/profile/[npub]`: profile view + their listings
- `/me`: publish/edit your profile + see your content
- `/create`: publish a listing (NIP-15 or NIP-99)
- `/listing/[id]`: listing detail, comments, zap/pay, message artist
- `/messages`: decrypt + read/send NIP-04 DMs
- `/notifications`: notifications feed
- `/pulse`: posts feed
- `/streams`: streams directory
- `/live`: NIP-28 channels + live chat
- `/artstack`: forum-style feed
- `/packs`: follow packs directory
- `/packs/create`: create/publish a follow pack
- `/d/[id]`: static follow-pack viewer (shareable link target)
- `/featured`: admin-curated featured artists list
- `/studios`: browse/publish virtual studios
- `/studios/[naddr]`: view a studio (guestbook + zaps + optional channel chat)
- `/events`: browse/publish event listings
- `/challenges`: browse/publish Zap Challenges
- `/challenges/[naddr]`: view a challenge + leaderboard

## Configuration

- **`BASE_PATH`**: URL base path for the SvelteKit app. Defaults to `/artist-hub` (see `svelte.config.js`).
- **`PUBLIC_BFTA_RELAYS`**: comma-separated relay list override (defaults in `src/lib/nostr/constants.ts`). Relay URLs are also cached client-side in `localStorage` (`bfta:artist-hub:relays`).
- **`PUBLIC_BFTA_ADMIN_NPUB`**: enables publishing “Featured Artists” curated set from `/featured` (writes `kind:30004` with `d=bfta-featured-artists`).
- **`PUBLIC_BFTA_FOLLOW_PACK_D`**: default follow-pack `d` tag for `/packs` (optional: **`PUBLIC_BFTA_FOLLOW_PACK_AUTHOR`** to pin the author pubkey).
- **`PUBLIC_SENTRY_DSN`** / **`PUBLIC_SENTRY_ENV`**: optional client error reporting via Sentry.

## Local development

Run locally (default base path is `/artist-hub`):

```bash
npm install
npm run dev
```

Then visit `/artist-hub` on the Vite dev server.

To run at the root path for convenience:

```bash
BASE_PATH="" npm run dev
```

## Build & deployment

- `npm run build` outputs a static SPA to `build/` using `@sveltejs/adapter-static` with an `index.html` fallback.
- `vercel.json` is configured to mount and SPA-rewrite the app under `/artist-hub` on Vercel (or any equivalent host with rewrites).

## Recent upstream commits

These are the most recent commits on `main` (kept here so the README reflects what’s currently deployed):

- `15f14c8` fix: messages inbox - show conversation inline on mobile
- `e4cf1fa` fix: disable profile hover tooltip on touch/mobile devices
- `ecd65a4` fix: rewrite header as two-row layout, fix scroll-under issue
- `d8d7125` feat: add Close button to notifications page
- `5aed7d0` feat: Packs page - persist last pack, show My Packs, auto-load following
- `236a3c4` fix: desktop header layout - prevent logout button from going off-screen
- `793f8e5` fix: followers tab crash - reduce query limits and throttle rendering
- `468ccb3` fix: rewrite relay connection - remove probing, connect directly
- `e96147a` perf: optimize d/[id] follow pack page load
- `7765d12` fix: relay connection - env var now takes precedence over localStorage
- `c7da47b` feat: integrate Sentry error monitoring into artist-hub
- `a4e80e1` perf: add MongoDB indexes and image error handling
- `263fe0b` perf: optimize notifications and add profile caching
- `fb4f5ff` fix: optimize profile pages and add error boundary
- `e6d945b` fix: add concurrency limits to relay queries to prevent browser crash
- `50d941c` Artist Hub: harden deadline collector against subscribe failures
- `698027a` Artist Hub: responsive npub modal, wider Pulse desktop, wolf pack icon
- `c42c1ec` Artist Hub: make follower list loading non-blocking and cancelable
- `9039f94` Artist Hub: hide emoji picker UI on mobile
- `5249543` Artist Hub: mobile layout fixes, avatar, tooltip close, stream timeouts

For the full history, use GitHub’s commit view or `git log`.

