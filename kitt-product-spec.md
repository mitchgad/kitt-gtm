# Kitt — Product & Feature Specification
## Kitt Clinician and Kitt Companion

**Compiled:** 20 July 2026
**Source:** `/Users/mitchgadek/Documents/GitHub/kitt` @ commit `99312bc3` (branch `main`, clean tree)
**Method:** Read-only code inspection. No files in the repository were created, modified, or deleted.

---

## 0. About this document

### What it is

An exhaustive technical inventory of the two Kitt product surfaces, derived from reading the codebase directly. It is intended as an internal source-of-truth document: what exists, what half-exists, what is documented but unbuilt, and what is built but unreachable.

### Source-of-truth policy

**Code is authoritative. Documentation is treated as a statement of intent, not fact.** This matters more than usual here: the repository contains a prior product review (`docs/product/FEATURE_BASELINE_REVIEW.md`, dated 2026-04-27) that is now wrong in several places, and multiple service READMEs that contradict their own implementations. Section 13 catalogues the drift.

Where this document asserts something plainly, a file was opened and read. Where it does not, the claim is marked.

**Out of scope: prices, plan pricing, and trial terms.** This document covers capability only. Commercial facts have one source — `MKT1 Work/marketing-strategy.md` § NAMING & PRICING (`truth.sh strategy "NAMING & PRICING"`), verified against the live site. Prices were removed from §2.4 on 2026-08-27 because the table there came from a kitt repo doc and contradicted live pricing. Do not quote a kitt price from this document; where billing mechanics still appear (§2.4, §3.9, §13), they describe how the code bills, not what kitt charges.

### Confidence conventions

| Marker | Meaning |
|---|---|
| *(unmarked)* | Read directly in code by at least one investigator, with file citation. |
| **[corroborated]** | Independently confirmed by two or more investigators working from different code paths. Highest confidence. |
| **[check]** | Single-source and consequential — verify before acting, especially security and commercial claims. |
| **[assumption]** | Inferred from naming, structure, or adjacent evidence. Not directly observed. |
| **[uncertain]** | Conflicting or insufficient evidence. Explicitly unresolved. |

### Coverage and limits

In scope: Kitt Clinician, Kitt Companion, and the shared platform beneath them.

Out of scope by instruction: the Chrome extension and the LiveKit voice agent as product surfaces. They appear here only where they touch the two platforms.

Not verified: runtime behaviour. Nothing was deployed, executed, or tested. Every claim is static analysis. Several findings below — particularly around environment variables and production configuration — can only be settled by inspecting the running environment.

---

## 1. Product architecture

### One codebase, two products, split by organisation type

Kitt is a single Node/TypeScript backend and a single React client serving two commercially distinct products. The discriminator is `Organization.type`, a Prisma enum with exactly two values:

```prisma
enum OrganizationType { PERSONAL, TEAM }   // default: TEAM
```

- **`TEAM`** → **Kitt Clinician**, the clinic-facing product
- **`PERSONAL`** → **Kitt Companion**, the patient/consumer-facing product

This single field drives routing, navigation, onboarding, settings visibility, billing defaults, and feature gating throughout both the client and the server.

### The split is visible from the first screen

`SignupProductSelection.tsx` presents two product cards at registration — "kitt Companion" for individuals and "kitt Clinician" for practices and teams — synchronised to a `?product=companion|clinician` query parameter. Companion carries a hardcoded "Most Popular" badge. OAuth buttons remain disabled until a product is chosen.

Signup then creates the organisation accordingly (`src/services/auth/controller.ts:588-603`):

- `mode: 'personal'` → organisation named `Companion – {name}`, type `PERSONAL`
- `mode: 'team'` → user-named organisation, type `TEAM`

The first user of any new organisation is always assigned `role: ADMIN`.

### Post-login divergence

`RootRedirect` in `src/client/src/App.tsx:102-113` routes users by organisation type:

```tsx
const RootRedirect = () => {
  const { user } = useAuth();
  if (user?.organizationType === 'PERSONAL') {
    return <Navigate to="/chat" />;
  }
  return <Navigate to="/dashboard" />;
};
```

Clinician users land on `/dashboard`. Companion users land on `/chat`.

### Deployment

Production is live at `https://kitt.zoneblue.ai` with the API at `https://api.zoneblue.ai`. The current production baseline is Release 4 (15 June 2026). Backend runs on Render; Redis backs BullMQ; Postgres with the pgvector extension backs storage and RAG.

Legal entity: Zone Blue Pty Ltd, ACN 684 928 265 (from `PrivacyPolicyPage.tsx`).

---

## 2. Shared platform foundations

### 2.1 Identity and roles

Three roles exist (`prisma/schema.prisma:1736-1740`):

```prisma
enum UserRole { ADMIN, MEMBER, SUPER_ADMIN }
```

Permission map at `src/middleware/permissions.ts:31-64`:

| Role | Capabilities |
|---|---|
| **MEMBER** | View organisation, generate letters, view letters. That is the complete set. |
| **ADMIN** | All MEMBER capabilities, plus: invite and manage users within own org, org settings, exercise CRUD, plan and note templates, soft-delete and retention admin routes, system memory/GC endpoints. |
| **SUPER_ADMIN** | All ADMIN capabilities, plus: platform-wide user list, impersonate any user, list and manage all organisations, internal support docs, super-admin analytics. |

Note that in the permission map itself, ADMIN and SUPER_ADMIN hold identical permission sets; SUPER_ADMIN's additional power comes from explicit `requireSuperAdmin` route guards rather than the map.

### 2.2 Authentication

**Password auth** — bcrypt at `saltRounds=10`. JWT access token (1h) and refresh token (7d). No refresh-token rotation and no server-side revocation or blacklist: a stolen refresh token remains valid for its full 7-day life regardless of logout.

**OAuth** — Google and Microsoft fully implemented with PKCE, state, and nonce (5-minute state expiry). **Apple is a stub** — environment variables are read, then the code logs `'Apple OAuth not yet implemented'` (`src/services/auth/oauth/providers.ts:59-64`).

OAuth callbacks return tokens in the URL hash fragment, which the client parses and writes directly to `localStorage` (`OAuthCallbackPage.tsx:15-39`). There is no server-side code exchange.

**Password policy** — minimum 8 characters, upper/lower/number/special, blocks common, sequential, and keyboard-walk patterns (`src/services/auth/password-validator.service.ts`). Applied to password reset and invitation acceptance. **Not applied to signup**, which enforces only a Zod `min(8)`.

**MFA/2FA — not implemented.** No TOTP, no two-factor anything. Confirmed by exhaustive grep.

**Email verification and invitation tokens** share the `verificationToken` field and **have no expiry check in code** (`src/services/auth/service.ts:243-294`), despite the invitation email telling recipients it expires in 7 days. **[check]**

**Anti-abuse on signup** — Cloudflare Turnstile, disposable-email blocking (`signupEmailBlock`), and geo-blocking (`signupGeoBlock`) are all active on the signup route.

### 2.3 Organisation type enforcement

`src/middleware/organization-guards.ts` enforces the product boundary server-side:

- `blockPersonalOrgInvites` — PERSONAL organisations cannot invite members
- `blockPersonalOrgMembership` — PERSONAL organisations cannot add or remove members
- `enforceSinglePersonalOrg` — a user may own only one PERSONAL organisation

Client-side, a single guard component draws the line (`App.tsx:70-113`):

```tsx
const TeamOnly = ({ children }) => {
  const { user } = useAuth();
  if (user?.organizationType === 'PERSONAL') {
    return <Navigate to="/dashboard" />;
  }
  return <>{children}</>;
};
```

There is no corresponding `PersonalOnly` guard. Companion routes (`/chat`, `/chat/plan/:planId`, `/program`, `/account`) are reachable at the routing layer by TEAM users; the boundary there is enforced only through navigation visibility. **[assumption]** — page-internal redirects were not verified.

### 2.4 Billing and plans

> **This spec carries no prices, plan pricing, or trial terms.** Those are commercial facts and the only source for them is
> `MKT1 Work/marketing-strategy.md` § NAMING & PRICING (`truth.sh strategy "NAMING & PRICING"`), which is verified against the
> live site. The price table formerly here was derived from `docs/features/pricingplans.md` — a kitt repo doc, i.e. exactly the
> class of source §13 records as drifting from the code — and it contradicted the live commercial pricing. Removed 2026-08-27
> at Mitch's instruction. Never quote a kitt price from this document.

Tiers and their monthly token allowances, corroborated against Stripe product IDs in `src/services/stripe/webhook-handlers.ts`:

| Tier | Monthly tokens | Stripe product ID |
|---|---|---|
| Free | 15,000 | — (default `Subscription` record) |
| Basic | 70,000 | `prod_SeWU78eK4zHvie` |
| Professional | 210,000 | `prod_SeWUvBTiuqnls2` |
| Enterprise | 420,000 | `prod_SeWUxQuA73ilVG` |

**Subscription is per-organisation, not per-seat.** `Subscription.organizationId` is `@unique`, and Stripe Checkout hardcodes `quantity: 1` (`src/services/stripe/index.ts:85`).

**There is no seat model in code.** `docs/personal-organizations/README.md:73-74` describes `STRIPE_PRICE_PERSONAL` and `STRIPE_PRICE_TEAM_PER_SEAT` environment variables. Neither string appears anywhere in `src/`. If per-seat Clinician pricing is part of the commercial plan, it is unbuilt.

**No trial period is set in code.** No `trialEndsAt` is ever set by the subscription webhook handlers. This is an observation about the billing implementation, not a statement of the commercial trial offer — for the trial terms kitt actually sells, read § NAMING & PRICING in the strategy doc.

Webhooks handled: `customer.subscription.created/updated/deleted`, `invoice.payment_succeeded/failed`, `checkout.session.completed`, `payment_intent.created/succeeded`. **Not handled:** `trial_will_end`, dunning retries.

Plan→token-limit mapping is duplicated in two places (a product-ID switch at `webhook-handlers.ts:322-374` and a separate `planType` switch at `:533-555`) — a drift risk. The payment-link path (`handleCheckoutSessionCompleted`) computes `currentPeriodEnd` as "now + 30 days" rather than reading it from Stripe.

### 2.5 Usage metering — and the enforcement gap

`src/services/usage/service.ts` implements generic metering via `recordTokenUsage(orgId, tokens, featureType, userId, metadata)`, covering: `letter`, `chat`, `document_tagging`, `patient_summary`, `document_title`, `chat_title`, `dictation_checklist`, `dictation_treatment_plan`, `dictation_speaker_id`, `dictation_realtime_checklist`.

**`checkUsageLimit()` is called in exactly one place: letter generation (`src/api/routes/letter.routes.ts:68`).**

Every other metered feature — chat, patient summaries, document tagging, document titles, and all dictation features — calls `recordTokenUsage()` to *log* consumption but never gates on the limit first. An organisation past its monthly token cap continues using everything except letter generation.

This is a direct commercial exposure: token spend against `anthropic/claude-sonnet-4` continues uncapped for most features. It also means `docs/features/PATIENT_360_TOKEN_METERING_PLAN.md` has met its instrumentation goals but explicitly failed its stated success criterion ("usage limits work with new token sources").

Separately, `src/services/auth/controller.ts:705` sets `monthlyTokenLimit: organization.type === 'PERSONAL' ? 15000 : 0` on one signup path — meaning TEAM organisations can be created with a **zero** token limit, diverging from the documented "15,000 for all". **[check]**

### 2.6 LLM infrastructure **[corroborated — three independent investigators]**

A single `LLMService` singleton (`src/services/llm/service.ts`) fronts nearly every generative feature.

| Purpose | Model ID | Route | Source |
|---|---|---|---|
| Primary generation | `anthropic/claude-sonnet-4` | **OpenRouter** (`https://openrouter.ai/api/v1`) | `src/services/llm/anthropic.ts:61` |
| Fallback generation | `gpt-4` | OpenAI SDK direct | `src/services/llm/openai.ts:27` |
| Vision OCR | `gpt-4o` | OpenAI direct | `src/services/document/vision-ocr.service.ts:29` |
| Embeddings | `text-embedding-3-small` | OpenAI direct | `src/services/rag/embedding.service.ts:25` |
| Speech-to-text | `scribe_v2_realtime` | ElevenLabs | `src/services/transcription/scribe.service.ts:20` |

Three things worth knowing:

1. **Despite the class being named `AnthropicProvider`, it never touches Anthropic's native API** — all Claude traffic is routed through OpenRouter's OpenAI-compatible endpoint.
2. **Model IDs are hardcoded, not environment-configurable.** Only API keys and base URLs are configurable. Changing model requires a code change and deploy.
3. **The fallback only exists if `LLM_API_KEY` is set.** Without it, an OpenRouter outage is a hard failure for every generative feature.

Fallback logic (`service.ts:169-211`): on primary failure, iterate all other registered providers, record a metric per attempt, throw an aggregated error only if all fail. Within-provider retries use exponential backoff capped at 10s, 3 attempts default.

Consumers span ~27 files: letter generation, notes, plans, letters, patient and Companion chat, Companion agentic-loop reasoning, dictation post-processing (checklist, speaker ID, treatment plan), document titles, patient summaries, physio program generation, auto-tagging.

### 2.7 Background workers

Five BullMQ workers (`bullmq: ^5.77.6`), each with an independent ioredis connection. All self-disable with a warning if `REDIS_URL` is absent rather than crashing.

| Worker script | Queue | Triggered by | Produces |
|---|---|---|---|
| `worker:rag` | `rag_ingestion` | `ingestionService` on document upload, note publish, chat, conversation summary | Chunked, embedded, encrypted vectors in `embedding_chunks` (pgvector) |
| `worker:note-generation` | `note_generation` | `enqueueNoteGenerationJob` | Generated clinical notes; marks note `FAILED` after 3 exhausted attempts |
| `worker:plan-generation` | `plan_generation` | `enqueuePlanGenerationJob` | Generated treatment plans |
| `worker:take-home-plan` | `take_home_plan` | `patient-plans.service.ts:590,623` | Patient-facing take-home plan content. `concurrency: 1` |
| `worker:sync` | integration sync | 4 on-demand route sites + `scheduled-sync.service.ts:133,201` | PMS sync execution. `concurrency: 1` |

Note and plan generation both **fall back to in-process execution if Redis is unavailable** — they degrade rather than drop. Whether RAG, sync, and take-home-plan silently drop jobs under the same conditions was not confirmed. **[uncertain]**

**Completion signalling is asymmetric between notes and plans.**

`src/services/generation/generation-events.ts` defines an in-process event bus with both `emitNoteGenerated`/`onNoteGenerated` and `emitPlanGenerated`/`onPlanGenerated`.

- **Notes get a real-time push.** `chat/websocket.service.ts` subscribes via `onNoteGenerated()` and emits `note-generated` / `note-generation-failed` to the `thread:{threadId}` Socket.IO room.
- **Plans do not.** `emitPlanGenerated()` is called from three sites in `chat-plan-generation.service.ts` (lines 619, 722, 811), but **`onPlanGenerated(` is never called anywhere server-side** — no WebSocket bridge exists. The client instead polls `PatientPlan.generationStatus` over REST (`plan-generation-utils.ts`, `PlansTab.tsx`).

The plan events are therefore emitted into a void. Plan generation still completes correctly and the status field still updates — this is a latency and polish gap, not a correctness bug — but the Release 4 notes describe "real-time client updates via WebSocket when generation completes", which is true for notes and not for plans. The take-home-plan worker emits no completion event at all.

Wiring `onPlanGenerated` into the existing WebSocket service is a small change that would close the gap. **[check]**

Scheduled jobs registered in `src/index.ts`:
- Daily exercise reminder email — cron `0 8 * * *`
- Daily admin digest — cron `30 5 * * *`, production only, Australia/Brisbane
- Companion agentic loop schedule sync — hourly
- Integration sync — every 30 min default (clamped 15–120 via `INTEGRATION_SYNC_INTERVAL_MINUTES`), plus daily full reconciliation

### 2.8 Global middleware stack

In order, from `src/index.ts`:

`helmet()` → `hpp()` → `ipBlocker` (in-memory) → `rateLimiter` (in-memory, 100/min prod API, 10/min prod auth) → `cors()` → `securityHeaders` (a second CSP overlapping helmet's) → `requestIdMiddleware` → Stripe raw-body webhook route → `express.json` → `setCsrfToken` → **`validateCsrfToken` — COMMENTED OUT** → metrics → memory monitor (warn 400MB, force GC 700MB) → static → routes → error handlers → Rollbar.

**CSRF validation is disabled** at `src/index.ts:340`:

```js
// app.use(validateCsrfToken); // Temporarily disabled
```

**[corroborated]** — confirmed independently by two investigators. `setCsrfToken` still issues `X-CSRF-Token` headers, so the application presents as CSRF-protected while nothing validates the token. The `CSRF_VIOLATION` audit event type is consequently dead code. This was flagged in the April 2026 baseline review and remains unresolved three months later.

---

## 3. Kitt Clinician

The clinic-facing product. Maps to `TEAM` organisations.

### 3.1 Route map

All routing lives in `src/client/src/App.tsx`. Protected routes are wrapped `RequireAuth > AppLayout > [guard] > Page`.

| Path | Component | Guard |
|---|---|---|
| `/dashboard` | DashboardPage | — |
| `/patients` | PatientsPage | `TeamOnly` |
| `/patients/:patientId` | Patient360Page | `TeamOnly` |
| `/patient-notes` | PatientNotesPage | — |
| `/patient-notes/:noteId` | PatientNoteViewPage | — |
| `/patient-cases` | PatientCasesPage | **none** |
| `/templates` | TemplatesPage | — |
| `/exercises` | ExerciseLibraryPage | `TeamOnly` |
| `/media` | MediaLibraryPage | `TeamOnly` |
| `/integrations` | IntegrationsPage | `AdminOnly` |
| `/users` | UsersPage | `TeamOnly` > `AdminOnly` |
| `/organization` | OrganizationPage | `TeamOnly` |
| `/settings` | SettingsPage | — |
| `/usage` | UsagePage | — |
| `/subscription`, `/subscription/plans`, `/subscription/billing` | Subscription pages | — |
| `/chats` | ChatsPage | — |
| `/chat-legacy` | ChatPage | — |
| `/super-admin` | SuperAdminDashboardPage | `SuperAdminOnly` |
| `/analytics` | AnalyticsPage | `SuperAdminOnly` |
| `/organizations`, `/organization/:id` | Org admin pages | `SuperAdminOnly` |
| `/support/*` | SupportDocsPage | `SuperAdminOnly` |
| `/api-keys` | **`PlaceholderPage`** | `AdminOnly` |

Public: `/login`, `/register`, `/oauth/callback`, `/forgot-password`, `/reset-password`, `/verify-email`, `/verification-sent`, `/accept-invitation`, `/accept-plan`, `/privacy-policy`, `/terms-of-service`, `*`.

**Routing defects:**
- `/patients` is declared **twice**, identically (lines 292-303 and 488-499). The second is unreachable.
- `/patient-cases` — which holds the actual patient notes workflow and therefore PHI — has **no `TeamOnly` guard**, unlike every other patient-data route. A PERSONAL user could reach it by direct URL. **[check — likely authorisation gap]**
- `/subscription/checkout` renders `CheckoutSuccessPage`, identical to `/checkout/success`. A "checkout" URL rendering a "success" page appears to be stale routing. **[assumption]**
- `/api-keys` renders a placeholder behind `AdminOnly` — the backend API-key routes exist and work, but there is no real UI.

### 3.2 The notes/cases naming inversion

This is the most confusing area of the Clinician codebase and worth stating precisely.

**`/patient-notes` → `PatientNotesPage.tsx`** is a *static navigation hub*. It makes **zero API calls**. It renders two cards ("Patient Cases", and "Note Templates" for admins) plus a three-step quick-start guide.

**`/patient-cases` → `PatientCasesPage.tsx`** contains the *actual* patient-selection → notes-list → note-editor workflow. The component declared inside that file is literally `const PatientNotesPage: React.FC` and exported as `export default PatientNotesPage` — an unrenamed copy-paste.

Consequences:
1. There is **no case-management UI anywhere** — no case creation, no case list — despite the hub card advertising it as a way to "track different conditions or treatment episodes."
2. The hub's admin "Note Templates" card navigates to **`/note-templates`, which is not a registered route**. `TemplatesPage` is mounted at `/templates`. The card is dead for every admin who clicks it.

**Why cases are missing is now answered.** `components/patient-record/CasesTab.tsx` is not an unfinished feature — it is a permanent deprecation notice. It renders the heading **"Cases No Longer Required"** with copy explaining that cases were removed following customer feedback, plus a button that switches the user to the Notes tab. The component exists solely to display that migration message.

So the case feature was **deliberately retired**, and what remains is stale surrounding copy:
- The `/patient-notes` hub still advertises cases as a way to track conditions and treatment episodes
- The route is still named `/patient-cases`
- `PatientDocumentsPanel` still renders a "Cases" tab (which shows the deprecation notice)
- Provider case sync still runs in the integration layer (§7.2), mirroring case IDs into `IntegrationMapping`

This is a copy-and-naming cleanup, not a missing feature. Worth correcting because anyone reading the hub page today is told a capability exists that was intentionally removed.

### 3.3 Patient 360 — the primary clinical workspace

`Patient360Page.tsx`, routed at `/patients/:patientId`, wrapped in a `SessionProvider`.

Composition:
- **`ChatPanel`** — the main AI thread UI, also the host for dictation and voice chat via a `startMode` prop
- **`Patient360Landing`** — shown when no thread is active; entry points for send-message, thread-click, start-dictation, start-voice-chat
- **`PatientDocumentsPanel`** — right rail for notes, letters, documents, cases (`hidePlansTab` always true here)
- **`PlansTab`** — full treatment-plan experience
- **`PatientSummaryPanel`** — AI-generated patient summary
- **`SessionBar`** — Nookal appointment/session selection
- **`Patient360Tour`** — guided onboarding overlay

Tabs: Chat / Plan on desktop; Chat / Plan / Documents on mobile.

Only one direct API call in the page itself (`patientAPI.getPatient`); all other data fetching is delegated to child components.

**Code health note:** the file carries extensive active `console.log` instrumentation tracking mount/unmount with a random `mountId`, and deliberately bypasses React Router with raw `window.history.replaceState` to avoid a remount race under `v7_startTransition`. Comments document this as a workaround for a known race condition.

### 3.4 Clinical documentation

**Patient notes** — `src/api/routes/patient-notes/`:
- Create manual draft, paginated retrieval per patient, auto-save draft content, publish (which also enqueues RAG ingestion), edit final with audit trail
- Generate from: dictation session, chat message, full conversation thread
- Nookal integration: case list, templates, mapping status, appointments, sync-to-Nookal, batch and single sync status
- Cliniko integration: mapping status, sync-to-Cliniko, sync status
- Note templates: full CRUD, clone, system-template seeding (admin-gated)
- Cost monitoring sub-router: cost summary, usage records, cost trends, cost limits (admin-gated)
- Regenerate sub-router: manual regenerate, batch regenerate (max 10), completion stats

`GenerationStatus` (`PENDING`/`COMPLETE`/`FAILED`) tracks async generation, with WebSocket completion events to the client.

**Patient letters** — `src/api/routes/patient-letters/`: create draft, list per patient, update, send, archive/unarchive, generate from chat message or conversation, sync-to-Cliniko as attachment, reverse lookup by generated document ID.

**Letter generation** — `POST /generate` on the letters route, gated by `UsageTrackingService`. This is the only endpoint in the entire application that enforces the token limit.

> **⚠ Probable live outage on one endpoint.** `POST /api/documents/generate-from-content` (`documents.routes.ts:1771`) registers the `auditPHI` middleware **factory uninvoked** — `auditPHI,` rather than `auditPHI({...})` as every other call site in the file does. `auditPHI` has signature `(context) => (req, res, next) => {...}`. Express therefore calls it as `auditPHI(req, res, next)`, which simply returns the inner function and **never calls `next()`**. Requests to this endpoint should hang until timeout.
>
> This is the "generate a document from pasted clinical text" feature — the path branching across `PROGRESS_NOTES` / `GP_LETTER` / `REFERRAL` / `TREATMENT_PLAN` / `DISCHARGE_SUMMARY` / `NDIS_REPORT`. If clinicians report that feature hanging or timing out, this is why. **[check — confirm against production logs; it is a one-character fix if reproduced.]**

**Documents** — `src/api/routes/documents.routes.ts` (~2,100 lines): multi-file upload (multer, 50MB, 10 files, MIME allowlist) with checksums and auto-tagging; paginated listing; presigned R2 download URLs (24h cap); text extraction across plain text, Word, CSV, RTF, with OCR fallback; soft and permanent delete with embedding cleanup; metadata patch; restore; tag preview; LLM title generation; sync to Nookal and Cliniko with duplicate-import protection; document generation from pasted clinical text branching across `PROGRESS_NOTES`/`GP_LETTER`/`REFERRAL`/`TREATMENT_PLAN`/`DISCHARGE_SUMMARY`/`NDIS_REPORT`; PDF generation from HTML.

### 3.5 Rehab Program Builder and treatment plans

The most developed feature area, shipped in Release 4.

**Plan lifecycle** — create draft, paginated list per patient (with archived/deleted filters), auto-save, publish (syncs to Companion), edit with audit trail and Companion re-sync, archive/unarchive, soft delete, PDF export, sync-to-Nookal and sync-to-Cliniko.

**Structured plan components**, each a sub-router mounted under `/:planId` and wrapped in `syncSharedPlanAfterWrite()` so every write propagates to shared Companion copies:

- **Goals** — list, add, reorder, update, delete
- **Progress measures** — measure type, custom label, unit, baseline, target, plus a time series of recorded values
- **Prescribed exercises** — add, update, delete, reorder; attach and detach media; set program start date; **copy-day**; **copy-week**; **distribute-unscheduled** across a week
- **Program day labels** — get, upsert by date
- **Program day notes** — get, upsert by date, length-validated
- **Program week labels** — get, upsert by week index (0–3)

Plan templates carry the same day/week label and note structure, keyed by relative `weekIndex`/`dayIndex` rather than absolute dates.

**Generation paths** — plan from chat message, plan from full conversation, plan from dictation session, regenerate take-home content.

**Companion-facing routes on the clinician side** — `companion-status` (checks for a `ClinicianPatientLink`), `companion-memory` (clinician view of Companion adherence), `share-to-companion`, `revoke-share`, `compliance` (adherence synced back from the patient).

**The builder UI** (`components/patient-plans/rehab-program/RehabProgramBuilder.tsx`, 1,660 lines) is a 4-week calendar grid, 7 columns per week:

- **Week controls** — inline rename (custom week label), and a copy-week action that arms a "copy mode": the source week gets a solid ring, every other week a dashed hover ring, and clicking a target week prompts `CopyScheduleConfirmDialog` ("Replace Week N?") if that week already has exercises.
- **Day controls** — inline custom day label (placeholder text suggests "Upper body", "Rest day"), day-level notes (textarea, 1,000 char max), and a copy-day action using the same arm/highlight/confirm pattern. Both support Enter-to-save and Escape-to-cancel, with Save disabled until the draft changes.
- **Exercise cards** — click to expand showing sets/reps/tempo/notes; inline editable fields persisting on blur with a 700ms debounce; **drag-and-drop across days** (native HTML5 DnD with drop-target ring) and reorder within a day (up/down arrows); per-exercise delete; per-day "Add" opening `ExerciseLibraryPicker` scoped to that date.
- **Hover preview** (`ExercisePreviewPopover.tsx`) — 250ms hover delay opens a portalled 320px popover, auto-repositioned to stay in viewport, showing an embedded YouTube/Vimeo iframe or a native autoplaying muted looped `<video>`, plus description. On touch devices it opens as a bottom sheet instead. Lazy-fetches full exercise detail via `exercisesAPI.getById` on first open.
- **Legacy auto-placement** — on mount, if every prescribed exercise has `scheduledDate === null` (a plan predating the calendar), the builder auto-runs `distributeUnscheduled` once to spread them onto dates.
- **Patient/read-only mode** — the drag handle is replaced by a green/amber/grey completion dot driven by compliance data synced back from Companion.

> **"Start from template" is hardcoded demo data.** The Templates dropdown is populated from `MOCK_REHAB_TEMPLATES` in `rehab-program/types.ts:39-87` — a hardcoded two-entry array ("Low back pain — Week 1", "Rotator cuff — Intro phase") with fabricated exercise data, consumed at `RehabProgramBuilder.tsx:794,892`. **No template-listing endpoint is called anywhere in the file**, and there is no backend rehab-template catalogue behind it.
>
> Applying a template does perform real writes (it loops the mock days calling the genuine label, note, and add-exercise endpoints), so the feature "works" — it just offers two fake starting points rather than a library. A clinician sees what looks like a template system. **[check — confirm whether a real template catalogue was intended; `PlanTemplate` exists in the schema and is unrelated to this dropdown.]**

### 3.6 Dictation and transcription

**Dictation is not a standalone product surface.** `DictationPage.tsx` — 1,267 lines — is routed nowhere in `App.tsx`, its navigation link is commented out (`Navigation.tsx:474-484`), and its only test suite is `describe.skip`. It carries a "Context analysis coming soon" stub and a `TODO: Remove in production` debug panel, uses raw `fetch` with ad-hoc base URL resolution instead of the shared axios instance, and duplicates its session-check-then-create logic near-verbatim across two functions.

Live dictation runs **inside `ChatPanel`**, reached from Patient360's "Start Dictation" button via a `startMode` prop. This resolves the April baseline's open question: dictation is embedded, and the standalone console is dead code.

The dictation *backend* is substantial and live:
- **Sessions** — recent, filtered/paginated, by ID, create, update, pause, resume, complete, cancel, active-session-for-patient, org stats
- **Transcripts** — chunks with filters, full transcript, add chunk (1000 char max), latest-after-sequence for polling, analysis, summary
- **Checklists** — LLM-generated dynamic checklist items from transcript (max 20), update, complete, delete, completion analysis, summary. This is an AI "what you haven't covered yet" prompt system.
- **Treatment plans** — get, LLM-generate, update (5000 char max), regenerate with additional notes
- **Speaker identification** — real-time speaker detection from partial transcript
- **Patients** — search/paginate, archive/unarchive, get by UUID, get by human-readable ID, create, update, soft delete, generate next patient ID

Transcription runs through ElevenLabs Scribe (`scribe_v2_realtime`) via a token endpoint issuing single-use realtime tokens for client-direct streaming.

**Known live bug:** `GET /api/dictation/sessions/stats` is shadowed by an earlier-registered `GET /:id` carrying UUID validation. Requests to `/stats` match `/:id` first, fail UUID validation, and return 400 — the stats handler is unreachable. **[check]**

### 3.7 Chat and RAG

Chat is **not** a REST endpoint for message generation — generation runs over Socket.IO (`src/services/chat/websocket.service.ts`). `chat.routes.ts` handles thread CRUD and history only.

A `ChatServiceRouter` (`chat-service-router.ts`) dispatches on `patientId === null`: null → `CompanionChatService`, non-null → `PatientChatService`. The same tables and endpoints serve both products.

Routes: create thread (with fire-and-forget LLM auto-title, summary, and RAG ingestion), generate title, list threads, get messages, message attachments (multer 50MB/5 files → R2 → RAG ingestion), temp message, update thread, delete thread (cascades RAG embedding cleanup via raw SQL), convert message to document (PDF via `pdfService` → R2).

**Stub:** `GET /messages/:messageId/attachments` always returns `[]` — the real Prisma query is commented out with `// TODO: Get attachments when ChatMessageAttachment model is added`, despite that model being actively written to elsewhere in the same file.

**RAG** — `rag.routes.ts`: semantic search over patient documents, force re-index of a document (deletes and re-enqueues, useful for triggering OCR on scanned PDFs), and a health endpoint. Vector storage is pgvector with an HNSW index added June 2026. Sources: `document`, `note`, `chat`, `exercise`, `conversation_summary`.

### 3.8 Exercise and media libraries

**Exercise library** (`/exercises`, `TeamOnly`) — search with query/body-region/tag/level/equipment/category filters, facets for filter population, create/update/delete (admin), variant append.

Two notable limitations:
- New exercises are created with a **hardcoded placeholder variant**: `{ name: 'Standard', level: 'Beginner', videoUrl: 'https://placeholder.local', cues: [] }`. There is no UI to set a real video, level, or cues at creation.
- **There is no edit UI on the page at all** — only create and delete. Video links explicitly filter out `https://placeholder.local`, confirming placeholder URLs are an expected state.

Practically, the exercise library can only be meaningfully populated by seeding or direct database work. An export script exists (`e9288c8a`).

**Media library** (`/media`, `TeamOnly`) — multi-file upload (100MB, 5 files, image/video), external YouTube/Vimeo link addition, search with category and tag facets, presigned thumbnail and download URLs, delete, metadata patch. `MediaAsset.organizationId` is nullable, where null means a **global asset** shared across all organisations.

No edit-metadata UI is wired on the page despite the API existing.

### 3.9 Analytics and dashboards

`AnalyticsPage` serves three audiences from one component: SUPER_ADMIN sees platform-wide business metrics; ADMIN sees org-scoped Overview / User Analytics / Business tabs; MEMBER sees a limited view.

Backend (`analytics.routes.ts`) uses a deliberate **fail-open** pattern — service errors return HTTP 200 with zeroed shapes rather than 5xx, explicitly commented as preventing dashboard breakage.

Endpoints: subscriptions, revenue, acquisition, churn, report (all `requireAdmin`); usage and dashboard (`authenticate`, branching internally for SUPER_ADMIN); super-admin dashboard, super-admin organisations, recent users (`requireSuperAdmin`); daily activity (contribution-calendar shape).

**`SuperAdminDashboardPage`** works correctly — MRR, new orgs, new subscribers, churn rate, customer health (healthy/at-risk/critical), subscription counts, retention (avg lifetime, ARPU), recent signups, organisations needing attention.

**`AdminDashboardPage` is effectively non-functional.** Its `transformToAdminDashboardData()` hardcodes `totalUsers: 0`, `newSignupsThisMonth: 0`, `activeUsersLast7Days: 0`, `activeUsersLast30Days: 0` and empty arrays for `topPractices`, `topUsers`, `recentSignups`, `recentActivity`, `allUsers` — discarding whatever the API returns. Only `documentsGeneratedThisWeek` is populated. The full UI renders, permanently empty. It also contains a fully-built `ActionsDropdown` component (View Details / Impersonate) that is **never mounted** in the table.

**[check]** — the super-admin MRR calculation uses a hardcoded `planPricing` placeholder map, with a comment noting "you'd lookup actual pricing from your pricing table". MRR figures should not be trusted without verifying that map against live pricing.

### 3.10 Settings and organisation management

`SettingsPage` provides the clearest product differentiation in the app. These sections are hidden when `organizationType === 'PERSONAL'`:
- **Terminology** — patients vs clients
- **Letter generation** — spelling convention (AU/UK/US), preserve formatting, preserve Nookal data fields
- **Professional context** — profession, experience level and years, specialisations, qualifications, clinical interests, communication style
- **Guided tour** — Patient 360 tour launcher

Always shown: organisation switcher (itself gated on `FEATURES.orgTypes` and >1 org), save preferences.

`OrganizationPage` — **four of five tabs are stubs.** Only "General" is implemented. Members, Invites, Billing, and Usage each render a centred heading reading "…functionality will be implemented here", explicitly commented `// Mock components for non-general tabs (to be implemented later)`. Fully-built equivalents exist as standalone pages (`UsersPage`, `BillingPage`, `UsagePage`), but the tabs are dead ends rather than links to them.

`UsersPage` — full CRUD: invite/add, edit, delete, reset password, activate/deactivate, plus SUPER_ADMIN cross-org directory with org filter, search, and pagination, and **user impersonation**.

---

## 4. Kitt Companion

The patient and consumer-facing product. Maps to `PERSONAL` organisations.

### 4.1 What Companion actually is

Positioned as an AI training and rehabilitation coach. In practice it serves two overlapping jobs that the codebase has not fully reconciled:

1. **A standalone AI fitness/training coach** — self-signup, chat-first, with an extensive sports-science system prompt
2. **A clinician-linked rehab companion** — receives shared treatment plans, logs exercise completions, syncs adherence back to the clinic

The second is materially more built out.

### 4.2 Route map

Four routed screens (not one, as the April baseline suggested):

| Path | Component | Purpose |
|---|---|---|
| `/chat` | `CompanionPage` | Primary landing — greeting → chat, with overview side panel |
| `/chat/plan/:planId` | `SharedPlanDetailPage` | Plan execution and logging |
| `/program` | `ProgramHomePage` | Redirect hop to first shared plan, or `/chat` |
| `/account` | `CompanionAccountPage` | Account menu |

Mobile navigation is a four-tab bottom bar (`CompanionMobileTabBar`, ENG-338): Program / Chat / History / Account. Visibility is gated on `organizationType === 'PERSONAL'` plus a path prefix allowlist.

### 4.3 CompanionPage (`/chat`)

Greeting screen with a time-based greeting and `CompanionInitialInput` composer, transitioning into a full chat interface on first message.

The composer supports **voice input** — it starts a dictation session against the user's `organizationId`, streams over WebSocket, and offers quick-action chips that differ by context: `REHAB_QUICK_ACTIONS` when the user has a shared plan, `FITNESS_QUICK_ACTIONS` otherwise.

Overview side panel (desktop) / modal (mobile) contains:
- `CompanionPushOptIn` — web push opt-in for morning check-ins
- `SharedPlansPanel` — plan list with inline completion logging
- **Device card — "Coming Soon"**, disabled, backed by a `mockDeviceStatus = { type: 'Garmin Connect', connected: true }` object that is never used for real state
- **Readiness card — "Coming Soon"**, all metrics render as `--`
- `OrganizationDocumentsPanel` — personal documents

### 4.4 SharedPlanDetailPage — the core Companion product

1,827 lines, the densest screen in either platform.

- Day-by-day exercise schedule with a day navigator showing per-day status dots (rest/pending/partial/complete)
- Per-exercise cards: body part, sets×reps or duration, per-week frequency, demo media (compact → full expand)
- Per-set logging with individual reps inputs, duration toggle, **pain score slider (0–10)**, **additional weight** field, and notes
- Multi-session support for exercises prescribed multiple times per week, with independently locked/active sub-sessions
- Future-day preview locking — "Preview only — logging opens on this day"
- Goals list, take-home instructions rendered from Markdown/HTML
- Sticky action bar with save-progress

**Gamification layer:** XP per set (10), session bonus (25), exercise bonus (50), all-done bonus (100); a six-tier level system; streak badge; confetti burst; XP toasts; randomised motivational message pools.

**Important limitation: XP, level, and streak are `useState` only and are never persisted.** They reset on every page reload. As built, this is a per-session animation rather than a retention mechanic. If gamification is being positioned as an adherence driver, this is a significant and probably cheap gap to close.

A second, lighter completion form (`PlanCompletionForm`, used by the sidebar panel) captures only sets/reps/duration/notes — **no pain score or weight** — an inconsistency with the detail page.

### 4.5 The Memory & Context Engine

Three-ring model, matching `docs/companion/context-package-mapping.md`:

| Ring | Model | Content | Status |
|---|---|---|---|
| **Ring 1** — static profile | `CompanionProfile` | sex, occupation, fitness background, injury history, diagnoses, goals, communication style | **Never written** |
| **Ring 2** — active context | via `planSharingService` | shared plans, goals, current program | Working |
| **Ring 3** — dynamic state | `CompanionDailyState` | mood rating, adherence ratio, mood note, reported symptoms, conversation summary | Working |

Assembly (`src/services/companion/context/assemble.ts`) runs rings in parallel via `Promise.allSettled` with per-ring degrade-on-failure and freshness provenance (`fresh`/`stale`/`missing`). `semantic-memory.ts` adds real pgvector RAG search filtered to `conversation_summary` and `chat` sources. `serialize.ts` renders a "COMPANION MEMORY BRIEF" text block with token-budget trimming (12k char default; trims Ring 2 plans first, then session notes, then semantic snippets, then truncates the conversation summary). `session-cache.ts` caches per-thread for 15 minutes.

**The critical gap: Ring 1 has no write path anywhere in the application.** `prisma.companionProfile.create/update/upsert` is called nowhere in the repository. The `CompanionOnboardingModal` that appears to collect this information is a pure product tour making **zero API calls**. The table exists, is read at assembly time, and is permanently empty for every user.

So the three-ring memory model ships as two rings. The architecture is sound; the collection UI was never built.

Observability is first-class: a Prometheus histogram (`companion_context_assembly_duration_seconds`), a Langfuse trace, and a `meta.assemblyDurationMs` field with a >500ms warning.

### 4.6 The agentic loop (Sense → Reason → Act → Learn)

Implemented per `docs/decisions/2026-06-12-companion-agentic-loop-scheduling.md` (BullMQ rather than Lambda/EventBridge).

| Step | File | Status |
|---|---|---|
| **SENSE** | `sense.ts` | **Built** — assembles a daily snapshot from context + plan detail + yesterday's adherence |
| **REASON** | `reason.ts` | **Built** — real LLM call producing a structured `DailyPositionAssessment` (session appropriateness, recovery trajectory, flag thresholds L1–L4, confidence), mode-aware (PRESCRIBED mode blocks load changes) |
| **ACT** | `act.ts` | **Built and delivering.** Plans `send_morning_check_in`, `modify_today_load`, `queue_practitioner_flag`, `celebrate_milestone`. [Updated 2026-08-26: delivery is now wired — `act.ts` calls `deliverMorningCheckIn(...)` via `check-in-delivery.service.ts` (ENG-221), sending over native push → web push → email. The `queue_practitioner_flag` at-risk leg also ships in production (product-owner confirmed; not yet reflected in the local `main` checkout @ `983ec51`, which still defers flags to the Clinical Flag Engine / ENG-232, so verify at next re-inspection). The stale "stub execution" docstring at `act.ts:133` predates this wiring.] |
| **LEARN** | `learn.ts` | **Built** — persists `CompanionLoopDecisionLog`, updates Ring 3 adherence, engagement outcome starts `pending` |

Scheduling (`scheduler.ts`, `queue.ts`, `worker.ts`, `idempotency.ts`) is real: per-user, per-timezone BullMQ job schedulers using `upsertJobScheduler` with cron pattern plus timezone, Redis `SET NX` idempotency locks keyed `userId:localDate`, and an hourly schedule-sync job.

**[Updated 2026-08-26: the loop now reaches users.]** The web-push infrastructure (`push.service.ts`, VAPID endpoints, `CompanionPushOptIn.tsx`) is now driven by the loop: `act.ts` → `deliverMorningCheckIn` → `check-in-delivery.service.ts` sends morning check-ins over native push → web push → email (ENG-221), and `reminder.service.ts` now calls `sendPushToUser` for daily reminders. Morning check-ins and reminders are reasoned about, decided, and delivered. (The July 2026 snapshot recorded these as decided-but-never-delivered; that is no longer true.)

**⚠ Operational item requiring verification.** The loop is gated by `COMPANION_AGENTIC_LOOP_ENABLED`, which per code **defaults on in production**. The Release 4 notes instruct setting it to `false` in production until clinical pilot — implying the safe state requires manual action. **If that variable is not explicitly set in the production environment, the loop is running against real users today**, making one LLM call per user per day and writing decision logs. It is inert in the sense that ACT cannot deliver, but it is consuming tokens and generating clinical reasoning records. **[check — verify the Render environment configuration directly.]**

### 4.7 The Companion AI coach

Routed via `ChatServiceRouter` on `patientId === null` → `CompanionChatService`, which injects the assembled memory brief into the system prompt on every reply.

The Companion system prompt (`src/services/chat/system-prompt.ts`, ~650 lines) contains:
- **Sports science grounding** — Friel periodisation, Gabbett acute:chronic workload ratio, Reaburn age-adjusted work:recovery ratios
- **Communication style rules** — decisive, non-hedging
- **A `<safety_protocols>` block** — progressive-overload percentage caps by age; an explicit red-flag list (chest pain, severe breathlessness, dizziness/fainting, neurological symptoms) with instruction to "STOP and seek medical help immediately"
- **A `<professional_standards>` block** — stating Kitt is not a doctor, physiotherapist, or dietitian, and must not diagnose or prescribe treatment

This is a genuine clinical-safety guardrail layer, not boilerplate.

### 4.8 Companion supporting services

`src/services/companion/`:

- **`plan-sharing.service.ts`** (1,485 lines) — the core engine. Covered in §5.
- **`sharing.service.ts`** — a *different* flow: shares a pre-signup preview session's data into a clinic org. Its `revokeSharing` admits in its own comment that RAG cleanup is incomplete.
- **`preview.service.ts`** — public marketing funnel. Generates a pre-signup training preview (LLM tip plus plan preview), persists a `PreSignupSession`, creates a shadow patient and note, and enqueues it into RAG.
- **`reminder.service.ts`** — daily email reminder cron reading `SharedPlanReminderPreference` (channel `email`), sending via `EmailService`.
- **`plan-sync-lock.ts`** — `runExclusivePlanSync`, a mutex preventing clinician plan edits from racing concurrent Companion syncs.
- **`context/`** and **`agentic-loop/`** — §4.5 and §4.6.

### 4.9 Companion API surface

Mounted twice in `src/index.ts`: `/api/public/companion` and `/api/companion` — the same router. **Authentication is enforced per-route inside the router, not by mount path**, so the "public" mount is not blanket-public.

`companion.routes.ts`:
- `GET /accept-plan/validate` — public, validates a `PlanShareToken`
- `GET /share-status` — public
- `POST /preview` — public, rate-limited, **Turnstile commented out**
- `POST /share` — public, rate-limited
- `GET /sessions` — authenticated
- `POST /accept-plan` — authenticated

`companion-plans.routes.ts` (all authenticated):
- `GET /documents` — user's own org-level documents only, filtered by `uploadedBy` to prevent cross-user leakage
- `GET|PUT /reminder-preferences`
- `GET /shared-plans`, `GET /shared-plans/:id`
- `POST /shared-plans/:id/completions`

### 4.10 Companion gaps

- **Device connectivity** — not built. Static "Coming Soon" cards with unused mock Garmin data.
- **Readiness metrics** — not built. Context types reserve `sleep`, `hrv`, `trainingLoad`, `connectedWearables` fields, permanently null. Owned by an unbuilt "Wearables project" (ENG-224/225/227 per docs). **No wearables model exists in the schema.**
- **Ring 1 profile capture** — no write path (§4.5)
- ~~**Agentic loop delivery** — stub (§4.6)~~ [Updated 2026-08-26: shipped — morning check-in and reminder delivery are wired (ENG-221); at-risk practitioner flag ships in production (product-owner confirmed). No longer a gap.]
- **Turnstile on public preview** — commented out (`companion.routes.ts:109-111`), leaving only rate limiting
- **Rate limiting is in-memory** — `companion-rate-limit.middleware.ts` notes "For production, consider Redis-backed implementation". It will not enforce correctly across multiple instances. Limits: 10/hr per IP, 5/24h per session, 3/min burst; applied only to `/preview` and `/share`.
- **Gamification not persisted** (§4.4)

---

## 5. The bridge: clinician → Companion plan sharing

This is the strongest cross-product axis in the codebase and the clearest strategic differentiator. Traced end-to-end in code.

### 5.1 The flow

1. **Clinician shares** — `POST /api/patient-plans/:planId/share-to-companion` → `sharePlanToCompanion`. If a `ClinicianPatientLink` already exists for (org, patient), the plan is pushed directly. Otherwise a `PlanShareToken` is minted — 32 random bytes, 7-day expiry — returning `https://<frontend>/accept-plan?token=…`.

2. **Patient opens the link** — `AcceptPlanPage` calls the public `GET /accept-plan/validate`. If logged out, it shows register/sign-in CTAs preserving the redirect. If logged in, it shows a consent screen.

3. **Patient accepts** — `POST /api/companion/accept-plan` → `acceptSharedPlan` creates or reuses a `ClinicianPatientLink` (the consent record) and calls `copyPlanToCompanion`.

4. **Copy with key re-wrapping** — `copyPlanToCompanion` decrypts take-home content and session context with the **source organisation's** encryption key and re-encrypts with the **Companion organisation's** key, respecting per-org AES key isolation. It generates a fresh LLM session summary (2–4 sentences) from the patient's latest FINAL note. Goals, prescribed exercises (with resolved presigned or external media URLs, 7-day expiry), program day/week labels, and day notes are copied row-by-row into the `SharedPlan*` tables.

5. **Ongoing sync** — `syncPlanToSharedPlans` re-pushes clinician edits to all active shares whenever a published plan is updated, guarded by `runExclusivePlanSync`.

6. **Adherence flows back** — patient logs completions → `logCompletions` → `syncComplianceToClinicianOrg` creates a `PatientPlanExerciseCompletion` on the clinician side in real time, idempotent on `sharedPlanCompletionId`.

7. **Revocation** — patient-initiated (`revokeSharedPlan`) or clinician-initiated (`revokeSharesForPlan`). Soft delete via `revokedAt`; completion history is retained for clinician audit.

### 5.2 What the clinician sees back

- `GET /:planId/compliance` — exercise compliance synced from Companion
- `GET /patient/:patientId/companion-memory` — adherence streaks and weekly pattern, surfaced in `CompanionMemoryPanel` on Patient 360
- `GET /patient/:patientId/companion-status` — whether a link exists

### 5.3 Assessment

This is a genuinely differentiated loop: prescribe → share → execute → log → adhere → report back, with proper consent records, per-tenant encryption boundaries, and real-time bidirectional sync. It is the most defensible thing in the product.

The weak links are at the edges rather than the core: gamification does not persist (§4.4), and the two completion forms capture different field sets (§4.4). [Updated 2026-08-26: reminders are no longer a weak link — they now deliver over push and email (§4.6).]

---

## 6. Data model reference

Single schema file, `prisma/schema.prisma` (1,854 lines), 28 migrations, Postgres with pgvector.

### 6.1 Enums (verbatim)

```prisma
enum UserRole { ADMIN, MEMBER, SUPER_ADMIN }
enum OrganizationType { PERSONAL, TEAM }
enum SubscriptionStatus { ACTIVE, PAST_DUE, CANCELED, TRIALING }
enum PaymentStatus { SUCCEEDED, FAILED, PENDING, REFUNDED }
enum SessionType { INITIAL, FOLLOW_UP }
enum SessionStatus { ACTIVE, PAUSED, COMPLETED, CANCELLED }
enum GenerationStatus { PENDING, COMPLETE, FAILED }
enum NoteStatus { DRAFT, FINAL }
enum PlanStatus { DRAFT, FINAL }
enum LetterStatus { DRAFT, SENT, ARCHIVED }
enum ChatRole { USER, ASSISTANT, SYSTEM, TOOL }
enum ProgramStatus { DRAFT, ACTIVE, COMPLETED, ARCHIVED }
enum IntegrationProvider { NOOKAL, CLINIKO, OTHER }
enum SyncStatus { PENDING, IN_PROGRESS, COMPLETED, FAILED, RATE_LIMITED }
enum DocumentStatus { ACTIVE, ARCHIVED, DELETED, PROCESSING, FAILED }
enum DocumentCategory {
  GENERAL, LAB_RESULTS, IMAGING, INSURANCE, REFERRALS, DISCHARGE_SUMMARY,
  TREATMENT_PLAN, CONSENT_FORMS, BILLING, CORRESPONDENCE, PRESCRIPTION,
  VITAL_SIGNS, PROGRESS_NOTES, OTHER
}
enum EmbeddingSourceType {
  DOCUMENT @map("document")
  NOTE @map("note")
  CHAT @map("chat")
  EXERCISE @map("exercise")
  CONVERSATION_SUMMARY @map("conversation_summary")
}
enum CompanionLoopEngagementOutcome { pending, engaged, not_engaged }
```

Note the thin workflow enums: `NoteStatus` and `PlanStatus` are `DRAFT`/`FINAL` only — no review or rejection state. Clinical sign-off is a single-step publish. (ENG-133, rehab builder clinician sign-off, was listed In Review at Release 4.)

### 6.2 Models by domain

**Identity / org** — `Organization` (tenant root; holds `encryptionKey`, `timezone` default `Australia/Brisbane`, `type`), `User` (role, org, preferences, `hasSeenOnboarding`, `hasSeenTour`, `selectedVoiceId` default `"marcus"`, per-PMS practitioner IDs, `firstLoginAt`, `emailVerifiedAt`), `Account` (OAuth linkage), `Tenant` (parallel legacy construct — see below), `CsrfToken`, `ApiKey`, `AlertNotification`, `PreSignupSession`.

**Patients** — `Patient` (heavily field-encrypted; `*Encrypted/*Iv/*Tag` triples plus `*Hash` for exact match and `firstNameSearchTokens`/`lastNameSearchTokens` HMAC prefix arrays for PHI-safe prefix search), `PatientSummary`, `ClinicianPatientLink` (the consent record bridging clinic patient to Companion user).

**Clinical documentation** — `DictationSession`, `TranscriptChunk`, `DynamicChecklistItem`, `NoteTemplate`, `PatientNote`, `PatientNoteAudit`, `PatientLetter`, `PatientLetterAudit`, `Document`, `ChatThread`, `ChatMessage`, `ChatMessageAttachment`, `ContentVersion`, `PHIAuditLog`, `AuditLog`.

**Plans / exercises** — `PlanTemplate`, `PatientPlan`, `PatientPlanGoal`, `PatientPlanProgressMeasure(Value)`, `PatientPlanPrescribedExercise`, `PatientPlanProgramDayLabel/WeekLabel/DayNote`, `PlanTemplateProgram*` equivalents, `MediaAsset`, `PatientPlanExerciseMedia`, `PatientPlanExerciseCompletion`, `Exercise`, `ExerciseVariant`, `PhysioProgram`, `PhysioProgramVersion`.

**Companion** — `CompanionProfile` (Ring 1), `CompanionDailyState` (Ring 3), `CompanionLoopDecisionLog`, `PushSubscription`, `SharedPlan` + `SharedPlanGoal`/`PrescribedExercise`/`ProgramDayLabel`/`WeekLabel`/`DayNote`/`ExerciseCompletion`, `SharedPlanReminderPreference`, `PlanShareToken`.

**Billing** — `Subscription`, `SubscriptionPayment`, `StripeCustomer`, `StripeProduct`, `StripePrice`, `UsageRecord`, `UserUsageRecord`.

**Integrations** — `Integration`, `IntegrationConfig`, `IntegrationMapping`, `SyncCheckpoint`, `IntegrationJob`, `IntegrationError`, `IntegrationSync`, `IntegrationSyncLog`.

**RAG** — `embedding_chunks` (pgvector, `Unsupported("vector")`, HNSW index).

**Extension** — `ExtensionInstallation`, `ExtensionActivity`.

### 6.3 Multi-tenancy

Enforced **by convention, not by database policy.** Every tenant-scoped table carries `organizationId` with `onDelete: Cascade` and org-leading indexes. There is no Postgres row-level security in the schema — isolation depends entirely on application code always filtering by `organizationId`. **[check — RLS applied outside Prisma cannot be ruled out from the schema alone.]**

This is the exact class of assumption that produced the October 2025 RAG leakage incident (§9.4).

Uniqueness constraints are org-scoped rather than global: `Patient` unique on `[patientId, organizationId]`, `Integration` on `[organizationId, provider]`, and so on.

`MediaAsset.organizationId` is deliberately nullable — null means a global shared asset, a documented escape hatch from strict isolation.

### 6.4 Encryption, audit, retention, versioning

**Encryption** — AES-256-GCM with per-organisation keys stored in `Organization.encryptionKey`. Encrypted: patient PII, transcript chunks, notes, summaries, dictation content, letters, chat content, embedding chunk content, Companion daily-state free text.

**Tamper evidence** — several encrypted fields carry parallel `*Hash`/`*Salt`/`*VerifiedAt` triples, apparently to detect post-generation modification of AI content. **[assumption]**

**Soft delete** (`deletedAt`/`deletedBy`/`deletionReason`) — on `Patient`, `PatientSummary`, `DictationSession`, `PatientPlan`, `PatientLetter`, `ChatThread`, `ChatMessage`. **Notably absent from `PatientNote`.** **[check]**

**Legal hold** (`legalHold`, reason, setAt, setBy) — on `Patient`, `PatientSummary`, `DictationSession`, `ChatThread`, `ChatMessage`. Absent from `PatientNote`, `PatientLetter`, `PatientPlan`. **[check]**

**Archive** (separate from delete) — `PatientPlan`, `PatientLetter`.

**Audit — three parallel systems:** entity-specific append-only audits (`PatientNoteAudit`, `PatientPlanAudit`, `PatientLetterAudit`), a PHI access log (`PHIAuditLog`, which records reads), and a generic system log (`AuditLog`).

**Versioning** — `PhysioProgramVersion` (numbered history), `ContentVersion` (generic polymorphic field-level history), plus simple integer `version` fields on `SharedPlan`, `NoteTemplate`, `PlanTemplate` with no history table.

### 6.5 Schema-level signals

- **`Tenant` is a parallel, apparently vestigial tenancy construct.** `User.organizationId` is required; `User.tenantId` is optional. `Tenant` has its own `apiKeys`, `usageRecords`, `subscriptionId`, `notifications`. Likely a deprecated precursor or a dormant reseller/MSP tier. **[check — confirm nothing depends on it before removal.]**
- **`PhysioProgram` is a parallel plan system.** Markdown/HTML program content with a public `shareToken`, coexisting with the structured `PatientPlan` system, with **no join between them**. Two of its routes return `501 Not Implemented`. **[corroborated — reached independently from schema analysis and from route analysis.]** Almost certainly a legacy generation path. It is additionally gated behind `requirePhysioFeature`, a per-org allowlist via `PHYSIO_ENABLED_ORGS`.
- **Two plan-sharing mechanisms exist** — token-link (`PlanShareToken`) and consent-based (`ClinicianPatientLink` → `SharedPlan`). Both are live and used in sequence, but the duplication is worth understanding.
- **`IntegrationProvider.OTHER`** is reserved and never used at runtime.
- **`DynamicChecklistItem`** backs a sophisticated real-time AI dictation-assist feature that may not be surfaced prominently in the UI.

### 6.6 Migration trajectory

The last ~15 migrations tell a clear story. After stabilising PMS integration correctness (provider-keyed checkpoints, practitioner mapping provenance, PHI-safe search) and completing the structured rehab plan and calendar builder, effort pivoted sharply to **Companion as an autonomous agent**: three-ring memory (`20260611000000_companion_memory_rings`), conversation-summary embeddings (`20260612000000`), the agentic loop decision log (`20260612120000`), web push (`20260615120000_push_subscriptions`), and an HNSW vector index for scale (`20260615120000_performance_indexes`).

**[corroborated]** — this matches the release-notes trajectory independently.

---

## 7. Integrations

### 7.1 Architecture

A clean adapter pattern: `IntegrationProvider` interface (`src/services/integrations/provider.ts`), implemented by `cliniko.provider.ts` and `nookal.provider.ts`, instantiated per-org from decrypted credentials by `factory.ts`. A single `sync-engine.ts` plus five provider-agnostic processors (patient, note, document, case, practitioner) drive sync without branching on provider identity.

### 7.2 Coverage and direction

| Entity | Nookal | Cliniko | Direction |
|---|---|---|---|
| Patients | ✅ | ✅ | Pull, plus optional push-back gated by per-org `pushPatientsEnabled` |
| Notes | ✅ | ✅ | Bulk pull-only; **manual per-note push** via explicit routes |
| Documents | ✅ | ✅ | Bulk pull-only; **manual per-document push** via explicit routes |
| Cases | ✅ | ✅ | Pull-only (Kitt has no native Case model; only mirrors IDs into `IntegrationMapping`) |
| Practitioners | ✅ | ✅ | Listing and mapping |

Source-of-record is configurable per entity per org via `IntegrationConfig` (`patientsSor`, `notesSor`, `documentsSor`, `casesSor`), each defaulting to `"provider"` — the PMS is authoritative by default.

### 7.3 Operational characteristics

- **Triggers:** scheduled every 30 min (staggered per org, gated on `INTEGRATION_SCHEDULED_SYNC_ENABLED`), manual on-demand (higher queue priority). **No webhook-driven or login-triggered sync.**

> **⚠ Conflicting evidence — daily reconciliation may not run.** One investigator reported a daily full reconciliation sync that clears checkpoints and forces a full re-fetch. A second investigator, reading `scheduled-sync.service.ts` directly, found that **`runReconciliationSync()` is fully implemented but never invoked anywhere** — neither `src/index.ts` nor any other file calls it; only its own definition and doc comment reference the name. Its doc comment describes a "daily at 03:00" schedule that does not appear to be wired up.
>
> If correct, incremental sync is the only sync that runs, and drift from the PMS would never be reconciled. Given the provider-keyed-checkpoint bug this mechanism was built to recover from, that matters. **[uncertain — resolve by grepping for `runReconciliationSync` call sites and checking the scheduler registration in `src/index.ts`.]**
- **Queue:** BullMQ `integration_sync`, 3 attempts with exponential backoff, `concurrency: 1` explicitly to avoid memory spikes and rate limits.
- **Rate limits:** token bucket, hardcoded — **Cliniko 200/min, Nookal 120/min**.
- **Errors:** structured `IntegrationError` with `retryCategory` (transient/permanent/rate_limit), plus an 8-category user-facing error translator.
- **Credentials:** encrypted at rest via `EncryptionService`. Nookal key rotation evicts stale OAuth cache and resets sync checkpoints.

### 7.4 Integrations UI

`IntegrationsPage.tsx` (1,403 lines) is the most complete single page in the application: provider selection, connection status, setup wizard, credential entry (Nookal needs Company ID + API key; Cliniko needs API key only with shard auto-detection), disconnect, per-entity source-of-record dropdowns, manual per-entity sync triggers, sync status tab with statistics and 30s auto-refresh, sync history with filters and pagination, and a practitioner mapping panel.

It also carries evidence of a prior security fix: explicit `localStorage` cleanup removing previously-stored API keys, with comments stating keys are intentionally never restored and always require re-entry.

### 7.5 No third PMS

No other practice-management system (Halaxy, Power Diary, etc.) is referenced anywhere in code or docs. There is no documented-but-unbuilt third integration.

---

## 8. Admin and super-admin

### 8.1 Built

- **Super Admin Dashboard** — MRR, new organisations, new subscribers, churn rate, customer health tiers, subscription counts, retention metrics, recent signups, at-risk organisations. (MRR uses a placeholder pricing map — §3.9.)
- **Organisation management** — list all with search and pagination, create, view detail, edit, toggle active status, delete (blocked if the org has users).
- **Cross-org user management** — list all, filter by org, update, delete, reset password, activate/deactivate.
- **User impersonation** — `POST /api/users/:id/assume` issues a new JWT carrying an `impersonation` block; `POST /api/users/exit-impersonation` reverses it. Self-impersonation is blocked. An `ImpersonationBanner` shows in the client.
- **Internal support docs** — `/api/support-docs`, SUPER_ADMIN only, path-traversal guarded. The UI hardcodes ~20 aspirational doc entries and greys out missing ones as "(Coming soon)".
- **Daily admin digest** — emails all active SUPER_ADMINs new signups and per-org token usage. Cron `30 5 * * *`, production only.
- **Retention admin** — policies, stats, per-class and global cleanup (with `dryRun`), scheduler control, export.
- **Soft-delete admin** — soft delete with required reason and legal-hold check, restore, legal-hold set/clear, stats, can-delete check.
- **System admin** — memory stats, forced GC (requires `--expose-gc`).

### 8.2 Not built

Per the feature docs' own "Future Enhancements" sections, confirmed absent in code: feature flags UI, billing overrides, system health monitoring, real-time dashboard updates, historical trend charts, dashboard export, time-limited impersonation sessions, notification to the impersonated user.

### 8.3 Impersonation is not audit-logged

**Neither the assume nor the exit route calls `logAuditEvent`. There is no `ImpersonationLog` model and no `AuditEventType` value for impersonation.** SUPER_ADMIN access to any user's account leaves no audit record. `USER_IMPERSONATION.md` admits this in its own Future Enhancements section.

For a product handling PHI under an Australian Privacy Act posture, this is the compliance gap most likely to matter under scrutiny. **[check — verify no logging exists at a layer not inspected.]**

---

## 9. Security and compliance posture

This section records what the code and repository documents actually say. It is not a security audit, and several items are single-source.

### 9.1 Compliance claims

**No certifications are claimed anywhere in the repository.**

- HIPAA and the Australian Privacy Act 1988 / APPs appear as **design targets**, not attainments
- ISO 27001 and SOC 2 appear only as **reference standards** in requirements docs, never as attained
- `docs/features/privacy/privacy_requirements.md` at time of writing **explicitly marked the app "❌ NON-COMPLIANT"** (no encryption at rest) and instructed removing false encryption claims from the public privacy policy
- The My Health Records Act is not mentioned at all
- Live user-facing policy pages cite the Privacy Act 1988 (Cth) and the APPs

Anyone making a compliance claim in marketing or sales should verify against current reality first — the repository does not support certification claims.

### 9.2 Encryption

AES-256-GCM, per-organisation keys. **The key is stored unencrypted in the same database as the ciphertext it protects** (`Organization.encryptionKey`). A database compromise defeats the encryption entirely. This is a meaningful architectural limitation rather than a bug — it protects against some threat models (backup exposure, partial exfiltration) and not others.

### 9.3 Findings requiring verification

All single-source unless marked. Ordered by my assessment of severity.

| # | Finding | Location | Confidence |
|---|---|---|---|
| 1 | **CSRF validation globally disabled.** `setCsrfToken` runs; `validateCsrfToken` commented out. App presents as protected while nothing validates. | `src/index.ts:340` | **[corroborated]** |
| 2 | **Backup encryption effectively broken.** Uses deprecated `crypto.createCipher` (not `createCipheriv`), and writes the key in plaintext at `${path}.key` beside the encrypted file. | backup service | **[check]** |
| 3 | **JWT secrets have insecure fallbacks.** Defaults to `'your-default-jwt-secret'` / `'your-default-refresh-secret'` if env vars are unset. | `src/services/auth/service.ts:39-40` | **[check]** |
| 4 | **Impersonation is not audit-logged.** | `users.routes.ts` | **[check]** |
| 5 | **`requireSuperAdmin` grants access whenever `req.user.impersonation` is truthy**, regardless of the impersonated user's actual role. | `src/middleware/auth.ts:195` | **[check]** |
| 6 | **API key routes have no admin gate.** Any authenticated org member can create, list, and delete the organisation's API keys. `requireSameOrganization` is imported but never applied. | `apikey.routes.ts` | **[check]** |
| 7 | **Plaintext temporary passwords returned in API responses** on user create and admin password reset, gated only by a comment, not code. Displayed in the UI with a copy button. | `users.routes.ts`, `UsersPage.tsx` | **[check]** |
| 8 | **`/patient-cases` (holding PHI) lacks the `TeamOnly` guard** applied to every other patient-data route. | `App.tsx:476` | **[check]** |
| 9 | **`admin/user-usage` `GET /top-users`** is documented Super Admin only but enforced with `requireAdmin` — any org admin may pull cross-org usage data. | `admin/user-usage.routes.ts` | **[check]** |
| 10 | **Two webhook endpoints have no auth middleware** — `alerts.routes.ts POST /process` and `notifications.routes.ts POST /webhook/alertmanager` (whose own doc comment claims `@access Private`). | as listed | **[check]** |
| 11 | **Debug routes have zero in-file auth**, gated only by a `NODE_ENV !== 'production'` check at registration — fully open on any staging or preview deploy. `debug-schema.routes.ts` dumps table column lists. | `debug.routes.ts`, `debug-schema.routes.ts` | **[check]** |
| 12 | **PHI audit logs auto-delete at 6 years** against a documented 7-year manual-review policy — destroying audit evidence early, without the manual gate the policy requires. | `PHIAuditRetentionService` | **[check]** |
| 13 | **Four overlapping auth middleware implementations**, one of which (`require-auth.ts`) bypasses the active-user/cache check the canonical one performs, and logs raw token payloads via `logger.info`. | `src/middleware/*` | **[check]** |
| 14 | **No PHI audit middleware** on dictation sessions, treatment plans, checklists, or note regeneration routes — inconsistent with coverage on notes, plans, letters, and transcripts. | `src/api/routes/dictation/*` | **[check]** |
| 15 | **`rag.routes.ts GET /health` has no authentication**, exposing config presence booleans. | `rag.routes.ts` | **[check]** |
| 16 | **No rate limiting on individual route files**, including LLM-generation endpoints — though a global `rateLimiter` does run (100/min prod). | global | **[corroborated]** |
| 17 | **Turnstile disabled on Companion public preview** (active on signup). | `companion.routes.ts:109-111` | **[corroborated]** |
| 18 | **Companion rate limiting is in-memory** and will not enforce across multiple instances. | `companion-rate-limit.middleware.ts` | **[check]** |
| 19 | **No refresh-token rotation or revocation.** Stolen refresh tokens remain valid 7 days regardless of logout. | auth service | **[check]** |
| 20 | **Email verification and invitation tokens have no expiry check**, despite the email promising 7 days. | `auth/service.ts:243-294` | **[check]** |

### 9.4 Prior incidents on record

- **`docs/security/RAG_DATA_LEAKAGE_INCIDENT.md`** (30 Oct 2025) — a SQL filter bug in `retrieval.service.ts` allowed org-level Companion chat to retrieve *any* patient's embedding chunks within the same organisation. Cross-patient PHI leakage, marked CRITICAL. Fixed by explicitly requiring `patient_id IS NULL`.
- **`EMBEDDING_CLEANUP_FIX.md`** — deleted documents' content could still surface via RAG to the same user. No cross-user or cross-org exposure per that document. Fixed by deactivating embeddings on delete.

Given §6.3 (tenancy by convention, no RLS), RAG isolation should remain a permanent regression-test target rather than a closed issue.

---

## 10. Observability

- **Metrics** — `prom-client`, `/metrics` endpoint. Tracks API, DB, system, tenant usage/subscription/token, LLM request duration/errors/fallback counts, WebSocket, BullMQ, auth attempts.
- **LLM tracing** — Langfuse (`https://us.cloud.langfuse.com` default), with PHI-safe I/O redaction gated by `LANGFUSE_LOG_LLM_CONTENT`.
- **Error tracking** — Rollbar with healthcare-specific field scrubbing (`patientId`, `diagnosis`, `medication`).
- **Tracing/logs** — OpenTelemetry → Grafana Cloud OTLP; `pino` → Grafana Cloud Loki in production, wrapped in a PHI-scrubbing `SafeLogger`.
- **Product analytics** — entirely in-house from Postgres. **No PostHog, Mixpanel, Amplitude, or Segment.** Client-side `gtag` events fire on onboarding completion/skip.

---

## 11. Gaps register

### 11.1 Built but unreachable

| Item | Detail |
|---|---|
| `DictationPage.tsx` | 1,267 lines, no route, nav link commented out, tests skipped |
| `src/api/routes/index.ts` | Orphaned route aggregator with `// ... existing imports ...` placeholders; nothing imports it |
| `extension.routes.ts` | Fully implemented, reachable only through the orphaned aggregator — **not mounted** |
| `analytics/business.routes.ts` + controller | Duplicate of 5 endpoints already live in `analytics.routes.ts` |
| `NavBar.tsx` | Complete alternate nav, zero imports |
| `MobileTable.tsx` | Zero imports |
| `components/chat/` (entire directory) | 5 files, zero external imports; superseded by `components/patient-record/chat/` |
| `ExerciseWithLogCard.tsx` | Zero imports |
| `pages/companion/CompanionDashboard.tsx` | Mock data, unrouted |
| `pages/companion/CompanionChatPanel.tsx` | Duplicate of the live component, unrouted |
| `stores/companionStore.ts` | Used only by the two dead Companion pages |
| `AdminDashboardPage` `ActionsDropdown` | Built with View Details / Impersonate, never mounted |
| `resource-auth.ts` `canAccessResource` | Exported, zero route callers |
| `integrations/middlewares.ts` guards | Defined, not wired to any route |
| `FEATURES.physioPrograms` | Flag defined, zero consumers |
| `patient-record/DocumentsTab.tsx` | 1,059 lines, zero importers. Superseded by `PatientDocumentsPanel.tsx`. Holds 7 of the client's 11 TODOs — all unreachable. |
| `patient-record/NoteGenerationChat.tsx` | 291 lines, zero importers |
| `integrations/setup/` — 5 wizard steps | `WelcomeStep`, `ProviderSelectionStep`, `ConfigurationStep`, `CredentialsStep`, `FirstSyncStep` fully built, zero importers. The live `SetupWizard` wires only `ConnectStep` and `SyncStep`. |
| Duplicate `NotificationCenter` | Two independent near-identical implementations in `components/notifications/` and `components/integrations/`, both polling sync status every 30s |
| `src/index.ts:20` | Static `emailPreviewRoutes` import never used; actual mount is a second dynamic import at `:478` |

### 11.2 Stubs and placeholders

| Item | Detail |
|---|---|
| `OrganizationPage` tabs | 4 of 5 render "will be implemented here" |
| `/api-keys` route | Renders `PlaceholderPage` |
| `physio.routes.ts` | `GET /programs/:id` and `GET /programs/:id/versions` return `501` |
| `chat.routes.ts` attachments GET | Always returns `[]`; real query commented out |
| Companion Device card | "Coming Soon", mock Garmin data |
| Companion Readiness card | "Coming Soon", all values `--` |
| Dictation "Context" tab | "Context analysis coming soon" |
| Agentic loop ACT delivery | [Updated 2026-08-26: SHIPPED — no longer a stub. Check-in/reminder delivery wired (ENG-221); at-risk flag ships in prod (product-owner confirmed). Only the `act.ts:133` docstring is stale.] |
| `CompanionProfile` (Ring 1) | No write path anywhere |
| Apple OAuth | Env read, logs "not yet implemented" |
| `src/services/prompt` | Types only, no implementation |
| `src/services/generation` | Status plumbing and pub/sub; no LLM work despite the name |
| Resend verification email button | No `onClick` handler; backend method exists unused |
| `rememberMe` on login | Checkbox drives nothing |
| Exercise creation | Hardcoded `https://placeholder.local` video; no edit UI |
| Support docs landing | ~20 hardcoded entries, missing ones greyed "(Coming soon)" |
| `user-usage` export | JSON only; comment notes CSV "can be added later" |
| Super-admin MRR | Hardcoded `planPricing` placeholder map |
| Rehab "start from template" | `MOCK_REHAB_TEMPLATES` — 2 hardcoded fake templates, no backend catalogue |
| Dictation audio level meter | `SessionControls.tsx:78-89` — driven by `Math.random()` every 100ms. Own comment: "replace with real audio level detection". Not connected to microphone input. |
| `IntegrationContext.tsx` "last seen" | Always returns the literal string `'Yesterday'` (lines 96-100) |
| `IntegrationContext.tsx` "recent notes" | Always returns the number `3` (lines 102-106) |
| `IntegrationContext.tsx` "View in Nookal/Cliniko" | Logs a click, navigates nowhere (line 79) |
| `ChatInput.tsx` file attachment | Picker opens; `handleFileChange` only logs "File selection triggered". Files are never attached (lines 92-99). |
| `SessionManager.tsx` | Hardcoded `practitionerId: 'current-user-id'` (line 81) |
| `SubscriptionDetails.tsx` | "The billing portal is not yet configured" fallback; free-plan users see a "Billing history available after upgrade" placeholder |

### 11.3 Broken or defective

| Item | Detail |
|---|---|
| `/note-templates` link | Route does not exist; templates are at `/templates`. Dead admin link. |
| `/patients` duplicate route | Declared twice; second unreachable |
| `GET /api/dictation/sessions/stats` | Shadowed by `GET /:id`; returns 400, never reached |
| `AdminDashboardPage` | Transform hardcodes zeros; page permanently empty |
| Companion gamification | XP/level/streak never persisted; resets on reload |
| Two completion forms | Sidebar form omits pain score and additional weight |
| `PatientCasesPage` internal name | Declared as `PatientNotesPage` |
| No case-management UI | Advertised on the hub, does not exist |
| `documents.routes.ts` | `router.post('/generate-pdf')` registered *after* `export default router` |
| `documents.routes.ts` | `/generate-from-content` passes `auditPHI` unconfigured, unlike every other call site |
| `dictation/patients` | `GET /patient-id/:patientId` (a read) wrapped in `auditPatientWrite()`; `DELETE /:id` has no audit at all |
| N+1 fetch patterns | `PatientsPage` (sync status per patient), `MediaLibraryPage` (URL per asset) |
| `CheckoutSuccessPage` | Creates subscriptions client-side as a dev-mode webhook substitute; errors silently swallowed |
| `POST /api/documents/generate-from-content` | `auditPHI` factory registered uninvoked → `next()` never called → **requests hang until timeout**. §3.4 |
| `POST /api/livekit/webhook` | Sits behind `authenticate` + `extractOrganizationContext`, so a real LiveKit-originated call (carrying LiveKit's own signed token, not a Kitt JWT) is rejected before the handler runs. It also reads `req.rawBody`, which **no middleware anywhere sets**, so signature validation falls back to `JSON.stringify(req.body)` — not byte-identical to what was signed. Its own comment says it must be registered before `express.json()`; it is registered at `index.ts:406`, well after `:335`. Appears non-functional for real webhook traffic. **[check]** |
| Plan generation WebSocket | `onPlanGenerated` never subscribed; events emitted into a void. §2.7 |
| `runReconciliationSync()` | Implemented, apparently never invoked. §7.3 **[uncertain]** |

### 11.4 Documented but unbuilt

| Item | Source |
|---|---|
| Per-seat Clinician pricing | `docs/personal-organizations/README.md` — env vars absent from `src/` |
| Wearables / device integration | Companion docs, ENG-224/225/227 — no model, no service |
| Readiness metrics | Companion docs — UI placeholder only |
| Agentic loop LEARN-driven delivery | [Updated 2026-08-26: ACT delivery now wired (ENG-221); this row is resolved.] |
| In-app change password | ENG-257, listed as a known gap at Release 4 |
| Reorderable day notes | ENG-192, In Progress at Release 4 |
| Rehab builder clinician sign-off | ENG-133, In Review at Release 4 |
| Multi-EMR support beyond Nookal/Cliniko | Referenced in extension docs; no third provider in code |

---

## 12. Commercial and product risk register

Distinct from §9 (security). These affect revenue, positioning, or clinical safety.

| # | Risk | Detail |
|---|---|---|
| 1 | **Uncapped token spend** | Only letter generation enforces the limit. Chat, summaries, tagging, titles, and dictation run past cap. Direct margin exposure against Sonnet-4 pricing. §2.5 |
| 2 | **TEAM orgs may start at 0 tokens** | One signup path sets `monthlyTokenLimit: 0` for TEAM. Potentially blocks new clinic trials at the door. §2.5 |
| 3 | **Agentic loop may be live in production** | Defaults on; safe state requires explicit env var. If unset, daily LLM calls per user, uncapped by §1. [Updated 2026-08-26: ACT now delivers to users (ENG-221), so a live loop is no longer inert — it sends real check-ins/reminders, raising the stakes of the env-var default.] §4.6 |
| 4 | **No model configurability** | Model IDs hardcoded. Cost or quality changes require a deploy. §2.6 |
| 5 | **Single LLM dependency** | Fallback exists only if `LLM_API_KEY` is set. OpenRouter outage is total generative failure otherwise. §2.6 |
| 6 | **Gamification doesn't persist** | The retention mechanic resets every reload. §4.4 |
| 7 | **Companion memory is 2 of 3 rings** | Ring 1 never populated — the coach knows nothing about who the user is beyond their plan and daily state. §4.5 |
| 8 | **Compliance claims unsupported** | No certifications. One doc explicitly marks non-compliance. §9.1 |
| 9 | **Product boundary unresolved** | Companion straddles B2C fitness coach and clinician-linked rehab companion. Signup, prompt, and quick-actions all fork on this. |
| 10 | **Exercise library not self-serviceable** | Placeholder video URLs, no edit UI. Content ops require DB access. §3.8 |

---

## 13. Documentation drift register

Where repository documentation contradicts code. Treat all of these as unreliable.

| Document | Claims | Reality |
|---|---|---|
| `docs/product/FEATURE_BASELINE_REVIEW.md` (2026-04-27) | Turnstile disabled; agentic loop absent; Companion has no dashboard | Turnstile shipped on signup (June); loop shipped (June); Companion has 4 routed screens. **Stale — do not cite.** |
| `src/services/llm/README.md` | OpenAI is production default | Anthropic via OpenRouter is primary. Code contradicts. |
| `src/services/llm/anthropic-README.md` | Lists claude-3-sonnet/opus/haiku as available | None appear in source. Documentation only. |
| `docs/personal-organizations/README.md` | Per-seat pricing env vars | Absent from `src/` entirely |
| `docs/plans/plan-sharing-clinician-to-companion.md` | Schema shape incl. `sourceExerciseId` | Schema evolved: `sourcePrescribedExerciseId`, plus tempo/painScore/additionalWeight and day/week label tables not in the doc |
| `docs/companion-ui-setup.md` | Zustand store, readiness/workout endpoints | Current app uses `CompanionPage` + chat/plan APIs. The Zustand store is dead code. |
| `src/services/integrations/README.md`, `INTEGRATION_EXAMPLE.md` | node-cron + `NookalSyncService` wrapper | Actual implementation is BullMQ + `scheduled-sync.service.ts` |
| `docs/features/patient-sync/IMPLEMENTATION_SUMMARY.md` | `PatientMapping` model | Replaced by generic `IntegrationMapping` |
| `RETENTION_POLICY_GUIDE.md` | Audit logs 7yr, manual review only | Code auto-deletes at 6yr |
| `USER_IMPERSONATION.md` | (Future Enhancements) audit logging planned | Confirmed absent in code |
| `docs/features/PATIENT_360_TOKEN_METERING_PLAN.md` | Success = limits work with new sources | Instrumentation done; enforcement not |
| `notifications.routes.ts` webhook | `@access Private` | No auth middleware |
| `admin/user-usage.routes.ts` `/top-users` | `@access Super Admin only` | Enforced with `requireAdmin` |
| `Navigation.tsx:458` comment | "visible to everyone in development, only super_admins in production" | Actually gated on `isTeamOrganization` |
| `provider.ts:86` comment | Practitioner listing unsupported | Both providers support it |

---

## 14. Open questions

Questions the code cannot answer, which need a product decision:

1. **Is Companion a B2C fitness coach or a clinician-linked rehab companion?** The code supports both and the split shows in signup, system prompt, quick actions, and the Companion→Clinician upsell in onboarding. The clinician-linked path is far more built.
2. **Is the agentic loop intended to be live?** If yes, ACT delivery (ENG-221) is the blocker. If no, the env default should be inverted so the safe state is the default.
3. **Should Ring 1 profile capture be built?** The memory architecture assumes it. Without it, the coach's personalisation is limited to plan and daily state.
4. **Is `PhysioProgram` being sunset?** Two 501s, no join to `PatientPlan`, gated behind an org allowlist. If it is dead, deleting it removes real confusion.
5. **Is `Tenant` load-bearing?** It looks vestigial but has relations across usage, API keys, and notifications.
6. **What is the CSRF re-enablement milestone?** Flagged in April, still disabled in July.
7. **Should per-seat Clinician pricing exist?** Documented, unbuilt, and the current model is flat per-org.
8. **Is dictation a product surface or an embedded feature?** The code has answered embedded; the 1,267-line standalone page should be deleted or revived.

---

## 15. Appendix: verification shortlist

If you act on nothing else, these are cheap to check and consequential. Ordered by expected value.

**Likely broken in production right now:**

1. **`POST /api/documents/generate-from-content` hangs** — `auditPHI` registered uninvoked. One-character fix. Check production logs for timeouts on that route. §3.4
2. **`COMPANION_AGENTIC_LOOP_ENABLED`** in the production Render environment — defaults on; if unset, the loop is running against real users. §4.6
3. **`runReconciliationSync()` may never run** — grep for call sites. If confirmed, PMS drift is never reconciled. §7.3
4. **`GET /api/dictation/sessions/stats`** returns 400 due to route shadowing. §3.6

**Security, cheap to verify:**

5. `JWT_SECRET` and `JWT_REFRESH_SECRET` are actually set in every environment — §9.3 #3
6. Whether `validateCsrfToken` can be re-enabled — §9.3 #1
7. The backup encryption path, and whether any backup has ever been relied upon — §9.3 #2
8. Whether `/patient-cases` should carry `TeamOnly` — §9.3 #8
9. Whether API-key routes should require admin — §9.3 #6

**Commercial:**

10. Whether `checkUsageLimit()` should gate chat and dictation — §2.5. Highest-value item on this list financially.
11. Whether TEAM signups are landing with `monthlyTokenLimit: 0` — §2.5
12. The super-admin `planPricing` map against live Stripe pricing — §3.9

**Copy and positioning:**

13. The `/patient-notes` hub still advertises cases, which were deliberately retired — §3.2
14. "Start from template" offers two hardcoded fake templates — §3.5

---

*Compiled from read-only inspection of the Kitt repository at commit `99312bc3` on 20 July 2026. No repository files were modified. Findings marked [check] are single-source and should be verified before action is taken on them.*
