# Onboarding Checklist

Repos (clone)
- Web (Angular): run `git clone git@github.com:BerjisTech/gifter-club.git`
- Android (Kotlin): run `git clone git@github.com:BerjisTech/GiftersClub.git`
- iOS (Swift/SwiftUI): run `git clone https://github.com/BerjisTech/GiftersClub-iOS.git`

Prerequisites
- Supabase project: URL and anon key, Edge Functions enabled.
- AWS: S3 buckets and CloudFront distribution per docs; access via IAM or presigned-function.
- Payments: Flutterwave public key (web), configured Edge Functions for secure processing.
- Tooling: Node 18+ and Angular CLI for web; Android Studio + SDK; Xcode 15+ for iOS.

Environment Configuration
- Angular: set `environment*.ts` with `supabase.url`, `supabase.key`, and any function URLs. Follow media setup in `gifter-club/README.md`.
- Android: set `SupabaseConfig` and `AwsConfig` (REST base, functions base, presign API, CloudFront domain). See `RetrofitClient.kt` for usage.
- iOS: set `SupabaseConfig` equivalents and CDN domain; implement `gifterclub://login-callback` URL scheme.

Secrets (Supabase Edge Functions)
- Run (from `gifter-club`) after creating AWS/media resources:
  - `supabase secrets set S3_BUCKET_PROFILE=<bucket>`
  - `supabase secrets set S3_BUCKET_POST=<bucket>`
  - `supabase secrets set S3_BUCKET_GIFT=<bucket>`
  - `supabase secrets set AWS_REGION=<region>`
  - `supabase secrets set CLOUDFRONT_DOMAIN=cdn.gifters.club`
  - Optional: `supabase secrets set SENDGRID_API_KEY=<key>`; `OPENAI_API_KEY=<key>`

Run Locally
- Angular: `cd gifter-club && yarn && ng serve` then open `http://localhost:4200`.
- Android: `cd GiftersClub`, open in Android Studio, set configs, build and run on device/emulator.
- iOS: `cd GiftersClub-iOS`, open `.xcodeproj`, set configs, run on simulator/device.

Developer Flow
- Make schema/policy/RPC changes in `gifter-club` migrations; deploy; validate on web; mirror models on Android/iOS.
- Use Edge Functions for privileged ops (uploads, gifting, purchases, notifications).

# Platform Quickstarts
- Angular (Web):
  - Config: edit `gifter-club/src/environments/environment*.ts` and set `supabase.url`, `supabase.key`; ensure media/CDN setup per `gifter-club/README.md`.
  - Run: `yarn && ng serve`.
  - Key features to review: `src/app/services/supabase.service.ts`, `src/app/pages/posts/posts-page.component.ts`, `src/app/components/ui/toast*.ts`.
- Android (Kotlin):
  - Supabase: `GiftersClub/app/src/main/java/club/gifters/giftersclub/SupabaseConfig.kt` set `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `REDIRECT_URI`, and `FLUTTERWAVE_PUBLIC_KEY`.
  - AWS/Media: `GiftersClub/app/src/main/java/club/gifters/giftersclub/AwsConfig.kt` set `API_URL` (presign endpoint), `REGION`, bucket names, and `CLOUDFRONT_DOMAIN`.
  - Networking: see `network/RetrofitClient.kt`, `network/PostApi.kt`, `network/FunctionsApi.kt`.
  - Create Post: `gifts/CreatePostFragment.kt` for camera/filters/publish.
- iOS (Swift/SwiftUI):
  - Supabase: `GiftersClub-iOS/GiftersClub/Sources/Config/SupabaseConfig.swift` set `url`, `anonKey`, and `redirectURL`.
  - UI spec: `GiftersClub-iOS/UI.md` for tabs, banners, drawers, gradients, create-post UX.
  - Auth/Client: `GiftersClub-iOS/GiftersClub/Sources/Services/SupabaseService.swift`.

# Edge Functions Catalog (Supabase)
- `upload-media` (POST): issues presigned PUT URL for S3 uploads.
  - Body: `{ fileName, fileType, bucket: 'profile'|'post'|'gift', overwrite?: boolean }`
  - Returns: `{ uploadUrl, key, publicUrl }`.
- `send-gift` (POST): securely sends a gift and records transactions/notifications.
  - Body: `{ gift_id, sender_id, recipient_id, tokens, reference_id? }`
- `purchase-tokens` (POST): records purchase of tokens after PSP callback.
  - Body: `{ user_id, tokens, transaction_reference }`
- `subscribe-creator` (POST): subscribes a user to a creator.
  - Body: `{ creator_id, subscriber_id, tokens, duration_type, tx_ref }`
- `purchase-post-access` (POST): buys access to a paid post.
  - Body: `{ post_id, user_id, price, tx_ref }`
- `send-notification-email` (POST): sends transactional email.
  - Body: `{ user_id, type, reference_id, message, templateData? }`
- `log-post-view` (POST): records a post view and optional duration.
  - Body: `{ post_id, user_id?, view_duration? }`
- `live-session` (POST/GET/PATCH): creates, fetches, updates live stream session metadata.
  - POST body example: `{ title, description?, mode, host_id, scheduled_at? }`
  - PATCH body: partial `{ title?, status?, ... }` with `?id` query.
- `update-interaction-settings` (POST): updates privacy/moderation settings.
  - Body: `{ user_id, settings: {...} }`
- `auth-log` (POST): records sign-in/auth events.
  - Body: `{ user_id, platform, version?, timestamp? }`
- `admin-reject-withdrawal` (POST, admin): rejects a withdrawal, logs audit, refunds tokens.
  - Body: `{ withdrawalId, notes }` (requires admin role).
- `admin-process-withdrawal` (POST, admin): processes a withdrawal payout; sends notifications.
  - Body: `{ withdrawalId }` (requires admin role).

# GiftersClub – Comprehensive Project Overview

This monorepo hosts the full GiftersClub product across Web (Angular), Android (Kotlin), and iOS (Swift/SwiftUI). Angular is the system of record for backend-facing changes (Supabase migrations, policies, Edge Functions, and operational docs). Mobile apps implement native UX while consuming the same Supabase backend and AWS media stack.

Contents
- Repo Layout and Roles
- Platform Stack (Supabase, AWS CloudFront/S3, Payments, Push)
- Authentication and Profiles
- Core Domain: Posts, Media, Comments, Reactions, Tags
- Gifting and Wallet (tokens, purchases, withdrawals)
- Feeds and Search (ranking, tabs, analytics)
- Live Streaming (plan and integration points)
- Admin/Moderation
- Mobile Architecture (Kotlin), Web Architecture (Angular), iOS Notes
- Developer Onboarding & Environments
- References (code paths in Angular/Kotlin)

----------------------------------------------------------------

Repo Layout and Roles
- `gifter-club/` (Angular Web PWA)
  - Source of truth for Supabase DB schema/migrations, RLS policies, and Edge Functions.
  - Shared data models and API semantics; reference implementations of flows (posting, gifting, admin).
  - Docs: media migration to AWS, feed scoring, policies, live-stream plan, etc.
- `GiftersClub/` (Android, Kotlin)
  - Native UI/UX for posts, create-post camera, chat, explore, profile, gifting, withdrawals, etc.
  - Retrofit-based clients for Supabase PostgREST and Edge Functions; S3 uploads via presigned URLs.
- `GiftersClub-iOS/` (iOS, Swift/SwiftUI)
  - Native UI aligned with Angular/Kotlin patterns; see `GiftersClub-iOS/UI.md` for detailed UI/UX spec.

----------------------------------------------------------------

Platform Stack
- Supabase (DB/Auth/Realtime/Edge)
  - Postgres with RLS; migrations and RPCs live under `gifter-club/migrations/`.
  - Edge Functions for secure operations (gifting, purchases, uploads, notifications, auth logs, etc.).
  - Auth: Google OAuth; JWT propagated to clients and functions.
- AWS for Media Delivery
  - S3 buckets: profile avatars, post media, gift images (see `gifter-club/policies.md`).
  - CloudFront CDN: `cdn.gifters.club` with behaviors for `/avatars/*`, `/posts/*`, `/gifts/*`.
  - Uploads via presigned PUT URLs from Edge Function `upload-media` (Angular and Android use this).
  - CORS applied to buckets; cache policies tuned for avatars vs. posts (see `gifter-club/README.md`).
- Payments
  - Angular uses Flutterwave (inline) for token top-ups within gifting flow (`send.component.ts`).
  - Server-side reconciliation via Edge Functions (records token transactions and gift sends).
- Push and Email
  - Email: SendGrid via Edge Function `send-notification-email`.
  - Mobile push: APNs (iOS) and FCM (Android) planned; notifications table in DB mirrors in-app alerts.

----------------------------------------------------------------

Authentication and Profiles
- Google OAuth is the primary provider across platforms.
  - Angular: `SupabaseService.signInWithGoogle` with redirect to `/dashboard` on web.
  - Android: tokens persisted in SharedPreferences, refreshed by `TokenRefreshAuthenticator`.
  - iOS: custom scheme `gifterclub://login-callback` handles OAuth return.
- New-user bootstrap:
  - On sign-in, clients ensure a `profiles` row exists and update email/metadata
    (Angular: `supabase.service.ts` → `handleProfile`; Kotlin mirrors this via REST calls).
- Deep links
  - Profiles: `/u/{username}` (and `/g/{username}`); Wishlists: `/wishlist/{id}`; handled per-platform (Angular router, Android intent filters, iOS URL scheme).

----------------------------------------------------------------

Core Domain
Tables (representative; see migrations for full details)
- `profiles`: user metadata, avatar URL (S3/CloudFront), counts, gifter level.
- `posts`: user posts with `access_type` (free/subscription/paid), price (for paid), optional product fields (see `post-as-product.md`).
- `post_media`: ordered media per post (photo/video) with S3/CloudFront URLs.
- `comments`, `post_reactions` (likes), `tags`, `post_tags`.
- `search_queries`: tracks typed queries, suggestion clicks, result clicks.
- `notifications`: in-app notifications; email dispatch triggered via Edge Function.
- `follows`: follower/following graph.
- `live_streams`, `live_streams_with_stats` (viewers/comments counts), plus auxiliary tables.

Media Uploads and Delivery
- Presigned uploads via Edge Function `upload-media` return `{ uploadUrl, publicUrl }`.
  - Angular: `SupabaseService.uploadProfileImage` PUTs the file to S3, then saves URL.
  - Android: `CreatePostFragment` presigns per media, uploads via OkHttp, then creates `post_media` rows.
- Delivery through CloudFront `cdn.gifters.club` with behaviors and cache settings documented in `policies.md` and `README.md`.

----------------------------------------------------------------

Gifting and Wallet
Concepts
- Gifts catalog (`gifts`) with token prices and popularity flags.
- Token balance on profile; token transactions for purchases/withdrawals.
- Gift sending creates `gift_sent` records and notifications (sender/recipient).

Angular flow (reference: `src/app/components/gifts/send/send.component.ts`)
- If user has sufficient tokens: call Edge Function to process gift send (`processGiftSendRpc`) and show success toast.
- If insufficient tokens: trigger Flutterwave inline payment, record token transaction as `initiated`, then on success:
  - Edge Function processes token purchase (`processPurchaseTokensRpc`) → then gift send (`processGiftSendRpc`).
  - Create notifications for both sender and recipient; show success toast.

Android flow (Kotlin)
- Retrofit clients call Supabase REST for catalog and balances, and Edge Functions for secure workflows (see `FunctionsApi` for `send-gift`, `purchase-tokens`).
- UI mirrors Angular: optimistic UI updates; background processing; top banners/snackbars per platform.

Withdrawals (Admin + User)
- Users submit withdrawal requests via RPC `request_withdrawal` with payment method/details; visible in Angular Admin.
- Admin UI (Angular: `admin/withdrawals/`) can process or reject withdrawals via Edge Functions, sending notifications and generating CSV exports.

----------------------------------------------------------------

Feeds and Search
Feed retrieval
- Angular: `supabase.service.getFeedPosts` and feed UI in `pages/posts/posts-page.component.ts`.
- Android: `PostApi.getFeedPosts` (RPC) with `FeedPost` models.

Ranking and behavior
- Scoring logic and roadmap documented in `post-feed-weight-calculation.md`:
  - Social proximity, engagement (likes/comments/shares), author popularity, hashtag affinity, view freshness penalty, plus random jitter per load.
  - Interleaving by author and deterministic ordering tuned by jitter.
- Optimistic interactions (like/comment/delete), refresh affected items and periodically refresh the feed.

Search
- Tabs: Top, Users, Videos, Photos, Live (Angular posts page implements this; Android explore implements similar categories).
- Typeahead suggestions from `search_queries` with click-through analytics; results recorded via `recordSearchEvent`.

----------------------------------------------------------------

Live Streaming
- Strategic plan in `live-stream-plan.md` (WebRTC SFU focus, HLS/LL-HLS fallback, managed services recommended initially).
- Data: `live_streams` + `live_streams_with_stats`, viewers/comments; Edge Functions manage session CRUD.
- Android: `LiveStreamActivity` uses CameraX; integrates with live session endpoints.
- Notifications: push when creators go live (APNs/FCM queues) and in-app displays.

----------------------------------------------------------------

Admin and Moderation
- Angular Admin pages (`src/app/pages/admin/*`):
  - Withdrawals management (approve/reject/export CSV, notify users).
  - Content, Reports, Users, Gifts dashboards (stubs or implemented depending on branch).
- RLS policies in migrations enforce per-role access; admin actions secured by Edge Functions when needed.

----------------------------------------------------------------

Mobile Architecture (Kotlin)
- Networking
  - Supabase REST via Retrofit with auth headers and token refresh (`RetrofitClient`, `TokenRefreshAuthenticator`).
  - Edge Functions via `/functions/v1` Retrofit client (`FunctionsApi`) for gifting, purchases, uploads, notifications, auth logs, live sessions.
  - S3 presign and upload flow used in `CreatePostFragment` for media.
- Create Post
  - `gifts/CreatePostFragment.kt` implements camera (CameraX), top-right controls (switch/flash/timer/filters), dual-row bottom controls, text post editor, filters, and background publishing (`CreatePostRequest` + `CreatePostMediaRequest`).
  - On success: navigate back to feed and show banner/Toast; on error: show error and keep state for retry.
- Explore & Social
  - `PostApi`, `ProfileApi`, follows, recent gifters, followers/following fragments; chat; notifications.

Web Architecture (Angular)
- Supabase service (`src/app/services/supabase.service.ts`) centralizes auth bootstrap, profile handling, data APIs (posts, tags, search, gifts, withdrawals, notifications, etc.).
- UI system: shimmer skeletons; toast container/service (`components/ui/toast*.ts`) used across app.
- Posts page: feed UI, search tabs, optimistic updates, and scrolling gestures.

iOS Notes
- UI/UX guide in `GiftersClub-iOS/UI.md` defines top tabs, search icon placement, toasts as bottom drawers, gradient buttons with animation, create-post multi-step flow, and optimistic actions.
- Swift/SwiftUI stack mirrors Kotlin/Angular semantics; implement shared token names for gradients and radii.

----------------------------------------------------------------

Developer Onboarding
1) Read docs in `gifter-club/` to understand backend and ops
   - Media stack and setup: `README.md`, `policies.md`, `supabas_to_aws_migration.md`
   - Feed ranking: `post-feed-weight-calculation.md`
   - Posts-as-Product: `post-as-product.md`
   - Live streaming: `live-stream-plan.md`
2) Configure environments
   - Supabase URL and anon key in each platform:
     - Angular: `src/environments/environment*.ts`
     - Android: `SupabaseConfig`/`RetrofitClient`
     - iOS: `SupabaseConfig`
   - AWS CloudFront domain and S3 bucket names in Edge Function secrets (Angular README includes commands)
3) Run locally
   - Angular: `yarn && ng serve` (or `npm run start`) and ensure Supabase connects
   - Android: open `GiftersClub` in Android Studio; set `SupabaseConfig` and `AwsConfig`
   - iOS: open `GiftersClub-iOS` in Xcode; set `SupabaseConfig`
4) Development flow
   - Propose DB/schema/RPC changes via `gifter-club` migrations; deploy and validate on web; then mirror models on mobile.

Operational Notes
- Secrets: store in Supabase secrets for Edge Functions (S3 bucket names, region, CloudFront domain, SendGrid, etc.).
- CORS: apply `cors.json` to S3 buckets used for direct PUTs.
- Cache invalidation: invalidate CloudFront on avatar/media replacements as needed (admin tooling or automated on upload).

----------------------------------------------------------------

References (Code Pointers)
- Angular
  - Media + CDN setup: `gifter-club/README.md`, `gifter-club/policies.md`, `gifter-club/supabas_to_aws_migration.md`
  - Feed/search UI and behavior: `src/app/pages/posts/posts-page.component.ts`
  - Toasts/Shimmer UI: `src/app/components/ui/toast*.ts`, `src/app/components/ui/shimmer/*`
  - Gifting flow (Flutterwave + Edge Functions): `src/app/components/gifts/send/send.component.ts`
  - Admin withdrawals: `src/app/pages/admin/withdrawals/*`
  - Supabase service APIs: `src/app/services/supabase.service.ts`
- Android (Kotlin)
  - Retrofit + auth + functions: `network/RetrofitClient.kt`, `network/PostApi.kt`, `network/FunctionsApi.kt`
  - Create Post camera and publishing: `gifts/CreatePostFragment.kt`, `model/PostRequests.kt`
  - Social lists and adapters: `social/*`, `explore/*`
- iOS
  - UI/UX specification: `GiftersClub-iOS/UI.md`

This overview is intended to onboard any developer (Angular, Kotlin, Swift) by providing architecture, stack, and concrete code references for each major feature.

## Gifts
- 1 token = 1 kes
- 

## Wishlists
- 

## Withdrawals
- 70% of requested amount eg request 500 get 350, flutterwave cost 100 and our profit 50

## Posts
- Can be video or photo
- Can be free for all, subscriptions only or paid (one time charge)


## Soft deletions
- Posts are moved to public.deleted_post, wishlists are moved to public.deleted_wishlists

## Profile
- image, name, username : refer public.profiles
