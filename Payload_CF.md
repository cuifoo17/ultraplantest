# Working Payload + Cloud Function Lifecycle

**Last updated:** 2026-04-07
**Purpose:** Technical reference for the request payload schema and the Cloud Function's end-to-end processing lifecycle. This is the developer-facing companion to the Plumbing Blueprint's Section 2. All types, field names, API formats, and error codes are implementation-ready.

---

## 1. Request Payload Schema

### Endpoint

`POST https://us-east1-{project-id}.cloudfunctions.net/chat`

Content-Type: `application/json`
Transport: HTTP with SSE response stream

### Payload Fields

```typescript
interface ChatRequest {
  auth_token: string;            // Firebase Auth ID token (JWT)
  conversation_id: string;       // UUID, generated client-side via UUID().uuidString. App creates on new conversation, reuses for all messages in that conversation. CF never generates these.
  message: ContentBlock[];       // Array of content blocks (text and/or image)
  ui_context: UIContext;         // Current screen state for scope injection
  timezone: string;              // IANA timezone (e.g. "America/Chicago")
  notification_permission: boolean; // iOS notification permission status
  payload_version: number;       // Always 1 for V1. Future-proofing for payload migrations.
}

interface ContentBlock {
  type: "text" | "image";
  text?: string;                 // Present when type === "text"
  url?: string;                  // Present when type === "image" — Firebase Storage download URL
}

interface UIContext {
  page: string;                  // "main_chat" | "cat_profile" | "settings"
  section?: string;              // "weight_graph" | "food_journal" | "habits" | etc.
  cat_id?: string;               // Firestore document ID of the cat being viewed (if on a cat-specific page)
}
```

### Payload Examples

**Text-only message:**
```json
{
  "auth_token": "eyJhbGciOiJSUzI1NiIs...",
  "conversation_id": "conv_a1b2c3d4",
  "message": [
    { "type": "text", "text": "Luna threw up after breakfast" }
  ],
  "ui_context": { "page": "main_chat" },
  "timezone": "America/Chicago",
  "notification_permission": true,
  "payload_version": 1
}
```

**Message with photo (image block before text block):**
```json
{
  "auth_token": "eyJhbGciOiJSUzI1NiIs...",
  "conversation_id": "conv_a1b2c3d4",
  "message": [
    {
      "type": "image",
      "url": "https://firebasestorage.googleapis.com/v0/b/claws-app.appspot.com/o/users%2Fuid123%2Fchat_images%2Fconv_a1b2c3d4%2Fimg_e5f6g7h8.jpg?alt=media&token=abc123"
    },
    { "type": "text", "text": "Look at this rash on Luna's ear, should I be worried?" }
  ],
  "ui_context": { "page": "main_chat" },
  "timezone": "America/Chicago",
  "notification_permission": true,
  "payload_version": 1
}
```

**Message from Baby Claws (cat profile page):**
```json
{
  "auth_token": "eyJhbGciOiJSUzI1NiIs...",
  "conversation_id": "conv_a1b2c3d4",
  "message": [
    { "type": "text", "text": "How much should Luna weigh?" }
  ],
  "ui_context": {
    "page": "cat_profile",
    "section": "weight_graph",
    "cat_id": "cat_x9y0z1"
  },
  "timezone": "America/Chicago",
  "notification_permission": true,
  "payload_version": 1
}
```

### Validation Constraints

| Field | Constraint | Enforced by |
|---|---|---|
| `auth_token` | Valid Firebase Auth JWT, not expired | CF Step 2 via `admin.auth().verifyIdToken()` |
| `conversation_id` | UUID string, generated client-side via `UUID().uuidString`. Security via Firestore path (`users/{uid}/`), not ID validation. App Check deferred to post-V1. | App generates on new conversation. CF uses as Firestore path key. |
| `message` | Array with ≥ 1 block | CF request parsing |
| `message` image blocks | Max 3 per message | App UI (photo button disabled after 3) + CF Step 3a |
| `message` text content | Max 6,000 characters (combined across all text blocks) | App UI (disabled at 4,000) + CF Step 3b (lock at 6,000) |
| `timezone` | Valid IANA timezone string | CF Step 7 (compared to stored value) |
| `payload_version` | Integer ≥ 1 | CF Step 1 |

### Pre-Send: Image Upload Flow (Client-Side)

Before the payload is sent, any attached images must already be in Firebase Storage:

1. User taps photo button → iOS `PhotosPicker` opens (out-of-process, no library permission)
2. User selects image → app receives `PhotosPickerItem`
3. App loads via `Transferable` protocol → `UIImage`
4. EXIF metadata stripped automatically (`UIImage` → `jpegData()` discards metadata)
5. Resize: `UIGraphicsImageRenderer`, longest edge ≤ 1,568px (Anthropic's optimal threshold — avoids server-side resize)
6. Compress: `.jpegData(compressionQuality: 0.75)` → typically 100-300 KB
7. Upload to Firebase Storage: `users/{uid}/chat_images/{conversation_id}/{uuid}.jpg`
8. Firebase returns download URL
9. URL included in `message` array as `{ type: "image", url: "..." }`

**Upload UX:** Image thumbnail in chat input, grayed out with spinner overlay + X cancel button. Cancel calls `StorageUploadTask.cancel()`, partial uploads auto-cleaned by Firebase. Send button disabled until all uploads complete.

**Storage security rules:**
```javascript
match /users/{userId}/chat_images/{conversationId}/{imageId} {
  allow read: if request.auth != null;
  allow write: if request.auth.uid == userId &&
                  request.resource.size <= 5 * 1024 * 1024;
  allow delete: if request.auth.uid == userId;
}
```

---

## 2. SSE Response Events

The CF streams responses as Server-Sent Events over the same HTTP connection.

```typescript
// Event types sent by CF → App
interface SSEEvents {
  text:   { data: string };                                          // Streamed text chunk
  status: { data: { tool: string, message: string, animation: string } }; // Tool execution in progress
  budget: { data: { used_monthly_allowance_percent: number } };      // Post-response budget update
  error:  { data: { code: string, message: string } };               // Error — see codes below
  done:   { data: {} };                                              // Response complete, close connection
}
```

### Error Codes

| Code | Message | Trigger | Recoverable |
|---|---|---|---|
| `auth_failed` | "Authentication failed" | Step 2 — invalid/expired Firebase JWT | Yes — app re-authenticates, retries |
| `too_many_images` | "Too many images" | Step 3a — > 7 image blocks (app caps at 3, CF tripwire at 7). Triggers account lock. | No — account locked |
| `invalid_image_url` | "Invalid image" | Step 3b — image URL doesn't match Firebase Storage pattern or user's uid | Yes — retry with valid images |
| `message_too_long` | "Message exceeds limit" | Step 3c — > 6,000 chars text. Triggers account lock. | No — account locked |
| `account_locked` | "Account locked" | Step 5 — `account_locked: true` on user doc | No — support only |
| `budget_exceeded` | "Monthly budget reached" | Step 6 — `used_monthly_allowance_cents >= monthly_allowance_cents`. App replaces text input with grayed-out "Upgrade plan to chat with Claws", keyboard disabled, tap opens paywall. | Yes — resets on `budget_reset_date` or tier upgrade |
| `safety_flagged` | TBD | TBD — safety pre-screen | TBD |
| `anthropic_error` | "Something went wrong, try again" | Anthropic 5xx, timeout, or stream drop | Yes — user resends |

---

## 3. Cloud Function Lifecycle (15 Steps)

### Overview

One Cloud Function. One HTTP connection. Stateless request/response with SSE streaming.

```
App → [HTTP POST] → CF → [Steps 1-11] → Anthropic API → [Steps 12-14 streaming + tool loop] → [Step 15 wrap-up] → App
```

**Ordering principle:** Stream to user first → save conversation second → charge budget last. Worst-case failure = free message, never a charge without a saved conversation.

---

### Step 1 — Receive and parse request

Parse JSON payload. Validate required fields are present. Check `payload_version` to determine parsing strategy (V1 for now — future versions may change `message` format).

### Step 2 — Auth token verification

```typescript
const decodedToken = await admin.auth().verifyIdToken(auth_token);
const uid = decodedToken.uid;
```

- Invalid/expired → `{ error: { code: "auth_failed" } }`, close.
- Valid → `uid` extracted. Continue.

### Step 3 — Message validation

**3a. Image count (CF tripwire at 7, app caps at 3):**
```typescript
const imageBlocks = message.filter(block => block.type === "image");
if (imageBlocks.length > MAX_IMAGES_CF_TRIPWIRE) { // MAX_IMAGES_CF_TRIPWIRE = 7
  await lockAccount(uid, "image_limit_bypass");
  return sendError("too_many_images");
}
```

**3b. Image URL validation:**
```typescript
const STORAGE_DOMAIN = "firebasestorage.googleapis.com";
for (const block of imageBlocks) {
  if (!block.url.includes(STORAGE_DOMAIN) || !block.url.includes(`/users%2F${uid}%2F`)) {
    return sendError("invalid_image_url"); // Soft rejection, no lock
  }
}
```

**3c. Text length:**
```typescript
const textContent = message
  .filter(block => block.type === "text")
  .map(block => block.text)
  .join("");
if (textContent.length > 6000) {
  await lockAccount(uid, "character_limit_bypass");
  return sendError("message_too_long");
}
```

### Step 4 — Read user document + cat profiles (parallel)

```typescript
const [userDoc, catProfiles] = await Promise.all([
  db.collection("users").doc(uid).get(),
  db.collection("users").doc(uid).collection("cats").get()
]);
```

Extracts: `account_locked`, `used_monthly_allowance_cents`, `monthly_allowance_cents`, `subscription_tier`, `ob_complete`, `session_count`, `timezone`, `grief_mode_override`, and all cat profile data.

1-4 Firestore reads total (1 user doc + up to 3 cat docs).

### Step 5 — Account locked check

```typescript
if (userDoc.data().account_locked === true) {
  return sendError("account_locked");
}
```

### Step 6 — Budget check

```typescript
const { used_monthly_allowance_cents, monthly_allowance_cents } = userDoc.data();
if (used_monthly_allowance_cents >= monthly_allowance_cents) {
  return sendError("budget_exceeded");
}
```

### Step 7 — Update timezone if changed

```typescript
if (request.timezone !== userDoc.data().timezone) {
  await db.collection("users").doc(uid).update({ timezone: request.timezone });
}
```

### Step 8 — Read all remaining context data (parallel)

```typescript
const [memories, reminders, routines, challenges, conversationHistory] = await Promise.all([
  getMemories(uid, catProfiles),           // Tier 1 + Tier 2 per cat
  getActiveReminders(uid, { withinDays: 3 }), // Only reminders due within 3 days
  getActiveRoutines(uid),
  getActiveChallenges(uid),
  getConversationHistory(uid, conversation_id) // Full message history for this conversation
]);
```

Conversation history may include image content blocks (stored as URLs) from previous turns — passed through as-is.

Reminder window (3 days) is a Tier 4 configurable variable. Claws can fetch all reminders via `get_reminders` tool on demand.

### Step 9 — Determine CP2 variant

```typescript
const hasGriefCat = catProfiles.some(cat =>
  cat.health_status === "passed" || cat.health_status === "terminal"
);

let cp2Variant: "grief" | "ob" | "normal";
if (hasGriefCat && !userDoc.data().grief_mode_override) {
  cp2Variant = "grief";
} else if (!userDoc.data().ob_complete) {
  cp2Variant = "ob";
} else {
  cp2Variant = "normal";
}

const cp2 = await getPersonaPrompt(cp2Variant); // Fetched from Firestore or cached in memory
```

One CP2 active at a time. Swapped, not layered.

**Account creation default:** `ob_complete` must be explicitly initialized to `false` when the user document is first created. CF also treats missing field as `false`: `const obComplete = userDoc.data().ob_complete ?? false`. Same principle for all boolean fields on the user document.

### Step 10 — Assemble CP3 (per-user content)

Build the per-user system prompt block from Steps 4 and 8 data:

```typescript
const cp3 = buildCP3({
  catProfiles,                    // All cats (up to 3), ~225 tokens each
  memoryTier1,                    // Core identity facts per cat
  memoryTier2,                    // Reinforced memories, capped per cat
  episodicSummaries,              // Last 3 session summaries, ~100-200 tokens each
  activeReminders,                // Due within 3 days only
  activeRoutines,
  activeChallenges,
  contextualFlags: deriveFlags(catProfiles),  // Kitten, geriatric, neutering flags
  scopeInjection: resolveScopeInjection(request.ui_context, catProfiles),
  careStandards,                  // Pinned veterinary facts
  sessionCount: userDoc.data().session_count,
  notificationPermission: request.notification_permission,
  localTime: computeLocalTime(request.timezone)
});
```

### Step 11 — Assemble full prompt and call Anthropic

```typescript
// Translate app message format → Anthropic content block format
const userMessage = message.map(block => {
  if (block.type === "text") {
    return { type: "text", text: block.text };
  }
  if (block.type === "image") {
    return { type: "image", source: { type: "url", url: block.url } };
  }
});

const response = await anthropic.messages.create({
  model: MODEL_ID,                // e.g. "claude-sonnet-4-5-20250929"
  max_tokens: MAX_RESPONSE_TOKENS,
  stream: true,
  system: [
    { type: "text", text: cp1Tools, cache_control: { type: "ephemeral" } },
    { type: "text", text: cp2, cache_control: { type: "ephemeral" } },
    { type: "text", text: cp3, cache_control: { type: "ephemeral" } }
  ],
  messages: [
    ...conversationHistory,       // CP4 — auto-cached progressively by Anthropic
    { role: "user", content: userMessage }
  ]
});
```

**Image token cost:** `(width × height) / 750` input tokens. Priced same as text. Typical resized photo (~1092×1092) = ~1,590 tokens = ~$0.005/image on Sonnet. Cached at 90% discount on subsequent turns.

### Step 12 — Stream text to the app

```typescript
for await (const event of response) {
  if (event.type === "content_block_delta") {
    if (event.delta.type === "text_delta") {
      sendSSE("text", event.delta.text);          // Stream to app
    }
    if (event.delta.type === "input_json_delta") {
      accumulateToolInput(event);                   // Buffer silently
    }
  }
}
```

App fades in each text chunk. Tool call JSON accumulated in memory — not sent to app.

### Step 13 — Check stop reason

```typescript
if (stopReason === "end_turn") goto Step 15;
if (stopReason === "tool_use") { loopCount++; goto Step 14; }
```

### Step 14 — Tool execution loop

**14a.** Parse tool call(s) from accumulated JSON.

**14b.** Send status event:
```typescript
sendSSE("status", { tool: "log_symptom", message: "Logging Luna's symptom...", animation: "working" });
```

**14c.** Execute tool(s) — Firestore writes. Parallel if multiple tools.

**14d.** Format tool results via template functions.

**14e.** Check `loopCount`:
```typescript
if (loopCount < 5) {
  // Call Anthropic again with conversation + tool_results appended
  // Resume streaming → back to Step 12
} else {
  goto 14f; // Safety cap
}
```

**14f.** Safety cap reached. Inject wrap-up message:
```typescript
const wrapUpMessage = "You have used the maximum number of tool calls for this message. Do not call any more tools. Respond to the user now.";
// Append as final message, one last API call
// This message is STRIPPED before saving to conversation history
```

**14g.** If Claws still returns `tool_use` after cap → ignore tool calls, proceed to Step 15 with whatever text was in the response.

#### Tool Loop Safety Layers

| Layer | What it does | Failure mode it covers |
|---|---|---|
| `loopCount` variable | Caps at 5 iterations | Normal operation — prevents excessive tool chaining |
| CF timeout (60s, Tier 4) | Kills function regardless of loop state | Counter bug — loop never terminates |
| Iteration logging | `tool_loop \| user: {uid} \| conv: {conv_id} \| iteration: {n} \| tools: {names}` | Observability — alerts on iteration 5, spots counter failures |
| Budget check (next message) | Rejects if over budget | Runaway cost on current message — prevents repeat |

**Worst-case cost within 60s timeout:** ~10-15 iterations × ~$0.007/iteration = **~$0.10**. Not a financial risk.

**Max API calls per user message:** 6 (1 initial + 5 tool loops). Accounted for in Step 15 budget calculation.

### Step 15 — Wrap up

**Ordering: stream → save → charge.**

**15a.** Save conversation turn to Firestore — batch write:
```typescript
await db.collection("users").doc(uid)
  .collection("conversations").doc(conversation_id)
  .collection("messages").doc(messageId)
  .set({
    user_message: message,              // Content block array (includes image URLs as-is)
    assistant_response: fullResponse,    // Text + any tool_use/tool_result blocks
    created_at: FieldValue.serverTimestamp()
  });
// 14f wrap-up message stripped — not part of conversation
// Image URLs stored in Anthropic content block format — no transformation needed on reload
```

**15b.** Calculate cost from Anthropic usage response:
```typescript
const totalInputTokens = accumulatedUsage.input_tokens;   // Across all loop iterations
const totalOutputTokens = accumulatedUsage.output_tokens;
const cacheReadTokens = accumulatedUsage.cache_read_input_tokens;
const cacheWriteTokens = accumulatedUsage.cache_creation_input_tokens;

const cost = calculateCost(MODEL_ID, totalInputTokens, totalOutputTokens, cacheReadTokens, cacheWriteTokens);
// Pricing rates from config table, keyed by model_id — never hardcoded
```

**15c.** Update budget:
```typescript
await db.collection("users").doc(uid).update({
  used_monthly_allowance_cents: FieldValue.increment(cost)
});
```

**15d.** Calculate percentage:
```typescript
const percent = Math.round((used_monthly_allowance_cents + cost) / monthly_allowance_cents * 100);
```

**15e.** Send budget event:
```
event: budget
data: {"used_monthly_allowance_percent": 48}
```

**15f.** Increment `session_count` if first message after session gap (session definition: Tier 4 variable).

**15g.** Send done event:
```
event: done
data: {}
```

**15h.** Close connection.

---

## 4. Firestore Paths Referenced

| Path | Who writes | Who reads | What it holds |
|---|---|---|---|
| `users/{uid}` | CF, RevenueCat webhook | CF (Step 4), App (launch checks) | User document — all user-level fields |
| `users/{uid}/cats/{cat_id}` | CF (via tools) | CF (Step 4), App (profiles) | Cat profile document |
| `users/{uid}/conversations/{conversation_id}/messages/{message_id}` | CF (Step 15a) | CF (Step 8), App (chat history UI) | Conversation turn — user message + assistant response |
| Firebase Storage: `users/{uid}/chat_images/{conversation_id}/{uuid}.jpg` | App (pre-send upload) | Anthropic (fetches URL in Step 11), App (chat history UI) | Uploaded chat images |

---

## 5. Config Variables (Tier 4 — Adjustable Without Deploy)

| Variable | Default | Where stored | What it controls |
|---|---|---|---|
| `MAX_IMAGES_APP` | 3 | iOS app (UI constraint) | Photo button disabled after 3 images attached |
| `MAX_IMAGES_CF_TRIPWIRE` | 7 | CF config / env var | Step 3a — CF-side lock tripwire. Gap between 3 and 7 accounts for bugs. |
| `MAX_TEXT_LENGTH` | 6000 | CF config / env var | Step 3b character limit (CF-side tripwire) |
| `TOOL_LOOP_CAP` | 5 | CF config / env var | Step 14e max iterations |
| `CF_TIMEOUT_SECONDS` | 60 | CF deployment config | Hard kill switch for runaway requests |
| `REMINDER_WINDOW_DAYS` | 3 | CF config / env var | Step 8 — how far ahead to load reminders into CP3 |
| `MODEL_ID` | `claude-sonnet-4-5-20250929` | Firestore config doc | Step 11 — which model to call |
| `SESSION_GAP_DEFINITION` | TBD | Firestore config doc | Step 15f — what counts as a new session |
