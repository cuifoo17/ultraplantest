# Purr.ai — Research Status

**Last updated:** 2026-04-07 (updated after April 5 flags/prompt session + April 6 notifications/OB session + April 7 budget/SSE/habits/lifecycle session)
**Purpose:** Living checklist of all architectural and product decisions, organized by how hard they are to change later. Tracks what's locked, what's changed, and what still needs a decision. Also serves as a session handoff document.

---

## Pressing Questions & Topics

Things I (Braulio) specifically want to dive deep into. May overlap with tier checklists below — this is the "what's on my mind right now" list.

- [ ] **User Firestore document — map every field.** Understand every piece of data stored on the user's Firestore document: what each field is, who writes it (Cloud Function, RevenueCat webhook, iOS app), when it gets read, and what breaks if it's wrong. Fields are accumulating across sessions (budget, session_count, ob_complete, account_locked, char_limit_rejections, subscription status, timezone, etc.) and need to be reviewed as a complete picture.

- [x] **Inline photos in chat messages — RESEARCHED AND INTEGRATED (April 7).** Full research in doc 25. Key decisions: `message` field is now an array of content blocks (text + image), not a plain string. Images uploaded to Firebase Storage by the app before sending, URL passed to CF, CF constructs Anthropic URL-based image content blocks (no base64). Images stored as URLs in conversation history, passed through on future turns, auto-cached by Anthropic (90% cheaper after first turn). Token formula: `(width × height) / 750`. Typical image: ~1,590 tokens (~$0.005). At 100K MAU: ~$5K/month additional cost. `payload_version` added to payload from day one. Lifecycle Steps 1, 3, 11, 15 updated in Plumbing Blueprint. All other steps unchanged.

- [x] **Conversation ID — DECIDED (April 8).** App generates it. When the user starts a new conversation, the app creates a UUID locally (`UUID().uuidString` in Swift — one line of code). The app uses it immediately for image uploads and message payloads. The CF receives it and uses it as the Firestore path: `users/{uid}/conversations/{conversation_id}/messages/{message_id}`. Security is in the Firestore path, not the ID format — conversations live under `users/{uid}/`, and the uid comes from the verified auth token (Step 2), not the payload. A malicious user sending a custom conversation_id can only create conversations under their own account. App Check (Firebase) is the long-term answer for verifying requests come from the real app — deferred to post-V1.

- [ ] **General Firestore structure — non-user documents.** The user document field inventory (first pressing question) covers per-user data. But the CF also reads from shared/global Firestore documents: CP2 persona prompts (normal, grief, OB), care standards block, contextual flag text, model pricing config table, and potentially more. Where do these live? What collection structure? Who writes them (manual deploy, admin tool, Firestore console)? How often do they change? Should the CF cache them in memory to avoid a Firestore read on every message? This is a separate mapping exercise from the user document inventory.

- [ ] **Cached context document — 1 read vs 12 reads per message.** Step 8 of the lifecycle reads memory tiers, reminders, routines, challenges, and conversation history in parallel. Without optimization, that's 8-12 separate Firestore reads per message. The cached context document (already listed in Tier 2 of Research_Status) would pre-assemble all per-user context into a single document, collapsing Step 8 to 2 reads (one context doc + one conversation history query). Not critical for V1 at low user counts, but at 100K MAU the difference in latency and Firestore cost adds up. Need to decide: when is this built (V1 or post-launch), what goes into the cached doc, who writes it, when does it refresh?

- [ ] **Conversation history and long conversations.** The conversation history query in Step 8 reads ALL prior messages for a conversation_id. Claws suggests starting a new conversation at turn 15-20 but never forces. If a user ignores the suggestion, history grows unbounded. Questions: (1) What's the practical context window limit before quality degrades? (2) Does the server-side compaction API (beta, from Research doc 06) solve this? (3) Should there be a hard cap where the CF refuses to load more history? (4) How do images in conversation history compound the size problem? (5) At what point does loading history become a latency issue for Step 8?

- [ ] **Cloud Function request/response lifecycle — end-to-end.** Walk through the full exchange: what the iOS app sends (payload shape), what the Cloud Function does with it (every check and action in order), what gets sent to Anthropic, what comes back, what the Cloud Function sends back to the app via SSE, and what a malicious actor could intercept at each step. Goal is to understand every piece of the pipeline in plain terms, including failure modes.

- [x] **Multi-model strategy — DECIDED.** Launch Sonnet-only. Build model routing into the architecture from day one (model_id variable in Cloud Function, never hardcoded). V1 always resolves to Sonnet, but the plumbing supports switching to Opus or Haiku per-user, per-conversation, or per-tier without any refactoring. Options explored and parked for future: (a) tier-based routing (Haiku/Sonnet/Opus by subscription tier), (b) dynamic routing by user behavior (power users get Opus, casual users get Sonnet), (c) context-based routing (grief conversations get Opus automatically), (d) Sonnet self-handoff to Opus via tool call for emotional escalation. All viable later because architecture is model-agnostic. Key insight: emotional attachment in a companion app forms primarily around memory + personality + knowing the user's cats (~85%), not raw model quality (~15%). Sonnet with good architecture beats Opus without it. Opus is a future optimization for high-value moments, not a launch requirement.

---

## Cost/Benefit Decision Framework

Every engineering decision in this app should be evaluated in this priority order:

1. **UX first.** Does this make the app feel better, more reliable, or more delightful to the user? If yes, strong bias toward doing it.
2. **Complexity second.** Does this reduce technical complexity, chances of bugs, or operational burden? Simpler systems are more reliable systems. Reliability IS UX.
3. **Cost third.** Only after UX and complexity are settled, optimize for cost. And always normalize cost against revenue, not in isolation.

**How to normalize cost:** Cost is always relative — compare it to the alternative, to existing costs, and to revenue. There's no magic threshold. A $2,000/month difference means nothing at $1M revenue but everything at $5K revenue. A free option that's 95% as good might still lose to a paid option if the 5% gap hits UX. Always ask: relative to what?

**The empty chair test:** Before any "should we do X to save money?" discussion, ask: does the user notice? Does the user care? A user paying $10-60/month for a cat care companion will churn over unreliable messages, weird tone shifts, or slow responses long before they churn over a price increase that funds better infrastructure. Optimize for the thing that retains users, not the thing that saves pennies per message.

**When cost DOES matter:** Tier pricing. The cost per message directly determines how many messages a user gets for their money. Token anxiety = churn risk. Every engineering decision should also be evaluated as "does this give the user more messages before throttle?"

---

## Braulio's Evolving Knowledge

What I've internalized so far and key mental models, so any new session can pick up where I left off.

### Cache Architecture (solid understanding)
- 4 breakpoints max per request — Anthropic hard limit, Tier 0 constraint
- Our allocation: CP1 (tools, ~3,500 tokens, global) → CP2 (persona variant, keep brief, global per variant) → CP3 (per-user content, per session) → CP4 (conversation history, auto-cached)
- CP2 = who Claws IS (swap for grief/OB/normal). CP3 = what Claws KNOWS about this user (cat data, memory, contextual modes like diet coaching). New features go in CP3, new personas go in CP2. Never burn a breakpoint on conditional content.
- Tools + persona are globally shared and always warm at 10K+ MAU. Per-user is a cache write every new session. Conversation auto-advances.
- Cache read refreshes TTL at no cost. First unlucky user of the day pays the cache write (~$0.03). Trivial at scale.
- Grief persona is a separate CP2 variant, not a conditional block. Zero contamination risk, zero breakpoint cost. Research confirms: separate prompts for high-stakes modes, conditional blocks only for low-stakes context.

### Cost Model (partial understanding)
- Tool cache read: ~$0.0011/message. Persona cache read: ~$0.0009/message. Combined global portion: ~$0.0020/message.
- Per-user cache write on first message of session: TBD (depends on per-user content size, still need to estimate)
- Output tokens are the biggest cost lever (Sonnet: $15/1M output vs $0.30/1M cached input = 50:1 ratio). Response brevity matters more than prompt trimming.
- Still need to build the full per-user per-month model to set tier pricing.

### Prompt Architecture (solid understanding)
- System prompt structure: `<persona>` → `<cat_profiles>` → `<memory>` → `<rules>` → `<examples>`. Static at top, dynamic at bottom.
- Voice-critical content (persona, tone, anti-sycophancy, few-shot examples) must be written by me. Functional content (tool guidelines, disclaimers, boundaries) can be LLM-drafted.
- Prompt quality is tested through conversation, not static review. Write → test → trace → revise → retest.
- Prompt text is Tier 4 (changeable in seconds via Firestore). Prompt structure/architecture is Tier 1.

### Haiku Removal (solid understanding)
- Warm Sonnet cheaper than cold Haiku. Cross-model caching doesn't exist. Eliminating Haiku saves weeks of dev time.
- Baby Claws replaced by contextual scope injection (same Sonnet, scoped instructions appended to user message).
- Budget throttling → no brevity injection (April 7). At 90% usage, app shows banner with upgrade link. At 100%: hard cap, send→upgrade button, all data still accessible.
- Background jobs (summarization, Tier 2 extraction, symptom/behavior key normalization, memory maintenance, safety pre-screen) default to Sonnet. Same model_id architecture — switchable to Haiku later as a cost lever if needed. Photo captioning still TBD (Haiku may be better fit for pure vision extraction).

### Notifications (solid understanding)
- Reactive only. Claws never initiates notifications — user always opts in during conversation ("want me to remind you?").
- No separate notification tool. Added a `notify` field to the existing `set_reminder` tool.
- Notification permission status is part of the iOS app payload to the Cloud Function. Claws knows whether the user has notifications enabled.
- Notification behavior controlled via CP3 flag (not tools or persona), avoiding instruction conflicts.
- Two Cloud Functions: one for prompt assembly (existing), one scheduled function that checks for due reminders and fires FCM notifications.
- All reminder timestamps stored in UTC, converted from user's local timezone by the Cloud Function.
- Every push notification also exists as an in-app reminder — the notify field just controls whether it also buzzes the phone via FCM.

### OB Claws (solid understanding)
- OB Claws is the same Claws with a background objective, not a separate experience. Same personality, just with added instructions to proactively gather core profile data and context that this is a brand new user.
- Still a CP2 variant (for caching), but written to feel identical to Normal Claws personality-wise.
- Users cannot edit data manually, only through Claws conversation (including Baby Claws on profile pages).
- Profile reveal happens during OB when minimum data set is complete, but doesn't trigger the OB → Normal transition.
- **Transition is Claws's decision, not the Cloud Function's.** OB Claws prompt includes criteria: (a) all core information collected, (b) at least 4 sessions logged, (c) user feels comfortable. When Claws judges the criteria are met, it calls `complete_onboarding` tool which sets `ob_complete: true` on the user's Firestore document.
- **Hard cap fallback:** If session count ≥ 12, Cloud Function sets `ob_complete: true` automatically regardless of whether Claws called the tool. Safety net, not the decision logic.
- **Two fields on user's Firestore document:** `session_count` (integer, incremented by Cloud Function each session) and `ob_complete` (boolean, set by Claws tool call or hard cap). Cloud Function checks `ob_complete` — if true, load Normal CP2; if false, load OB CP2.
- Session count surfaced to Claws in CP3 so it can evaluate the "at least 4" criterion. Claws can see the count but the Cloud Function doesn't make the transition decision (except at the hard cap).
- What defines a "session" is a Tier 4 variable — build the counter architecture, leave the trigger configurable.
- New cats added after OB ends don't retrigger OB. Normal Claws handles it naturally with a Tier 2/3 note like "Tiger was just adopted, still adjusting."

### UI → Cloud Function Relationship (early understanding)
- UI sends minimal payload: auth token, conversation_id, message, ui_context, timezone, notification permission status. No cat_id at top level — Claws determines which cat from conversation context. Cat ID only appears inside `ui_context` when on a cat-specific profile page.
- Cloud Function does all heavy lifting: auth verification, budget check, persona selection (including OB session counter check), context assembly, cache breakpoint placement, API call, SSE relay, Firestore writes.
- Scope injection: UI reports which page/section the user is on, Cloud Function translates to prompt content. Clean separation.
- Complexity spectrum: main chat (trivial) → scope injection (simple) → real-time profile population (complex) → floating bubble (complex, possibly defer).

### Emotional Attachment & Model Quality (solid understanding)
- In a raw chat interface (claude.ai), emotional depth IS the product. Attachment forms around model quality because that's all there is.
- In a companion app with persistent memory, attachment forms differently: ~40% knowing the user's cats, ~25% consistent personality, ~20% proactive behaviors (follow-ups, pattern recognition), ~15% raw emotional depth of responses.
- The first three are architecture, not model quality. Sonnet handles them identically to Opus.
- "Claws remembers Luna's birthday" creates more emotional attachment for a normal user than "Claws used unusually nuanced sentence structure."
- Opus is meaningfully better for sustained emotional conversations (grief over multiple sessions), creative cross-cat reasoning, and varied language over months of daily use. Gap is real but concentrated in ~10-15% of interactions.
- Decision: launch Sonnet, build model routing into architecture from day one (one variable, one conditional), deploy Opus selectively later if churn data shows the quality gap matters for retention.

### Cloud Functions Reliability (hard-earned lesson)

My previous app (Origamies) had Cloud Functions that failed consistently for 6 weeks straight, including through the App Store launch. I will not repeat this. This means:

- **Cloud Function logic must be kept as simple as possible.** Every layer of complexity is another thing that can silently break in production. If there's a simpler way to do something, default to that — but if the complex way is genuinely, considerably better, I'm open to being convinced.
- **Every Cloud Function must be explained to me in plain terms.** Not just what it does, but what happens if it fails. "This function does X. If it fails, the user sees Y. The data is in state Z. Here's how it recovers." If you can't explain the failure mode simply, the function is too complex.
- **Reliability is not a feature — it's the product.** A user who sends a message and gets no response, or gets an error, or loses data, will not come back. Every engineering decision should be evaluated as "what's the blast radius if this breaks?"
- **Fewer functions is the default, not the rule.** Each separate Cloud Function is a separate thing that can fail, timeout, cold-start, or have a permissions issue. Start from "can one function do this?" and only split when there's a clear, explainable reason to.

### Reliability (need to understand)
- Stateless HTTP request/response is inherently reliable. No persistent connections to manage.
- Key principle: operations should fully succeed or fully fail, no in-between state.
- Ordering matters: stream to user first → write to Firestore second → adjust budget last. If anything fails after user saw the response, reconcile later.
- Need to go deep on every failure mode before building.

---

## Decision Tiers (How We Prioritize)

We use a 5-tier system based on cost of changing your mind:

- **Tier 0 — Constraints.** Can never change. Getting these wrong means starting over.
- **Tier 1 — Structural decisions.** Load-bearing walls. Getting these wrong means rebuilding major systems.
- **Tier 2 — Systems that are hard to change.** Weeks of work to modify, ripples into other things.
- **Tier 3 — Tunable systems.** Architecture is in place, adjusting knobs. Days of work.
- **Tier 4 — Polish and config.** Pricing numbers, copy, small UI tweaks. Hours. Expect these to change.

---

## Tier 0 — Constraints

These are decided and confirmed by research. Not revisiting.

- [x] iOS only, Swift/SwiftUI
- [x] Firebase (Firestore, Cloud Functions v2, Storage, Auth)
- [x] Anthropic Claude API — **Sonnet only** (Haiku removed as runtime model, see "Key Change" below)
- [x] iOS 18 minimum (required by TextRenderer for streaming text fade-in)
- [x] Direct HTTP streaming (SSE) — Cloud Function as SSE proxy
- [x] No offline support (cat pun screen when offline, no Firestore offline persistence)
- [x] Sign in with Apple + Google before onboarding
- [x] Deploy to us-east1 (closer to Anthropic's AWS infrastructure)
- [x] RevenueCat for subscriptions
- [x] 19 tools with strict: true, all always-loaded (~3,500-4,000 tokens, cached after first call). April 6 additions: `notify` field on `set_reminder` (schema changed to `due_datetime`), `get_reminders` (read tool), `complete_onboarding` (OB transition), `complete_reminder` (mark reminder done). April 7: `log_habit`/`get_habits` replaced by `log_routine`/`get_routines`/`log_challenge`/`get_challenges` (habits split into routines and challenges — see Plumbing_Blueprint_Ongoing.md Section 1.1). Still well under the threshold where deferred loading matters.
- [x] Prompt caching mandatory from day one
- [x] 4 cache breakpoints max per request (Anthropic hard limit). Allocated: CP1 (tools), CP2 (persona), CP3 (per-user content), CP4 (conversation history, auto-cached). Zero spare.
- [ ] Safety pre-screen on every user message — now defaults to Sonnet with all other background jobs. Same model_id architecture, switchable to Haiku as a cost lever if needed.

### Key Change: Haiku Removed

Previously the architecture had 2 models: Sonnet (main chat) and Haiku (Baby Claws, profile editing, cheapest tier, summarization). Research showed:

- A warm Sonnet message ($0.0063) is cheaper than cold-starting Haiku ($0.0071)
- Cross-model caching doesn't exist — maintaining two models means two separate cache pools
- Haiku needed its own system prompt, prompt engineering, few-shot examples, test suites, and quirk workarounds (37% anti-sycophancy overtriggering)
- Eliminating Haiku saves weeks of development time

**What replaces it:**
- Profile editing → **Contextual scope injection.** Same Sonnet, same cache, same conversation. Scoped instruction appended to user message ("User is viewing Luna's weight graph. Keep responses under 2 sentences.").
- Budget throttling → **No brevity injection (April 7).** At 90% usage, app shows a banner with usage percentage and upgrade link — no prompt changes, user decides how to manage remaining budget. At 100%: hard cap, send button becomes upgrade button, all data still accessible, no chat.
- Adding Haiku back later is a Tier 2 decision if ever needed. Firestore structure unchanged, tools identical, conversation format same.

---

## Tier 1 — Structural Decisions

### Locked

- [x] **Cache architecture (highest-priority Tier 1 item).** 4 cached layers, each separated by a breakpoint:
  - **Cache 1 (CP1):** Tool schemas (~3,500 tokens) — globally shared across all users, always warm at 10K+ MAU
  - **Cache 2 (CP2):** Persona variant — globally shared per variant, swapped not conditional. V1 variants: Normal Claws, Grief Claws, OB Claws. Keep as brief as possible while including all relevant information; no hard upper limit from Anthropic, but shorter = better cache economics and less attention drift.
  - **Cache 3 (CP3):** Per-user content (cat profiles, memory tiers, contextual state, flags) — per-user, cache write on every new session
  - **Cache 4 (CP4):** Conversation history — automatic progressive caching, advances forward each turn
  - **Key principle: CP2 = who Claws IS (personality, tone, rules). CP3 = what Claws KNOWS about this user (cat data, memory, contextual modes like diet coaching).** New persona modes (grief, OB, future variants) are CP2 swaps. New per-user context (diet plans, life stage info, scope injection, flags) goes in CP3. This means adding contextual features never consumes a breakpoint.
  - Persona contamination research confirms separate prompts over conditional blocks for high-stakes modes (grief). Low-stakes contextual info (diet, scope injection, flags) is safe in CP3.
- [x] Up to 3 cats: all profiles always loaded in system prompt (~675 tokens max)
- [x] **CP3 flags system (April 5, revised April 6).** Contextual instruction blocks injected into CP3 based on cat data or UI state. Additive — can stack multiple flags simultaneously. Triggered by: (a) data-derived (age < 12mo → kitten mode, age > 11yr → geriatric mode, spayed_neutered false + age ~4-5mo → neutering flag), (b) conversation-set (Claws activates via tool call — diet, leash training, car exposure). Flag text stored in Firestore, fetched by Cloud Function at prompt assembly. Cloud Function only does simple, binary, unambiguous checks. **Household state removed as a flag trigger (April 6)** — multi-cat dynamics are contextual, not binary. Cat profiles in CP3 already contain adoption dates and cat count; Claws can infer multi-cat situations on its own. Adding a Cloud Function-derived flag risks injecting incorrect context that conflicts with what Claws can see in the data.
- [x] **Flags vs scope injection — same concept (April 5).** Both are conditional CP3 content. Flags trigger from cat data (kitten, diet). Scope injection triggers from UI navigation state (viewing weight graph, editing food library). Not worth treating as separate systems.
- [x] **Care standards block in CP3 (April 5).** Small reference list of pinned facts Claws should be consistent about (socialization window 2-14 weeks, spay/neuter 4-6 months, free feeding transition 6-8 months, etc.). Lives in CP3 because it's factual reference, not personality. Prevents Sonnet from freelancing on facts that should be consistent across every conversation. Exact list TBD.
- [x] **CP2 token budget (April 5).** No hard upper limit from Anthropic — checked official docs and 3 external sources, none publish a ceiling. Keep as brief as possible while including all relevant information. Test empirically. Attention drift and cache economics are the practical constraints, not a token number.
- [x] **Sonnet for all background jobs (April 5).** Summarization, Tier 2 extraction, symptom/behavior key normalization, memory maintenance, safety pre-screen all default to Sonnet. Eliminates Haiku-specific prompts, testing, and quirk management. Same model_id architecture — switchable to Haiku as a cost lever if needed. Cost difference is ~3x (not 5x — that's Opus vs Sonnet), well within acceptable range relative to revenue. Sonnet makes better judgment calls on extraction quality, which directly impacts whether Claws "remembers well."
- [x] **Don't over-flag (April 5).** Test Sonnet's baseline knowledge first on geriatric cats, leash training, diet planning, etc. Only write flags where Sonnet's default behavior falls short of what Claws should deliver. Flags are for tone/pushiness and brand-specific behavior, not for teaching Sonnet things it already knows. Care standards block handles factual consistency.

- [x] **Notifications — DECIDED (April 6).** Reactive only — Claws never initiates, user always opts in during conversation. No separate tool — `notify` field added to existing `set_reminder` tool. Notification permission status is part of iOS payload to Cloud Function. Behavior controlled via CP3 flag. Two Cloud Functions: prompt assembly (existing) + scheduled function for firing FCM notifications. All timestamps UTC. Every push notification also exists as an in-app reminder.
- [x] **OB Claws → Claws transition — DECIDED (April 6, revised later same session).** Transition is Claws's decision, not the Cloud Function's. OB Claws prompt includes criteria (core info collected, ≥4 sessions, user comfortable) and calls `complete_onboarding` tool when ready. Hard cap at session 12 — Cloud Function auto-sets `ob_complete: true` as safety net. Two Firestore fields on user document: `session_count` (integer) and `ob_complete` (boolean). Session count surfaced in CP3 so Claws can evaluate criteria. Cloud Function just checks `ob_complete` for CP2 selection. Session definition is Tier 4 — build counter, leave trigger configurable. **April 8 addition:** `ob_complete` must be explicitly initialized to `false` on account creation. CF also treats a missing field as `false` as a safety net. Same principle for all boolean fields on the user document.
- [x] **Users cannot edit data manually (April 6).** All data edits go through conversation — either main Claws or Baby Claws on profile pages. No direct edit buttons.
- [x] **Flags architecture is not load-bearing (April 6).** Adding new flags later is routine Tier 3/4 work. No need to anticipate every flag at launch.

### Need Discussion
- [ ] **4-tier memory system** — Tier 1: Core Identity, Tier 2: Reinforced Memory, Tier 3: Episodic Summaries, Tier 4: Structured Data via read tools. General structure proposed by research but needs discussion to lock.
- [ ] **Tier 2 memory cap** — Research suggested 30, April 4 session raised to 50. Gate at extraction layer vs storage. Needs decision.
- [ ] **Household-level memory block** — 10 cross-cat facts like "owner works night shifts." New finding from memory research. Needs confirmation.
- [ ] **4-10 cats roster system (Crazy Cat Lady)** — 3 most recent fully loaded + compact roster for rest. May be more engineering trouble than it's worth. Needs discussion on whether to simplify or cut.
- [x] **Floating Claws bubble (ZStack above NavigationStack)** — Keep it. Purely UI, no architectural impact. Easy to cut if it's buggy or takes too long to polish — just delete the view and add a back button. Worth attempting because it makes the app feel premium.
- [ ] **Usage-based free trial (~$0.50)** — Server-side Firestore tracking, not Apple's time-based trial. Is this really Tier 1? Placed here for now, needs discussion.
- [ ] **Realistic prompt sizes and cache hit rates** — Need to settle on actual token counts for system prompt, tool schemas, memory tiers, and expected cache hit percentages to normalize all cost projections.
- [x] **AI contexts — DECIDED.** 3 CP2 variants: Normal Claws, Grief Claws, OB Claws. Baby Claws is NOT a separate AI context — it's same Sonnet with scope injection in CP3.
- [x] **Baby Claws as a brand / UX concept — DECIDED.** Removed as a subscription tier name. Kept as a UX concept for data editing on profile pages (scope injection produces curt responses, which fits the "baby" brand). Makes main Claws feel richer by contrast.
- [ ] **Grief mode design** — Research provides framework (tone shifts, "never say" list, memorial page, resources). Needs founder's creative decisions. Should be V1, not deferred. **April 5 decision:** V1 approach is simple — if ANY cat in household has health_status `passed` or `terminal`, load grief CP2 for the whole household. Hybrid grief+normal (grieving one cat while being playful about another) deferred to post-launch. **April 7 addition:** User can opt out of grief mode via a toggle in settings (`grief_mode_override` field on user document). Toggle only visible when a terminal/passed cat exists. When overridden, Normal/OB Claws loads instead — no CP3 note needed, cat profiles already show health_status and Sonnet handles it contextually. Add a one-line CP3 note only if testing shows Sonnet is too cavalier (Tier 4 change). App shows a grief mode banner in chat when active (links to settings toggle). Prompt content still TBD.
- [ ] **Claws voice / system prompt** — The single most important thing in the app. Research provides structure (XML tags, few-shot examples, persona reinforcement every 6 turns). Actual voice needs to come from founder.
- [ ] **OB Claws question flow** — Exact sequence of onboarding questions, how pushy about photos, how habits are introduced. Transition is locked (session counter, 4 sessions) but the actual question sequence still needs design.
- [ ] **Paywall communication** — "Same AI, more messages" needs clear communication.
- [ ] **Baby Claws prompt scope** — Restricted to current page, or contextually primed but unrestricted? Founder's instinct: contextually primed. Needs testing.
- [ ] **Only 2 RevenueCat entitlements (claws_standard, claws_plus)?** — Middle tiers differ only in Firestore budget. Needs confirmation.

---

## Tier 2 — Systems (Hard to Change)

These are confirmed as the right approach but details need implementation.

- [ ] **Subscription tiers** — Baby Claws removed as a tier name (April 5). Remaining tiers and pricing TBD. Tier count and structure are harder to change than the actual numbers.
- [ ] **Subscription tier pricing** — April 4 session explored lower numbers. Lock or keep testing?
- [ ] **Prompt caching implementation** — BP architecture locked in Tier 1 (3 explicit + auto). Implementation details: Swift sorted keys bug (SR-13414) — pass raw JSON bytes. Treat tool_use.input as opaque JSON, never decode to Dictionary. Monitor cache_read_input_tokens on every response — zero = production incident.
- [x] **Cache heartbeat warming — DECIDED (April 6).** Not needed. Instead of engineering warm caches, the generous budget cap (~15% above breakeven) absorbs the occasional cold-start cache write cost. Eliminates a scheduled Cloud Function, its failure modes, and $50-150/month in cost. At high MAU caches are naturally warm anyway.
- [x] **Budget tracking — DECIDED (April 6).** Simple check-then-charge, no reserve-then-adjust pattern. Cloud Function checks if user is over their monthly cap before calling Anthropic — if yes, reject (app shows paywall). If no, proceed. After response completes, calculate actual cost from Anthropic's usage response and add to `used_monthly_allowance_cents` in Firestore. One read, one write. If the Cloud Function crashes mid-request, the user got a free message — acceptable. Key decisions: (1) Track dollar cost, not tokens or messages — dollar cost is the only fair unit given wildly different token type costs. (2) Model-agnostic cost calculation — pricing rates stored in a config table keyed by model ID, Cloud Function looks up rates for whatever model was used, no hardcoded prices. (3) Generous cap instead of engineered precision — caps set ~15% above breakeven per tier to absorb variance, adjusted based on real user data after launch. (4) 90% usage warning — Cloud Function includes `used_monthly_allowance_percent` in final SSE event, app reads budget fields on launch, banner appears at 90%+ with upgrade link, no prompt changes or brevity injection. (5) 100% hard cap — app disables all chat, send button becomes upgrade button, user can still type but can't send, all existing data remains accessible via direct Firestore reads, no punitive lockout. (6) Three Firestore fields + one computed: `monthly_allowance_cents` (integer, set by RevenueCat webhook), `used_monthly_allowance_cents` (integer, updated by CF after each response), `budget_reset_date` (timestamp), `used_monthly_allowance_percent` (computed from the first two, sent via SSE and readable on launch).
- [x] **Message length validation + account locking — DECIDED (April 6).** Two layers: iOS app disables send at 4,000 characters (UX guardrail), Cloud Function rejects at 6,000 characters (security tripwire). The 2,000-character gap eliminates edge cases — anyone hitting the CF limit bypassed the app and is not a normal user. On first offense, Cloud Function sets `account_locked: true`, `locked_reason: "character_limit_bypass"`, and `locked_at` on the user's Firestore document. Every future request checks `account_locked` first — if true, reject immediately. The app has a single locked-account screen (no navigation, no app access) with a link to the support website. Not a separate Cloud Function — it's two `if` statements at the top of the main chat function, before budget or Anthropic calls. Zero added latency (account_locked is read from a document already being fetched; message length is a string check in memory).
- [ ] **Circuit breaker** — Sonnet → canned responses (no longer Sonnet → Haiku → canned). Needs shared storage at scale.
- [ ] **Conversation history management** — Subcollection model. UI history separate from AI context. Server-side compaction API (beta) for Sonnet. Suggest new conversation at turn 15-20, never force.
- [ ] **Account deletion flow** — 7-step order of operations. Do NOT delete knownIdentities record (prevents free trial re-abuse).
- [ ] **System prompt versioning** — Firestore for prompt text, Remote Config for routing. Version-pin conversations. Rollback in <10 seconds.
- [ ] **Auth token verification** — Manual verification via Express middleware (since we chose HTTP streaming over callable functions).
- [ ] **Firestore data optimization** — Switch profile page to one-time reads. Build cached context document (1 read per API call instead of 18). Disable offline persistence.
- [ ] **Image pipeline** — Phone to Firebase Storage to Claude via URL. Photo captioning defaults to Sonnet with other background jobs, switchable to Haiku via model_id. EXIF stripping via UIImage. Send URLs not base64.
- [ ] **Tier 2 memory maintenance** — Monthly Sonnet maintenance job reviews for staleness. Switchable to Haiku via model_id if needed as cost lever.
- [ ] **Notification delivery chain (April 6)** — Full chain: user opts in during conversation → Claws fires `set_reminder` with `notify: true` → Firestore write → scheduled Cloud Function checks for due reminders → fires FCM notification → in-app reminder always exists regardless of notify field. Needs implementation.
- [ ] **OB transition implementation (April 6, revised)** — Two Firestore fields: `session_count` (integer, Cloud Function increments) and `ob_complete` (boolean, set by `complete_onboarding` tool call OR hard cap at session ≥12). Cloud Function checks `ob_complete` for CP2 selection. Session count surfaced in CP3 for Claws to evaluate. `complete_onboarding` tool needs to be added. Tool count now 19 (after April 7 habit schema redesign — see Plumbing_Blueprint_Ongoing.md Section 1.1). Session trigger definition is Tier 4.
- [ ] **Tier 3 episodic summaries** — Last 3 session summaries, ~100-200 tokens each. Biggest memory system cost at scale (~$12K/month at 100K MAU). Changes every session, invalidates system prompt cache.

---

## Research Completed (20 Documents)

All research lives in `/Users/brau/Desktop/Claws-Ai/research/`. Every doc was written from primary sources. Each follows: Summary, What I Learned, Best Practices, Risks, Recommendations, Sources.

| # | File | Key Finding |
|---|---|---|
| 02 | prompt-caching.md | Swift sorted keys bug (SR-13414) silently kills cache. Pass raw JSON bytes. |
| 03 | prompt-engineering.md | Persona drift 30%+ after 8-12 turns. Inject reminders every 6 turns. |
| 04 | api-at-scale.md | Sonnet/Haiku have separate rate limit pools. Pre-fund $400 for Tier 4 immediately. |
| 05 | streaming-responses.md | Firestore listener = cost trap. Use direct HTTP streaming. TextRenderer (iOS 18+) for fade-in. |
| 06 | conversation-history.md | Server-side compaction API (beta). Context rot at 10-15 turns. Tool call bloat is real. |
| 07 | cold-start-latency.md | minInstances:1 = $3.24/month. Warm path 1.5-2.1s. Deploy to us-east1. |
| 08 | error-handling.md | No stream resumption. Circuit breaker needs shared storage. Every write tool must be idempotent. |
| 09 | multi-cat-context.md | 3 cats always loaded = negligible cost. No cache invalidation on cat switch. |
| 10 | image-handling.md | Photo captioning = $3,600/month at 100K MAU, not $200. EXIF stripping free via UIImage. |
| 11 | system-prompt-versioning.md | Firestore for prompts, Remote Config for routing. Rollback <10s. |
| 12 | security-prompt-injection.md | Haiku pre-screen + XML trust boundaries + rate limiting = sufficient. |
| 13 | memory-architecture.md | 4-tier system validated. Cap Tier 2 at 50 facts/cat. Add household memory (10 facts). |
| 14 | apple-app-store-guidelines.md | Must name Anthropic in consent. Report button required. Account deletion required. |
| 15 | firestore-cost-gotchas.md | No-offline worsens listener reconnection. Switch to one-time reads. Cached context doc. |
| 16 | behavioral-design-grief.md | Grief mode lasts forever. Memorial page V1. "Would a vet's office send this?" notification test. |
| 17 | testing-ai-quality.md | 60 test cases for V1. Start monitoring at 10-15%. Promptfoo. Don't use Claude to judge Claude. |
| 18 | swiftui-chat-ui.md | iOS 18 minimum. WWDC24 session 10151. Floating bubble in ZStack. |
| 19 | data-privacy.md | Anthropic DPA is automatic. Pet health data not regulated. 8 pre-launch items. |
| 20 | subscription-management.md | Usage-based trial is server-side only. Only 2 entitlements needed. 16-day billing grace. |
| 21 | authentication-identity.md | No anonymous auth. Apple sub survives deletion. DeviceCheck survives factory resets. |

Also kept from old research:
- `01-ai-companion-landscape.md` — market research, competitor analysis
- `03-Updated-Tool-Calling.md` — proved the audit approach works

---

## Prompt Engineering Execution Strategy

The system prompt contains two fundamentally different types of content that require different quality control approaches.

**Voice-critical content** — the persona block, tone guidelines, anti-sycophancy calibration, and few-shot examples — defines how Claws feels to the user and is the primary differentiator of the product. **Functional content** — tool use guidelines, health disclaimers, topic boundaries, food rating definitions, summary templates — defines how Claws operates mechanically.

**Voice-critical content should be written manually by the founder, not drafted by an LLM.** An LLM will produce language that sounds like a generic AI assistant rather than the specific character being designed. Every word choice in these sections compounds across every response the model generates, so precision matters. This represents roughly 30% of the prompt by volume but carries disproportionate impact on user experience and retention.

**Functional content can be drafted by an LLM and reviewed once** for correctness and clarity. These sections need to be accurate, not inspired. Spending extensive time polishing tool use instructions or boundary definitions yields minimal return compared to the same time spent on voice work.

**The primary quality control method for prompts is conversational testing, not static review.** The execution loop is: write → test with realistic multi-turn conversations → identify where the tone or behavior feels wrong → trace the issue back to specific prompt language → revise → retest. Static review of prompt text in isolation has limited value because prompt behavior is emergent — it only becomes visible in the context of real interactions.

The prompt text itself is Tier 4 infrastructure — changeable in seconds via a Firestore write with instant rollback. This means the iterative testing loop is cheap and fast once the Tier 1 architecture (XML structure, cache ordering, injection points, persona variant design) is locked. Perfectionism should be applied liberally to voice-critical sections during the tuning phase, with the understanding that no amount of upfront drafting replaces empirical testing against real conversation patterns.

---

## Process Status

1. Product decisions first (done — Core_Features.md (renamed from Core_Features_User_Perspective.md), needs updating for Sonnet-only and Baby Claws tier removal)
2. Pass 1: Research all topics from primary sources (done — 20 docs)
3. **Pass 2: Lock Tier 0/1 decisions, cross-check for conflicts (in progress)**
   - April 5 session: Locked CP naming convention, flags architecture, care standards block, Sonnet for background jobs, Baby Claws as UX concept not tier, grief V1 approach, floating bubble, don't-over-flag principle. Drafted Normal Claws persona prompt and kitten mode flag as working examples.
   - April 6 session (claude.ai): Locked notifications (reactive only, notify field on set_reminder, FCM via scheduled Cloud Function), OB Claws transition (session counter, 4 sessions, same Claws with background objective), no manual data editing (all through conversation), flags not load-bearing (Tier 3/4 to add new ones).
   - April 6 session (Claude Code): Revised OB transition — Claws decides when to transition via `complete_onboarding` tool call (not Cloud Function counter logic). Hard cap at session 12 as safety net. Two Firestore fields: `session_count` + `ob_complete`. Tool count now 15.
4. Pass 3: Synthesize into master build plan (not started)

### TBDs from April 5-6 sessions
- [ ] Which flags make V1 (3-15 flags — test Sonnet baseline first, only flag where it falls short)
- [ ] Grief mode CP2 prompt content (founder's creative decision)
- [ ] OB Claws CP2 prompt content (same personality as Normal, just with onboarding objective)
- [ ] Care standards pinned facts list (the specific facts to pin in CP3)
- [ ] Core_Features.md update pass for Sonnet-only architecture and Baby Claws tier removal
- [x] ~~What defines a "session" for Tier 3 summarization triggers~~ — Architecture decided: build the counter, leave the trigger definition as a Tier 4 configurable variable. Deliberately deferred to testing.
- [ ] Safety pre-screen (Tier 0) — last unchecked Tier 0 item. Confirm it stays with Sonnet or explicitly flag as open.

---

## Key Cost Numbers (From April 4 Deep Dive)

| Metric | Value |
|---|---|
| First message of conversation | ~$0.055 (cache write) |
| Subsequent messages | ~$0.007 (cache read) |
| 12-message chain total | ~$0.13 ($0.011/msg average) |
| Profitability at 1K MAU | $9,490 revenue, $1,714 API cost, 81.9% margin |
| Corrected API cost at 100K MAU | ~$16,987/month (48% higher than original estimate) |
| Net margin at scale | ~70% (still healthy) |
