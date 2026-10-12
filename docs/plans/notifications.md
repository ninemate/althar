# Notifications plan

> **Temporary.** A product and implementation plan, not architecture. Move lasting decisions into an ADR and architecture docs when implementation begins.

Research checked against upstream `af3023a` and external documentation on 12 October 2026.

## Goal

Tell the person when something in any project deserves attention, without
starting every project's coordinator or turning routine work into a chorus.
Keep a cross-project record they can return to, with OS notifications as a
selective prompt to look.

This plan covers task and step state, lead reports, review conclusions,
actionable requests, and provider usage warnings. It does not change how a
coordinator receives news for its next turn.

## What exists today

| Surface | Current behavior | Evidence in this checkout |
| --- | --- | --- |
| OS notifications | Runtime `Nudges` reach Electron's main process. It tells configured `call`, `stopped`, and `ready` events only when no `BrowserWindow` is focused and `Notification.isSupported()` is true. Clicking opens the referenced thread. | `apps/desktop/src/runtime/runtime.ts:216-220`; `apps/desktop/src/main/main.ts:219-255`; `packages/runtime/src/Nudges.ts:25-41` |
| Nudge events | Open task attention requests and project-coordinator questions are collected under stable keys; stuck work becomes `stopped`; ready tasks become `ready`. New keys are announced once after startup and changes are gathered for 250 ms. There is no progress nudge. | `packages/runtime/src/Nudges.ts:22-41,106-168,204-245` |
| Coordinator threads | Each project has its own coordinator agent session/thread (ADR-004). Its news is read from the ledger before that coordinator's next turn, covering plan and issue-draft changes since its prior turn; this is not a cross-project notification dispatcher. | `docs/decisions/004-coordinator-is-an-agent-session.md`; `packages/runtime/src/coordinatorNews.ts:10-27,59-78,140-157` |
| Task and run records | Runs have implement/review/settle/publish nodes, attempt success/failure events, `step_result` thread items, and succeeded/failed/cancelled run events. A stuck step opens an attention request. | `packages/runtime/src/Runs.ts:366-418,464-523,680-725` |
| Review and external change state | Review results are step results with verdict/findings; external change snapshots include open/merged/closed state and aggregate check outcomes, and are recorded as `external_link.seen`. | `apps/desktop/src/renderer/features/home/HomeView.tsx:83-96`; `packages/runtime/src/Changes.ts:78-112,767-782` |
| Home activity | Home already spans projects and shows what waits, what runs, and what the loop did since the person last left. Its activity includes step results, dealt-with limit/stall notices, and counts of automatic permission answers. It is a since-last-look feed, not a durable, individually readable notification inbox. | `apps/desktop/src/renderer/features/home/HomeView.tsx:18-28,64-97`; `packages/runtime/src/Queries.ts:1311-1350,1353-1403`; `apps/desktop/src/renderer/features/home/useHome.ts:12-20` |
| Board, cards, and project chrome | The board separates Up next, Running, Needs you, and Settled; work cards show task status and step; project tabs/switcher surface where calls wait. These show current state in context, not cross-project event history. | `packages/ui/src/board/Board/Board.tsx:10-16,28-39,84-98`; `packages/ui/src/board/WorkCard/WorkCard.tsx:27-38,41-48,85-104`; `packages/ui/src/chrome/ProjectTabs/ProjectTabs.tsx:93-124`; `packages/ui/src/chrome/ProjectSwitcher/ProjectSwitcher.tsx:113-115,224-230`; `packages/ui/src/chrome/WorkStatus/WorkStatus.tsx:9-14,27-34` |
| Badges and edge | The Dock count is the current number of waiting calls/ready tasks when enabled. The edge tray mark has a waiting dot and opens a sheet; Linux may have no usable tray, and Settings shows the available edge. | `apps/desktop/src/main/main.ts:184-196,239-243`; `apps/desktop/src/main/notify.ts:24-25`; `apps/desktop/src/main/edgeWindows.ts:70-88,235-255,272-288,313-327`; `apps/desktop/src/renderer/features/settings/useEdgePlace.ts:8-15,38-48`; `packages/ui/src/setup/EdgeMini/EdgeMini.tsx:9-16` |
| Settings | There are switches for calls, ready tasks, stopped work, badge count, and sound. All notification kinds can be switched together. Defaults enable the three notification kinds and the badge. | `apps/desktop/src/main/appPreferences.ts:39-48,59`; `apps/desktop/src/renderer/features/settings/SettingsPanel.tsx:147-155,300-306`; `packages/ui/src/setup/Preferences/Preferences.tsx:190-238` |

The Home feed and task cards already provide the raw material for useful
activity. The missing piece is a persistent, cross-project place that records
notification-worthy events, tracks whether the person has read them, and
decides centrally whether an OS prompt is warranted.

## Surfaces considered

| Surface | What it is good at | Cost or limitation | Role in the design |
| --- | --- | --- | --- |
| Coordinator thread only | Gives an event context and lets the coordinator explain it in project language. | A coordinator speaks only when its thread gets another turn. Waking each project for routine state would create the chorus the person wants to avoid, add agent work, and make delivery depend on sessions. | Keep ledger news for the next natural coordinator turn. Never wake coordinators just to announce events. |
| OS notification | Timely, outside the app, with a click that can bring the relevant task forward. | Easy to miss, platform-specific, and not a history. Support does not guarantee the OS will display it. | A selective, centrally coalesced prompt for high-priority events. |
| In-app activity/inbox | Persistent, searchable or filterable, cross-project context; works when OS delivery is unavailable. | Needs read/unread state, navigation, retention, and a calm grouping model. | Canonical notification record and the place to review history. Extend Home's existing cross-project activity and expose a dedicated Activity view/panel from Home and the edge. |
| Badges | A quiet glance that something waits. | No context; counts can become noise and conflate unread history with work needing a decision. Linux tray and some app-icon badges are not universal. | Keep the current needs-you count semantics. Add at most an unread Activity dot/count in Althar's own chrome; do not make a tray or OS badge the inbox. |

## Recommended design

### One app-wide activity stream

The runtime records notification-worthy source events once, with a stable
source identity, project/task references, event time, priority, and read
state. A single Althar dispatcher consumes that stream across all projects.
The Activity view is the source of truth; a toast is a delivery attempt, not
the record itself. Opening a toast deep-links to the relevant request, task,
or thread and marks that activity item read.

Reuse Home's event vocabulary and cross-project project grouping, but give
Activity its own read cursor and retention policy. Home can continue to show
the short “since you last left” summary. Settings configures delivery; it is
not the notification inbox. The existing edge sheet is a convenient entry
point while the tray exists, but a window/menu entry must also reach Activity.

Do not send one agent message per event. Coordinators retain their current
ledger-based news and learn about changes on their next ordinary turn (ADR-004;
`packages/runtime/src/coordinatorNews.ts:59-78`).

### Event classes and delivery

| Event class | Source/eligibility | Activity view | OS default |
| --- | --- | --- | --- |
| Needs-you request: permission, plan/question, or coordinator question | A new open attention request. Include project and task/thread destination. | Immediate, actionable, remains unread until opened or answered. | **On.** High priority; batch simultaneous arrivals app-wide. |
| Work stopped / could not continue | New stuck request, including a confirmed usage-limit stop. | Immediate, with reason and next action. | **On.** High priority; deduplicate by request/run. |
| Task ready | First transition to ready after the current ready criteria hold. | Immediate, with task summary and project. | **On.** One per ready cycle. |
| Step or run state | Started, resumed, paused, succeeded, failed, cancelled. State can come from `node_attempt.*` and `run.*`. | Compact timeline; update the same task's current-state row where appropriate. | **Off.** Never toast on routine progress. A failure that opens a stuck request is covered above. |
| Lead report / step result | A saved `step_result` for implement, settle, or publish. | Activity entry, grouped beneath its task. | **Off.** Optional digest for people who want completion summaries. A ready event is the task-level prompt. |
| Review concluded | Review step result with pass, findings, or a request for direction. | Activity entry. If the task also becomes ready, show one task-ready prompt and link to the review result. | **Off** by default to avoid a second toast for the same task. Optional in the digest. |
| Usage warning before a limit | Only a fresh, structured provider snapshot with a known window, usage, and reset time. | Activity entry keyed to account and quota window, not to every task sharing that account. | **On when supported**, once per window at a proposed 80% threshold; confirmed exhaustion uses the stopped event. Never infer quota percentage from ACP context tokens. |
| External review/check/change state | A material change in a listened-to external change snapshot, not every poll. | Later stage: Activity entry for a changed verdict or merge/close. | **Off** until the core inbox and event semantics ship. |

The runtime already records several inputs: run attempt and run transitions
(`packages/runtime/src/Runs.ts:464-523`), step results (`:498-500`),
attention requests (`:680-725`), and external-link snapshots
(`packages/runtime/src/Changes.ts:767-782`). A “lead report” or a concluded
review is therefore a product classification over existing results, not a
claim that a separate notification event exists today.

### Coalescing and noise controls

1. **Deduplicate by source identity and transition.** Use an attention-request
   ID, task ID plus ready-cycle, attempt/run ID plus terminal transition, or
   account ID plus quota-window/reset. Repeated snapshots update the existing
   activity item. They do not create new rows or toasts.
2. **Aggregate globally, then group by project in the inbox.** Coordinators
   remain independent, but the dispatcher sees all projects. During a short
   proposed 5-second high-priority window, emit one toast for a single request
   or one summary toast (for example, “3 things need you across 2 projects”)
   for a burst. The summary opens Activity, where each item remains actionable.
3. **Digest low-priority events.** Step reports and review conclusions update
   Activity immediately but can be summarized at most once per chosen digest
   interval. Do not emit a toast for every step in a plan.
4. **Coalesce by task.** Several changes to one task during the batch window
   become one latest-state line. If `ready` follows a review result, the ready
   toast subsumes the review toast; both details remain in Activity.
5. **Respect focus and preferences, not visibility of history.** When an Althar
   window is focused, record events and update Activity, but suppress OS toasts
   as today. Turning off OS notifications never removes inbox entries. Quiet
   hours defer low-priority digests; actionable requests remain visible in-app.
6. **Keep badges distinct.** The Dock/tray “needs you” count remains the count
   of outstanding calls and ready tasks. An Activity unread marker is separate
   and should not inflate that count.

Five seconds and 80% are starting defaults to validate with use, not platform
limits. The dispatcher should be configurable without changing event records.

### Settings proposal

- **Master switch:** OS notifications on/off; Activity recording stays on.
- **Needs you:** calls/questions, stopped work, and tasks ready. On by default.
- **Usage:** early warning on when a provider supplies a trustworthy quota
  snapshot; choose threshold and quiet/digest behavior. Confirmed limit notices
  remain part of stopped-work delivery.
- **Activity summaries:** step/lead reports and review conclusions. Off for
  immediate OS delivery; a daily or hourly digest is opt-in.
- **Projects:** mute a project from OS delivery or mute low-priority summaries;
  never hide its in-app Activity history or a direct action required to unblock
  work without an explicit project-level choice.
- **Sound:** retain the existing sound preference. Keep alert content short
  and avoid putting sensitive report text in lock-screen previews; clicking
  opens the full item in Althar.
- **Badges:** keep the existing needs-you count toggle; make Activity unread
  affordance separate and optional.

## Electron, desktop, and Flatpak constraints

Research checked against Electron and Flatpak documentation on 12 October
2026.

- Electron's `Notification` is a **main-process** API. `isSupported()` reports
  platform support; it does not establish that the host will display a given
  notification. Calling `.show()` requests delivery, and the instance `click`
  event handles activation while the object is available. On Windows,
  `Notification.handleActivation()` can centrally handle clicks, replies, and
  actions even when the original object is gone, including cold start. On macOS,
  notifications use `UNNotification` and require a signed app. Electron
  documents that Windows toasts need a Start Menu shortcut with an
  AppUserModelID and ToastActivatorCLSID. On Linux Electron uses `libnotify` for desktops that
  follow the freedesktop Desktop Notifications specification. See [Electron
  Notification API](https://www.electronjs.org/docs/latest/api/notification)
  and [platform tutorial](https://www.electronjs.org/docs/latest/tutorial/notifications).
- Notification buttons are documented on macOS and Windows; selection actions
  are Windows-only, and macOS only treats the first button as primary. Do not
  put required controls inside the toast: a click to open the Activity item is
  the portable action. See [Electron notification actions](https://www.electronjs.org/docs/latest/api/structures/notification-action).
- Electron `id`/`groupId` are documented for macOS and Windows, not Linux;
  `groupTitle` is Windows-only. Native grouping can improve presentation but
  cannot implement cross-platform coalescing. Do that in Althar before calling
  `.show()`.
- Electron's tray uses StatusNotifierItem on Linux by default with a
  GtkStatusIcon fallback. Tray activation behavior varies by desktop.
  StatusNotifier needs a `org.kde.StatusNotifierWatcher` bus permission in
  Flatpak, and Flatpak notes status icons only appear on some distributions.
  A notification inbox must not depend on the tray. See [Electron Tray API](https://www.electronjs.org/docs/latest/api/tray),
  [Flatpak desktop integration](https://docs.flatpak.org/en/latest/desktop-integration.html),
  and the [StatusNotifierItem specification](https://specifications.freedesktop.org/status-notifier-item/latest/status-notifier-item.html).
- Flatpak lists showing notifications as a portal feature and documents
  Electron support for Linux desktop notifications. The XDG notification
  portal can update or withdraw notifications, but an app cannot know whether
  the host actually presented one. Treat OS delivery as best effort and the
  persisted Activity feed as authoritative. See [Flatpak desktop
  integration](https://docs.flatpak.org/en/latest/desktop-integration.html)
  and [the notification portal](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.Notification.html).

The separate tray-permission task owns any Flatpak manifest or tray changes.
This design assumes the tray is present where available and does not change
`package-linux`.

## ACP and provider usage data

### What ACP gives us

Althar already uses ACP for Claude Code through `claude-agent-acp`, Codex
through `codex-acp`, and OpenCode through `opencode acp` (ADR-002,
`docs/decisions/002-acp-for-every-agent.md:9-20`). Electron pins the first two
adapters at `apps/desktop/package.json:24-26`. The connection sends the
protocol version from its SDK and stores the version negotiated at initialize
(`packages/provider-adapters/src/AgentConnection.ts:530-560`); the current
adapter contract asserts protocol version 1
(`packages/provider-adapters/tests/contract.ts:76-83`). Package version and
wire protocol version are different facts.

ACP is useful for normal session events and, where implemented, context
window/cost usage. The ACP repository's v2 `UsageUpdate` schema has context
tokens, context size, and optional cumulative session cost; it has no provider-plan
window percentage or reset time ([ACP v2 schema](https://github.com/agentclientprotocol/agent-client-protocol/blob/main/docs/protocol/v2/schema.mdx),
checked 12 October 2026). A full context window is not a depleted Claude or
ChatGPT subscription allowance.

### Harnesses and quota signals

| Harness path in Althar | What the current evidence says | Plan/quota warning via standard ACP? |
| --- | --- | --- |
| Claude Code → `claude-agent-acp` | The adapter is pinned at `0.88.0`. Althar's adapter architecture says Claude's SDK `rate_limit_event` side channel is not read yet; usage limits currently come from errors (`packages/provider-adapters/ARCHITECTURE.md:44-49`). An upstream adapter issue asks to expose Claude subscription usage over ACP. | **Not demonstrated.** The documented native event is a future adapter capability, not a current ACP signal. |
| Codex → `codex-acp` | Pinned at `2.1.1`. The upstream adapter issue says it ingests/stores native `RateLimitSnapshot`, but exposes it as `/status` text rather than structured ACP data; the issue was open when checked 12 October 2026. | **Not demonstrated.** Do not parse `/status` markdown for a warning. |
| OpenCode → `opencode acp` | ACP session is used; Althar's current architecture recognizes provider rate-limit errors/retry hints. It says API-key use has no plan windows (`docs/architecture/03-agent-runtime-and-auth.md:402-406`). | **No quota window identified.** Do not assume another account-status API or infer plan allowance from token use. |

The ACP repository described protocol 1 as its current stable wire version
when checked. Its v2 migration guide describes a stable v2 baseline while
labeling the full v2 surface draft; the TypeScript SDK's stable entry point is
still v1 and its v2 entry point is experimental. Althar's current contract
checks v1. No shipped Claude, Codex, or OpenCode harness in this checkout can
be responsibly listed as supporting the full ACP v2 protocol for quota usage.
This means “not established by the checked evidence,” not a claim that no
experimental implementation exists.
See [ACP protocol repository](https://github.com/agentclientprotocol/agent-client-protocol),
[v2 migration notes](https://github.com/agentclientprotocol/agent-client-protocol/blob/main/docs/protocol/v2/migration.mdx),
and [TypeScript SDK version status](https://github.com/agentclientprotocol/typescript-sdk).

Two upstream adapter issues make the quota gap concrete: [Codex ACP #227](https://github.com/agentclientprotocol/codex-acp/issues/227)
(opened 23 June 2026; checked 12 October 2026) describes native rolling
windows held by the adapter but not emitted as structured ACP data; [Claude
Agent ACP #625](https://github.com/agentclientprotocol/claude-agent-acp/issues/625)
requests the corresponding Claude usage display (checked 12 October 2026).
These are implementation status reports, not guarantees of future support.

### Recommendation for usage warnings

Keep ACP as the session transport. Do not wait for every harness to adopt ACP
v2, and do not treat ACP `UsageUpdate` context use as subscription use. Put
quota status behind an optional typed adapter capability. When a provider has
a documented native signal and the person has authenticated the relevant
account, adapt that signal inside the provider adapter; Althar's architecture
already describes Claude SDK status and Codex app-server rate-limit reads as
the intended side channels, while its provider-adapters gap list confirms
they are not implemented (`docs/architecture/03-agent-runtime-and-auth.md:395-406`;
`packages/provider-adapters/ARCHITECTURE.md:44-49`). For a harness without a
trustworthy pre-limit signal, notify only when a structured limit error
confirms exhaustion. A documented provider API/account-status endpoint is a
reasonable interim source when its authentication and reset semantics are
clear, but keep it in that provider's adapter; this is a per-harness fallback,
not a universal endpoint. Do not scrape localized status text or poll
undocumented provider endpoints from the desktop app.

This gives the product useful warning coverage provider by provider without
making basic task, review, or attention notifications wait on ACPv2.

## Staged implementation outline

1. **Define event contract and policy.** Catalog source IDs, event transitions,
   priorities, project/task links, suppression rules, and the event-to-surface
   table above. Derive events from ledger records; do not call coordinators.
2. **Persist Activity state.** Add a durable cross-project activity record or
   index with source identity, read state, and bounded retention. Preserve Home's
   current since-last-look summary behavior.
3. **Build the in-app Activity surface.** Add an Activity panel/view reachable
   from Home and the edge. Group by project, show unread and needs-you states
   separately, and deep-link to the existing task/thread actions.
4. **Add the app-wide OS dispatcher.** Centralize preference/focus checks,
   event coalescing, digest scheduling, stable notification IDs, and click
   routing. Use native group IDs only as a presentation aid; test best-effort
   delivery across signed macOS, packaged Windows, and Linux desktop/Flatpak.
5. **Extend settings and migrate preferences.** Add master/category/digest,
   project mute, threshold where supported, and separate badge behavior while
   retaining current switches and defaults for calls, ready, stopped, sound,
   and needs-you count.
6. **Add usage signals incrementally.** First expose only structured snapshots
   from documented provider APIs in adapters, with source freshness and reset
   semantics. Enable warnings only for adapters that pass those checks; retain
   error-based confirmed-limit handling for every harness. Revisit ACP when a
   stable, negotiated protocol expresses plan-quota windows.

## Open decisions before implementation

- Confirm the default warning threshold and whether people need a second
  threshold per provider/window.
- Decide whether the optional lead/review digest is hourly, daily, or user-set.
- Set Activity retention and whether read/dismiss state is device-local or
  shared with another Althar instance.
- Validate the 5-second urgent aggregation window and whether a burst summary
  should include project names in OS previews.
