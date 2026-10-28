# Etengram

A self-hosted social platform with server-enforced privacy, real-time chat, and moderation. This is a full rewrite: authentication, authorization, posts, follows, messages, notifications, uploads, and moderation all live on a Node.js server backed by SQLite. The browser never receives password hashes or credentials it can use to bypass privacy rules.

## Features

- **Accounts** — registration, login, logout, profile editing, avatar emoji or image upload, cover image, website link, password change, account deletion, and password reset through a single-use email link.
- **Private accounts** — new followers must be approved. Only the owner and accepted followers can see follower-only posts.
- **Posts** — text plus up to four images/videos in a carousel, public or followers-only visibility, hashtags, mentions, inline editing with an “edited” marker, reactions, comments and threaded replies, reposts, bookmarks, sharing, delete, and reporting.
- **Feeds** — Following and For You, cursor-based pagination with de-duplication, skeletons and end-of-feed states, and a transparent ranking formula (recency, engagement, affinity, two posts per creator per page) that can be switched off for chronological order. No machine learning is used.
- **Publishing** — text and up to four photos or videos, public or followers-only visibility, a posting progress state, a duplicate-submit guard, inline success confirmation, and failure feedback that keeps the draft and offers retry.
- **Discovery** — Explore with trending hashtags, public post search, hashtag topics, and people search with follow suggestions.
- **Social graph** — follow/unfollow, follow requests, followers/following lists, mute, restrict, blocks that hide both directions.
- **Profile** — edit display name, bio, website, avatar, and cover in Settings; your own profile also has a camera button on the avatar for a direct picture change (JPEG/PNG/GIF/WebP, 4 MB limit), with upload progress and immediate header refresh.
- **Stories** — photo or video stories that disappear 24 hours after posting. A story can hold up to 10 slides; the viewer steps through them in order with per-slide progress, tap/hold pause, keyboard navigation, per-story view tracking, 24-hour server-side expiry (rows and slide uploads are purged), and owner delete. The composer supports reordering slides, removing a slide, retrying one failed slide without re-uploading the others, cropping/zooming/rotating photos (fit or fill with a background color), text layers (size, font, alignment, color, highlight, drag to position), and a searchable sticker tray (mention, hashtag, date/time, and http/https links work; location, poll, question, and music are shown disabled with a reason). Each story also stores an audience: `public` (visible wherever the author's profile is visible) or `followers` (accepted followers only). Private accounts always publish followers-only and cannot select Everyone; public accounts default to Everyone and may switch. The chosen audience is enforced on the server for the tray, profile story list, view endpoint, mention notifications, and every slide file. Mentions in captions, text layers, and mention stickers notify the mentioned account once after publishing, only if that account is allowed to see the story (account privacy, story audience, block, suspension). Private accounts expose stories only to accepted followers, and media is served through the authenticated media route.
- **Story drafts** — save an unpublished story as a private draft, edit its caption or audience later, resume it from the tray, publish it, or discard it. Drafts keep every slide, its order, its overlays, and its audience (`story_drafts` plus `story_draft_slides`, up to 20 drafts and 10 slides each), appear only to their author, and never enter anyone's tray or feed until published. Draft uploads stay `pending` in `media_assets`, so the existing media gate serves them to the uploader alone; publishing converts the same files and rows into a story inside one transaction, and discarding a draft or slide, deleting the story, or deleting the account removes the files. Drafts have no expiry — they last until published or discarded.
- **Messaging** — one-to-one conversations, unread counts, image attachments, live delivery over Server-Sent Events, report message. Any signed-in user can open a chat with a public account; a private account can only be messaged once it follows you (existing conversations stay open). Long-press, right-click, or the actions button on any message opens a menu to copy text, report, or delete; deletion is sender-only, enforced on the server, and removes the attachment.
- **Notifications** — likes, comments, reposts, mentions, follows, follow requests, and moderation outcomes, with live badges, per-item delete, and a separate Messages badge for chats. Clicking anywhere on a notification opens the related post or profile; avatar and action buttons keep their own targets.
- **Moderation** — report queue, dismiss/remove content/suspend account with a required resolution note, user search, suspend/restore.
- **UI** — responsive React app with dark/light themes, keyboard-friendly modals, focus states, and reduced-motion support.

## Stack

| Layer | Technology |
| --- | --- |
| Server | Node.js (ESM), Express 4, Helmet, Multer, Nodemailer |
| Database | SQLite via built-in `node:sqlite` (WAL, foreign keys, versioned migrations) |
| Client | React 18, React Router 6, Vite 5 |
| Realtime | Server-Sent Events (`/api/events`) |
| Auth | scrypt password hashes, SHA-256 session and reset-token hashes, httpOnly cookies |

Node **22.9 or newer** is required because the server uses `node:sqlite` and `--env-file-if-exists`.

## Quick start

```bash
npm install
cp .env.example .env        # Windows: copy .env.example .env
```

`npm start`, `npm run dev:server`, and `npm run seed` load `.env` automatically when the file exists. Edit it and set an administrator (optional but recommended):

```env
PORT=4000
APP_ORIGIN=http://localhost:4000
ADMIN_EMAIL=admin@example.com
ADMIN_PASSWORD=use-a-long-unique-password
ADMIN_USERNAME=admin
```

Add SMTP settings to enable password reset email:

```env
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=mailer@example.com
SMTP_PASS=use-a-secret-from-your-provider
SMTP_FROM=Etengram <no-reply@example.com>
```

`SMTP_URL` may be used instead of the individual fields. SMTP credentials stay on the server and are never sent to the browser.

Build the client and start the server:

```bash
npm run build
npm start
```

Open http://localhost:4000. The same process serves the API and the built client.

### Development

Run the API and Vite dev server separately:

```bash
npm run dev:server   # nodemon-style watch on port 4000
npm run dev:client   # Vite on port 5173, proxies /api and /media
```

Then open http://localhost:5173.

### Demo data

```bash
SEED_DEMO=true SEED_DEMO_PASSWORD='Demo1234!' npm run seed
```

Demo seeding is opt-in; never enable it in production with a known password.

## Environment variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `PORT` | `4000` | HTTP port |
| `APP_ORIGIN` | request origin | Allowed `Origin` for state-changing requests |
| `COOKIE_SECURE` | `false` | Set `true` behind HTTPS so session cookies require TLS |
| `TRUST_PROXY` | `false` | Set `true` only behind a trusted reverse proxy |
| `DATABASE_PATH` | `data/etengram.db` | SQLite file location |
| `UPLOADS_PATH` | `uploads/` | Upload directory; point it at persistent storage in production |
| `ADMIN_EMAIL` | — | Admin account email; triggers create/promote on startup |
| `ADMIN_PASSWORD` | — | Initial admin password (minimum 8 characters) |
| `ADMIN_USERNAME` | `admin` | Username used when creating the admin |
| `SEED_DEMO` | `false` | Enable demo data when running `npm run seed` |
| `SEED_DEMO_PASSWORD` | `Demo1234!` | Password for seeded demo users |
| `SMTP_URL` | — | Full SMTP connection string; takes precedence over the `SMTP_*` fields |
| `SMTP_HOST` | — | SMTP server host |
| `SMTP_PORT` | `587` | SMTP port (`465` enables implicit TLS) |
| `SMTP_SECURE` | `false` | Set `true` for implicit TLS |
| `SMTP_USER` | — | SMTP username |
| `SMTP_PASS` | — | SMTP password or app password; keep it server-side |
| `SMTP_FROM` | — | From address, e.g. `Etengram <no-reply@example.com>` |

Without SMTP configuration, password reset requests still return a generic response, but no email is sent and the server logs a warning. The reset link uses `APP_ORIGIN`, so set it to the public site URL in production.

## Scripts

| Command | What it does |
| --- | --- |
| `npm run dev:server` | Start the API with `node --watch` |
| `npm run dev:client` | Start Vite with API/media proxy |
| `npm run build` | Build the client into `dist/` |
| `npm start` | Serve the API and `dist/` |
| `npm run seed` | Create opt-in demo accounts and posts |
| `npm test` | Run the API smoke suite (server must already be running) |

## API smoke suite

Start the server, then in another terminal:

```bash
node server/tests/smoke.mjs
# or against another host:
BASE_URL=https://your-host.example node server/tests/smoke.mjs
```

The suite creates throwaway users and runs 170 checks: registration and auth, generic password-reset responses, unknown-token and weak-password rejection, anonymous access denial, text and multi-image posts, MIME/magic-byte rejection, follow requests, duplicate-follow and self-follow rejection, private visibility, reactions and comments, bookmarks and reposts, notification delivery, feed pagination and personalization toggles, conversations and unread counts, conversation detail authorization, message deletion authorization (non-member and recipient rejection, sender success, repeat-delete 404), reports and admin resolution, blocks, mute, restrict, password change and session rotation, suspension invalidation, stories (creation, media validation, follower-only visibility, authenticated media access, idempotent views, non-owner delete rejection, owner delete and tray cleanup), story audiences (followers-only and public round-trips through drafts and publish, invalid audience rejection, non-follower suppression on profile/view/media, follower access, private-account default), story drafts (author-only list/edit/publish/discard, non-owner 404 on every route, draft media denied to followers, unpublished drafts absent from feeds, publish converts the same media into a story, repeat publish 404), multi-slide drafts (adding slides, slide order, overlay round-trip and unsafe-link rejection, non-owner slide 404s, publish keeps order and overlays, per-slide file cleanup, slide removal and draft discard delete only their own files), story mentions (no notification before publish, exactly one notification for an accepted follower, privacy and block suppression, notification after unblock), and account deletion.

The fixture accounts sign in and delete themselves at the end of the run, including their posts, messages, stories, drafts, and uploads. Point `BASE_URL` at a disposable instance anyway, since rate limits and any interrupted partial runs can leave data behind. Do not run the suite twice inside the same minute against one process: the API allows 240 requests per minute per IP and a second run will be throttled. The suite does not read email; verifying that a reset link arrives requires SMTP configuration and a mailbox.

## Database migrations

Schema changes ship as numbered migrations in `server/migrations.js`. They run automatically at startup inside a transaction, and applied versions are recorded in `schema_migrations`. Back up `data/etengram.db` before deploying a version that adds migrations.

## Security model

- Passwords are hashed with scrypt (N=16384, r=8, p=1) using a random 16-byte salt and compared in constant time. No endpoint returns a password or password hash.
- Session tokens are 32 random bytes; only their SHA-256 hash is stored. Cookies are `httpOnly`, `SameSite=Strict`, and `Secure` when `COOKIE_SECURE=true`.
- Changing a password destroys every existing session, then issues a fresh one to the current browser.
- Password reset tokens are 32 random bytes; only their SHA-256 hash is stored. They expire after 60 minutes, work once, and a successful reset destroys every session for that account. Reset requests return the same message whether or not an account exists, and never return the token in the response.
- Suspended accounts fail session lookup immediately.
- Every privacy decision (feed, profile posts, conversation membership, block relationships, story audience, admin actions, comment ownership) is enforced in SQL on the server, not in the client.
- State-changing requests check `Origin` against `APP_ORIGIN`; IP rate limits cover auth, password reset requests, and API traffic.
- Uploads are limited to JPEG/PNG/GIF/WebP/MP4/WebM, capped at 8 MB for images and 50 MB for video, renamed with random UUIDs, and checked against magic bytes before being kept.
- `/media/<uuid>` requires a signed-in session and applies the asset's visibility rules (post visibility, private-account follows, blocks, suspended authors). Files are not public URLs.
- Helmet sets a strict Content Security Policy, `nosniff`, frame denial, and `same-origin` resource policy.

### Known limitations

- For You uses a fixed, explainable formula. It is not machine learning, and no AI-generated content is published without review because no AI posting feature exists.
- Email is only used for password resets. There is no email verification or other transactional mail. If SMTP delivery fails, the server logs the error but does not retry or queue it; the browser still receives the generic response.
- Rate limits are in-memory and per process; use a shared store when running multiple instances.
- SSE and SQLite target a single Node instance. There is no clustering or job queue.
- No transcoding: uploaded video must already be browser-playable.
- Stories are hidden at read time once expired and purged at startup and hourly; the tray is unpaginated (all active authors are returned in one request).
- Story audiences are limited to `public` and `followers`. Close friends and custom-audience lists are shown disabled in the composer because no list management or audience table exists yet. Migration 7 backfills existing stories and drafts from private accounts to `followers`; stories from accounts that are public today stay `public`, matching the behavior they already had. The audience is evaluated at read time together with current account privacy, so switching an account to private still hides all of its stories from non-followers.
- Story drafts never expire and are not purged; only the 20-per-account cap limits storage. A draft's media cannot be replaced — swap it by discarding the draft and starting a new one. Cropping is offered only for photos that have not been uploaded yet (once a slide is saved to a draft, its file is fixed), and videos cannot be cropped or rotated in the browser.
- Location, poll, question, and music stickers have no backend provider, so they are shown disabled with an explanation instead of pretending to work. Mention, hashtag, date/time, and link stickers do work.
- Story mention notifications are one per story and mentioned account; there are no per-notification-type preferences yet. Suspended accounts, blocked accounts, and accounts that cannot view the story are skipped.
- Overlays are stored as JSON on each slide (`story_slides.overlays`, `story_draft_slides.overlays`). Rows created before this feature, or posted through the single-slide API without overlays, keep an empty array.
- The password reset flow does not reveal whether an email is registered. This is intentional; support workflows must not add an “email not found” response.

## Deployment

This app needs a **long-running Node process** and a **persistent disk** for SQLite and `uploads/`.

Suitable targets: Render, Railway, Fly.io, a VPS, or any container host with a volume.

1. Set `NODE_ENV=production`, `APP_ORIGIN=https://your-domain`, and `COOKIE_SECURE=true`.
2. Mount persistent storage for SQLite and uploads, then set `DATABASE_PATH` and `UPLOADS_PATH` to paths on that mount. Both directories are created automatically.
3. Run `npm ci && npm run build`, then `npm start`.
4. Put HTTPS in front of it. If the proxy terminates TLS, set `TRUST_PROXY=true` only when that proxy is trusted.
5. Set `ADMIN_EMAIL`, `ADMIN_PASSWORD`, and `ADMIN_USERNAME` for the first boot, then keep the password secret.
6. Set the `SMTP_*` variables (or `SMTP_URL`) so password reset links can be delivered. Use an app password or provider-specific credential, never a personal mailbox password.

### Render

`render.yaml` describes a single Node web service with a 1 GB disk mounted at `/var/data`:

- `DATABASE_PATH=/var/data/etengram.db`
- `UPLOADS_PATH=/var/data/uploads`
- `COOKIE_SECURE=true` and `TRUST_PROXY=true`
- health check at `/api/health`

Steps:

1. Push this folder to a GitHub or GitLab repository. `.env`, `data/`, `uploads/`, `dist/`, and `node_modules/` are ignored, so no secrets or local data are pushed.
2. In Render, choose **New → Blueprint**, select the repository, and apply `render.yaml`.
3. When prompted, enter `APP_ORIGIN` (the service URL, e.g. `https://etengram.onrender.com`), `ADMIN_EMAIL`, and a strong, unique `ADMIN_PASSWORD`. Do not reuse a demo password.
4. Add `SMTP_URL` and `SMTP_FROM` (or the individual `SMTP_*` values) if password reset email is required.
5. The first boot creates the admin account and runs migrations automatically. Redeploys keep the same disk, so accounts, posts, and uploads survive.

Render requires a paid instance type (Starter or above) to attach a disk. Without a disk the service still runs, but the database and uploads are lost on every deploy. The blueprint sets Node `22.14.0`; the app requires Node 22.9 or newer. Source changes deploy automatically from the connected branch.

### Netlify

Netlify can host the **static client only**. It cannot run this server as-is:

- SQLite needs a writable, persistent filesystem; Netlify Functions are ephemeral and serverless.
- SSE (`/api/events`) is not a good fit for function timeouts and connection limits.
- Uploads need persistent object storage, not a function filesystem.

To use Netlify for the frontend, deploy `dist/` there and proxy API/media requests to a separate Node host:

```toml
# netlify.toml
[build]
  command = "npm run build"
  publish = "dist"

[[redirects]]
  from = "/api/*"
  to = "https://your-api-host.example/api/:splat"
  status = 200
  force = true

[[redirects]]
  from = "/media/*"
  to = "https://your-api-host.example/media/:splat"
  status = 200
  force = true

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

Set `APP_ORIGIN=https://your-netlify-site.example` and `COOKIE_SECURE=true` on the Node host, and use HTTPS on both sides. Cookies are `SameSite=Strict`, so the site and API must be same-site for authenticated requests; proxy redirects keep the browser on one origin. Running the whole platform on Netlify would require replacing SQLite, uploads, and SSE with Netlify-compatible services.
