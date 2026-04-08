# Purr.ai — Plumbing Blueprint (Ongoing)

**Last updated:** 2026-04-07
**Purpose:** Master reference for all plumbing — data structures, Cloud Function logic, UI views, security, integrations, and the contracts between them. This document should contain everything needed to build the entire system from the ground up.

---

## 1. Tool Calls

Reference: `Tool_Use.md` in this folder for full schemas and definitions.

Current tool count: 19 (as of April 7). **NOTE: `log_habit` and `get_habits` have been replaced by 4 new tools (see below). Tool_Use.md needs to be updated to reflect this change and verified against the latest decisions in Research_Status.md.**

### 1.1 Habit Schema Redesign (April 7)

`log_habit` and `get_habits` are replaced by `log_routine`, `get_routines`, `log_challenge`, and `get_challenges`. Habits split into two concepts:

**Routines** — ongoing, no end date. General cat maintenance (brushing, walking, litter box, vet visits, treats, etc.)
- Display: "X times in the last 7 days" + 7 dots (daily/weekly) or "Last logged: March 23rd — 15 days ago" (monthly)
- No streaks in V1

**Challenges** — time-bound, has a start and end date. Socialization goals, diet plans, training programs, medication courses, etc.
- Display: "83 days left — 8 times in 40 days" (counts from start date, not rolling 7 days)
- Archives when end date passes

**`log_routine`**

| Field | Type | Required | Notes |
|---|---|---|---|
| `cat_id` | string | yes | Which cat |
| `routine_name` | string | yes | User-defined name |
| `period` | enum | yes | `daily`, `weekly`, `monthly` |
| `completed_date` | string | yes | ISO 8601. Empty string if just creating, not logging a completion. |
| `actual_value` | number, nullable | no | What happened — 50 (minutes). Null = simple checkbox. |
| `target_value` | number, nullable | no | The goal — 45 (minutes). Null = no target. |
| `unit` | string | yes | `minutes`, `times`, `ml`, etc. Empty string = no unit. |
| `notes` | string | yes | Empty string = no notes. |

**`log_challenge`**

| Field | Type | Required | Notes |
|---|---|---|---|
| `cat_id` | string | yes | Which cat |
| `challenge_name` | string | yes | User-defined name |
| `period` | enum, nullable | no | `daily`, `weekly`, `monthly`. Null = no recurring rhythm, just a deadline. |
| `start_date` | string | yes | ISO 8601. When the challenge begins. |
| `end_date` | string | yes | ISO 8601. When the challenge ends. Claws calculates from context (e.g. "when Yeye turns 6 months"). |
| `completed_date` | string | yes | ISO 8601. Empty string if just creating. |
| `actual_value` | number, nullable | no | What happened. Null = simple checkbox. |
| `target_value` | number, nullable | no | The goal. Null = no target. |
| `unit` | string | yes | `minutes`, `times`, `ml`, etc. Empty string = no unit. |
| `notes` | string | yes | Empty string = no notes. |

**Read tools:** `get_routines` and `get_challenges` replace `get_habits`. Schemas TBD but follow the same pattern as existing read tools (cat_id + filters).

**Prompt guidance needed:** When a user asks to track something recurring, Claws should clarify whether they want a routine (ongoing tracking), a reminder (one-time nudge), or both. Voice-critical — founder writes this.

**Dieting:** Not a routine or challenge. Diet rules live in Claws's memory (Tier 2). Daily intake is tracked via `log_food` (food preference library). Weight progress tracked via `log_weight`. No separate diet tool.

---

## 2. Cloud Function Request/Response Lifecycle — End-to-End

*(Full lifecycle finalized April 7. 15 steps, one Cloud Function, one HTTP connection.)*

### 2.0 Scope Injection Plumbing Note (April 7)

When `ui_context` includes a `cat_id`, the Cloud Function must resolve it to the cat's natural language name (and include both the name and the Firestore ID) when writing the CP3 scope injection. E.g. "User is viewing Luna's (cat_abc123) weight graph." The Cloud Function already has all cat profiles loaded from Step 4 — no extra read needed, just a lookup. Which format Claws responds best to (name only, ID only, or both) is Tier 4 — test and adjust. The plumbing must support both from day one.

Additionally: every UI view that has a chat input (main chat + Baby Claws on each profile section) must know its own identity and report it in `ui_context`. The Cloud Function must have a mapping from each `ui_context` value to the corresponding CP3 injection text. If either side is missing, scope injection silently fails — Baby Claws behaves like main Claws. Build both sides together.

### 2.1 Dictionary

*(Variable and field naming reference. Every variable used across the Cloud Function, SSE events, and Firestore documents should be listed here with its type, who writes it, who reads it, and what it means. This will eventually become a helper file during development.)*

| Variable | Type | Who writes it | Who reads it | What it means |
|---|---|---|---|---|
| `conversation_id` | String (UUID) | iOS app (generated locally when user starts a new conversation via `UUID().uuidString`) | Cloud Function (Step 8 — reads conversation history; Step 15a — writes conversation turn), Firebase Storage (image upload path) | Unique identifier for a conversation. Created by the app, never by the CF. Used as the Firestore document path: `users/{uid}/conversations/{conversation_id}/messages/{message_id}`. Also used in the Firebase Storage image upload path: `users/{uid}/chat_images/{conversation_id}/{uuid}.jpg`. Security is in the Firestore path (uid from verified auth token), not the ID format. App reuses the same ID for all messages in a conversation. |
| `message` | Array of content blocks: `[{ type, text?, url? }]` | iOS app (included in every request payload) | Cloud Function (Step 3 — text extraction for length check; Step 11 — translated into Anthropic content blocks) | The user's message. Always an array, even for text-only messages. Each block is either `{ type: "text", text: "..." }` or `{ type: "image", url: "https://firebasestorage..." }`. Image blocks contain Firebase Storage download URLs — the image was uploaded by the app before sending. CF translates to Anthropic's format in Step 11. Stored as-is in conversation history (Step 15a) for passthrough on future turns. |
| `ui_context` | Object: `{ page, section, cat_id }` | iOS app (included in every request payload) | Cloud Function (Step 10 — translates into CP3 scope injection) | Reports which screen the user is currently on. `page`: top-level view (e.g. "main_chat", "cat_profile", "settings"). `section`: subsection if applicable (e.g. "weight_graph", "food_journal", "habits"). `cat_id`: Firestore document ID of the cat being viewed, if on a cat-specific page. Cloud Function resolves `cat_id` to the cat's name using already-loaded profiles. Main chat sends `{ page: "main_chat" }` with no section or cat_id — no scope injection. |
| `payload_version` | Integer | iOS app (included in every request payload) | Cloud Function (Step 1 — determines how to parse the payload) | Starts at 1. Allows the CF to handle multiple app versions during transitions after an App Store update. If the payload shape changes in a future version, the CF checks this field to know which format to expect. |

### 2.2 Full Lifecycle

#### Step 1 — App sends request

**Before sending:** If the user attached a photo, the app has already uploaded it to Firebase Storage and received the download URL. The upload flow: user taps the photo button → PhotosPicker opens (no library permission needed) → app strips EXIF metadata, resizes to longest edge ≤ 1568px, compresses to JPEG at 0.75 quality → uploads to `users/{uid}/chat_images/{conversation_id}/{uuid}.jpg` → receives download URL. Send button stays disabled until upload completes. If upload fails, user sees an error and can retry or remove the image.

User hits send. The app opens an HTTP connection to the Cloud Function and sends the payload:

**Text-only message:**
```
{
  auth_token: "Firebase auth token",
  conversation_id: "conv_abc123",
  message: [
    { type: "text", text: "Luna threw up after breakfast" }
  ],
  ui_context: { page: "main_chat" },
  timezone: "America/Chicago",
  notification_permission: true,
  payload_version: 1
}
```

**Message with photo:**
```
{
  auth_token: "Firebase auth token",
  conversation_id: "conv_abc123",
  message: [
    { type: "image", url: "https://firebasestorage.googleapis.com/v0/b/claws-app.appspot.com/o/users%2Fuser123%2Fchat_images%2Fconv456%2Fimg789.jpg?alt=media&token=abc" },
    { type: "text", text: "Look at this rash on Luna's ear, should I be worried?" }
  ],
  ui_context: { page: "main_chat" },
  timezone: "America/Chicago",
  notification_permission: true,
  payload_version: 1
}
```

Seven fields. `message` is always an array of content blocks — text-only messages are just an array with one text block. Image blocks come before text blocks (Claude processes content in order). `payload_version` is included from day one so the CF can handle future payload shape changes gracefully during app update transitions (see research doc 25).

`conversation_id` is generated by the app — a UUID created locally when the user starts a new conversation (`UUID().uuidString` in Swift). The app stores it and reuses it for all messages in that conversation, including image upload paths. The CF never creates conversation IDs — it uses whatever the app sends. Security is in the Firestore path (`users/{uid}/conversations/{conversation_id}`), not the ID format. The uid comes from the verified auth token (Step 2), so a user can only create conversations under their own account. App Check (Firebase) deferred to post-V1 for verifying requests originate from the real app.

No cat_id at the top level — Claws figures out which cat from the conversation. If the user is on a profile page, ui_context includes section and cat_id for scope injection.

#### Step 2 — Auth token verification

Verify the Firebase auth token is valid and extract the user ID. Manual verification via Express middleware (required because we chose HTTP streaming over callable functions).

- Invalid → reject immediately, send `{ error: { code: "auth_failed" } }`, close connection.
- Valid → we now have the user's ID. Continue.

#### Step 3 — Message validation

Three checks, all instant, no Firestore reads:

**3a. Image count check.** Count image blocks in the `message` array. More than 7?
- Yes → set `account_locked: true`, `locked_reason: "image_limit_bypass"`, and `locked_at` on the user's Firestore document. Send `{ error: { code: "too_many_images" } }`. Reject immediately. Same logic as the character limit — the app caps at 3, so 7 is a generous gap for edge cases or bugs. Anyone sending 7+ images bypassed the app. First offense = lock. The CF-side cap (7) is a config variable (Tier 4).
- No → continue.

**3b. Image URL validation.** For each image block, check that the URL matches the expected Firebase Storage pattern and contains the authenticated user's uid in the path. Does the URL start with the Firebase Storage domain AND include `/users/{verified_uid}/` in the path?
- Any URL fails → reject, send `{ error: { code: "invalid_image_url" } }`. No account lock. This could theoretically trigger from a Storage URL format change (unlikely), so keep it a soft rejection, not a lock. Easy to remove entirely if it causes issues — it's one check that nothing else depends on.
- All pass → continue.

**3c. Text length check.** Extract the text content from the `message` array (skip image blocks) and check total character count. Over 6,000 characters?
- Yes → set `account_locked: true`, `locked_reason: "character_limit_bypass"`, and `locked_at` on the user's Firestore document. Send `{ error: { code: "message_too_long" } }`. Reject immediately. First offense = lock, because a real user can't get past the app's 4,000 character limit plus the 2,000 character gap. We can now write to their document because Step 2 gave us their user ID.
- No → continue.

Image URLs are not counted toward the character limit — only the user's typed text.

#### Step 4 — Read user document and cat profiles

Two parallel reads from Firestore:

1. **User document** — one read. Gets: `account_locked`, `used_monthly_allowance_cents`, `monthly_allowance_cents`, `subscription_tier`, `ob_complete`, `session_count`, `timezone`, `grief_mode_override`, and any other user-level fields.
2. **Cat profiles** — one read per cat (up to 3). Gets: name, breed, age, weight, `health_status`, `spayed_neutered`, indoor/outdoor, and all other profile fields.

These fire at the same time. Cat profiles are needed for both CP2 selection (grief check in Step 9) and CP3 assembly (Step 10), so we load them now rather than later. Total: 1-4 Firestore reads depending on number of cats.

The rest of CP3 data (memory tiers, reminders, routines, challenges, conversation history) is read later in Step 8, because we first need to pass the gate checks in Steps 5-6. No point reading all that data for a locked or over-budget user.

#### Step 5 — Account locked check

Is `account_locked` true on the user document from Step 4?

- Yes → reject immediately, send `{ error: { code: "account_locked" } }`. App shows the locked screen (see Section 8.2). Close connection.
- No → continue.

#### Step 6 — Budget check

Is `used_monthly_allowance_cents` >= `monthly_allowance_cents`?

- Yes → reject, send `{ error: { code: "budget_exceeded" } }`. App shows the paywall state (see Section 8.3). Close connection.
- No → continue.

#### Step 7 — Update timezone if changed

Compare the timezone from the request payload to the stored timezone on the user document. If different, update it. Handles travel. If unchanged, skip — no Firestore write needed.

#### Step 8 — Read all remaining context data

Now that we've passed all gate checks (auth, length, lock, budget), it's worth reading the rest. Fire these reads in parallel:

- Memory Tier 1 — core identity facts per cat
- Memory Tier 2 — reinforced memories per cat
- Tier 3 episodic summaries — last 3 session summaries
- Active reminders — pending reminders with due dates **within 3 days only** (Tier 4 variable — adjustable after testing). Reminders beyond this window are not loaded into CP3. Claws can still fetch all reminders on demand via the `get_reminders` tool. The 3-day window prevents Claws from mentioning far-future reminders unprompted, and prevents the double-remind problem (reminder sits in CP3 for weeks → Claws mentions it → conversation history prunes that mention → Claws sees the reminder again and mentions it as if new).
- Active routines and challenges — current habits
- Conversation history — all prior messages for this `conversation_id`

Cat profiles were already loaded in Step 4. How many Firestore reads this totals depends on the data model — if the cached context document is built (1 read instead of 18, see Research_Status Tier 2), this whole step could collapse into 2 reads (one context doc + one conversation history query). That's a Tier 2 implementation decision, but the lifecycle works either way.

#### Step 9 — Determine CP2 variant

Using the cat profiles from Step 4 and the user document fields:

Does any cat have `health_status`: `"passed"` or `"terminal"`?
- Yes → check `grief_mode_override` on user document:
  - Override is false (default) → load **Grief Claws** CP2.
  - Override is true (user opted out in settings) → load **Normal/OB Claws** as usual. No CP3 note needed — cat profiles already show health_status in CP3, and Sonnet with Normal Claws is contextually aware enough to be appropriate. If testing shows otherwise, a one-line CP3 note can be added (Tier 4 change).
- No → is `ob_complete` false?
  - Yes → load **OB Claws** CP2.
  - No → load **Normal Claws** CP2.

One CP2 active at a time. Swapped, not layered. The Cloud Function does simple binary checks — it doesn't interpret grief or evaluate readiness.

**Account creation default:** `ob_complete` must be explicitly initialized to `false` when the user's Firestore document is first created (during sign-up). This ensures Step 9 always has a definitive value to check. As a safety net, the CF should also treat a missing `ob_complete` field as `false` — belt and suspenders. Same principle applies to all boolean fields on the user document: initialize them explicitly on account creation, never rely on "field doesn't exist" meaning false.

#### Step 10 — Assemble CP3 (per-user content)

The Cloud Function builds the per-user section of the system prompt using the data from Steps 4 and 8. All of this goes into one block between CP2 (persona) and CP4 (conversation history).

Contents:

- **Cat profiles** — all cats (up to 3), from Step 4. Name, breed, age, weight, health status, spayed/neutered, indoor/outdoor, etc. ~225 tokens each.
- **Memory Tier 1** — core identity facts per cat. "Luna can open cabinets." "Mochi is afraid of the vacuum." Permanent, never pruned.
- **Memory Tier 2** — reinforced memories. Important observations that have come up multiple times or were flagged as significant. Capped per cat.
- **Tier 3 episodic summaries** — last 3 session summaries. "Last session: discussed Luna's vomiting, logged symptom, set vet reminder." ~100-200 tokens each.
- **Active reminders** — pending reminders due within 3 days (see Step 8 note) so Claws can mention them naturally. "[rem_abc] Luna flea medication — due Apr 10."
- **Active routines and challenges** — current habits so Claws has context. "Walk Yeye: weekly routine. Socialize Yeye: challenge, 43 days left."
- **Contextual flags** — triggered by cat data. Age < 12 months → kitten mode flag. Age > 11 years → geriatric flag. `spayed_neutered: false` + age ~4-5 months → neutering flag. Only fires where Sonnet's default behavior falls short.
- **Scope injection** — from `ui_context` in the payload. Cloud Function resolves `cat_id` to the cat's name using the already-loaded profiles. "User is viewing Luna's (cat_abc123) weight graph. Keep responses focused and concise." If main chat, no injection.
- **Care standards block** — pinned facts Claws should be consistent about. Socialization window 2-14 weeks, spay/neuter 4-6 months, etc.
- **Session count** — so OB Claws can evaluate transition criteria.
- **Notification permission** — from the payload. Tells Claws whether to offer push notifications.
- **User's current local time and timezone** — from the payload. "User's local time: 3:42 PM CDT, America/Chicago."

#### Step 11 — Assemble full prompt and call Anthropic

Put it all together:

- **CP1:** Tool schemas (19 tools, cached globally)
- **CP2:** Persona variant (from Step 9, cached globally per variant)
- **CP3:** Per-user content (from Step 10, cache write on new session)
- **CP4:** Conversation history (from Step 8, auto-cached progressively). Previous turns may include image content blocks (stored as URLs) — these are passed through as-is and get cached by Anthropic automatically.
- **The new user message** appended at the end. The CF translates the app's `message` array into Anthropic's content block format:
  - Text block: `{ type: "text", text: "..." }` — passed through directly.
  - Image block: `{ type: "image", source: { type: "url", url: "https://firebasestorage..." } }` — the app's `url` field is wrapped in Anthropic's source format. Images are positioned before text (Claude processes content in order).
  - Text-only messages: just one text content block, identical to before.

Call Anthropic API with `stream: true` and the appropriate `model_id`.

**Image token cost:** Each image costs `(width × height) / 750` input tokens, priced the same as text. A typical phone photo resized to 1568px = ~1,590 tokens (~$0.005 on Sonnet). Cached on subsequent turns at 90% discount. See research doc 25 for full cost modeling.

#### Step 12 — Stream text to the app

As `content_block_delta` events arrive from Anthropic with `text_delta`, the Cloud Function relays them as custom SSE events:

```
event: text
data: "Oh no, poor Luna!"
```

The app fades in each chunk. Tool call data (`input_json_delta`) is accumulated silently in memory — not sent to the app until the stream finishes and `stop_reason` is known.

#### Step 13 — Check stop reason

Anthropic's stream ends with a `stop_reason`:

- `"end_turn"` → go to Step 15 (wrap up).
- `"tool_use"` → increment `loopCount`, go to Step 14 (tool execution loop).

#### Step 14 — Tool execution loop

**14a.** Parse the tool call(s) from the accumulated data.

**14b.** Send a status event to the app:
```
event: status
data: {"tool": "log_symptom", "message": "Logging Luna's symptom...", "animation": "working"}
```

**14c.** Execute the tool(s) — write to Firestore. If multiple parallel tool calls, execute concurrently.

**14d.** Format tool result(s) using template functions (e.g. "Logged vomiting for Luna on Apr 7, 2026. First occurrence.").

**14e.** Check `loopCount`:
- **Under 5** → call Anthropic again with the full conversation so far plus the tool results appended. This is a full API call — same CP1/CP2/CP3/CP4 plus the growing conversation, so it hits cache on everything except the new tool results and whatever text Claws generated since the last cached turn. Resume streaming — go back to Step 12.
- **Equal to 5 (safety cap)** → go to 14f.

**14f. Safety cap reached.** Inject a wrap-up message after the final tool results: *"You have used the maximum number of tool calls for this message. Do not call any more tools. Respond to the user now."* This message is a hardcoded constant in the Cloud Function code — not in Firestore, not in the prompt, not voice-critical. One final API call. Claws sees its tool results, sees the instruction, and writes a natural closing response acknowledging what it did and didn't get to. Then go to Step 15. **This injected message is stripped before saving conversation history** — it's internal plumbing, not part of the conversation.

**14g. If Claws still returns `tool_use` after the cap message**, ignore the tool calls and proceed to Step 15 with whatever text was in the response.

##### Tool Loop Safety Layers

The `loopCount` variable is a simple local integer that increments each time through the loop. It's trivial code — hard to get wrong. But if it somehow fails, three backstops exist:

1. **Cloud Function timeout (primary backstop).** CF v2 has a configurable max execution time. Set to 60 seconds (Tier 4 — adjustable). If the function is still alive at 60 seconds, it gets killed regardless of loop state. The user sees an `anthropic_error`. One bad message, bounded cost.

2. **Worst-case cost within the timeout.** Each loop iteration takes ~4-6 seconds (Anthropic API call is the bottleneck). In 60 seconds: ~10-15 iterations max. Cost per iteration: ~$0.007 (mostly output tokens). **60-second runaway worst case: ~$0.10.** A 30-second timeout halves that. Not a financial risk — the timeout prevents the loop from running for minutes or hours, which is where real damage would happen.

3. **Logging every iteration.** Every time the CF enters Step 14, it logs: `tool_loop | user: {user_id} | conv: {conv_id} | iteration: {n} | tools: {tool_names}`. This enables:
   - Searching logs for any request that hit iteration 4 or 5 (should be rare)
   - Setting up a Cloud Monitoring alert if any request hits iteration 5
   - Spotting counter failures immediately (iteration numbers climbing past 5)

4. **Budget check on next message.** Won't save the current runaway message, but prevents a second one.

**Pressure testing:** During development, send absurdly tool-heavy messages (15+ potential tool calls) and watch the logs. Verify the cap fires at iteration 5, the wrap-up message produces a natural response, and Claws acknowledges what it did and didn't get to. Also test the timeout independently by temporarily setting the cap to 999 and confirming the CF dies at the timeout threshold.

##### Tool Loop Cost Implications

Each loop iteration is a full Anthropic API call. A message that triggers 3 sequential tool calls costs ~4x a pure text response (1 initial + 3 tool rounds). With the safety cap, the theoretical max is 6 API calls per user message (1 initial + 5 tool loops). This is accounted for in Step 15 budget calculation — all iterations are accumulated.

#### Step 15 — Wrap up

**Ordering principle: user sees the response → conversation is saved → budget is charged.** If the CF crashes at any point after streaming, the worst case is a free message — never a charge without a saved conversation.

**15a.** Save the complete conversation turn to Firestore — the user's message (including any image content blocks as URLs, not base64), the full assistant response (including any `tool_use` and `tool_result` blocks from the loop), all in one batch write. Never save incrementally during streaming. The 14f wrap-up message (if it was injected) is stripped — it's not part of the conversation. Image URLs are stored in Firestore in the same content block format used by Anthropic's API, so loading conversation history on future turns requires no transformation — just pass it through as CP4.

**15b.** Calculate actual cost from Anthropic's usage response (accumulated across all loop iterations). Look up the `model_id` pricing from the config table. Multiply tokens by rates.

**15c.** Update `used_monthly_allowance_cents` on the user's Firestore document.

**15d.** Calculate `used_monthly_allowance_percent`.

**15e.** Send budget event:
```
event: budget
data: {"used_monthly_allowance_percent": 48}
```

**15f.** Increment `session_count` if this is the first message after a session gap (definition TBD — Tier 4 variable).

**15g.** Send done event:
```
event: done
data: {}
```

**15h.** Close the connection.

---

## 3. Firestore Data Structure

*(All collections and subcollections, document shapes, who writes to each, who reads from each.)*

### 3.1 User Firestore Document

*(Every field on the user document, its type, who writes it, when it's read, and what breaks if it's wrong.)*

---

## 4. Security Rules

*(Firestore security rules — read/write permissions per collection, the principle that the app is read-only and only Cloud Functions write via admin SDK.)*

---

## 5. SSE Event Types

The Cloud Function translates Anthropic's complex SSE events into a simple set of custom events for the iOS app. The app doesn't see Anthropic's format — only these.

| Event | Data | When it fires | What the app does |
|---|---|---|---|
| `text` | String (the words) | Each chunk of Claws's response | Append to screen with fade-in animation |
| `status` | `tool`, `message`, `animation` | Cloud Function is executing a tool call | Show loading animation + friendly text (e.g. "Logging Luna's symptom..."). The `animation` field maps to a pre-built animation asset in the app — supports multiple animation types for different tools. Build the infrastructure to support different animations from day one, even if V1 only ships with one default animation. |
| `budget` | `used_monthly_allowance_percent` | After the full response completes | Update usage banner if above 90% |
| `error` | `{ code, message }` | Something went wrong | Handle by error code (see table below) |
| `done` | — | Response is fully complete | Stop listening, connection closes |

**Braulio has signed off on this structure (April 7).** Status event animation infrastructure confirmed as a priority.

### 5.1 Error Codes

| Code | Message | When | Recoverable? |
|---|---|---|---|
| `account_locked` | "Account locked" | Lifecycle Step 5 — `account_locked: true` on user document | No. Only support can unlock. |
| `budget_exceeded` | "Monthly budget reached" | Lifecycle Step 6 — `used_monthly_allowance_cents` >= `monthly_allowance_cents` | Yes. Resets on `budget_reset_date`, or user upgrades tier. |
| `auth_failed` | "Authentication failed" | Lifecycle Step 2 — invalid or expired Firebase auth token | Yes. App should re-authenticate and retry. |
| `too_many_images` | "Too many images" | Lifecycle Step 3a — more than 7 image blocks in message. App caps at 3; 7 is the CF tripwire. Triggers account lock. | No. Account is now locked. |
| `invalid_image_url` | "Invalid image" | Lifecycle Step 3b — image URL doesn't match expected Firebase Storage pattern or doesn't contain the user's uid. Soft rejection, no lock. | Yes. Retry with valid images. |
| `message_too_long` | "Message exceeds limit" | Lifecycle Step 3c — message over 6,000 characters. Also triggers account lock. | No. Account is now locked. |
| `safety_flagged` | TBD | TBD — safety pre-screen rejects the message | TBD (depends on safety pre-screen design) |
| `anthropic_error` | "Something went wrong, try again" | Anthropic API returns an error or times out | Yes. User can resend. |

---

## 6. Third-Party Integration Points

*(RevenueCat webhooks, FCM notification delivery, Anthropic API config and pricing table by model_id, any other external services.)*

---

## 7. Error Handling

*(What happens at each step when things fail, what the user sees, how the system recovers. Every Cloud Function must have its failure mode documented here.)*

### 7.1 Account Locked

**Trigger:** Cloud Function finds `account_locked: true` on the user's Firestore document (lifecycle Step 5), OR sets it during Step 3 (character limit bypass).

**What the CF sends:** `{ event: "error", data: { code: "account_locked", message: "Account locked" } }`. Connection closes immediately.

**What the app shows:** Locked account screen (see Section 8.2). No navigation, no access to any app features. Single link to the support website. This screen should check `account_locked` on app launch too — don't wait for a message attempt to discover it.

**Recovery:** Manual only. Support reviews the case and clears `account_locked` on the Firestore document. There is no in-app unlock flow.

### 7.2 Budget Exceeded

**Trigger:** Cloud Function finds `used_monthly_allowance_cents` >= `monthly_allowance_cents` on the user's Firestore document (lifecycle Step 6).

**What the CF sends:** `{ event: "error", data: { code: "budget_exceeded", message: "Monthly budget reached" } }`. Connection closes immediately.

**What the app shows:** Budget exceeded state (see Section 8.3). All existing data remains accessible (cat profiles, conversation history, reminders, routines, challenges — all via direct Firestore reads). The text input area is replaced with grayed-out text ("Upgrade plan to chat with Claws"), keyboard cannot be opened, send button grayed out. Tapping opens RevenueCat paywall. User can still browse everything, just can't send new messages.

**Recovery:** Automatic on `budget_reset_date` (monthly cycle), or immediate if user upgrades to a higher tier (RevenueCat webhook updates `monthly_allowance_cents`).

**Note:** The app should also check budget status on launch by reading the user document — show the grayed-out state immediately instead of letting the user type a message they can't send.

### 7.3 Auth Failed

**Trigger:** Firebase auth token in the request payload is invalid, expired, or missing (lifecycle Step 2).

**What the CF sends:** `{ event: "error", data: { code: "auth_failed", message: "Authentication failed" } }`. Connection closes immediately.

**What the app shows:** This should rarely be visible to users. The app should handle token refresh silently. If it does surface, show a generic "Please sign in again" screen and re-trigger the auth flow.

**Recovery:** Automatic. App refreshes the Firebase auth token and retries.

### 7.4 Message Too Long (+ Account Lock)

**Trigger:** Message exceeds 6,000 characters (lifecycle Step 3). This means the sender bypassed the app's 4,000 character UI limit — not a normal user.

**What the CF sends:** `{ event: "error", data: { code: "message_too_long", message: "Message exceeds limit" } }`. Also writes `account_locked: true` to the user's document. Connection closes immediately.

**What the app shows:** On this request, the error message. On the next request (or next app launch), the locked account screen from 7.1 / 8.2.

**Recovery:** Same as 7.1 — manual support review only.

### 7.5 Safety Flagged (TBD)

**Trigger:** Safety pre-screen rejects the user's message. Design pending — this is the last unchecked Tier 0 item.

**Open questions:**
- Does this lock the account, or just reject the single message?
- Does the user get a friendly explanation, or a generic error?
- Is there a strike system (3 flags = lock)?
- What does the app show?

Placeholder until the safety pre-screen decision is made.

### 7.6 Anthropic Error

**Trigger:** Anthropic API returns a 5xx error, times out, or the stream drops mid-response.

**What the CF sends:** `{ event: "error", data: { code: "anthropic_error", message: "Something went wrong, try again" } }`. Connection closes.

**What the app shows:** Friendly error message in the chat (see Section 8.4). The message the user typed should still be in the input field so they can resend without retyping.

**Recovery:** User taps send again. If it fails repeatedly, circuit breaker kicks in (Tier 2, design TBD).

**Failure timing matters:**
- Error before any text streamed → user sees error message, nothing else. Clean.
- Error mid-stream (partial response already on screen) → user sees a partial message plus the error. The partial text should remain visible but be visually marked as incomplete (e.g. faded, or with an error indicator). The incomplete message is NOT saved to conversation history — lifecycle Step 15 never fires, so nothing gets written.

---

## 8. UI Views

*(All app screens and their states — main chat, cat profiles, settings, onboarding, locked account screen, paywall, 90% usage banner, offline screen, etc.)*

**NOTE:** Every view that contains a chat input (main chat + Baby Claws on each profile section) must report its identity via `ui_context` in the request payload. This means each chat-enabled view needs to know its own `page`, `section`, and `cat_id` (if applicable) and include them when sending a message. Build this into every chat-enabled view from day one — retrofitting it later means touching every screen. See Section 2.0 (Scope Injection Plumbing Note) and Section 2.1 (Dictionary) for the `ui_context` spec.

### 8.1 Offline Screen

Cat pun screen when offline. No Firestore offline persistence. No functionality — just a friendly dead end until connection returns.

### 8.2 Locked Account Screen

**When:** `account_locked: true` on the user's Firestore document. Checked on app launch AND on every message attempt.

**What the user sees:** Full-screen takeover. No navigation bar, no tab bar, no access to any part of the app. Just:
- A message explaining the account is locked
- A link to the support website
- Nothing else

**Why it's aggressive:** The only way to trigger this is by bypassing the app's UI (character limit exploit). These are not normal users.

### 8.3 Budget Exceeded State

**When:** `used_monthly_allowance_cents` >= `monthly_allowance_cents`. Checked on app launch (read user document) AND returned as an error code if they try to send a message.

**What the user sees:** The app remains fully navigable. All existing data is accessible — cat profiles, conversation history, reminders, routines, challenges. The chat screen looks almost the same, with one change:
- The text input area is replaced with grayed-out text that says "Upgrade plan to chat with Claws" (copy TBD — Tier 4). The user cannot tap on it to open the keyboard. The send button is grayed out.
- Tapping the grayed-out area or the send button opens the RevenueCat paywall.
- If approaching budget (90%+), a softer warning banner appears first with usage percentage and upgrade link — before they hit the cap.

**Key principle:** Never punitive. The user paid for a tier and used what they paid for. Reward continued engagement by keeping data accessible. The upgrade path should feel like an invitation, not a wall. No repeated rejections, no error messages, no Claws being the one asking for money — the state change is immediate and clear.

### 8.4 Anthropic Error / Chat Error State

**When:** `anthropic_error` received during a message attempt, or stream drops mid-response.

**What the user sees:**
- **Error before any text:** An error bubble appears in the chat (not an alert/popup). Friendly copy: "Claws got distracted by a laser pointer. Try sending that again." (exact copy TBD — Tier 4). The user's message stays in the input field so they can tap send to retry without retyping.
- **Error mid-stream:** The partial response remains visible but is visually marked as incomplete (faded, or with a small error indicator at the end). An error bubble appears below it. The user's original message is NOT still in the input field (it was already sent), but they can tap a "Retry" button on the error bubble to resend the same message.

### 8.5 Reminders Carousel (Home Page)

**Where:** Home page, accessible without navigating into any cat's profile.

**What the user sees:** A horizontal carousel of all active reminders, sorted by due date (soonest first). Each reminder card shows:
- **Description** — "Apply Yeye's tick medicine"
- **Days until due** — "3 days" / "Tomorrow" / "Today" / "Overdue"
- **Cat name tag** — if the reminder is cat-specific (e.g. "Luna" or "Yeye"). No tag if it's a household reminder.
- **Recurring indicator** — if it's a recurring reminder (e.g. a small repeat icon or "Monthly")

**Tap into detail view (deepest page):** Shows full reminder info — description, due date/time, notes (if any, e.g. "Fast Luna 12 hours before"), recurring interval (if any), which cat (if any), and a done button.

**Why home-level:** Some reminders are household-level (`cat_id: ""`), not tied to any cat. Nesting reminders inside cat profile pages leaves household reminders with nowhere to go. One centralized view avoids that problem and gives the user a single place to see everything coming up.

**V1 scope note:** If a power user accumulates many reminders, the carousel could get long. Acceptable for V1 — if users feel strongly, they'll tell us. Filtering/grouping is a post-launch improvement.

### 8.6 Photo Attachment in Chat Input

**Where:** Chat input area, on every screen that has a chat input (main chat + Baby Claws on profile sections).

**Attaching a photo:**
- Photo button (camera or photo icon) next to the text input field
- Tapping opens the system photo picker (no library permission needed — it runs outside the app)
- After selecting a photo, a thumbnail preview appears in the chat input area above the text field
- The thumbnail is grayed out with a small spinning circle on top while uploading to Firebase Storage
- Small X button in the top-right corner of the thumbnail — tapping it cancels the upload (if still in progress) and removes the image. The upload stops, any partial file is automatically cleaned up by Firebase, nothing is left behind.
- If the internet drops during upload, the upload fails on its own. The spinning circle stops and the thumbnail shows an error state. User can tap X to remove it, or wait for connection to return and retry.
- Once upload completes, the thumbnail is no longer grayed out (full color), the spinning circle disappears, and the send button enables.
- Maximum 3 images per message. After 3 are attached, the photo button is hidden or disabled. This is a UX and budget decision — images cost tokens, and 3 is enough for Claws to see what's going on. Quality over quantity.

**Sending:** When the user hits send, the message payload includes the Firebase Storage download URLs for any attached images (see lifecycle Step 1). The images are already uploaded — send is instant.

**No photo attached:** The chat input works exactly as before. Text-only messages are just an array with one text block. The photo button is always available but never required.

### 8.7 Grief Mode UI (April 7)

- **Grief mode banner** in chat: shows only when a terminal/passed cat exists AND `grief_mode_override` is false (default). Text: "Claws is in grief mode. Change in Settings." Tapping goes to the toggle.
- **Grief mode toggle** in settings: only visible when a terminal/passed cat exists. Toggling it on sets `grief_mode_override: true` on the user's Firestore document. Cloud Function Step 9 checks this — if override is true, loads Normal/OB Claws instead of Grief Claws. No CP3 note needed — cat profiles already show health_status in CP3, and Sonnet with Normal Claws is contextually aware enough to be appropriate. If testing shows otherwise, a one-line CP3 note can be added (Tier 4 change).
- Both the banner and the toggle depend on the app knowing cat health statuses (already read from Firestore for profiles) and the `grief_mode_override` field on the user document.

---
