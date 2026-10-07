# BrandRelay (VelorQ): Bugs & Issues Report, File by File

**Date:** 2026-10-06  ·  **Branch:** `feature/duplicate-main`  ·  **Mode:** read-only audit, no code changed
**Companion doc:** [PRODUCTION_READINESS.md](PRODUCTION_READINESS.md) covers deployment and architecture. This file lists concrete bugs and the change each one needs.

---

## How to read this

| Severity | Meaning |
|---|---|
| **CRITICAL** | Security hole or data leak. Fix before any public launch. |
| **HIGH** | Feature is broken, or users see false or wrong information. |
| **MEDIUM** | Wrong behaviour in some cases, performance, or cost. |
| **LOW** | Code quality, dead code, lint, small UX issues. |

Every entry gives `file:line` (approximate, as of this audit), what is wrong, and what to change.

**Coverage:**
- **Read line by line:** all backend routes and services, `models.py`, `dependencies.py`, `main.py`; the frontend app shell (`src/app/*`), services, utils, and every page with business logic.
- **Checked by pattern scan, lint, and build:** the 36 CSS files, landing and marketing components, `src/data/*`, schemas, and migrations.
- **Tests:** the backend suite was run (182/186 pass); oxlint and `vite build` were run.

---

## 1. Summary (approximate counts)

| Area | Critical | High | Medium | Low |
|---|---|---|---|---|
| Backend: security & auth | 7 | 4 | 6 | 2 |
| Backend: features & data | — | 9 | 18 | 8 |
| Frontend: data correctness | — | 14 | 15 | 6 |
| Frontend: UX, dead code, lint | — | 2 | 8 | 20+ |
| Styles / CSS | — | — | 4 | 3 |
| Repo / tests / config | 2 | 1 | 3 | 3 |

**The pattern behind most bugs.** Many frontend services hide failures. When an API call fails, they quietly save to `localStorage`, use bundled demo data, or show a success message anyway. Users are then told things worked when they did not. This affects publishing, calendar, team members, analytics, integrations and billing. Fixing this pattern alone removes about 15 of the High bugs.

---

## 2. Fix these first (top 25)

| # | Sev | Where | Problem |
|---|---|---|---|
| 1 | CRITICAL | `backend/data/*.json` | Real Meta `EAA…` access tokens are committed and pushed to GitHub. Rotate them, then purge them from history. |
| 2 | CRITICAL | ~15 route files | The user is identified by an `email` sent from the browser instead of the session cookie. Anyone can read or change anyone's data. |
| 3 | CRITICAL | `routes/settings.py:297` | The account-deletion OTP is **always** returned in the API response, and the UI displays it. |
| 4 | CRITICAL | `services/social_publisher.py:166,216-231` | Takes a webhook URL from the client and attaches the server's Meta tokens to the request. Unauthenticated. |
| 5 | CRITICAL | `services/cloudinary_service.py:59-82` | Uploads any local file path to Cloudinary. Unauthenticated. |
| 6 | CRITICAL | `routes/auth.py:57`, `services/auth.py:22,115` | Login OTP is returned when `APP_ENV` is unset, logged in plain text, never emailed. `AUTH_SECRET` falls back to a default. |
| 7 | CRITICAL | `services/email.py` | User text is put into email HTML without escaping, so anyone can inject content into emails sent from your domain. |
| 8 | HIGH | `routes/reviews.py:93` | The review link in emails is built from the request's `Origin` header, so a branded email can point to any site. |
| 9 | HIGH | `pages/CreativeReview.jsx:61-75` | The "Looks Good" email link auto-approves when the page loads, so email link scanners approve creatives on their own. |
| 10 | HIGH | `pages/Publish.jsx:421-437` | Shows "Published successfully" even when posting to Instagram/Facebook failed. |
| 11 | HIGH | `pages/Publish.jsx:470-490` | The "Load Free Test Creative" button posts a stock photo to the real connected brand page. |
| 12 | HIGH | `routes/workspaces.py` (`list_current_workspace_members`) | Tuple unpacked in the wrong order, so **Settings → Team is always empty**. |
| 13 | HIGH | `pages/CompanyProfile.jsx:155-178` | Every profile save creates a **new workspace** and switches the user into it. |
| 14 | HIGH | `pages/CreativeStudio.jsx:567` | If Market Intelligence suggests "image", Creative Studio is a dead end: its button loops back to the same page. |
| 15 | HIGH | `services/composition.py:234` + `profile.py` | The brand logo is **never** drawn on generated ads (wrong path, and the asset list is never passed in). |
| 16 | HIGH | `services/composition_pipeline.py:76` | Brand context is cut at 4,000 characters, so the JSON is invalid and brand colors/fonts are silently dropped for larger profiles. |
| 17 | HIGH | `utils/campaignFlow.js:167-205` | After any page refresh, the stored image is a truncated "stripped" string. Scheduled posts are then saved with a broken image. |
| 18 | HIGH | `services/calendarService.js:125-168` | When saving fails (including permission denied), the app invents a local event and shows success. |
| 19 | HIGH | `services/workspaceMembersService.js` | Adding, changing or removing members fails silently: changes are saved to localStorage and shown as done. |
| 20 | HIGH | `services/socialInsightsService.js:55,191,207` | When the API fails, Analytics shows bundled sample data as if it were the customer's, labelled "Live". |
| 21 | HIGH | `pages/CreativeEditor.jsx:38,805` & `pages/BrandKit.jsx:102,196` | Customers are shown **VelorQ's own logos and demo assets** as if they were their brand kit. |
| 22 | HIGH | `services/quality_validation.py:43,141` + `layout_templates.py` | The legacy layout uses key `supporting` but validation reads `supporting_message`, so generation crashes with a KeyError. |
| 23 | HIGH | `services/campaign_status.py:253-260` | "Technical quality" values (resolution, file size) are **made up by the LLM** and shown as measurements. |
| 24 | HIGH | `routes/campaign_status.py:191` + `scheduler.py` | "Publish" does not cancel a pending scheduled post, so the same creative is posted twice. |
| 25 | HIGH | `routes/security.py:43,73` | 2FA can be disabled or reset with only a session, without a TOTP code. |

---

## 3. Backend: file by file

### `backend/app/main.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| ~40 | MEDIUM | `Base.metadata.create_all()` runs at import alongside Alembic, so the schema drifts. | Remove it; run `alembic upgrade head` on deploy. |
| ~45 | MEDIUM | The scheduler loop runs inside the web process. With several workers it duplicates publishes. | Run it as a separate single worker. |
| ~68 | HIGH | `/uploads` is served publicly with no auth, including support screenshots and SVGs. | Use object storage on a separate domain; make support files private. |
| ~100 | MEDIUM | CORS allows localhost only, hard-coded. | Read `CORS_ORIGINS` from env. |
| — | LOW | Env-dependent module constants rely on whichever module calls `load_dotenv` first. | Load config once at startup (pydantic-settings). |

### `backend/app/api/dependencies.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 35 | MEDIUM | Writes `last_seen_at` and commits on **every** authenticated request. | Update at most once every ~5 minutes. |

### `backend/app/api/routes/auth.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 57 | CRITICAL | `APP_ENV` defaults to `development`, so the OTP is returned in the response. | Default to production; never return the OTP. |
| 48 | HIGH | No rate limit on `request-otp` or `verify-otp`. Requesting a new OTP resets the attempt counter, so codes can be brute-forced. | Rate-limit per email and per IP. |
| 48 | MEDIUM | Logging in with an unknown email silently creates an account. | Separate sign-up from sign-in, or confirm before creating. |

### `backend/app/services/auth.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 22 | CRITICAL | `AUTH_SECRET` falls back to `"velorq-development-auth-secret"`. | Refuse to start without it in production. |
| 115 | CRITICAL | Logs every OTP in plain text. | Delete the log line. |
| 94 | CRITICAL | The OTP is never emailed; there is no send call and no template. | Send it with `services/email.py`. |
| — | LOW | `ensure_user_workspace()` exists but is never called. | Call it after successful OTP verification (replaces three frontend copies). |

### `backend/app/api/routes/settings.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 297 | CRITICAL | `OtpRequestResponse(development_otp=code, otp=code)` is returned in every environment, ignoring line 277. | Return nothing; email the code only. |
| 283, 291 | LOW | The email says the code expires in 10 minutes; the real default is 5. | Use `OTP_EXPIRY_MINUTES`. |
| 362 | MEDIUM | After `db.rollback()`, `current_user.avatar_url` reloads the **old** value, so the code deletes the previous avatar and leaves an orphan file. | Delete `f"/uploads/{storage_key}"`. |
| 199 | MEDIUM | Data export is incomplete: no support tickets, chats, feedback, scheduled posts or calendar events. It also includes all base64 media in memory. | Add the missing tables; export media as links; run as a background job. |
| 300 | MEDIUM | Soft delete keeps all personal data, and the email stays reserved, so the user can never sign up again. | Add a hard-delete or anonymise job with a retention period. |

### `backend/app/api/routes/security.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 73 | HIGH | `/2fa/disable` needs no TOTP code. | Require the current TOTP code or a re-authentication. |
| 43 | HIGH | `/2fa/setup` while 2FA is on overwrites the secret and disables 2FA. | Block it while enabled. |
| — | MEDIUM | TOTP secret stored in plain text; no replay protection; other sessions stay logged in when 2FA is enabled. | Encrypt the secret; store the last used step; revoke other sessions. |

### `backend/app/api/routes/workspaces.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| `list_current_workspace_members` | HIGH | `_, workspace = _get_authenticated_workspace(...)` takes the membership instead of the workspace, so the team list is **always empty**. No test covers it. | Use `workspace, _ = ...` and add a test. |
| `create_workspace_member` | HIGH | Legacy endpoint identifies the caller by email; allows `role="owner"`. | Remove it, or require session auth and block `owner`. |
| member lookups | MEDIUM | `member_email` is not lower-cased, so `Bob@x.com` is not found. | Normalise emails. |
| member lookups | MEDIUM | "Member user not found" reveals which emails have accounts; members are added without an invite or consent. | Use an invite flow with email acceptance. |
| `create_workspace_endpoint` | HIGH | Identifies the user by email and creates users for any address. | Use the session. |
| `list_workspaces`, `retrieve_workspace`, `list_workspace_members` | HIGH | Identified by email. | Use the session. |

### `backend/app/api/routes/profile.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| all | CRITICAL | Every brand profile, kit and asset endpoint identifies the user by email: anyone can read, overwrite or delete them. | Use `Depends(get_current_user)`. |
| 154 | MEDIUM | PUT `/brand` replaces the whole `profile_data`, losing `brand_tone` written by the kit settings; empty colors wipe the kit colors. | Merge instead of replace. |
| GET routes | MEDIUM | GET `/brand` and `/brand/kit` create users and commit, so reads have side effects. | Read-only lookup. |
| assets | HIGH | Logos are stored only in the `BrandAsset` table, but the ad renderer reads `profile_data.brand_kit.assets`, which is never written. | Pass the real logo assets into the generation context. |

### `backend/app/api/routes/campaign_status.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| all | CRITICAL | Identified by email (select, approve, **publish**, **schedule**, regenerate). | Use the session. |
| 191 | HIGH | `/publish` only sets the database status. It doesn't post, and it doesn't cancel a pending `ScheduledPost`. | Cancel scheduled posts; make publish do the real dispatch, or rename it. |
| 174 | MEDIUM | Approve has no status check: it works from draft without a compliance check or review, and flips "published" back to "approved". | Enforce the allowed status transitions. |
| 132 | MEDIUM | Returns `str(error)` to the client. | Return a generic message. |
| 267-370 | MEDIUM | Regenerating a video runs 2+ Veo jobs synchronously inside the request. | Use a background job. |

### `backend/app/services/persistence.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 32 | HIGH | `get_or_create_user` ignores `is_deleted`, so deleted accounts are reused by every email-based route. A race on the unique email causes a 500. | Exclude deleted users; handle `IntegrityError`. |
| 592 | MEDIUM | `approve_campaign_creative` only checks `profile.asset_id`, which is always auto-generated, so the gate is meaningless. | Check compliance and review status. |
| 224, 228, 241 | HIGH | `_truncate_json_text` cuts JSON mid-token, sending invalid JSON to the LLM and the renderer. | Trim fields before serialising. |
| 311 | MEDIUM | Full base64 images and videos are stored in `MediaVariant.media_url`, and again inside `campaign_data`. | Use object storage and store URLs. |
| 429, 508 | LOW | Deprecated `datetime.utcnow()`; naive and timezone-aware datetimes are mixed. | Use `datetime.now(timezone.utc)`. |
| — | LOW | `MarketIntelligenceRecord` grows forever. | Add a retention policy. |

### `backend/app/services/scheduler.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 147 | HIGH | No row locking, so two or more workers publish the same post. Posts stuck in `processing` are never retried. | `FOR UPDATE SKIP LOCKED`, plus a timeout and retry. |
| 96 | MEDIUM | The calendar event is found by **title**, so campaigns with the same headline overwrite each other's events. | Link the event by `scheduled_post_id`. |
| 215 | MEDIUM | On publish, it marks the **first** due "scheduled" event in the workspace, not this post's event. | Same: use the direct link. |
| 147 | MEDIUM | Synchronous DB queries inside the async loop block the server. | Use a worker process or `to_thread`. |
| 68 | LOW | Workspace is the first membership, not the active one. | Pass the workspace context. |
| 187 | MEDIUM | Any HTTP 2xx from n8n counts as "published", even if posting to Meta failed. | Have n8n return the Meta post ID and check it. |

### `backend/app/services/social_publisher.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 166, 374 | CRITICAL | Client-supplied `webhook_url` lets callers make the server send requests anywhere (SSRF). | Read the URL from env only. |
| 216-231 | CRITICAL | Sends the server's Meta tokens to that URL. | Remove token fields from the client API. |
| 111, 115 | HIGH | Auto-fit opens local file paths and fetches any URL; base64 is decoded with no size limit. | Restrict input sources; enforce size and pixel limits. |
| 189, 275, 289 | MEDIUM | Errors expose the internal n8n URL and exception text. | Return generic messages. |
| 311 | HIGH | One cache file inside `app/data` is shared by **all tenants**, written non-atomically, and fails on a read-only filesystem. | Per-workspace cache in the DB. |
| 341 | LOW | `int(p.get("likes"))` crashes on non-numeric values. | Parse safely. |

### `backend/app/api/routes/social_publisher.py`, `analytics.py`, `cloudinary.py`
| Sev | Issue | Change |
|---|---|---|
| CRITICAL | No authentication on any endpoint. | Require the session and workspace. |
| HIGH | Analytics serves one global n8n account's data to every user, and `/refresh` triggers Meta API calls. | Scope per workspace; rate-limit. |
| MEDIUM | `analytics.py:236` and `cloudinary.py:62,105` return `str(err)`. | Return generic errors. |
| LOW | `platform` isn't validated; at most 100 posts are read, so totals undercount. | Validate the parameter; paginate. |

### `backend/app/services/cloudinary_service.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 59-82 | CRITICAL | `resolve_local_file_path` accepts any path, including `/uploads/../`. | Only data URIs, or paths that resolve inside `uploads/`. |

### `backend/app/services/composition.py` (ad renderer)
| Line | Sev | Issue | Change |
|---|---|---|---|
| 234 | HIGH | `/uploads/x` is mapped to `backend/x`, missing the `uploads` folder, so logos never load. | Use `parents[2] / "uploads" / ...`. |
| 226, 235 | HIGH | `requests.get` on a URL from the brand context (SSRF), and `Image.open` on any local path. | Load only from your own storage. |
| 296 | LOW | `headline_case="title"` is never applied. | Apply title-casing. |
| 305-314 | MEDIUM | Pure-Python pixel averaging runs about 9 times per creative, which is slow. | Use `ImageStat` or downscale first. |
| 403-410 | LOW | Identical if/else branches. | Merge them. |

### `backend/app/services/composition_pipeline.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 76 | HIGH | `json.loads` on brand context truncated to 4,000 characters fails, and the brand is silently ignored. | Pass a dict, not truncated JSON. |
| 57, 69 | MEDIUM | KeyError when the LLM returns mismatched IDs. | Validate the ID sets match. |
| 120 | MEDIUM | Quality issues are only logged; bad compositions are still returned. | Surface a quality flag to the UI. |

### `backend/app/services/layout_templates.py` / `quality_validation.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| QV 43, 141 | HIGH | The legacy template key is `supporting`, but validation reads `supporting_message`, causing a KeyError. | Use one key name. |
| QV 61 | MEDIUM | The overlap check only flags *identical* boxes, never overlapping ones. | Test for rectangle intersection. |
| LT 341 | MEDIUM | The upper zones overlap, and the LLM can put two text blocks in the same zone. | Reject or reassign duplicate zones. |

### `backend/app/services/fonts.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 16-26 | HIGH | Only Windows font paths. `app/assets/fonts` contains only a README, so on a Linux server every ad uses Pillow's default font. | Bundle TTF files (Playfair, Inter). |
| 40 | LOW | `Path(value)` from the brand context loads an arbitrary file path. | Only allow bundled font names. |

### `backend/app/services/campaign_status.py` (compliance)
| Line | Sev | Issue | Change |
|---|---|---|---|
| 253-260 | HIGH | Resolution, file size and "loading optimized" are LLM guesses ("plausible value") shown as facts. | Compute them from the image bytes. |
| 225 | MEDIUM | The compliance check never sees the rendered image, only the prompt text. | Label it clearly, or use a vision model. |
| 162 | MEDIUM | The status response always includes the full base64 images. | Return URLs. |

### Other AI services
| File:Line | Sev | Issue | Change |
|---|---|---|---|
| `ad_studio.py:141` | MEDIUM | `candidate.content` can be `None` when Gemini's safety filter blocks the image, causing an AttributeError and a generic 502. | Detect safety blocks and show a clear message. |
| `ad_studio.py:126` | MEDIUM | Images are generated one after another, with no timeout or retry. | Run in parallel, with a timeout. |
| `art_direction.py:79, 290` | MEDIUM | Prompt says "pick two" when count > 2; count and ID uniqueness aren't enforced; duplicate keys waste tokens. | Validate the count; trim the schema. |
| `copy_strategy.py:28` | LOW | `generated_data["headline"]` can raise KeyError, and there is no length check. | Validate with Pydantic. |
| `video_prompt.py:58` | LOW | `["prompt"]` isn't validated. | Validate. |
| `media_prompts.py:19`, `prompt_writer.py` | LOW | KeyError when IDs don't match. | Validate. |
| `creative_brief.py:36` | LOW | Always produces 2 briefs, ignoring `variant_count` (legacy endpoint). | Use the count, or remove the endpoint. |
| `video_generation.py:31` | LOW | Default model differs from `.env.example`. | Align them. |
| `video_generation.py:47` | HIGH | `time.sleep` polling for up to 10 minutes **per video** inside the HTTP request. | Use a job queue and have the frontend poll. |
| `website_watch.py:23` | HIGH | Fetches any URL with redirects followed (SSRF); downloads an unbounded body. | Block private IPs; cap the size. |
| `deepseek.py:40` | HIGH | Synchronous 90-second call made from `async def` routes, blocking the event loop. | Use plain `def` routes or `httpx.AsyncClient`. |

### Remaining route files
| File:Line | Sev | Issue | Change |
|---|---|---|---|
| `ad_studio.py`, `video_studio.py`, `media_prompts.py`, `creative_faqs.py`, `suggestions.py`, `website_watch.py`, `market_intelligence.py` | CRITICAL | Identified by email. | Use the session. |
| same | HIGH | `async def` handlers make blocking AI calls. | Use `def` handlers or async clients. |
| `video_studio.py:44` | HIGH | Always runs at least 2 sequential Veo jobs; the whole MP4 is returned as base64 JSON and stored in the DB. | Generate 1 video by default; use storage. |
| `video_studio.py:89` | MEDIUM | Raw `RuntimeError` text is returned to the user. | Return a generic message. |
| `media_prompts.py:41`, `creative_faqs.py:41` | LOW | Every exception is reported as "DeepSeek returned invalid data", hiding real bugs. | Catch specific exceptions. |
| `whats_new.py:38` | MEDIUM | A GET creates users; identified by email; read-all races on the unique constraint. | Session auth; upsert. |
| `help_center.py:47` | HIGH | Support request is unauthenticated and creates a user for any email (spam). | Require the session. |
| `reviews.py:93` | HIGH | Link base URL comes from the `Origin` header. | Use a `FRONTEND_URL` env var. |
| `reviews.py:89` | LOW | Self-review check is case-sensitive. | Normalise emails. |
| `reviews.py` (owner routes) | CRITICAL | Identified by `owner_email` or the `email` query parameter. | Use the session. |
| `assistant.py:43` | MEDIUM | Intent keywords are too narrow; 2 tests fail. | Widen the keyword lists or use an LLM. |
| `assistant.py:192` | LOW | Reads two JSON files from disk on every message. | Cache them. |
| `assistant.py:308` | HIGH | Support screenshots are publicly served from `/uploads/support`. | Private storage with signed URLs. |
| `assistant.py:469` | LOW | Admin users list runs 2 extra queries per user and nothing is paginated. | Aggregate in one query; paginate. |
| `assistant.py:378` | MEDIUM | An admin reply never notifies the user. | Email or in-app notification. |
| `calendar.py` | MEDIUM | Moving or deleting a social event doesn't change its `ScheduledPost`; listing returns everything. | Link the two; require a date range. |

### `backend/app/services/reviews.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 332 | MEDIUM | A reviewer can change their decision any number of times; there's no rate limit; the submitter can overwrite `reviewer_name`. | Lock the decision once submitted (or version it); rate-limit. |
| — | MEDIUM | Reviewer decisions never affect campaign status. | Block approval while changes are requested (optional setting). |
| 141 | LOW | The full base64 image is embedded in the email, which Gmail clips. | Link to a hosted image. |

### `backend/app/services/email.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 148, 177, 185, 211, 215 | CRITICAL | No `html.escape()` on user text in the HTML. | Escape every interpolated value. |
| 49 | MEDIUM | Returns `True` when SMTP isn't configured, so the UI says "email sent". | Return `False`; show "not sent". |
| — | MEDIUM | Synchronous SMTP with no retry queue. | Use a background job or a provider API. |

### `backend/app/services/dashboard.py`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 85 | MEDIUM | Campaigns are counted per user, but the calendar per workspace, so teammates' work is missing. | Add `workspace_id` to campaigns. |
| 85 | MEDIUM | Loads every campaign row with its heavy JSON just to count. | SQL aggregates. |
| 154 | LOW | A "scheduled" status shows as Draft or In Review. | Add a Scheduled label. |
| 270 | MEDIUM | "N creatives awaiting **your** approval" actually counts invites the user *sent*. | Fix the wording or the query. |
| 353 | LOW | Primary audience text is replaced by an age bracket such as "25-34". | Keep both. |
| 298 | LOW | "Today" uses UTC. | Use the user's timezone. |

### `backend/app/services/api_keys.py`, workspace settings
| Sev | Issue | Change |
|---|---|---|
| HIGH | `resolve_api_key` is never called, so API keys can be created but nothing accepts them. | Add API-key auth or hide the feature. |
| HIGH | `require_approval_before_publish` and `allow_public_sharing` are stored but never enforced. | Enforce them in publish/schedule and the review routes. |
| MEDIUM | `AUTH_SECRET` falls back to a default here too (line 137). | Use shared config. |

### Upload services (`brand_assets.py`, `profile_avatar.py`, `workspace_logo.py`)
| Sev | Issue | Change |
|---|---|---|
| HIGH | Client `content_type` is trusted; no magic-byte check; SVG is allowed (stored XSS on the API origin). | Verify with Pillow; sanitise or reject SVG. |
| LOW | Delete helpers don't confirm the path stays inside the uploads folder. | Add an `is_relative_to` check. |

### `backend/app/db/models.py`
| Sev | Issue | Change |
|---|---|---|
| MEDIUM | `CampaignHistory` has no `workspace_id`, so teams can't share campaigns. | Add the column and a migration. |
| LOW | Base64 media is stored in `Text` and JSON columns across several tables. | Move to object storage. |

---

## 4. Frontend: file by file

### `src/app/AppRoutes.jsx`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 58-150 | HIGH | Only `/dashboard`, `/calendar` and `/admin/*` are guarded; ~15 app pages are public. | Wrap all app routes in `<RequireAuth>`. |
| 43 | MEDIUM | Support admins are redirected to `/admin/support` from **every** path, so they can't use the product or open review links. | Only guard `/admin`. |
| 42 | MEDIUM | Returns `null` until `/auth/me` responds, so the public landing page is blank. | Render public routes immediately. |
| 99, 150 | MEDIUM | `/image-studio` and `/portfolio` are placeholder `<div>`s. | Remove them or build them. |
| 147 | LOW | The Integrations page is unreachable (redirects to dashboard). | Delete it or restore it. |
| 153 | LOW | No 404 page. | Add one. |

### `src/app/AuthProvider.jsx` + `src/services/authService.js`
| Line | Sev | Issue | Change |
|---|---|---|---|
| authService 62 | HIGH | Logout leaves `velorq_market_intelligence`, `velorq_active_campaign`, `velorq_onboarding`, `velorq_team_members`, `velorq_integrations`, `velorq_settings`, `velorq_conv_*`, and more, so **the next user on the same browser sees the previous user's data**. | Clear every `velorq_*` key on logout. |
| authService 21 | MEDIUM | `console.log` of the OTP. | Remove it. |
| AuthProvider 25 | LOW | A network error on `/me` is treated as logged out. | Distinguish 401 from a network error. |
| — | LOW | Context values aren't memoised. | `useMemo`. |

### `src/app/WorkspaceProvider.jsx`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 171 | HIGH | Auto-creates a workspace using the (possibly stale) email from localStorage. | Let the backend create it at login (`ensure_user_workspace`). |
| 151 | MEDIUM | One failing call wipes the workspace list. | Use `Promise.allSettled`. |

### `src/app/NotificationsProvider.jsx`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 297 | MEDIUM | Calls `GET /api/calendar-events` **every 5 seconds per tab**, plus the notifications call every 15 s. | Every 60 s+, or server push. |
| 236 | MEDIUM | Reminders fire after the start time, up to 2 hours late, still saying "Starting Now". | Remind before the start; correct the wording. |
| — | LOW | Duplicate toasts across tabs; no cap on stored notifications; a failed "mark read" is silently reverted. | BroadcastChannel; cap; retry. |
| 276 | LOW | `/favicon.ico` doesn't exist. | Use `/favicon.svg`. |
| 371 | LOW | Sits outside the router, so `window.location.assign` reloads the whole app. | Move it inside `BrowserRouter`. |

### `src/pages/CreativeStudio.jsx`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 567 | HIGH | When the MKI deliverable is `image`, the page always blocks, and its button navigates back to itself. | Remove the block; treat image as poster. |
| 313-318 | HIGH | An empty supporting message falls back to the MKI **audience description**, which gets printed on the ad. The headline falls back to a field that doesn't exist, then to "Make your next move count." | Require the user's copy or let the AI write it. Never use strategy text. |
| 182, 186 | LOW | Reads `marketIntelligence.context` and `.recommendations`, which don't exist. | Remove. |
| 373 | HIGH | Up to 10 videos can be requested in one request (each up to 10 minutes). | Cap video at 1–2; use a job queue. |
| 863 | MEDIUM | "Download video" always downloads variant 1, not the selected one. | Use the selected variant. |
| 421, 465 | MEDIUM | Writes the full base64 campaign to localStorage; the quota error is silently swallowed. | Store IDs and fetch from the server. |
| 527 | LOW | Several programmatic downloads in a loop; browsers block all but the first. | Zip them, or download one by one with user clicks. |
| — | MEDIUM | Chosen type and variant count are lost on refresh. | Keep them in the URL. |
| — | MEDIUM | No abort or timeout on generation requests lasting 1–10 minutes. | Job ID plus polling. |

### `src/pages/Publish.jsx`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 421-437 | HIGH | The social publisher response isn't checked, so success is shown even when posting failed. | Check `res.ok` and the returned post ID; show the error. |
| 366 | HIGH | Marks the campaign "published" **before** posting. | Mark it only after posting succeeds. |
| 470-490 | HIGH | The test-creative button posts a stock photo to the real page. | Remove it from production builds. |
| — | HIGH | "Publish now" doesn't cancel an existing scheduled post. | Cancel it server-side. |
| 146-155, 912-995 | MEDIUM | UTM, campaign URL and first-comment controls are never sent. | Implement or remove. |
| 94 | MEDIUM | Shows a fake connected account, "@velorq.brand". | Show "Not connected". |
| — | LOW | "Save Draft" only saves to localStorage. | Save server-side. |

### `src/pages/BrandQualityCheck.jsx`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 279 | MEDIUM | A new paid compliance check runs on **every visit**, and the score changes each time. | Reuse the stored result; add a "Re-run" button. |
| 243 | MEDIUM | Regenerate pays for 2+ variants and silently keeps the first. | Let the user choose, or generate 1. |
| — | MEDIUM | The check evaluates the original, not the edited image the user sees. | Upload the edit, then check it. |
| 375 | LOW | Breadcrumb `span role=button` has no keyboard support. | Use `<button>`. |

### `src/pages/CreativeEditor.jsx`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 38, 805, 943 | HIGH | Logo and color pickers use **VelorQ's demo brand**, not the customer's. | Load the user's brand kit. |
| 879 | HIGH | The edited image is never saved to the backend, so reviewers and compliance see the original. | Upload the edit and store it on the campaign. |
| — | MEDIUM | Video trim is stored as metadata only and never applied. | Implement it (ffmpeg server-side) or hide it. |
| lint | LOW | 6 unused imports/variables; unnecessary hook dependency (line 505). | Clean up. |

### `src/services/calendarService.js` + `src/pages/Calendar.jsx`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 125, 150, 168 | HIGH | Create, update and delete fake success on any error, including 403 for read-only users. | Show the error; queue offline changes explicitly. |
| 102 | MEDIUM | Returns cached localStorage events on 401/403/409. | Show a sign-in or no-access state. |
| Calendar | MEDIUM | Local `T00:00:00` keys versus UTC timestamps, so events near midnight land on the wrong day. | Convert with the user's timezone. |
| 530 | LOW | `window.confirm` for delete. | Use `ModalOverlay`. |

### `src/pages/BrandKit.jsx`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 102, 196, 217 | HIGH | Empty slots are filled with VelorQ logos and demo assets ("Hero Banner — Summer Launch"). | Show empty states. |
| 271 | CRITICAL | Identified by `?email=`. | Use the session. |
| lint | LOW | `faSliders` and `emptyLogoState` unused; `applyKit` missing from hook dependencies. | Clean up. |

### `src/pages/Analytics.jsx` + `src/services/socialInsightsService.js`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 55, 207 | HIGH | Any error shows bundled sample analytics as the user's own. | Show an error or empty state. |
| 191 | HIGH | Real posts from the fallback endpoint are discarded (`raw` unused). | Use `raw`. |
| Analytics 183 | MEDIUM | The badge says "Live n8n Synced" for fallback data. | Label correctly. |
| `data/fallbackInsights.json` | MEDIUM | Real-looking post data and a localhost URL are shipped in the public bundle. | Delete it. |

### `src/pages/CreativeReview.jsx` (public reviewer page)
| Line | Sev | Issue | Change |
|---|---|---|---|
| 61-75 | HIGH | `?quick_decision=approved` auto-submits an approval on load. | Pre-select only; require a click. |

### `src/pages/CompanyProfile.jsx`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 155-178 | HIGH | Creates a new workspace and switches to it on every save. | Only on first setup, and better done server-side. |
| 128 | CRITICAL | Profile save is identified by email. | Use the session. |

### `src/services/workspaceMembersService.js` + `components/settings/TeamMembersSection.jsx`
| Line | Sev | Issue | Change |
|---|---|---|---|
| 118+ | HIGH | API failure is saved locally and shown as success. | Show the error. |
| 60 | HIGH | Falls back to the local list, which is what users always see because of the backend bug. | Fix the backend; remove the fallback. |
| 12 | MEDIUM | `velorq_team_members` isn't scoped per user or workspace. | Remove local storage of the team. |

### `src/components/settings/*`
| File | Sev | Issue | Change |
|---|---|---|---|
| `BillingCreditsSection.jsx` | HIGH | All fake: "12,450 credits", "Visa •••• 4242"; Upgrade and Update do nothing. | Hide until billing exists. |
| `DataPrivacySection.jsx:139,280` | CRITICAL | Shows the deletion OTP on screen; says "10 mins". | Remove; tell the user to check their email. |
| `ApiWebhooksSection.jsx` | MEDIUM | Webhooks exist only in React state; API keys are unusable. | Implement or hide. |
| `PreferencesSection.jsx` / `hooks/useSettings.js` | LOW | Mixes server settings with an unscoped `velorq_settings` key. | Server only. |
| `SecuritySection.jsx`, `SessionsSection.jsx` | LOW | `window.confirm` (5 places). | Use `ModalOverlay`. |

### Auth pages
| File:Line | Sev | Issue | Change |
|---|---|---|---|
| `VerifyOtp.jsx:308` | CRITICAL | Shows "Development OTP" whenever the backend returns it. | Remove from production builds. |
| `Register.jsx:27` | MEDIUM | The name is saved only to localStorage, never to the profile; no Terms/Privacy consent. | Send it to the backend; add a consent checkbox. |
| `VerifyOtp.jsx:26` | LOW | Hard-coded 5-minute timer; interval recreated every second. | Use the server's expiry; one interval. |

### Other pages and components
| File:Line | Sev | Issue | Change |
|---|---|---|---|
| `Suggestions.jsx:18` | MEDIUM | Paid AI call on every visit. | Cache; generate on button click. |
| `Integrations.jsx:71` | MEDIUM | "Connect" is a 600 ms timer that fakes success (and the page is unreachable). | Implement OAuth or delete it. |
| `CustomPublisher.jsx:55` | MEDIUM | No file-size limit; videos are sent as base64 JSON. | Limit size; use multipart or direct upload. |
| `ReviewApproval.jsx:125` | MEDIUM | Owner approves without checking reviewer responses. | Show a warning when changes are requested. |
| `components/review/HumanReviewPanel.jsx` | CRITICAL | Identified by `owner_email` from localStorage. | Use the session. |
| `WhatsNew.jsx:37` | MEDIUM | Email in the query string. | Use the session. |
| `HelpCenter.jsx:123` | LOW | Silent fallback to static content. | Fine, but log it. |
| `WebsiteWatch.jsx`, `Suggestions.jsx` | LOW | Navigate to `/onboarding`, which is just a redirect. | Link directly to `/creative-studio/brief`. |
| `CampaignBrief.jsx` | LOW | Legacy page still routed and used as the "Back" target. | Remove or retarget. |
| `Dashboard.jsx:744,866` | MEDIUM | A third copy of the workspace auto-create logic. | Remove (backend handles it). |
| `components/common/AssistantWidget.jsx` | LOW | `isTyping` is unused (no typing indicator); unused imports. | Wire it up or remove. |
| `hooks/useBrandProfile.js` | MEDIUM | Fetches on every page via AppShell with `?email=`. | Session plus a cache. |
| `utils/campaignFlow.js:167-205` | HIGH | Stored images are truncated to `[stripped_for_storage]`, and pages later use them as real images. | Store only IDs and refetch; never reuse stripped data. |
| `utils/time.js` | LOW | Future dates show "1 min ago"; invalid dates show "NaN". | Handle both. |
| 17 files using `fetch` | HIGH | No `credentials: "include"`, so they will break once the backend requires the session. | One shared API client. |
| 34 files | LOW | `API_BASE_URL` declared separately in each. | One config module. |

### Landing / marketing
| Item | Sev | Issue | Change |
|---|---|---|---|
| `pages/Landing.jsx`, `components/marketing/*` (16 files), `hooks/useSiteContent.js`, `services/siteContentService.js`, `content/landingFallback.js`, backend `/api/site-content`, migrations 0022/0023 | LOW | **Dead code**: the live landing page uses `pages/Landing/Landing.jsx` + `components/landing/*`, so the CMS is never shown. | Delete one of the two systems. |
| `components/landing/*` | HIGH | Claims the product doesn't support: "search demand indexed in real time", LinkedIn/Google/TikTok publishing, SSO and audit log, direct API access, a 14-day trial and ₹7,900 plan (no billing), "compliance certificate", "100% brand kit lock", "14 days → 1 day". | Rewrite to match real features (consumer-protection risk). |
| `components/landing/*.jsx` | LOW | Each component is a single very long line. | Format and split. |

### Dead or empty files
| File | Issue |
|---|---|
| `src/app/App.jsx` | Empty file (the real one is `src/App.jsx`). |
| `src/app/AppContext.jsx`, `src/app/appContext.js` | Unused. |
| `backend/temp.py` | Script that adds an endpoint returning `DEEPSEEK_API_KEY`. Delete it. |
| `mediapublisher.md` | Stray notes at the repo root. |

---

## 5. Styles (`src/styles/*.css`, 36 files, ~21k lines)

| Sev | Issue | Where | Change |
|---|---|---|---|
| MEDIUM | **Dark mode is incomplete**: the theme switches CSS variables, but many stylesheets hard-code hex colors that never change (help-center 181, brand-kit 129, support-admin 98, creative-editor 81, market-intelligence 62 occurrences). Only 5 stylesheets have their own `[data-theme]` overrides. | most page CSS | Replace hard-coded colors with the theme variables from `index.css`; check each page in dark mode. |
| MEDIUM | Weak mobile support: 13 stylesheets have 0–1 media queries (login, company-profile, custom-publisher, assistant, notifications, whats-new, support-admin…). | listed files | Add breakpoints; test at 375px. |
| MEDIUM | Focus outlines removed in 25 places with `outline: none/0` and no replacement. | analytics, brand-kit, calendar, company-profile, creative-studio… | Add `:focus-visible` styles. |
| LOW | z-index values range up to `999999` (toast) and `9999` (brand-kit). | notification-toast.css:6, brand-kit.css:990 | Define a z-index scale. |
| LOW | `verify-otp.css` has 55 `!important`. | verify-otp.css | Fix specificity instead. |
| LOW | Google Fonts `@import` inside `landing.css` blocks rendering. | landing.css:1 | Use `<link rel="preconnect">` in `index.html`. |
| LOW | `index.html` has no favicon link, meta description or Open Graph tags. | index.html | Add them. |

---

## 6. Repo, tests, config

| Sev | Issue | Change |
|---|---|---|
| CRITICAL | Meta tokens are in `backend/data/veloq.postman_collection.json` and `backend/data/Social Insights Fetcher.json` (pushed to GitHub). | Rotate, purge from history, add gitleaks. |
| CRITICAL | User uploads and support screenshots are committed under `backend/uploads/**`. | Remove from git; add to `.gitignore`. |
| HIGH | Required files are untracked: `alembic/versions/0027_user_soft_delete.py`, `services/opportunity_score.py`, `src/utils/brandColors.js`, and tests. | Commit them. |
| MEDIUM | 4 failing tests: `test_assistant` ×2 (intent), `test_social_publisher` ×2 (webhook URL). | Fix them. |
| MEDIUM | No test for the Team list endpoint, auth/IDOR, Publish failure handling, or calendar permissions. | Add regression tests. |
| MEDIUM | `requirements.txt` is unpinned; `pytest` isn't installed (tests run with `unittest`). | Pin versions; add dev dependencies. |
| LOW | `backend/app/data/cached_social_insights.json` is runtime state kept in git. | Ignore it. |
| LOW | `.gitignore` lists individual `.pyc` paths (redundant with `*.py[cod]`). | Clean up. |
| LOW | Build produces a single 1.3 MB JS chunk. | Lazy-load routes. |
| LOW | 25 oxlint warnings (unused imports and variables, hook dependencies). | Fix during the cleanup pass. |

---

## 7. Suggested fix order

**Sprint 1: security (Critical items)**
- Rotate the Meta tokens and purge them from git history.
- Fix the deletion-OTP and login-OTP leaks, and send the OTP by email.
- Remove client-supplied webhook URLs and tokens.
- Lock down the Cloudinary local-path upload.
- Escape user text in email HTML.
- Use the session on every route.
- Add `credentials: "include"` through one shared API client.
- Clear all `velorq_*` keys on logout.
- Commit the untracked files.

**Sprint 2: stop lying to users (fake-success bugs)**
- Publish: check the social-publisher result; mark published only after success; remove the test-creative button; cancel scheduled posts on publish.
- Calendar, team members and analytics: show errors instead of local or demo fallbacks.
- Remove the demo brand and logo defaults from Brand Kit and the Editor.
- Remove the auto-approve email link behaviour.
- Hide fake Billing, webhooks and Integrations, and unusable API keys.
- Stop inventing "technical quality" values.
- Rewrite landing-page claims to match real features.

**Sprint 3: broken features**
- Team list tuple bug.
- Company Profile creating duplicate workspaces.
- Creative Studio "image" dead end.
- Logo path and brand-context JSON truncation (brand on ads).
- Layout `supporting` KeyError.
- Bundle fonts for Linux.
- Stripped-image storage bug.
- Scheduler locking and calendar linking.
- Status transition rules.
- Enforce workspace settings.

**Sprint 4: performance & cost**
- Blocking AI calls and the video job queue.
- Polling intervals.
- Compliance re-run on every visit; Suggestions on every visit.
- Move base64 media to object storage.
- Dashboard SQL aggregates.

**Sprint 5: polish**
- Dark mode variables, mobile breakpoints, focus styles.
- Remove dead code (second landing system, empty files).
- Lint fixes.
- 404 page.
- Data export completeness and hard delete.
