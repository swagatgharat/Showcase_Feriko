# Feriko – Trip Expense Splitter

[![Production CI/CD Pipeline](https://github.com/swagatgharat/Feriko_Official/actions/workflows/deploy.yml/badge.svg)](https://github.com/swagatgharat/Feriko_Official/actions/workflows/deploy.yml)
[![Node Version](https://img.shields.io/badge/node-%3E%3D22-green.svg)](https://nodejs.org/)
[![Package Manager](https://img.shields.io/badge/pnpm-11.15-orange.svg)](https://pnpm.io/)

Feriko helps groups track shared costs and work out who owes whom. It has two modes:

- **Personal trips** – travel with family or friends: log expenses, split them, settle up, attach shared photo-album links, and pin visited places to a personal travel **Passport**.
- **Corporate ride pools** – colleagues sharing cabs or daily commutes split the costs of each ride pool.

The project is a pnpm monorepo with an Express API, a Next.js web app and an Expo (React Native) mobile app.

---

## Links

- **Live Demo**: [https://feriko.onrender.com/](https://feriko.onrender.com/)

---

## Features

### Accounts and security
- Registration on the **website and in the mobile app**, two ways: **email + password** (6-digit email code; expires after 10 minutes and unverified sign-ups are deleted automatically) or **Continue with Google** (Google-verified email, then a short date-of-birth/Terms step). Date of birth is required and users must be **14 or older**.
- **Strict sign-in methods:** one account per email, and one method per account. A Google account signs in only with Google, a password account only with its password; the other method is refused with a message naming the right one. Google accounts confirm sensitive actions (deleting the account, 2FA, signing out sessions) with a fresh Google sign-in.
- **No Firebase:** the website uses Google Identity Services and the Android app uses `@react-native-google-signin/google-signin`; the backend verifies Google's ID token with `googleapis` and stores everything in MongoDB.
- The **Android app is available to every user**, whatever their plan; the free plan can be activated from inside the app, and paid passes are bought on the website. **Admin accounts are website-only**: the API refuses admin sign-in, token use and refresh from the app.
- **Signed in when the app opens the website:** "Upgrade on Web", Contact and Appeal open the site through a one-time link (60 seconds, single use, only a hash stored) that the website exchanges for its own session, so users don't sign in again. 2FA isn't repeated (it was passed in the app), and at the session cap older website sessions are dropped before the phone's.
- Passwords (min. 8 characters) are stored as bcrypt hashes. Optional **two-factor authentication** by emailed code; turning it on or off requires the account password (or Google confirmation for Google accounts).
- JWT access tokens (15 min, kept in memory) and rotating refresh tokens (7 days; HTTP-only cookie on web, `expo-secure-store` on mobile). Concurrent sessions are capped (default **3**) and can be reviewed and revoked.
- One account per device: sign-up/sign-in record a device identifier (FingerprintJS + a random device ID on web, a random device ID on mobile) to stop repeat free-plan/trial abuse.
- Rate limiting on auth, OTP and export endpoints; Helmet security headers; a prohibited-words filter on submitted text.
- Account deletion with a **7-day grace period**, after which a scheduled job erases the account (see `backend/services/accountPurgeService.js`).

### Trips, expenses and settlement
- Each expense has a payer and is either **split equally** among selected members or marked as a **personal** expense. (There are no percentage or custom-amount splits.)
- Members can be registered users (added by Feriko ID), people invited by email, or name-only guests. Guests can later register and take over their expenses.
- Settlement uses a greedy debt-simplification algorithm to minimise the number of payments. Members can mark a payment as settled; if expenses change afterwards, the settlement is shown as pending again.
- Trip creators (pool creators for ride pools) control membership, join requests, whether members may add expenses, and (on supporting plans) handing over the creator role (`PATCH /api/trips/:id/transfer-creator`). "Admin" always means the platform administrator, never a trip role.
- Shared **album links** (links to albums hosted elsewhere, not uploaded photos), with a per-plan limit.
- **Exports**: PDF and CSV trip summaries, gated by plan, optionally creator-only, with a daily limit (default 25).
- Real-time updates for trip members via Server-Sent Events.

### Passport and recommendations
- Pin visited places (with photos) to a personal scratch map; place search uses OpenStreetMap Nominatim.
- Publish travel recommendations that all users can browse; authors are shown by name only to friends or if they opt in.

### Plans and payments
Plans, prices and limits are stored in the database and editable by admins; the defaults live in [`backend/constants/defaultTier.js`](backend/constants/defaultTier.js), and the Pricing page always shows the current values.

- **Personal trip packs** – one-time payments for a number of trips (with 2x/3x/5x multi-packs). Creating a trip uses one allotment; deleting an active trip with no expenses returns it.
- **Corporate passes** – one-time payments for 1, 3, 6 or 12 months. They **do not auto-renew**; renewal opens in the last 2 days and extends from the current end date. Expired passes lock active ride pools until renewal. A fair-use limit (default 20) applies to active pools.
- Each account gets **one** free plan, once: either the free personal plan or the free corporate plan, not both. Paying customers can still claim it (only when they have no active plan of that type). Trial passes can be bought once, and disappear once any paid plan has been bought.
- Plan limits are snapshotted at purchase, so later admin changes don't affect an active pass.
- Payments via **Dodo Payments** (hosted checkout, webhook + return-URL verification) or **manual UPI** (transaction reference + receipt screenshot, reviewed by an admin). Receipt screenshots are deleted after approval or ~7 days.
- Archived trips on free/trial plans are auto-deleted after their plan's retention period (default 30 days personal, 60 days corporate, counted from archival); paid plans keep them.

### Administration
- Admin dashboard: users, subscriptions, payments and manual-payment review, tier/pricing configuration, reviews, bans and appeals, account deletion, device lookup, and system/cron/email logs.
- Terms of Service and Privacy Policy are stored in the database and editable by admins. The built-in defaults are in [`backend/constants/defaultLegal.js`](backend/constants/defaultLegal.js); bumping `DEFAULT_LEGAL_REVISION` republishes them as a new version on next start (unless an admin has edited them), and users are asked to accept.
- The admin account is created on startup for `ADMIN_EMAIL` (default `GMAIL_USER`) and signs in with Google only: it has no password, and any password stored by older versions is removed on startup. Its first Google sign-in with that verified email links the Google account; only that Google account can sign in afterwards.

---

## Tech stack

| Part | Stack |
| :--- | :--- |
| **Backend** (`backend/`) | Node.js 22 (ES modules), Express 4, MongoDB with Mongoose 8, `jsonwebtoken`, `bcryptjs`, `express-validator`, `helmet`, `express-rate-limit`, `pdfkit`, `pino`, Gmail API via `googleapis` (OAuth2), ImageKit, Dodo Payments SDK |
| **Web** (`frontend-next/`) | Next.js 15 (App Router, standalone output), React 19, Tailwind CSS 4, Zustand, Axios (with automatic token refresh), Recharts |
| **Mobile** (`mobile/`) | Expo SDK 57, Expo Router, React Native 0.86, NativeWind, Zustand, `expo-secure-store` |

---

## Repository layout

```text
.
├── backend/                 # Express API
│   ├── config/              # Env config, MongoDB, ImageKit, Dodo Payments
│   ├── constants/           # Default tiers, limits, legal documents, ImageKit folders
│   ├── controllers/         # Route handlers (auth, trips, expenses, subscriptions, admin, …)
│   ├── middlewares/         # Auth, admin, trip/expense permissions, plan limits, validation
│   ├── models/              # Mongoose schemas
│   ├── routes/              # Express routers
│   ├── services/            # Email, ImageKit, PDFs/receipts, SSE, cleanup, archive, account purge
│   ├── utils/               # Logger, caches, generators, helpers
│   └── validators/          # express-validator rules
├── frontend-next/           # Next.js web app (src/app, src/components, src/stores, src/api)
├── mobile/                  # Expo app (app/, components/, stores/, services/)
├── runner.js                # Production supervisor: runs the API and Next.js in one container
├── Dockerfile               # Multi-stage production image (backend + web; mobile excluded)
└── .github/workflows/       # CI/CD pipeline
```

---

## Environment variables

### Backend (`backend/.env`, see [`backend/.env.example`](backend/.env.example))

| Variable | Required | Description |
| :--- | :---: | :--- |
| `MONGODB_URI` | ✅ | MongoDB connection string |
| `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET` | ✅ | Random secrets, at least 32 characters each |
| `ADMIN_EMAIL` | | Google account of the platform admin (default `GMAIL_USER`; a separate Gmail with 2-Step Verification is safer). Changing it moves the admin to the new address |
| `IMAGEKIT_PUBLIC_KEY`, `IMAGEKIT_PRIVATE_KEY`, `IMAGEKIT_URL_ENDPOINT` | ✅ | ImageKit credentials |
| `GMAIL_USER` | ✅ | Sending address (also the admin account email unless `ADMIN_EMAIL` is set) |
| `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `OAUTH_REFRESH_TOKEN` | ✅ | Google OAuth2 credentials for the Gmail API |
| `GOOGLE_CLIENT_ID` | | **Web** OAuth client ID for "Continue with Google" (website, and the token audience for the Android app). Google sign-in is hidden when unset |
| `PORT` | | API port (default `5000`) |
| `NODE_ENV` | | `development` or `production` |
| `FRONTEND_URL` | prod | Public website address (e.g. your own domain). Emails, invites, payment redirects, the APK download page, the mobile app's web links and the search-engine `robots.txt`/`sitemap.xml` all follow it, so a new domain only needs this changed. On Render it falls back to `RENDER_EXTERNAL_URL`; default `http://localhost:3000` in dev |
| `APP_RELEASE_SECRET` | | Shared secret the mobile release workflow uses to publish new APKs (`POST /api/app/releases`). Publishing is disabled when unset |
| `CORS_ALLOWED_ORIGINS` | | Extra comma-separated origins allowed in production (same-origin and `FRONTEND_URL` are always allowed) |
| `DODO_PAYMENTS_API_KEY`, `DODO_PAYMENTS_ENVIRONMENT` | | Dodo Payments key and `test_mode` / `live_mode` |
| `DODO_PAYMENTS_WEBHOOK_SECRET` | prod | Webhook signing secret. **Required in production** – unsigned webhooks are rejected |
| `LOG_MONGO_URI` | | Separate MongoDB for system/cron logs (optional) |
| `PING_TARGET_URL` | | Keep-alive ping target (default `${FRONTEND_URL}/health`) |

### Web (`frontend-next/`)
| Variable | Description |
| :--- | :--- |
| `NEXT_PUBLIC_API_URL` | API base URL (default `http://localhost:5000/api`; the Docker build uses `/api`) |

### Mobile (`mobile/`)
| Variable | Description |
| :--- | :--- |
| `EXPO_PUBLIC_API_URL` | API host. For builds it is an **EAS environment variable** (`eas env:create --environment production --name EXPO_PUBLIC_API_URL --value https://<api-host> --visibility plaintext`), not stored in `eas.json`; for local development put it in `mobile/.env` |
| `EXPO_PUBLIC_WEB_URL` | Optional. Website links in the app come from the server's `FRONTEND_URL`; this is only the fallback before the app has reached the server |

---

## Local development

Requires Node.js 22+ and pnpm 11 (`corepack enable`).

```bash
pnpm install                         # installs all workspace packages
cp backend/.env.example backend/.env # then fill in real values

pnpm run dev          # API on :5000, then the web app on :3000
pnpm run dev:all      # API + web + Expo
pnpm run dev:backend  # API only
```

Quality checks (the same ones CI runs):

```bash
pnpm run check:syntax
pnpm run lint         # backend ESLint + Next.js lint
pnpm run type-check   # web TypeScript
```

There is currently no automated test suite.

---

## Deployment

Production runs as a **single Docker container** on Render:

- `runner.js` starts the Express API on the public port (`PORT`, default 10000) and the Next.js standalone server on an internal port (3000). Express serves `/api/*` and `/health` and reverse-proxies all other requests to Next.js, so the web app and API share one origin.
- The multi-stage `Dockerfile` builds Next.js in standalone mode and installs only production backend dependencies; the final image runs as the unprivileged `node` user with a health check on `/api/health`.
- A background scheduler (every 12 hours) prunes old notifications, sends 7-day reminders, deletes expired archived trips and erases accounts past their deletion grace period.

### CI/CD (`.github/workflows/deploy.yml`)
On every push to `main`:
1. Install dependencies, then run `check:syntax`, `lint` and `type-check`.
2. Build a `linux/amd64` image and push it to GitHub Container Registry (`:latest` and `:<commit-sha>`).
3. Trigger the Render deploy hook (`RENDER_DEPLOY_HOOK_URL` secret; skipped with a warning if not set).

Pushes that change **only Markdown files** (`**/*.md`) or **only the mobile app** (`mobile/**`) do not run the pipeline. It can also be started manually (`workflow_dispatch`).

### Android releases (`.github/workflows/mobile-release.yml`)
The app is distributed as an APK from the website (`/download`), not through the Play Store. On every push to `main` that changes `mobile/**` (or manually, with optional release notes, a minimum supported version and a notify switch):
1. Type-check the app, then build the APK with EAS (`apk` profile; the versionCode increases automatically).
2. Upload it to ImageKit as `/feriko/app-releases/build-<versionCode>/Feriko-<version>.apk` (users download `Feriko-1.0.0.apk`; the build folder gives every build a new address, so the CDN never serves an old cached file).
3. Publish it to the backend (`POST /api/app/releases`), which shows it on `/download`, sends an in-app "App update available" notification to users who use the app (tapping it opens the download page), and keeps only the newest APK on ImageKit (the workflow then deletes every other build folder and file in the release folder, including leftovers from before a database reset).

The **Download APK** button uses `GET /api/app/download`, which streams the latest APK through the server with the Android app type, so phones save `Feriko-<version>.apk` instead of `.apk.zip` (ImageKit labels APKs as ZIP files).

**"What's new"** is written for users, never copied from commit messages: notes typed when running the workflow by hand are used first, then any `Release-Notes: ...` lines in the pushed commit messages (one line per point). Otherwise a plain message is chosen from the commit types: `feat` → new features and improvements, `fix` → bug fixes, `perf` → performance improvements, anything else → bug fixes and performance improvements. Bump `version` in `mobile/app.json` for releases users should notice.

Installed apps also check `GET /api/app/latest` when opened and offer the update. **Forced updates:** set a minimum supported versionCode on a release only when older builds would break (changed API, security or data fix). The server then answers `426 Upgrade Required` to app requests from older builds (the app sends its versionCode in `X-App-Version`; builds without the header count as older), and the app shows a blocking update screen. The update check and logout stay available. Repository secrets: `EXPO_TOKEN`, `IMAGEKIT_PRIVATE_KEY`, `APP_RELEASE_SECRET`; the API address is read from the EAS `EXPO_PUBLIC_API_URL` variable.

---

## License

No license file is included in this repository, so all rights are reserved by the author unless a license is added.
