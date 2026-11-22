# Verve&#32;

Version 1.0 · November 2026 · Audience: AI coding agent and human reviewer

## 1. How to use this document

- Build exactly what is described here. Do not add features outside this PRD. If something is ambiguous, choose the simplest option, record it in `DECISIONS.md`, and continue.
- Follow the milestones in section 19 in order. Each milestone ends with passing tests and a working demo of that slice.
- Priorities: P0 = must ship, P1 = should ship, P2 = only if time remains. Untagged items are P0.
- Conventions: TypeScript strict mode everywhere. Money is stored as integer paise (INR). Timestamps are UTC ISO strings. IDs are Mongo ObjectIds serialized as strings. Every API input is validated with zod schemas imported from `packages/shared`.
- All changes to `Collaboration.status` go through one module (`collaboration.service.ts`) that owns the transition map. No other code writes that field.
- All background jobs must be idempotent and safe to retry.
- Write the README as you build: architecture diagram, state machine diagram, design trade-offs, local setup, demo accounts.

## 2. Product overview

### 2.1 Problem

D2C brands (skincare, fashion, food, gadgets) rely on micro-influencers (5k–100k followers). Managing dozens of them over DMs and spreadsheets fails. Brands pay, and creators delete the post days later or forget the tracking link. Creators deliver on time, then chase brands for weeks for payment.

### 2.2 Solution

CreatorFlow is a workflow plus escrow and compliance engine. Brands fund campaigns into escrow. Creators apply, submit drafts, and receive timestamped feedback in one workspace. After the post goes live, background workers verify it on a schedule. The payout unlocks automatically only if the post stays live and compliant for the contracted duration. If it is removed, the payout freezes and the brand is alerted.

### 2.3 Goals

- A deployable, demo-able portfolio project with a deep backend: state machine, queues, idempotent workers, high-throughput redirect service.
- A complete brand-to-creator flow from campaign creation to unlocked invoice.
- A live demo that shows a full verification cycle, including a breach, in about 5 minutes using accelerated intervals.

### 2.4 Non-goals

- Real money movement. The wallet is a simulated ledger. Razorpay test mode is an optional P2 for deposits. Never describe this as a real payments system.
- Guaranteed Instagram verification. YouTube is the primary live platform. Instagram is best-effort and behind a feature flag.
- Native mobile apps, in-app chat, multi-currency, real tax filing.

### 2.5 Success criteria

- The full flow works end to end on the deployed site with seeded demo accounts for both roles.
- Verification runs on a configurable schedule (default 3 daily checks; any duration 1–30 days is supported).
- The redirect route is load-tested and the README reports requests per second and p95 latency.
- State machine and worker logic have automated tests, including 5-day and 10-day simulated-time scenarios.

## 3. Tech stack and repository

### 3.1 Stack

- Frontend: Next.js (App Router), TypeScript, Tailwind CSS, shadcn/ui, TanStack Query, react-hook-form, zod, Recharts, socket.io-client.
- Backend: Node.js, Express, TypeScript, Mongoose, zod, pino logging, helmet, express-rate-limit, jsonwebtoken, bcrypt, Socket.IO.
- Queue: BullMQ with ioredis. Redis is self-hosted in Docker. Do not use Upstash free tier (BullMQ polling exhausts its command quota).
- Database: MongoDB Atlas M0.
- Object storage: Cloudflare R2 through the S3 API (`@aws-sdk/client-s3`, `s3-request-presigner`).
- Video processing: ffmpeg through `fluent-ffmpeg`. PDFs: `pdfkit`. Email: Resend (log to console when no API key).
- Testing: Vitest, supertest, mongodb-memory-server, a real Redis service in CI, k6 or autocannon for load tests.
- CI/CD: GitHub Actions. Deployment is free-tier only (section 18).

### 3.2 Monorepo layout (pnpm workspaces)

- `apps/web` — Next.js app.
- `apps/api` — Express API, Socket.IO server, BullMQ workers (workers run as a separate process entry point `worker.ts` from the same codebase).
- `packages/shared` — zod schemas, enums, TypeScript types, status transition map, constants.
- `docker-compose.yml`, `Caddyfile`, `.github/workflows`, `README.md`, `DECISIONS.md`.

### 3.3 Authentication approach

- Email and password. Passwords hashed with bcrypt. A signed JWT (HS256, 7 days) is stored in an httpOnly, Secure, SameSite=Lax cookie.
- Next.js rewrites proxy `/api/*` to the API so the cookie is first-party. Next.js middleware verifies the JWT with `jose` using the shared `JWT_SECRET` and enforces role-based route access.
- Socket.IO cannot go through Vercel rewrites. The browser connects directly to the API origin using a short-lived token from `GET /api/v1/auth/socket-token`.

## 4. Roles and permissions

- Creator (`accountType = CREATOR`): browse marketplace, apply, upload drafts, view feedback, generate tracking link, submit live URL, view own earnings and invoices, edit own profile.
- Brand Owner: everything in the organization, including team management, wallet, and campaigns.
- Brand Campaign Manager: create and edit campaigns, screen applicants, accept or reject, review drafts, view analytics. Cannot access wallet actions or team settings.
- Brand Finance Reviewer: view wallet, deposit funds, view and mark invoices paid, resolve breaches, view analytics read-only. Cannot edit campaigns or review drafts.
- Every API route checks role and organization membership. A user can only access resources of organizations they belong to, and a creator only their own collaborations.

## 5. Domain model (MongoDB, Mongoose)

### 5.1 User

`email` (unique, lowercase), `passwordHash`, `name`, `accountType` (BRAND | CREATOR), `currentOrgId` (brand users), `onboardingComplete`, `createdAt`.

`creatorProfile`: `displayName`, `bio`, `niches[]`, `handles[]` (`platform` INSTAGRAM | YOUTUBE, `handle`, `url`, `followers`, `engagementRate`, `verified`), `audience` (`ageRanges`, `topCities`, `genderSplit`), `portfolio[]` (`title`, `videoUrl`, `thumbUrl`), `reliabilityScore` (0–100), `stats` (`completed`, `breached`, `onTimeDrafts`, `totalDrafts`), `payoutDetails` (`method` UPI | BANK, `upiId`, `accountHolder`, `accountLast4`, `ifsc`; always masked in API responses).

### 5.2 Organization

`name`, `slug` (unique), `planTier` (FREE | PRO | SCALE, cosmetic), `walletBalance` (available, paise), `lockedBalance` (escrow, paise), `members[]` (`userId`, `role` OWNER | CAMPAIGN\_MANAGER | FINANCE\_REVIEWER), `createdAt`.

### 5.3 Invite

`orgId`, `email`, `role`, `tokenHash` (store a hash, email the raw token), `expiresAt` (7 days), `acceptedAt`, `invitedBy`.

### 5.4 Campaign

`orgId`, `title`, `briefMarkdown`, `productCategory`, `targetAudience`, `niches[]`, `deliverables[]` (`type` REEL | SHORT | STORY | POST, `count`), `requiredTags` (`mentions[]`, `hashtags[]`), `compensationType` (FIXED | AFFILIATE\_CPC | BARTER\_CASH), `payoutPerCreator`, `cpcRate`, `maxBonusPerCreator`, `barterDescription`, `targetUrl`, `liveDurationDays` (1–30, default 3), `draftTurnaroundDays` (default 7), `applicationDeadline`, `slots`, `budgetTotal`, `budgetCommitted`, `conversionKey` (secret for postbacks), `status` (DRAFT | OPEN | ACTIVE | COMPLETED | CANCELLED), timestamps.

### 5.5 Collaboration (core state machine)

- Identity: `campaignId`, `creatorId`, `orgId`, `pitch`, `status`, `acceptedAt`, `draftDueAt`, `escrowLocked` (paise).
- `drafts[]`: `version`, `rawKey`, `previewUrl`, `thumbUrl`, `durationSec`, `fps`, `processingStatus` (PENDING | PROCESSING | READY | FAILED), `uploadedAt`, `decision` (PENDING | CHANGES\_REQUESTED | APPROVED), `decidedAt`, `decidedBy`, `comments[]` (`_id`, `timestampSec`, `authorId`, `text`, `resolved`, `createdAt`).
- Tracking: `trackingSlug` (unique, created on first request, idempotent), `verifiedClicks`, `conversions`.
- Live monitoring: `livePostUrl`, `livePlatform` (YOUTUBE | INSTAGRAM | DEMO), `monitoringStartedAt`, `contractedDays`, `daysVerified`, `nextCheckAt`, `verificationLogs[]` (`checkedAt`, `dayNumber`, `outcome`, `captionMatched`, `httpStatus`, `detail`), `breach` (`reason`, `at`, `resolution` REFUNDED | RELEASED | null).
- Watchdog flags: `draftReminderSentAt`, `lastBrandNudgeAt`.

### 5.6 Invoice

`collaborationId`, `orgId`, `creatorId`, `number` (for example CF-2026-0001), `baseAmount`, `clickBonusAmount`, `totalAmount` (gross), `tdsRate`, `tdsAmount`, `netAmount` (to creator), `status` (LOCKED | UNLOCKED | PAID | VOID), `pdfKey`, `auditLog[]` (`at`, `event`, `detail`), `paidAt`.

### 5.7 Supporting collections

- `WalletTransaction`: `orgId`, `type` (DEPOSIT | ESCROW\_LOCK | ESCROW\_RELEASE | PAYOUT | REFUND), `amount`, `collaborationId`, `invoiceId`, `balanceAfter`, `createdAt`.
- `ClickEvent`: `collaborationId`, `campaignId`, `slug`, `ts`, `deviceType`, `os`, `browser`, `referrer`, `visitorHash`, `isUnique`, `isBot`. TTL index of 90 days.
- `Notification`: `userId`, `type`, `title`, `body`, `link`, `readAt`, `createdAt`.
- `ConversionEvent` (P2): `collaborationId`, `orderId` (unique per collaboration), `orderValue`, `ts`.

### 5.8 Indexes

Unique on `User.email`, `Organization.slug`, `Collaboration.trackingSlug`, and `(campaignId, creatorId)` on Collaboration. Also index `Collaboration` on `(orgId, status)`, `(creatorId, status)`, `nextCheckAt`, and `Campaign` on `(orgId, status)` and `(status, applicationDeadline)`.

## 6. Collaboration state machine

Statuses: APPLIED, ACCEPTED, DRAFT\_SUBMITTED, CHANGES\_REQUESTED, APPROVED\_TO\_POST, LIVE\_MONITORING, BREACHED, COMPLETED, REJECTED, RELEASED. Terminal: BREACHED, COMPLETED, REJECTED, RELEASED.

Allowed transitions (anything else throws `InvalidTransitionError`, HTTP 409):

- APPLIED → ACCEPTED: brand (Owner or Campaign Manager). Locks escrow, sets `acceptedAt` and `draftDueAt`. Fails if slots are full or the wallet is short.
- APPLIED → REJECTED: brand. APPLIED → withdrawn (P1): creator deletes own application.
- ACCEPTED → DRAFT\_SUBMITTED: creator, after draft v1 finishes processing (READY).
- ACCEPTED → RELEASED: watchdog only. Returns escrow, reopens the slot.
- DRAFT\_SUBMITTED → CHANGES\_REQUESTED or APPROVED\_TO\_POST: brand, on the latest draft.
- CHANGES\_REQUESTED → DRAFT\_SUBMITTED: creator uploads the next version.
- APPROVED\_TO\_POST → LIVE\_MONITORING: creator submits a live URL and the initial check passes. If the initial check fails, the status stays APPROVED\_TO\_POST and the failure reasons are returned so the creator can fix the post and resubmit.
- LIVE\_MONITORING → BREACHED: verification worker, after a confirmed deletion, privacy change, or caption non-compliance.
- LIVE\_MONITORING → COMPLETED: verification worker when `daysVerified === contractedDays`.

Every transition writes an audit entry, emits a Socket.IO event, and creates notifications for affected users.

## 7. Escrow and money rules

- Deposit (Owner or Finance Reviewer): `walletBalance += amount`, writes a DEPOSIT transaction. Simulated by default. Razorpay test-mode checkout is P2.
- Accept applicant: lock `payoutPerCreator + maxBonusPerCreator`. `walletBalance -= lock`, `lockedBalance += lock`, `collaboration.escrowLocked = lock`, create an invoice with status LOCKED. Reject the accept if `walletBalance < lock` or if committed budget would exceed `budgetTotal`.
- Unlock (worker, on COMPLETED): `clickBonus = min(verifiedClicks * cpcRate, maxBonusPerCreator)`. `total = base + clickBonus`. `tds = round(total * TDS_RATE)` (default 10 percent, configurable, labeled illustrative). `net = total - tds`. Move `lock` out of `lockedBalance`, return `lock - total` to `walletBalance`, set invoice UNLOCKED, generate the PDF, write ESCROW\_RELEASE.
- Mark paid (Owner or Finance Reviewer): invoice UNLOCKED → PAID, writes a PAYOUT transaction (informational), notifies the creator.
- Breach: funds stay locked and the invoice stays LOCKED. Owner or Finance Reviewer resolves with REFUND (invoice VOID, lock returned to wallet, `breach.resolution = REFUNDED`) or RELEASE (manual override, invoice UNLOCKED, audited, `RELEASED`).
- Released slot or rejected-after-accept: return the lock and void the invoice.
- All balance changes happen in a MongoDB transaction (or an equivalent guarded atomic update) together with the transaction record. Balances can never go negative.
- Compensation types: FIXED has no click bonus (cpcRate 0). AFFILIATE\_CPC pays base plus bonus per verified click. BARTER\_CASH shows barterDescription and pays a smaller cash base.

## 8. REST API

Base path `/api/v1`. JSON in and out. Errors use `{ error: { code, message, details? } }`. Lists accept `page` and `limit`. Public exceptions are `/r/:slug` (outside the base path), `/demo-post/:id`, `/health`, and auth routes.

### 8.1 Auth and onboarding

- `POST /auth/register`, `POST /auth/login`, `POST /auth/logout`, `GET /auth/me`, `GET /auth/socket-token`.
- `POST /orgs` — brand user creates an organization and becomes OWNER.
- `PUT /creators/me/onboarding` — handles, niches, payout details; sets `onboardingComplete`.
- `GET /invites/:token`, `POST /invites/:token/accept`.

### 8.2 Organization and team

- `GET /orgs/:orgId`, `PATCH /orgs/:orgId` (Owner).
- `GET /orgs/:orgId/members`, `POST /orgs/:orgId/invites`, `PATCH /orgs/:orgId/members/:userId` (role), `DELETE /orgs/:orgId/members/:userId`. An org must always keep at least one Owner.

### 8.3 Campaigns (brand)

- `POST /campaigns`, `GET /campaigns?status=`, `GET /campaigns/:id`, `PATCH /campaigns/:id` (full edit in DRAFT, limited fields once OPEN).
- `POST /campaigns/:id/publish` (DRAFT → OPEN), `POST /campaigns/:id/close` (stop applications; OPEN → ACTIVE).
- `GET /campaigns/:id/applicants?niche=&followerTier=&minEngagement=&minReliability=&status=&sort=`. Follower tiers: NANO under 10k, MICRO 10k–50k, MID 50k–100k.
- `GET /campaigns/:id/collaborations`, `POST /collaborations/:id/accept`, `POST /collaborations/:id/reject`.
- `GET /campaigns/:id/analytics` and `GET /campaigns/:id/analytics/timeseries?interval=day`.
- `GET /creators/:id/public-profile` (media kit, reliability, stats) for the applicant drawer.

### 8.4 Marketplace and collaboration (creator)

- `GET /marketplace/campaigns?q=&niche=&compensationType=&deadlineBefore=&page=`, `GET /marketplace/campaigns/:id`.
- `POST /campaigns/:id/apply` (body: `pitch`). One application per creator per campaign.
- `GET /creator/collaborations`, `GET /collaborations/:id` (role-scoped view).

### 8.5 Drafts and review

- `POST /collaborations/:id/drafts/upload-url` — body `{ filename, contentType: video/mp4, sizeBytes }`, max `MAX_UPLOAD_MB`. Returns `{ version, uploadUrl, key }`. Creates a draft with PENDING status.
- `POST /collaborations/:id/drafts/:version/complete` — verifies the object exists in R2, queues transcoding.
- `GET /collaborations/:id/drafts/:version` — metadata, signed preview URL, comments.
- `POST /collaborations/:id/drafts/:version/comments` (`timestampSec`, `text`), `PATCH .../comments/:commentId` (edit text or toggle `resolved`), `DELETE .../comments/:commentId`. Brand roles only; creators read only.
- `POST /collaborations/:id/drafts/:version/request-changes` and `.../approve`. Request changes requires at least one comment. Only the latest version can be decided.

### 8.6 Tracking and live post

- `POST /collaborations/:id/tracking-link` — idempotent. Returns `{ slug, shortUrl, utmUrl }`. Allowed from ACCEPTED onward.
- `POST /collaborations/:id/live-post` — body `{ url }`. Allowed only in APPROVED\_TO\_POST. Detects platform, runs the initial check synchronously, then queues the schedule.
- `POST /collaborations/:id/verify-now` — runs the next check immediately. Allowed when `DEMO_MODE=true` or for brand Owner.
- `POST /collaborations/:id/resolve-breach` — body `{ action: REFUND | RELEASE }`.
- Demo target (only when `DEMO_MODE=true`): `POST /demo/posts` creates a controllable fake post, `POST /demo/posts/:id/delete` flips it to deleted, and `GET /demo-post/:id` (public) returns 200 with a caption or 404.

### 8.7 Wallet and invoices

- `GET /orgs/:orgId/wallet` (balance, locked, available), `POST /orgs/:orgId/wallet/deposit`, `GET /orgs/:orgId/wallet/transactions`.
- `GET /orgs/:orgId/invoices?status=`, `GET /invoices/:id`, `GET /invoices/:id/pdf` (redirects to a short-lived signed URL), `POST /invoices/:id/mark-paid`.
- `GET /creator/earnings` — collaborations with milestones, projected bonus, and invoices.

### 8.8 Dashboards, profile, notifications, conversions

- `GET /brand/dashboard` (KPIs and recent alerts), `GET /creator/dashboard`.
- `GET /creators/me/profile`, `PUT /creators/me/profile`, `POST /creators/me/portfolio/upload-url`.
- `GET /notifications`, `POST /notifications/read`.
- P2: `POST /conversions` (public, authenticated with `conversionKey` and `slug`, dedupes on `orderId`).
- `GET /health` returns status of Mongo and Redis.

## 9. Background pipelines (BullMQ)

Queues: `verification`, `settlement`, `watchdog`, `media`, `notifications`. Default job options: 3 attempts with exponential backoff, `removeOnComplete` with age limit, `removeOnFail` retained for inspection. Workers log with a correlation ID and write failures to a dead-letter log.

### 9.1 Pipeline 1: live post verification

- Trigger: `POST /collaborations/:id/live-post`.
- Initial check (synchronous, does not count as a verified day): the URL parses to a supported platform, the post returns public and available, and the caption or description contains every required mention and hashtag (case-insensitive) plus `#ad` or the campaign hashtag. On failure return reasons and do not change status.
- On success: set LIVE\_MONITORING, `contractedDays = campaign.liveDurationDays`, `daysVerified = 0`, and enqueue the first delayed job `verify:{collabId}:1` with delay `VERIFY_INTERVAL_MS` (default 24 hours).
- Each job: run the provider check, append a verification log, and then follow the rules below.
  - OK: `daysVerified += 1`. If `daysVerified === contractedDays`, enqueue settlement. Otherwise enqueue `verify:{collabId}:{n+1}`.
  - Transient error (timeout, 5xx, 429, network): retry with backoff. A transient error never causes a breach.
  - DELETED, PRIVATE, or NON\_COMPLIANT: run one confirmation re-check after 60 seconds. If it fails again, breach.
- Breach handling: status BREACHED, invoice stays LOCKED (payout frozen), cancel pending verification jobs, notify the brand by Socket.IO and email, reduce the creator reliability score, write an audit entry.
- Idempotency: job IDs are `verify:{collabId}:{dayNumber}`. Before applying a result, the worker checks that `dayNumber === daysVerified + 1` and that the status is still LIVE\_MONITORING. Duplicate or late jobs are no-ops.
- Providers (adapter interface `PostProvider.check(url, requirements)` returning `{ outcome, captionMatched, httpStatus, detail }`):
  - `YouTubeProvider`: YouTube Data API `videos.list` with `part=snippet,status`. An empty result means DELETED. `privacyStatus = private` means PRIVATE. Match requirements against title plus description. Shorts URLs are normalized to video IDs.
  - `DemoProvider`: calls the internal `/demo-post/:id` endpoint.
  - `InstagramProvider`: best-effort oEmbed or HTML metadata, behind `ENABLE_INSTAGRAM`. Document its limits in the README.

### 9.2 Pipeline 2: settlement and invoice unlock

- Trigger: verification worker enqueues `settle:{collabId}` once.
- Steps: in one transaction, set COMPLETED, compute bonus from Redis and Mongo click counts, release escrow per section 7, set the invoice UNLOCKED, update creator stats and reliability score. Then generate the PDF and upload it to R2 (retryable independently; the unlock must not depend on PDF success). Notify creator and brand finance.
- Idempotent: if the invoice is already UNLOCKED or PAID, do nothing.
- PDF content: invoice number, brand and creator details, campaign, base fee, click bonus (with click count and rate), gross total, TDS line and rate, net payable, and a campaign audit log (key events with timestamps, verification days and results).

### 9.3 Pipeline 3: deadline and stalled-workflow watchdog

- Trigger: repeatable job every `WATCHDOG_INTERVAL_MS` (default 6 hours).
- ACCEPTED with no v1 draft: send one reminder when `now >= draftDueAt - 48h` and `draftReminderSentAt` is empty. When `now >= draftDueAt + 72h`, transition to RELEASED, return escrow, reopen the slot, and notify both sides.
- DRAFT\_SUBMITTED with no decision for more than 72 hours: send the brand an escalation nudge, repeated at most once every 72 hours (`lastBrandNudgeAt`).

### 9.4 Pipeline 4: video transcoding and thumbnails

- Trigger: `drafts/:version/complete`.
- Download from R2, run ffmpeg to produce a 720p H.264 MP4 preview with `+faststart` and a JPEG thumbnail at 1 second, read duration and fps with ffprobe, upload outputs, set `processingStatus = READY`, then delete the raw file to save storage. On final failure set FAILED and keep the raw file.
- When READY, emit `draft.processed`. If this is the creator's submitted version, move the collaboration to DRAFT\_SUBMITTED.

## 10. Redirect and attribution service

- Route: `GET /r/:slug` on the Express API, outside `/api/v1`, mounted before heavy middleware. The public short link is `SHORT_LINK_BASE_URL/c/:slug`, which maps to the same handler.
- Behavior: look up `slug → { collaborationId, campaignId, targetUrl }` in Redis (cache on miss from Mongo with a 1-hour TTL), respond with a 302 to `targetUrl` plus UTM parameters (`utm_source=creatorflow`, `utm_medium=influencer`, `utm_campaign={campaignId}`, `utm_content={slug}`), and record the click asynchronously.
- Click processing: parse user agent for device type, OS, and browser; read referrer. Compute `visitorHash = sha256(ip + userAgent + dailySalt)`. Mark as bot if the user agent matches known bot or crawler patterns or the same hash exceeds a rate threshold. Unique means `SET NX` on `uniq:{collabId}:{visitorHash}` with a 24-hour TTL succeeds.
- Counters in Redis: `HINCRBY clicks:{collabId}` for `total`, `unique`, and `bot`. Push the click event to a Redis list that a worker flushes to MongoDB `ClickEvent` in batches. A periodic job syncs `verifiedClicks` (unique and non-bot) to `Collaboration`.
- Unknown slug: 404 page. Never expose internal errors. Redis failure must not break the redirect (fall back to Mongo lookup and skip counting).
- Target: p95 under 25 ms on the VM for cached slugs. Load-test and record results.

## 11. Real-time notifications

- Socket.IO server authenticated with the socket token. Each user joins `user:{id}` and each brand member joins `org:{id}`.
- Events: `notification.created`, `collaboration.updated`, `verification.checked`, `breach.detected`, `draft.processed`.
- The notification bell in the shared layout shows an unread badge, a dropdown of the latest 20, mark-all-read, and links to the relevant page. Breach, review nudge, draft approved, changes requested, payout unlocked, and slot released all produce notifications. Critical ones also send email.

## 12. Frontend specification

### 12.1 General

- Desktop-first, responsive. Creator pages must work well on a phone. Light theme is required.
- Every data view needs loading skeletons, empty states, and error states. Forms use react-hook-form with the shared zod schemas and show inline errors. Mutations show toasts.
- Status badges use consistent colors. Creator-facing milestone labels map as: APPLIED → Applied; ACCEPTED, DRAFT\_SUBMITTED, CHANGES\_REQUESTED → Draft in review; APPROVED\_TO\_POST → Approved to post; LIVE\_MONITORING → Live verification active (Day X/N); COMPLETED → Payout unlocked.
- Shared components: `AppShell` (sidebar, header, notification bell), `DataTable`, `StatusBadge`, `MilestoneStepper`, `DaysProgress`, `EmptyState`, `ConfirmDialog`, `VideoReviewPlayer`, `ApplicantDrawer`, `CampaignForm`, `MoneyText` (formats paise as INR).
- Quality priority: the video review page, creator collaboration studio, campaign command center, and analytics tab get the most polish. Other pages stay clean and simple.

### 12.2 Public and auth pages (4)

- `/` Landing: hero with the value proposition (escrow plus verified payouts), a three-step how-it-works, and two CTAs (I'm a brand, I'm a creator) linking to `/register?role=`. Static, no API calls.
- `/login`: email and password. After login, redirect by account type to `/brand/dashboard` or `/creator/dashboard`, or back to onboarding if incomplete. When `NEXT_PUBLIC_DEMO_MODE` is true, show one-click buttons that fill demo credentials.
- `/register`: step 1 is role, name, email, password. Step 2 is onboarding. Brand: organization name (slug auto-generated and editable). Creator: handles (platform, handle, followers, engagement rate), niches, payout details (UPI or bank). Step 2 resumes if the user returns before finishing.
- `/invite/[token]`: shows organization and role. New users set name and password, existing users log in, then accept and land on the brand dashboard. Handle expired or already-used tokens.

### 12.3 Brand workspace (7)

- `/brand/dashboard`: KPI cards (active campaigns, locked escrow, pending draft reviews, total verified clicks) and a live feed of compliance and breach alerts driven by Socket.IO. Links to the relevant campaign or review.
- `/brand/campaigns`: table of campaigns with status filter tabs (DRAFT, OPEN, ACTIVE, COMPLETED), budget utilization bar, applicant and collaboration counts, and quick actions.
- `/brand/campaigns/new` (also used for editing a draft): sectioned form — basics (title, category, audience, niches), markdown brief with preview, deliverables, required mentions and hashtags, compensation (type, payout, CPC rate, max bonus), live duration days, draft turnaround, application deadline, slots, target URL, budget. Shows an escrow requirement summary and a wallet check. Save as draft or publish.
- `/brand/campaigns/[campaignId]` (command center, tabs):
  - Brief: campaign details, edit link, publish and close actions.
  - Applicants: filterable table (niche, follower tier, engagement rate, reliability score) with accept and reject. Clicking a row opens the applicant drawer with media kit, handles, audience, portfolio, reliability breakdown, and pitch.
  - Collaborations: every creator with status badge, days verified progress, next action, and link to the review page when a draft awaits review.
  - Analytics: per-creator table of verified unique clicks, conversions, cost per click (committed payout plus bonus divided by clicks), daily compliance status, and charts (clicks over time, clicks by creator). Includes a total ROI summary.
- `/brand/collaborations/[collabId]/review`: the video review page (section 12.5).
- `/brand/wallet`: balance, locked, and available cards; deposit dialog (simulated); transaction history; invoices table with status filter, PDF download, TDS breakdown, mark-as-paid, and breach resolution actions. Visible to Owner and Finance Reviewer only.
- `/brand/settings/team` (minimal): organization name and plan, invite by email with role, change role, remove member. Owner only.

### 12.4 Creator portal (6)

- `/creator/dashboard`: active deals, pending revision requests, compliance countdowns (Day X/N), upcoming and unlocked payouts, and a notification feed.
- `/creator/marketplace`: search and filters (niche, compensation type, deadline), campaign cards showing payout, deliverables, slots left, and deadline.
- `/creator/campaigns/[campaignId]`: full brief rendered from markdown, deliverables, required mentions and hashtags, payout terms, live duration, and an Apply dialog with a pitch. Shows the application state afterward.
- `/creator/collaborations/[collabId]` (submission and tracking studio): a single workspace with these panels.
  - Milestone stepper and days-verified progress.
  - Draft upload: drag and drop `.mp4`, direct presigned upload to R2 with a progress bar, processing status, version list.
  - Review: read-only player with pinned feedback markers and a comment list; clicking a comment seeks the video.
  - Tracking link: one-click generate, copy button, shows the short URL and UTM URL.
  - Live post: URL input enabled only in APPROVED\_TO\_POST, shows initial-check errors, then the live monitoring log after submission.
- `/creator/earnings`: for each collaboration, the milestone stepper, projected click bonus, invoice status, and PDF download. Totals for locked, unlocked, and paid.
- `/creator/profile`: media kit editor (portfolio, handles, demographics), payout details (masked), and the reliability score with a breakdown of on-time rate and compliance rate.

### 12.5 Video review player

- Custom component around a native HTML5 `<video>` using the preview URL. Custom timeline bar with a scrubber, buffered range, and comment markers positioned by `timestampSec / duration`.
- Clicking the timeline seeks to that time and opens a comment composer pre-filled with the timestamp (shown as mm:ss.ff). Saving pins a marker. Hovering a marker shows the text.
- Frame accuracy: show current time with frame number using the stored fps, support previous and next frame buttons and left and right arrow keys, space for play and pause, and use `requestVideoFrameCallback` where available.
- Version switcher (v1, v2, …) with each version's comments. Comments list with resolve toggle, filter for unresolved, and click-to-seek.
- Actions: Request changes (requires at least one comment) and Approve to post. Only the latest version is actionable, and older versions are read-only.
- The same component renders in read-only mode for creators.

## 13. Integrations

- Cloudflare R2: presigned PUT for uploads and short-lived presigned GET for playback and PDFs. Configure CORS for the web origin. Limits: `MAX_UPLOAD_MB` default 100, MP4 only, keep total storage under 10 GB by deleting raw files after transcoding.
- YouTube Data API v3: `YOUTUBE_API_KEY`. Cache results for at most 30 seconds to protect quota.
- Resend: transactional email for invites, breach alerts, nudges, and invoices. Fall back to console logging in local development.
- Razorpay test mode (P2): deposit checkout only. Label everything as test mode.

## 14. Configuration

Environment variables (validate at startup with zod and fail fast):

- Core: `NODE_ENV`, `PORT`, `WEB_ORIGIN`, `API_ORIGIN`, `SHORT_LINK_BASE_URL`, `JWT_SECRET`, `MONGODB_URI`, `REDIS_URL`.
- Storage: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET`, `MAX_UPLOAD_MB`.
- Integrations: `YOUTUBE_API_KEY`, `RESEND_API_KEY`, `ENABLE_INSTAGRAM`.
- Timing: `VERIFY_INTERVAL_MS` (default 86400000), `VERIFY_CONFIRM_DELAY_MS` (default 60000), `WATCHDOG_INTERVAL_MS` (default 21600000).
- Money: `TDS_RATE` (default 0.10).
- Demo: `DEMO_MODE` (enables the demo post endpoints, verify-now, and simulate deletion) and `NEXT_PUBLIC_DEMO_MODE`.
- Demo mode defaults for the live demo: `VERIFY_INTERVAL_MS=60000`, `liveDurationDays` 3 to 5, so each day passes in about a minute.

## 15. Non-functional requirements

- Security: helmet, CORS limited to `WEB_ORIGIN`, rate limiting on auth and write routes, input validation on every route, no secrets in the repo, hashed invite tokens, masked payout details, signed URLs for private files, authorization checks on every resource.
- Reliability: idempotent workers, graceful shutdown that lets in-flight jobs finish, structured logs, `GET /health`.
- Performance: redirect p95 under 25 ms for cached slugs; list endpoints paginated; dashboards use indexed queries or aggregation.
- Observability: a minimal admin view or log output of queue counts and recent failures (P1).
- Accessibility: keyboard-reachable controls, labels on form fields, and visible focus states, including the video player controls.

## 16. Testing

- Unit and integration tests (Vitest, supertest, mongodb-memory-server) run in CI on every push, with a Redis service container for BullMQ.
- State machine: every allowed transition succeeds, every disallowed transition throws, and role checks are enforced.
- Escrow: lock on accept, insufficient-funds rejection, release with bonus capped and unused amount returned, refund on breach, no negative balances, no double release.
- Verification worker with a fake clock and mocked provider responses:
  - 5-day happy path ends in COMPLETED and UNLOCKED with the correct amounts.
  - 10-day happy path (and a 30-day case) ends the same way.
  - Post deleted on day 3 of 10 leads to BREACHED, a frozen payout, cancelled jobs, and a reliability penalty.
  - Private post and missing hashtag produce the correct breach reasons.
  - A single failed check that passes on re-check does not breach.
  - Transient errors retry and never breach.
  - Duplicate or late jobs do not double count a day or double unlock.
- Watchdog: reminder at 48 hours before the deadline, release at 72 hours overdue, brand nudge after 72 hours, no repeated spam.
- Redirect: unknown slug, UTM parameters appended, unique versus duplicate clicks, bot filtering, and Redis-down fallback.
- Pipeline 4: transcoding job produces a preview and thumbnail from a small fixture video.
- Load test (k6 or autocannon) for `/r/:slug`: record requests per second and p95 latency in the README.
- One end-to-end happy-path test through the API, from registration to an unlocked invoice, using the demo provider.

## 17. Seed data and demo script

### 17.1 Seed script (`pnpm seed`)

- Two organizations. One brand Owner (`owner@demo.com`), one Campaign Manager, and one Finance Reviewer, all with password `Demo@1234`.
- Twenty creators with varied niches, follower counts, engagement rates, and reliability scores. One featured demo creator (`creator@demo.com`).
- Five campaigns across DRAFT, OPEN, ACTIVE, and COMPLETED, plus collaborations in every status.
- Generated click events over the past two weeks for the analytics charts, a few invoices in each status, a funded wallet, and a few drafts with comments (use a small sample video).

### 17.2 Five-minute live demo flow

1. Brand publishes a campaign and accepts the demo creator (escrow locks).
2. Creator uploads a draft; brand adds timestamped comments and requests changes; creator uploads v2; brand approves.
3. Creator generates the tracking link and submits a live post URL (a YouTube Short or the demo post). The initial check passes.
4. Show the progress counter ticking (Day 1/3, Day 2/3) at accelerated intervals.
5. Delete the post. The next check breaks compliance: the brand dashboard shows BREACHED, the payout is frozen, a bell alert appears, and the reliability score drops.
6. Show the 5-day and 10-day happy paths in the passing test run and the invoice PDF from a seeded completed collaboration.

Provide a Check now button and a Simulate deletion button (demo mode only) so the demo never stalls.

## 18. Deployment (free tier only)

- Frontend: Vercel Hobby. Set `NEXT_PUBLIC_API_URL`, `JWT_SECRET`, and the rewrite target.
- Backend VM: Oracle Cloud Always Free (Ubuntu, Docker). Docker Compose services: `api`, `worker`, `redis` (AOF persistence), and `caddy` (HTTPS reverse proxy). The Ampere A1 free allowance is about 2 OCPU and 12 GB, which is enough. Capacity is often unavailable, so retry or start with a smaller shape.
- Database: MongoDB Atlas M0 with the VM IP allowed.
- Files: Cloudflare R2 bucket with CORS for the web origin.
- Domain: a free subdomain (DuckDNS) works; a cheap custom domain with Cloudflare DNS is preferred for the short-link domain.
- Fallback if Oracle is unavailable: a single Render free web service running the API with an in-process worker, an UptimeRobot ping every 10 minutes to prevent sleeping, and a small Redis instance. Document it as a degraded option.
- CI/CD: GitHub Actions runs lint, type-check, and tests, then deploys the API by SSH (`git pull && docker compose up -d --build`). Vercel deploys the frontend from GitHub.
- Provide a `docker-compose.yml` that also runs the full stack locally with seeded data.

## 19. Milestones

1. Foundation: monorepo, shared package, Mongo and Redis connections, auth, role middleware, register, login, onboarding, invites.
2. Campaigns and applications: campaign builder, marketplace, apply, applicant screening, accept and reject with escrow lock, wallet and ledger basics.
3. Collaboration core: state machine module with tests, presigned uploads, transcoding worker, review player, comments, request changes and approve.
4. Attribution: tracking link generator, redirect service, Redis counters, click flushing, load test.
5. Verification and settlement: providers, verification worker, breach handling, settlement, invoice PDF, notifications and Socket.IO.
6. Dashboards and analytics: brand and creator dashboards, analytics tab, earnings, wallet and invoices pages, watchdog worker.
7. Hardening and delivery: seed script, demo mode, full test suite, deployment, README with diagrams and benchmark numbers, demo video.

## 20. Acceptance checklist

- A new brand can register, create an organization, invite a teammate, fund the wallet, and publish a campaign.
- A creator can register, complete onboarding, apply, and be accepted; escrow is locked correctly.
- Draft upload goes directly to R2, produces a preview and thumbnail, and appears in the review player. Timestamped comments appear as pinned markers for the creator. Revisions create v2 and approval is only possible on the latest version.
- The tracking link redirects with UTM parameters, counts unique non-bot clicks, and the analytics tab shows live numbers.
- Submitting a live URL runs the initial check and then scheduled checks. Deleting the post causes a confirmed breach, a frozen payout, a notification, and a reliability score reduction.
- A fully verified collaboration unlocks the invoice with the correct base, click bonus, TDS, and net amounts, and a downloadable PDF.
- The watchdog sends reminders, releases stalled slots, and nudges slow brand reviewers.
- Role permissions are enforced on the API and in the UI. Invalid state transitions return 409.
- All tests pass in CI, the load test results are in the README, and the app is live on free-tier hosting with demo accounts.
