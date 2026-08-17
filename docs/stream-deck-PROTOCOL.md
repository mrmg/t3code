# T3 Code client protocol — findings for the Stream Deck plugin

Phase 0 deliverable. Everything below was read out of the local T3 Code clone at
`../t3code-6cac9819/` (worktree `t3code/6cac9819`). File references are relative to that
repo root. Nothing in the T3 Code repo was modified; this document is the only artifact.

**TL;DR of decisions**

- **Import strategy: re-implement a minimal wire client** (~300 LOC). `@t3tools/contracts`
  and `@t3tools/client-runtime` are `private: true` (not on npm), export raw `.ts` source,
  and resolve `effect` via a pnpm-workspace `catalog:` protocol that does not exist outside
  the t3code workspace. A `file:` dependency would import all of those problems.
  Details in §6.
- **Auth (decided with the maintainer): pairing token → `/oauth/token` exchange →
  least-privilege Bearer access token → `Authorization: Bearer` header on the WS upgrade.**
  The user runs one CLI command (`t3 auth pairing create --json`) and the plugin does the
  rest itself. `t3 auth session issue` was considered and **rejected** (hardcoded admin
  scopes). Details in §4.
- **Desktop focus: no deep-link exists.** `open -a "T3 Code"` (relaunch) only reveals the
  main window; it cannot navigate to a thread. The web UI *does* have a per-thread URL
  route, which is the only working "focus" fallback. Details in §7. This is the main GAPS.md
  candidate.

---

## 1. Transport & wire framing

The client-facing protocol is **effect-rpc over a single WebSocket**, JSON-serialized.

- Server endpoint: `GET /ws` (upgrade) — `apps/server/src/ws.ts:2302-2364`.
  The handler authenticates the upgrade, then mounts
  `RpcServer.toHttpEffectWebsocket(WsRpcGroup, …)` with `RpcSerialization.layerJson`
  (`ws.ts:2321-2326`).
- JSON serialization (`RpcSerialization.json`,
  `.repos/effect-smol/packages/effect/src/unstable/rpc/RpcSerialization.ts:58-71`):
  **one JSON object per WebSocket text frame**. The decoder also accepts a JSON *array* of
  messages per frame (batching), so clients must handle both. No length framing — WebSocket
  frames do it.

### Message envelopes (`.repos/effect-smol/.../rpc/RpcMessage.ts`)

Client → server:

| Shape | Purpose | Ref |
|---|---|---|
| `{ _tag:"Request", id, tag, payload, headers, traceId?, spanId?, sampled? }` | RPC call. `tag` is the method string (e.g. `"orchestration.dispatchCommand"`). `id` is `string \| number`, client-chosen. `headers` is `Array<[string,string]>` — we send `[]`. | L60-69 |
| `{ _tag:"Ack", requestId }` | **Required** acknowledgement for streamed chunks (see below). | L119-122 |
| `{ _tag:"Interrupt", requestId }` | Cancel an in-flight request. | L130-133 |
| `{ _tag:"Ping" }` | Keepalive (see below). | L153-171 |
| `{ _tag:"Eof" }` | Client finished sending (we never need this). | L142-144 |

Server → client:

| Shape | Purpose | Ref |
|---|---|---|
| `{ _tag:"Chunk", requestId, values: […] }` | One batch of streamed values (≥1). For `stream: true` RPCs each `values[i]` is one stream item (e.g. one `OrchestrationShellStreamItem`). | L229-233 |
| `{ _tag:"Exit", requestId, exit }` | Terminal. `exit` is `{ _tag:"Success", value }` or `{ _tag:"Failure", cause: [{ _tag:"Fail", error } \| { _tag:"Die", defect } \| { _tag:"Interrupt", fiberId }] }`. Non-streaming RPCs answer with a single `Exit` whose `value` is the result. | L256-286 |
| `{ _tag:"Defect", defect }` | Connection-level fatal error. | L321-324 |
| `{ _tag:"Pong" }` | Keepalive reply. | L391-401 |
| `{ _tag:"ClientProtocolError", error }` | We misbehaved; log it. | L295-298 |

### Rules a client must follow (verified against the vendored effect source)

1. **Ack every streamed chunk.** The server enables client acks by default
   (`RpcServer.ts:115`, `401-405`): for a `stream: true` request it parks on a per-request
   latch after each chunk batch until the client answers `{ _tag:"Ack", requestId }`
   (`RpcServer.ts:191-194`). The stock client sends the Ack immediately after processing
   each chunk (`RpcClient.ts:564-567`). **If we don't Ack, subscriptions deliver exactly one
   batch and silently stall.** This is the single easiest bug to write into a custom client.
2. **Ping every ~5 s, expect Pong.** The stock client sends `{"_tag":"Ping"}` on a 5-second
   cadence and treats a missed Pong as a dead connection (`RpcClient.ts:1145-1167`); the
   server replies `{"_tag":"Pong"}` (`RpcServer.ts:732-734`). WS-level pings also exist at
   the transport layer, but the effect-rpc ping is application-level JSON — we must emit it.
3. **Request ids are ours to choose.** Numbers are fine (`0, 1, 2, …`); they're branded but
   the wire value is a plain string/number (`RpcMessage.ts:43-51`).
4. **Disconnect/retry:** stock policy is exponential 500 ms ×1.5 capped at 5 s
   (`RpcClient.ts:1140-1143`) — good defaults to copy.

### The RPC method set

`WS_METHODS` (`packages/contracts/src/rpc.ts:196-316`) + `ORCHESTRATION_WS_METHODS`
(`packages/contracts/src/orchestration.ts:26-35`), assembled into `WsRpcGroup`
(`rpc.ts:985-1085`). **The deck needs exactly 6:**

| tag | kind | payload → success | required scope |
|---|---|---|---|
| `server.probe` | unary | `{}` → `{}` | `orchestration:read` |
| `server.getConfig` | unary | `{}` → `ServerConfig` (providers, models, auth descriptor) | `orchestration:read` |
| `subscribeServerConfig` | stream | `{}` → `ServerConfigStreamEvent` (live provider/model changes) | `orchestration:read` |
| `orchestration.subscribeShell` | stream | `OrchestrationSubscribeShellInput` → `OrchestrationShellStreamItem` | `orchestration:read` |
| `orchestration.subscribeThread` | stream | `OrchestrationSubscribeThreadInput` → `OrchestrationThreadStreamItem` | `orchestration:read` |
| `orchestration.dispatchCommand` | unary | `ClientOrchestrationCommand` → `{ sequence }` | `orchestration:operate` |

Scope requirements are enforced per-method in
`apps/server/src/auth/RpcAuthorization.ts:23-138` (dispatch→operate L24; subscriptions,
probe, getConfig→read L29-33). Errors come back as `Exit/Failure/Fail` with a typed error
(`OrchestrationDispatchCommandError` or `EnvironmentAuthorizationError { message,
requiredScope }`).

Nice-to-haves (not needed for v1): `orchestration.searchThreads`, `getTurnDiff`,
`getArchivedShellSnapshot`, `serverGetSettings`. There are also **HTTP equivalents** for the
bootstrap reads — `GET /api/orchestration/shell`, `GET /api/orchestration/threads/:threadId`,
`POST /api/orchestration/dispatch` (`packages/contracts/src/environmentHttp.ts:502-535`) —
useful as a reconnect fast-path or debugging, but the WS subscription alone is sufficient.

---

## 2. Orchestration subscription protocol

### `orchestration.subscribeShell` — the all-threads feed

Input (`orchestration.ts:544-560`):

```json
{ "afterSequence": 0, "requestCompletionMarker": true }   // both optional
```

Stream items (`OrchestrationShellStreamItem`, `orchestration.ts:532-542`), delivered in this
order (server impl `apps/server/src/ws.ts:1187-1293`):

1. `{ kind:"snapshot", snapshot: OrchestrationShellSnapshot }` — full state:
   `{ snapshotSequence, projects: OrchestrationProjectShell[], threads:
   OrchestrationThreadShell[], updatedAt }` (`orchestration.ts:500-506`).
2. `{ kind:"synchronized" }` — **only if** we sent `requestCompletionMarker: true`
   (`ws.ts:1229-1240`). This is our "resync complete, now render" signal.
3. Live events forever:
   - `{ kind:"thread-upserted", sequence, thread: OrchestrationThreadShell }` — full shell row
     each time (server coalesces bursts into refetches, so every upsert is authoritative,
     `ws.ts:1191-1201`)
   - `{ kind:"thread-removed", sequence, threadId }`
   - `{ kind:"project-upserted" | "project-removed", … }`

Resume: on reconnect we can pass `afterSequence = lastSeenSequence` to get a bounded replay
instead of a full snapshot; if the gap is too large the server falls back to a fresh snapshot
itself (`ws.ts:1248-1281`). Simplest correct behavior for the deck: always resubscribe
without `afterSequence` and treat the snapshot as ground truth (the brief's "always resync
from snapshot" requirement matches this).

### `orchestration.subscribeThread` — the selected-thread detail feed

Input (`OrchestrationSubscribeThreadInput`, `orchestration.ts:562-586`):

```json
{ "threadId": "…", "requestCompletionMarker": true }   // note: NO environmentId —
// thread ids are server-local; we connect to one server at a time
```

Stream items (`OrchestrationThreadStreamItem`, `orchestration.ts:1492-1505`):

1. `{ kind:"snapshot", snapshot: { snapshotSequence, thread: OrchestrationThread, page? } }`
   — full thread including `messages[]`, `activities[]`, `session`, `checkpoints[]`
   (`OrchestrationThread`, `orchestration.ts:378-423`).
2. Optional `{ kind:"synchronized" }`.
3. `{ kind:"event", event: OrchestrationEvent }` — typed events (`thread.activity-appended`,
   `thread.session-set`, `thread.turn-start-requested`, … full list at
   `orchestration.ts:1058-1089`).

**We need this feed for one thing: pending approval details.** The shell row only carries
the boolean `hasPendingApprovals`; the `requestId`/`kind`/`summary` needed to answer an
approval live in `thread.activities` (§3.3).

### `OrchestrationThreadShell` — what the deck renders from

`orchestration.ts:448-498`. Fields that matter to us:

| field | type | deck usage |
|---|---|---|
| `id` | ThreadId | key-slot identity, subscribeThread target |
| `projectId` | ProjectId | join against `projects[]` for the project name |
| `title` | string | dial strip / key label |
| `modelSelection` | `ModelSelection` | provider + model display (§5) |
| `runtimeMode` | `"approval-required" \| "auto-accept-edits" \| "auto" \| "full-access"` | runtime-mode key icon |
| `interactionMode` | `"default" \| "plan"` | (optional) badge |
| `latestTurn` | `null \| { state: "running"\|"interrupted"\|"completed"\|"error", startedAt, completedAt, … }` | done/error/idle timing |
| `session` | `null \| OrchestrationSession` | working/error status; `session.providerInstanceId`, `session.lastError` |
| `hasPendingApprovals` | boolean | **amber flashing state** |
| `hasPendingUserInput` | boolean | separate attention state (v1: render like approval, no respond UI) |
| `hasActionableProposedPlan` | boolean | (optional) badge |
| `backgroundLiveness` | `"working" \| "monitoring" \| null` (optional) | keeps "working" after turn settles |
| `archivedAt` / `deletedAt` | `null \| ISO` | drop from key slots |
| `snoozedUntil` / `settledAt` / `pinnedAt` | optional | slot ordering/filtering |
| `updatedAt` | ISO | recency ordering for slot assignment |

### Status mapping (steal the web app's precedence verbatim)

`resolveSidebarThreadStatus`, `apps/web/src/components/Sidebar.logic.ts:475-499`:

```
hasPendingApprovals        → needs-approval   (amber)
hasPendingUserInput        → input            (v1: treat as needs-approval visually)
session.status running|starting → working     (pulse)
session.status error       → error            (red)  // outranks backgroundLiveness
backgroundLiveness working → working
backgroundLiveness monitoring → monitoring    (v1: render as idle-dim or its own state)
else                       → idle/ready
```

"Done" isn't a session status — derive it from `latestTurn.state === "completed"` (with
`session.status` in `ready`/`idle`) for the green-flash-then-steady behavior.

---

## 3. Commands we dispatch

All via `orchestration.dispatchCommand`, payload = one `ClientOrchestrationCommand`
(union at `orchestration.ts:940-965`). Every command carries a client-generated
`commandId` (any unique string — it's the correlation id, `orchestration.ts:163-164`) and
mostly a `createdAt` ISO timestamp. Success response is `{ sequence }`
(`DispatchResult`, `orchestration.ts:1566-1569`) — fire-and-forget; the *effect* arrives
through the subscriptions.

| Deck action | command | payload (beyond `type`, `commandId`, `createdAt`) | ref |
|---|---|---|---|
| New thread + first turn | `thread.turn.start` **with `bootstrap.createThread`** | `threadId` (client-generated), `message: { messageId, role:"user", text, attachments: [] }`, `runtimeMode`, `interactionMode`, `bootstrap: { createThread: { projectId, title, modelSelection, runtimeMode, interactionMode, branch, worktreePath, createdAt } }` | L825-863, L799-823 |
| (alt) create then start | `thread.create` then `thread.turn.start` | create: `threadId, projectId, title, modelSelection, runtimeMode, interactionMode, branch, worktreePath` | L667-681 |
| Approve / deny | `thread.approval.respond` | `threadId, requestId, decision: "accept" \| "acceptForSession" \| "decline" \| "cancel"` | L873-880, L134-140 |
| Interrupt | `thread.turn.interrupt` | `threadId` (`turnId` optional — omit to hit the active turn) | L865-871 |
| Cycle runtime mode | `thread.runtime-mode.set` | `threadId, runtimeMode` | L783-789 |
| Stop session | `thread.session.stop` | `threadId` (+ optional `onlyIfSettled`) | L899-910 |

Notes:

- **`thread.turn.start` with bootstrap is the one-call new-thread flow** — no separate
  `thread.create` needed. `branch`/`worktreePath` can be `null` (server picks). 
- Approval `requestId` comes from the thread's activities (next section). Responding with a
  stale/unknown id is a safe no-op server-side (the web app explicitly guards against those
  error strings, `apps/web/src/session-logic.ts:376-383`) — the deck should still re-render
  from events, not assume success.
- There is also `thread.user-input.respond` (free-form question answers,
  `orchestration.ts:882-889`) — out of scope for v1 keys, but `hasPendingUserInput` threads
  should not be mistaken for approval-pending ones.

### Pending approval details (the Approve/Deny key labels)

From the selected thread's `activities` (subscribeThread snapshot + `thread.activity-appended`
events). The web app's derivation, which we mirror
(`apps/web/src/session-logic.ts:386-424`):

- `activity.kind === "approval.requested"` with `activity.payload.requestId` (string),
  `payload.requestKind ∈ "command" | "file-read" | "file-change"` (fall back to mapping
  `payload.requestType`: `command_execution_approval`/`exec_command_approval`/
  `dynamic_tool_call`→command, `file_read_approval`→file-read,
  `file_change_approval`/`apply_patch_approval`→file-change, L354-364), and human text in
  `payload.detail`; `activity.summary` is a pre-rendered one-liner.
- `activity.kind === "approval.resolved"` with the same `requestId` closes it.
- Maintain an ordered map requestId→approval; the *oldest open* one is what the Approve/Deny
  keys act on (matches web behavior).

---

## 4. Auth handshake (the important part)

### Policies and capabilities

Server auth posture is advertised in `serverGetConfig → config.auth: ServerAuthDescriptor`
(`packages/contracts/src/auth.ts:138-144`) and at the unauthenticated discovery endpoint
(§4.2). Policy is derived from host/mode (`apps/server/src/auth/EnvironmentAuthPolicy.ts:16-42`):

| policy | when | bootstrap methods |
|---|---|---|
| `desktop-managed-local` | desktop app, loopback | `desktop-bootstrap` |
| `loopback-browser` | `npx t3 serve` on loopback | `one-time-token` |
| `remote-reachable` | non-loopback host | `desktop-bootstrap`+`one-time-token` (desktop) / `one-time-token` |
| `unsafe-no-auth` | explicit escape hatch (no flag found in `apps/server/src`; reserved in contracts L29-35) | — |

Every normal deployment requires credentials; `sessionMethods` always includes
`bearer-access-token` (and `dpop-access-token`, which we do **not** need — DPoP is for the
relay/mobile path; binding keys would be pure overhead on a loopback plugin).

### Scopes

`auth.ts:76-110`. `AuthStandardClientScopes` = `orchestration:read`,
`orchestration:operate`, `terminal:operate`, `review:write`, `relay:read` — exactly covers
the six methods in §1. (`access:*` and `relay:write` are administrative; not needed.)

### Server discovery (how `t3 pair` finds a running server)

`apps/server/src/cli/pair.ts:250-301`:

1. Candidate state dirs: inside a linked worktree its own `.t3` first, then `$T3CODE_HOME`,
   then the default base dir (`~/.t3`). For each, both `userdata/` and `dev/` variants.
2. Read `<stateDir>/server-runtime.json` (`PersistedServerRuntimeState`,
   `apps/server/src/serverRuntimeState.ts:11-21`; path assembled at
   `apps/server/src/config.ts:130`): `{ version:1, pid, host?, port, origin, devUrl?, startedAt }`.
3. Guard: the recorded `pid` must be alive (protects against a stale file whose port was
   reused).
4. Probe `GET {origin}/.well-known/t3/environment` → `ExecutionEnvironmentDescriptor`
   `{ environmentId, label, platform, serverVersion, capabilities }`
   (`packages/contracts/src/environment.ts:86-93`). Unauthenticated; doubles as the
   environment-identity check so the deck can pin an `environmentId` in settings and warn if
   the server on that port is suddenly a different environment.

For the deck: **default discovery = read `~/.t3/userdata/server-runtime.json`** (desktop app
and `npx t3 serve` both land there), with a Property-Inspector override for
host/port/remote URL. Dev servers write the same file under the worktree's `.t3/userdata/`
(useful for the smoke test).

### The pairing flow the deck will use

**One-time setup command (run by the user):**

```bash
npx t3 auth pairing create --json --label "Stream Deck"
# → { "id": "…", "credential": "<bootstrap token>", "scopes": ["orchestration:read", …],
#     "expiresAt": "…" }        (apps/server/src/cli/auth.ts:84-114; JSON shape at
#                                apps/server/src/cliAuthFormat.ts:44-56)
```

This command works against the server's local database directly (via
`resolveCliAuthConfig` → `ServerConfig`, `apps/server/src/cli/config.ts:394-404`) — the
server only needs to have run once; it does **not** need to be answering HTTP at that
moment (unlike `t3 pair`, which requires a live server and prints a QR/URL).
Default one-time-token TTL is **5 minutes** (`PairingGrantStore.ts:241`) — pass
`--ttl 1h` if we want a more relaxed setup window.

**Then the plugin, once, at first launch (or via a `t3sd-pair` helper script):**

```
POST {origin}/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=<bootstrap credential>
&subject_token_type=urn:t3:params:oauth:token-type:environment-bootstrap
&requested_token_type=urn:ietf:params:oauth:token-type:access_token
&client_label=Stream Deck
&client_device_type=bot
&client_os=macOS
```

Contract: `AuthTokenExchangeRequest` (`auth.ts:175-185`, form-encoded);
`POST /oauth/token` route (`environmentHttp.ts:426`). The exchange **consumes** the
bootstrap credential (single-use) and issues a session
(`EnvironmentAuth.ts:691-744`). Response `AuthAccessTokenResult` (`auth.ts:187-194`):

```json
{ "access_token": "<base64url>.<hmac>", "issued_token_type":
  "urn:ietf:params:oauth:token-type:access_token", "token_type": "Bearer",
  "expires_in": 2592000, "scope": "orchestration:read orchestration:operate …" }
```

Default access-token TTL: **30 days** (`SessionStore.ts:403`); tokens are
`base64url(payload).signature` HMAC'd with a server-side key (`SessionStore.ts:647`).
We persist `{ origin, environmentId, access_token, expiresAt }` in the plugin's credential
file (mode 0600), and re-run pairing when it expires.

**Rejected alternative — `npx t3 auth session issue --json`:** one command, prints a
ready-made bearer token (`{ sessionId, token, method, scopes, subject, client, expiresAt }`,
`cliAuthFormat.ts:118-132`), works against the DB directly just like pairing create. But it
has **no scope flag** — scopes are hardcoded to `AuthAdministrativeScopes`
(`cli/auth.ts:176-177`), which adds `access:read`/`access:write`/`relay:write` on top of the
standard set. A leaked admin token can mint new pairing tokens for arbitrary clients
(`POST /api/auth/pairing-token`, `http.ts:339`), list and revoke any session including the
user's own browsers (`http.ts:388-427`), and install/configure the relay client. The
pairing + exchange path yields the same UX (one command, one pasted string) with
least-privilege scopes that can't self-propagate. Decision: **pairing + exchange only**;
`session issue` is not offered in the README. (Side note verified while evaluating: both
CLI paths share the running server safely — the SQLite layer sets `busy_timeout=5000` and
WAL with an explicit "CLI and server write from separate processes" comment,
`persistence/Layers/Sqlite.ts:33-41`; both run migrations on open, so the CLI must be the
same install version as the running server.)

### Per-connection WebSocket auth

Two ways in; both end at `authenticateWebSocketUpgrade`
(`apps/server/src/auth/EnvironmentAuth.ts:941-961`):

1. **Ticket (what every official client does):**
   `POST /api/auth/websocket-ticket` with header `Authorization: Bearer <access_token>`
   → `{ ticket, expiresAt }` (5-minute TTL, `SessionStore.ts:404`;
   contract `auth.ts:196-200`, route `environmentHttp.ts:434`). Then connect to
   `ws(s)://host/ws?wsTicket=<ticket>` (`EnvironmentAuth.ts:502,945-956`; client-runtime
   reference `packages/client-runtime/src/authorization/remote.ts:185-191`).
   A ticket is minted **per connection attempt** — on reconnect, fetch a fresh one.
2. **Direct header:** since the ticket check falls through to the normal HTTP
   authenticator (`EnvironmentAuth.ts:960`), a Node WebSocket client can also put
   `Authorization: Bearer <access_token>` on the upgrade request and skip the ticket call.
   Fewer moving parts; equally supported. (Browsers can't set that header, which is why the
   ticket path exists.)

Recommendation: **use the direct `Authorization` header** (we're Node, we can), keep the
ticket flow as documented fallback. Sanity check after connect: `server.probe` (requires
`orchestration:read`; an auth failure surfaces as a failed `Exit`).

### Token expiry / reconnect behavior

- 401 on the ticket endpoint or a failed WS auth → mark credential bad, surface
  "re-pair needed" on the keys, keep retrying discovery cheaply.
- Access-token expiry is knowable in advance (`expires_in`); proactively re-pair or warn
  the user a day before. There is no refresh-token flow — re-issue is the flow.

---

## 5. Providers, models, and options (multi-provider requirement)

### Identity model

`ProviderDriverKind` = implementation slug (`"codex"`, `"claudeAgent"`, `"cursor"`,
`"grok"`, `"opencode"` — note **claudeAgent**, not `claude`;
`packages/contracts/src/model.ts:130-134`). It is an **open branded slug**, not a closed
union — forks can add drivers and old payloads must still parse
(`packages/contracts/src/providerInstance.ts:58-71`). → The deck must theme known drivers
and fall back to neutral gray for unknown ones, exactly as the brief's resilience section
says.

`ProviderInstanceId` = user-defined routing slug; a user can run two instances of one
driver (`codex_personal`, `codex_work`). **Threads bind to instances, not drivers.**
Default instance id = driver kind (`defaultInstanceIdForDriver`, `providerInstance.ts:148-149`).

### Enumeration at runtime

`server.getConfig → ServerConfig.providers: ServerProvider[]`
(`packages/contracts/src/server.ts:161-198`, config at L420-449). Each entry:

```
instanceId, driver, displayName?, accentColor?, badgeLabel?,     // ← theming!
enabled, installed, status: ready|warning|error|disabled,
auth: { status: authenticated|unauthenticated|unknown, … },
models: ServerProviderModel[]  // slug, name, shortName?, subProvider?,
                               // isCustom, isDefault?, isLegacy?,
                               // capabilities: ModelCapabilities | null
```

Live updates arrive via `subscribeServerConfig` events (`providerStatuses`,
`server.ts:517-523`) — wire this so keys re-theme when a provider dies or a model list
changes.

- **Per-model options** (reasoning effort etc.) come from
  `model.capabilities.optionDescriptors: ProviderOptionDescriptor[]`
  (`model.ts:7-44`, `125-128`): each is `{ id, label, type: "select"|"boolean",
  options?: [{id,label,isDefault}], currentValue? }`. The Property Inspector's per-provider
  defaults should be built from these descriptors, not hardcoded.
- **Selection wire shape:** `ModelSelection = { instanceId, model, options? }` where
  `options` is the **canonical array** `[{ id, value: string|boolean }]`
  (`ProviderOptionSelections`, `model.ts:90-94`; a legacy object form exists but the server
  normalizes it — we only ever *send* the array).
- Defaults to pre-fill in the PI: `DEFAULT_MODEL_BY_PROVIDER` (`model.ts:150-156`) —
  codex `gpt-5.6-sol`, claude `claude-sonnet-5`, cursor `auto`, grok `grok-build`,
  opencode `openai/gpt-5`; slug aliases at `model.ts:168-215`; display names at
  `model.ts:219-225`.
- `ModelSelection` on a thread shell tells us exactly which instance+model every thread
  runs — the status-key "provider AND model" requirement is fully covered by
  `subscribeShell` alone.

---

## 6. Import strategy decision (brief item 3)

Options considered:

1. **npm import `@t3tools/contracts` / `@t3tools/client-runtime`** — ❌ Both are
   `"private": true` (`packages/contracts/package.json`, `packages/client-runtime/package.json`),
   nothing is published.
2. **`file:` dependency into the local clone** — ❌ Rejected:
   - Their `exports` map points at raw TypeScript source (`"./src/index.ts"`), so our build
     would need to typecheck/transpile their world.
   - `contracts` depends on `"effect": "catalog:"` — a pnpm-workspace-only protocol that
     fails to resolve outside the t3code repo.
   - It pins the plugin to wherever the user happens to keep the clone — wrong for a
     distributable plugin, and it breaks the moment the clone updates schemas mid-dev.
3. **Re-implement the minimal wire subset** — ✅ Chosen.

What we re-implement (small, and all pinned down in §1–§4):

- ~150 LOC: WS client with `{_tag}` envelope demux, request-id map, Ack-after-Chunk,
  5 s Ping, Exit handling, reconnect with the stock backoff (500 ms ×1.5, cap 5 s).
- ~150 LOC: hand-ported TypeScript *types* (no schemas, no runtime validation dependency)
  for the ~12 shapes we consume: `OrchestrationThreadShell`, `OrchestrationShellStreamItem`,
  `OrchestrationThreadStreamItem` (activities subset), the 6 command payloads,
  `ServerConfig.providers`, `ModelSelection`, auth request/response bodies.
- Validation posture: **parse defensively, never hard-fail on unknown fields/kinds**
  (matches the contracts' own forward-compat posture — `ForwardCompatibleArray`,
  open driver slugs, optional new fields everywhere). Log-and-ignore unknown event kinds.

Coupling cost: if T3 Code changes the wire, we update our types. That is the correct price
for zero fork coupling, and it's what the mobile app effectively does through its own
runtime layer anyway.

---

## 7. Focusing the T3 Code desktop app (brief item 5)

**Finding: there is no deep-link, CLI, IPC, or RPC to navigate the desktop app to a
specific thread.** Verified by absence:

- `t3code://` is the desktop app's *internal* Electron content scheme
  (`apps/desktop/src/electron/ElectronProtocol.ts:112-123`, `protocol.handle`) — it serves
  the bundled web UI; it is not an OS-level deep-link into navigation.
- No `open-url`/`deeplink` handler exists anywhere in `apps/desktop` (grep: only menu
  actions like `open-settings`).
- Launching a second instance only **reveals the main window** (`second-instance` handler,
  `apps/desktop/src/window/DesktopClerk.ts`); no argv/URL is routed to a thread.
- The `t3` CLI has no `open` command (subcommands: start/serve/pair/auth/project/service/
  connect).

**What does exist:** the web app's thread route is `/{environmentId}/{threadId}`
(`apps/web/src/threadRoutes.ts:42-66`). So the realistic focus behaviors are:

- **Desktop app:** `open -a "T3 Code"` — brings the app forward, lands wherever it was.
  Honest limitation; goes in GAPS.md with the minimal upstream fix (an `open-url` handler
  that parses `t3code://<env>/<thread>` and navigates the router — ~20 lines in the
  desktop shell).
- **Web surface:** `open "http://<host>:<port>/<environmentId>/<threadId>"` — full
  navigation, works today. Worth a PI toggle: "focus target: desktop app | browser tab |
  none".

---

## 8. Resilience & edge cases (protocol-level)

- **Resync:** drop all local state on reconnect; subscribeShell without `afterSequence`;
  wait for `synchronized` before declaring keys live. `snapshotSequence` is still worth
  tracking for debugging.
- **Offline:** WS close/error or ticket 401 → gray "offline" keys; retry with the stock
  backoff; re-mint ticket (or just re-send header) per attempt.
- **Staleness:** with the push model, "no events for N minutes" is normal on idle systems —
  do **not** dim merely for silence. Instead: if `session.status` says running but the
  *process* is gone (WS dropped), the offline state already covers it. (The heartbeat the
  brief asks for is the connection liveness, not per-thread freshness.)
- **Unknown event kinds / drivers:** ignore-and-log; neutral gray icon; never crash —
  the contracts are explicitly forward-compatible (`providerInstance.ts:16-32`).
- **Archived/deleted threads** disappear from the shell stream (`thread-removed` /
  `archivedAt` set) → reclaim the key slot.
- **Approval race:** if the user answers from the deck and the web UI simultaneously, the
  second response is a harmless no-op (server logs "unknown/stale pending approval
  request"); the resolved event reconciles both UIs.
- **Multiple servers:** out of scope for v1 (one environment per profile); the protocol
  supports it by opening one WS per server and merging slot registries.

## 9. Open questions / assumptions to validate in the smoke test

1. Confirm a streamed `subscribeShell` stalls without Acks (validates §1 rule 1 in
   practice) — the unit-test fixture recorder will show this immediately.
2. Confirm `thread.turn.start` + `bootstrap.createThread` is accepted from a standard-scope
   token (web uses it, but our token has no cookie session — scope check says fine).
3. Exact `activity.summary` text quality on 72×72 (SD) / 200×100 (SD+ strip) — may need to
   prefer `payload.detail` truncated.
4. Whether `subscribeServerConfig` is worth it in v1 vs. re-`getConfig` on reconnect
   (lean: reconnect-only; decide during build).

---

### Appendix A — minimal session transcript (shapes, abbreviated)

```jsonc
→ {"_tag":"Request","id":1,"tag":"server.probe","payload":{},"headers":[]}
← {"_tag":"Exit","requestId":1,"exit":{"_tag":"Success","value":{}}}

→ {"_tag":"Request","id":2,"tag":"orchestration.subscribeShell",
   "payload":{"requestCompletionMarker":true},"headers":[]}
← {"_tag":"Chunk","requestId":2,"values":[{"kind":"snapshot","snapshot":{…}}]}
→ {"_tag":"Ack","requestId":2}                                   // required!
← {"_tag":"Chunk","requestId":2,"values":[{"kind":"synchronized"}]}
→ {"_tag":"Ack","requestId":2}
← {"_tag":"Chunk","requestId":2,"values":[{"kind":"thread-upserted","sequence":8123,
    "thread":{"id":"…","hasPendingApprovals":true,…}}]}
→ {"_tag":"Ack","requestId":2}

→ {"_tag":"Request","id":3,"tag":"orchestration.dispatchCommand",
   "payload":{"type":"thread.approval.respond","commandId":"sd-…",
              "threadId":"…","requestId":"…","decision":"accept",
              "createdAt":"2026-08-17T10:00:00.000Z"},
   "headers":[]}
← {"_tag":"Exit","requestId":3,"exit":{"_tag":"Success","value":{"sequence":8124}}}

→ {"_tag":"Ping"}   ← every ~5 s
← {"_tag":"Pong"}
```
