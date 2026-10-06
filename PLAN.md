# IM Fleet on OpenMausBot — Build Plan

**For:** Claude Code or OpenCode (running on GLM through LiteLLM), on the user's work Mac.
**Goal:** a fleet of Android phones, each driven by a persona bot in OpenMausBot, that holds realistic, human‑like conversations with each other over IM apps (WhatsApp, Telegram, Signal, …), one‑to‑one and in groups, for durations the user chooses. Built to start with 4–5 devices and grow to 25–50.

---

## 0. Read this first: how to work with this plan

1. **Never guess anything about this Mac.** Folder locations, MCP names, tool names, model IDs, ports, the LiteLLM address, the login token location: discover them, or ask the user. Phase 0 collects them into `fleet.config`; every later step reads from there. Nothing environment‑specific is hardcoded.
2. **Names in this plan are roles, not exact names.**
   - "**android-core MCP**" = the user's existing MCP with the ADB functions (device list, device details, phone number extraction, …).
   - "**android-im MCP**" = the user's existing MCP that controls the IM apps (includes a self‑healing fallback that uses GLM Flash vision on screenshots).
   Both **already exist**. Do not rebuild them. Find their real names and tools in Phase 0 and confirm them with the user.
3. **Items marked ⚠️ must be verified on the user's installed OpenMausBot before relying on them.** Items marked ✅ were verified in the OpenMausBot source. Items marked 🔧 are things we build around OpenMausBot.
4. **Stop at every stage gate** (end of each phase) and show the user the result. Do not start the next phase without the user's OK.
5. **Keep `README.md` at the fleet root up to date** as you build: a short map of what lives where and how to run things.
6. **When a detail is missing:** try the tools first; if that fails, ask the user once; save the answer so it is never asked again.
7. **Keep it simple.** Do not add components that the plan does not need (for example, no decision model like Jev/Kev unless a clear need appears later and the user agrees).

---

## 1. The system in one picture

```
 User ──"I connected a new device" / "start a WhatsApp group with 3 devices for 10 min"──▶
        ┌─────────────────────────────── OpenMausBot (installed desktop app) ───────────────────────────────┐
        │  Section "IM Fleet"                                                                               │
        │   ┌──────────────────────┐   creates / delegates    ┌──────────────┐ ┌──────────────┐            │
        │   │ Chief of Staff       │ ───────────────────────▶ │ Device bot A │ │ Device bot B │ …          │
        │   │ (coordinator)        │                          │ (persona)    │ │ (persona)    │            │
        │   └─────────┬────────────┘                          └──────┬───────┘ └──────┬───────┘            │
        └─────────────┼──────────────────────────────────────────────┼────────────────┼────────────────────┘
                      │ android-core (read-only)                     │ android-im + android-core,
                      │ fleet MCP                                    │ pinned to its own serial; fleet MCP
                      ▼                                              ▼
              ┌───────────────┐   reads runs   ┌───────────┐  wakes bots (OpenMausBot MCP)  
              │ shared-memory │ ◀────────────▶ │  Pacer    │ ───────────────────────────────▶ device bots
              │ (registry,    │                │ (service) │ ── checks unread via android-im ─▶ phones
              │ personas,     │                └───────────┘
              │ ledger, runs) │
              └───────────────┘
```

**The split of responsibilities:**
- **Bots decide *what* and *whether*** (what to write, whether to reply, which topic, asking for a recap).
- **The pacer decides only *when* and *where*** (wakes the right bot, in the right app and chat, at human‑like times).
- **Tools do the mechanical work** (ADB, IM actions, registry bookkeeping). Bots operate tools; they don't do mechanical work token by token.

---

## 2. Decisions already made (do not reopen without the user)

| Topic | Decision |
|---|---|
| Builder vs. product | This plan is executed by Claude Code/OpenCode. OpenMausBot is where the finished bots live and run. The user talks to the Chief of Staff for daily use. |
| OpenMausBot build | The user runs the **installed (packaged) desktop app**, not dev mode. |
| Engine | Bots run on the **Claude Code engine** with GLM / GLM Flash through LiteLLM. |
| Coordinator | One persistent **Chief of Staff** in its own OpenMausBot section. |
| Device bots | **One bot per enrolled device**, bound to the device's **ADB serial** (never to a cable or USB port). Bots stay in the UI, idle when not in a run. A bot is created only when a new device is enrolled. |
| Device discovery | **Never automatic.** The user says "I connected a new device" in plain words. The Chief finds unknown devices itself (via android-core), describes them in human terms (model, number ending, apps), and adds only what the user confirms. Skipped devices go on an ignore list. The user never types a serial. |
| Which devices are touched | Only **enrolled** devices, and only **during runs the user started**. Unknown devices are invisible to the fleet. |
| Missing details | Tools first (e.g. android-core's phone‑number extraction), then the IM app's profile screen, then **ask the user** once; save the answer. |
| Persona | Created automatically at enrollment; **ask the user** if a required detail is missing. Persona is tied to the device and kept across reconnects. |
| Replaced device (e.g. after a ban) | **New persona.** Old persona, registry entry and bot are archived as "retired", not deleted. |
| Shared memory | **Local** shared folder with Markdown files, behind a **fleet MCP**. SQLite backend later if needed, without changing bots. Bitbucket maybe later. |
| Readiness | **Start with whoever is ready.** Devices that aren't ready (missing contacts, disconnected, newly enrolled) **join later** when ready. |
| Contact details | Always read from the **shared registry**, never requested live from a device that is busy chatting. |
| Late joiner in a group | Asks for a short recap like a human ("just got here, what did I miss?"); **only one member** answers; vary the behavior (sometimes it just says hi and picks up from recent messages). |
| Run length | Defined by **duration** (e.g. 10 minutes, 8 hours), not by message count. Mixed groups and durations at the same time are allowed. |
| Reset | **Standard reset** (default): removes **only contacts/groups the system created** (from the creation ledger); keeps bot memory. **Clean‑slate reset**: the same, plus clears bot memory. Both show a preview and wait for the user's OK. |
| Testing | System tests on **real devices only**, growing step by step. Unit tests are fine for plain code (fleet MCP, pacer logic). |
| Bans | Accepted risk for a legitimate test setup; the user swaps in another device. |
| Decision model (Jev/Kev) | **Not included.** Only revisit if a clear need appears. |
| Other projects | Other bot teams (e.g. automation tests) run on the same OpenMausBot. The fleet must stay isolated (see §9). |

---

## 3. What OpenMausBot gives us (verified in the source) and what we build

Verified against the OpenMausBot repository (October 2026). Re‑verify ⚠️ items on the installed version in Phase 0.

| Need | Status | Detail / source |
|---|---|---|
| Persistent coordinator | ✅ | Chief of Staff per section. Tools include `list_bots`, `create_bot`, `create_room`, `delegate_bot`, `send_to_bot`, `wait_delegation`, `propose_team_setup`, `propose_routine` (`server/drivers/agents-catalog.ts`, `server/chief-of-staff.ts`). |
| Chief creates device bots | ✅ / ⚠️ | Chief's `create_bot` accepts **name, role, instructions (soul), model, and working folder (`cwd`, must already exist)**. Max 4 new bots per turn. Connected apps and auto‑approvals start disabled. ⚠️ Check whether the user gets an approval card for each creation. |
| Scripted API writes | ❌ | On the installed app, plain HTTP writes from scripts are refused (403, "must come from the desktop app or a paired device") — `server/request-auth.ts`. Do **not** plan around raw API calls. |
| External control (for the pacer) | ✅ / ⚠️ | The app ships an **OpenMausBot MCP server** (`docs/mcp-server.md`). With a one‑time **local pairing** (Settings → Phone → set up → pairing code → exchange on `127.0.0.1`), a client can `list_bots`, `send_bot_message`, `wait_for_conversation`, `create_channel`, `update_bot_profile`, `set_bot_model`, … It **cannot** approve requests, delete data or import teams. ⚠️ Verify pairing works locally without exposing phone access on the network (the user keeps remote/companion access off at work). |
| Per‑bot MCP binding | ✅ / ⚠️ | Claude‑engine bots load **`<working folder>/.mcp.json`** and project settings (`--setting-sources project`) from their working folder (`docs/custom-mcp-servers.md`, `server/drivers/claude.ts`). ⚠️ Verify both load on the installed version. |
| Pre‑approving MCP tools | ✅ / ⚠️ | `<working folder>/.claude/settings.json` with `permissions.allow` (e.g. `mcp__<server-name>`). ⚠️ Verify exact rule format for the user's MCP names. |
| Global MCP registry | ✅ (avoid for fleet MCPs) | Globally enabled servers are offered to **every bot that hasn't narrowed its list**. So fleet MCPs must be attached per bot through `.mcp.json`, not globally. |
| Rooms / group chat between bots | ✅ | Channels/rooms with members, bulletin, default responder. |
| Memory | ✅ | Per bot: `~/.openmausbot/workspaces/<botId>/MEMORY.md` (first 200 lines / 24 KB load every turn), topic notes, archive, daily logs. Shared across **all** the bot's threads, plus a short "recent work" brief and `session_search` (`docs/memory.md`). |
| Long threads | ✅ | Claude Code compacts its own session (`--autocompact`). Continuity lives in memory, not in one thread. |
| Soul (standing instructions) | ✅ | Canonical in OpenMausBot's bot record; `SOUL.md` in `~/.openmausbot/bots/<botId>/` is a mirror — editing it creates "drift" that only applies after the user confirms. Do not edit it behind the user's back. |
| Conversation → skill | ✅ | "Draft a reusable skill from this conversation" / "Save as skill". |
| Team file | ✅ (limited) | Carries bots (name, title, description, soul), skills (`SKILL.md` text only), rooms, routines (paused), memory notes, Chief of Staff, remote MCP addresses. **Not** models, folders, approvals, or local command‑based MCPs. Useful for backup/sharing, not for wiring devices. |
| Routines | ✅ (no durations) | One‑time, interval and cron schedules. **No "stop after X"**. Run duration is handled by the pacer (🔧). |
| Shared fleet memory | 🔧 | Not built in. We build it (fleet MCP + shared folder). |
| Pacer (timing, unread detection, wake‑ups) | 🔧 | Not built in. We build it (small background service). |
| Device discovery / enrollment | 🔧 | Built from android-core tools + fleet MCP + Chief instructions. |
| OpenMausBot's own data folder | — | `~/.openmausbot`. **We never put our files there**; we only point bots at our folders. |

---

## 4. Folder layout (one home for everything)

Root location is chosen in Phase 0 (a sensible place, confirmed by the user — e.g. under the user's projects folder, **not** inside `~/.openmausbot` and not inside any bot's working folder). Placeholder below: `<fleet-root>`.

```
<fleet-root>/
├── README.md                 map of this folder (kept up to date)
├── PLAN.md                   this plan
├── fleet.config              discovered settings (paths, MCP names, tool names, model IDs, limits)
│
├── shared-memory/            the fleet's common knowledge (written only through the fleet MCP)
│   ├── registry/             one file per device: <serial>.md
│   ├── personas/             one file per persona: <serial>.md (active) 
│   ├── ledger/               creation ledger: what the system created (contacts, groups), per device
│   ├── snapshots/            pre-existing contacts/groups per device, taken at enrollment
│   ├── runs/                 active and closed runs
│   ├── ignore-list.md        devices the user said are not part of the fleet
│   └── archive/              retired devices, personas and ledgers
│
├── fleet-mcp/                the fleet MCP server (source, tests, its own README)
├── pacer/                    the pacer service (source, tests, launchd definition, its own README)
│
├── bots/
│   ├── chief-of-staff/       the Chief's working folder (.mcp.json, .claude/settings.json)
│   └── devices/
│       ├── _template/        template for device bot folders
│       └── <serial>/         one working folder per device bot, created from the template
│
├── skills/                   SKILL.md files (procedures the bots follow)
├── logs/
│   ├── calls/                call log: every tool call through the fleet, one file per day
│   ├── pacer/                pacer decisions (wake‑ups, skips, pauses)
│   └── test-runs/            results of each test stage
└── team/                     team file exports and backups
```

**Rules**
- Secrets (pairing token, LiteLLM token) are **never** stored in this tree in plain text inside shared files; keep them where `fleet.config` points (owner‑only file permissions), and never in `shared-memory/` or logs.
- Device folders are named by **serial**. Human names (persona names, phone models) appear inside the files, not in folder names.

---

## 5. Data model

For every piece of data: what it is, where it lives, when it is reused, when it is removed.

### 5.1 Persona (who the device "is") — `shared-memory/personas/<serial>.md`
- **Source of truth.** Copied into the device bot's soul when the bot is created.
- **Fields:**
  - display name (should match the name shown in the IM apps; if it doesn't, ask the user whether to adapt the persona or the app profile)
  - age range, background in one or two lines
  - personality, writing style: message length, emoji use, typos/slang, punctuation habits
  - language(s)
  - favorite topics, things they avoid
  - chattiness, typical reply speed, active hours (used by the pacer)
  - relationship hints to other personas (optional, grows over time)
- **Created:** automatically at enrollment, varied so personas across the fleet are distinct. Show a short summary to the user; the user can accept or tweak. If a required field can't be decided sensibly (e.g. language), **ask the user**.
- **Changed:** edit the persona file, then update the bot's soul. ⚠️ Verify the supported route (Chief's team‑setup/profile tools, OpenMausBot MCP `update_bot_profile` for profile fields, or the UI for the soul). Never edit `SOUL.md` silently.
- **Reused:** always; a reconnected device is the same persona.
- **Retired:** when a device is replaced, the persona moves to `archive/`. A replacement device gets a **new** persona.

### 5.2 Device identity (registry) — `shared-memory/registry/<serial>.md`
- serial, phone model, phone number, owner/profile name, installed IM apps (and which are logged in), bot ID, enrollment date
- status: `enrolled-connected` / `enrolled-disconnected` / `retired`; last seen
- in‑use marker: which run(s) the device is currently in
- **Reused:** always. Only status, last seen and in‑use change.

### 5.3 Relationships, ledger and snapshots
- **Snapshot** (`snapshots/<serial>.md`): contacts and groups the phone **already had** at enrollment, marked pre‑existing. Never deleted by any reset.
- **Ledger** (`ledger/<serial>.md`): every contact or group **the system created**: app, contact name + number (or group name + members), run ID, time, status (`active` / `removed` / `skipped-changed`).
- **Relationships view:** who has whom, per app; groups and members. Derived from snapshot + ledger.

### 5.4 Bot memory — OpenMausBot's per‑bot memory
- Short recollections: who they talked to, about what, running jokes, plans mentioned. Written by the bot through OpenMausBot's memory tools.
- **Reused** across runs for continuity ("they know each other from last time").
- **Cleared** only by the clean‑slate reset (§7.9). ⚠️ Verify the supported way to empty it (the bot emptying its own `MEMORY.md` via its memory tools, or the user via the Memory panel). `MEMORY.md` must be emptied, never deleted.

### 5.5 Runs — `shared-memory/runs/<run-id>.md`
- run ID, devices (serials), app, conversation type (one‑to‑one / group / mixed), group name, topic(s), start time, **end time**, status (`starting` / `active` / `paused` / `closed`), devices waiting to join, short summary at close.
- Closed runs stay as history.

### 5.6 Logs — `logs/`
- **Call log:** every tool call that goes through the fleet: time, caller (bot), tool, device serial, result (ok/error), short detail. One file per day.
- **Pacer log:** every wake‑up, skip (with reason), pause.
- **Test results:** per stage.
- Kept for a configurable number of days (in `fleet.config`), then deleted.

### 5.7 Never saved anywhere
Passwords, verification codes, tokens (outside their protected config location), full chat transcripts in shared memory (summaries only), screenshots — except on failures, saved under `logs/` and deleted with the logs.

---

## 6. Components

### 6.1 Chief of Staff (coordinator)
- **Where:** its own OpenMausBot section, e.g. "IM Fleet". Working folder: `bots/chief-of-staff/`.
- **Tools:** OpenMausBot's built‑in Chief tools; **fleet MCP (full)**; **android-core MCP, read‑only tools only** (device list, device details, phone number). **No android-im** — the Chief never operates IM apps itself.
- **Responsibilities:** understand the user's commands; device discovery and enrollment; persona creation; creating device bots (`create_bot` with `cwd` = the device folder); starting, monitoring and closing runs; contact‑setup coordination; resets (preview → user OK → execute through device bots); reporting.
- **Approval mode:** Ask, with fleet MCP and android-core read‑only tools pre‑approved in its folder's `.claude/settings.json`.

### 6.2 Device bots (one per enrolled device)
- **Where:** same section, named with a recognizable fleet prefix. Working folder: `bots/devices/<serial>/`, created from `_template/`.
- **Soul:** generated from the persona + the standard device‑bot rules below.
- **Tools (via its folder's `.mcp.json`):**
  - android-im and android-core, **pinned to this device's serial**. ⚠️ Discover how the user's MCPs take a device: a per‑call serial parameter, or an environment variable. If they call `adb` underneath, setting `ANDROID_SERIAL` in the server's `env` usually pins `adb` to one device — verify on the user's MCPs.
  - fleet MCP, **device mode** (read registry/personas/run; record ledger entries; write its own call‑log entries; no enrollment/reset/run control).
- **Pre‑approved** in `.claude/settings.json`: those MCP tools. Approval mode: Ask (everything else still asks).
- **Standard rules in every device bot's soul:**
  - Act **only** in the app and chat named in the wake‑up note. Never touch other apps, other chats, or non‑fleet contacts.
  - Stay in persona: style, language, pace. Not every message needs a reply; short and imperfect is human.
  - Read contact details from the fleet registry, never ask another device live.
  - Record every contact/group you create in the ledger, right after creating it.
  - On joining a group late: sometimes ask for a short recap, sometimes just say hi and pick up from the latest messages.
  - If the IM app shows a block (login wall, verification, rate limit, account warning), stop and report — don't try to "heal" through it.
  - After the turn: update memory with anything worth remembering (who, what topic), briefly.

### 6.3 Fleet MCP (🔧 build)
A small MCP server over `shared-memory/`. Markdown files now; designed so the storage can be swapped to SQLite later without changing any tool. Two modes (two `.mcp.json` entries or an argument): **coordinator** (full) and **device** (restricted).

Suggested tools (adjust names in Phase 1, document them in `fleet-mcp/README.md`):
- **Registry:** `list_fleet`, `get_device`, `find_new_devices(connected_serials)` (returns serials not enrolled and not ignored), `enroll_device`, `ignore_device`, `set_device_status`, `retire_device`
- **Personas:** `get_persona`, `save_persona`, `list_personas` (for diversity)
- **Snapshots/ledger:** `save_snapshot`, `record_created`, `list_created(serial, filters)`, `mark_removed`, `mark_skipped`
- **Runs:** `create_run`, `get_run`, `update_run`, `close_run`, `active_runs`, `add_waiting_device`
- **Contact setup queue:** `queue_contact_setup(serial, targets, pace)`, `next_setup_tasks` (used by the pacer, throttled)
- **Reset:** `preview_reset(level, devices)`, `reset_plan_for_device(serial)` (returns exactly the ledger items to remove)
- **Device lock:** `claim_device(serial, holder, timeout)`, `release_device(serial)` — one actor per phone at a time (pacer or bot)
- **Logging:** `log_call` (or automatic logging inside every tool)

Writes are atomic (no half‑written files). Concurrent writes are serialized.

### 6.4 Pacer (🔧 build)
A small background service on the Mac (macOS launchd agent, starts at login). **No model calls.**
- **Reads** active runs from the fleet MCP / `shared-memory/runs/`. **No active runs → does nothing.**
- **Doorbell:** every N seconds (configurable), for each device in an active run, asks the **android-im MCP** for unread messages **in that run's app and chats only**. Messages in other apps, other chats or from non‑fleet contacts are ignored.
  - ⚠️ Phase 0 discovers what android-im can report: per app + per chat (best); per app only (bot then checks fleet chats in that app); or nothing (fallback: read notifications via android-core/ADB, which show app and sender/group — test on the user's phones).
- **Clock:** for each device, schedules random "feels like texting" moments inside the persona's active hours and chattiness.
- **Wake‑up:** sends the device bot a precise note through the **OpenMausBot MCP server** (paired once), e.g.:
  `"[fleet wake] run #12 · WhatsApp · group 'Weekend plans' · 2 unread"` or `"[fleet wake] run #12 · WhatsApp · chat with Yossi · your move: start or continue a conversation"`
- **Setup tasks:** wakes device bots for queued contact‑setup and reset tasks at a throttled pace.
- **One actor per phone at a time:** the pacer never checks a device while its bot is awake, and never wakes two bots for the same device. Use a per‑device lock in the fleet MCP (`claim_device` / `release_device`, released automatically after a timeout). If unread checking needs to open the app's UI, it must leave the phone where it found it.
- **Limits:** concurrency cap (max bots awake at once, from `fleet.config`) to leave capacity for other projects; per‑device minimum gap.
- **End:** stops waking anyone in a run once its end time passes; tells the Chief to close the run.
- **Disconnects:** a device that disappears from the device list is marked disconnected; its wake‑ups pause; it rejoins when back.
- **Token check:** before waking bots, checks that model access works (e.g. a cheap LiteLLM call). If not (expired daily login → 401/403), **pause** active runs and notify the user through the Chief; resume when access works again.
- **Logs** every decision to `logs/pacer/`.

### 6.5 Skills (`skills/*/SKILL.md`)
Procedures the bots follow; they call MCP tools, they don't contain scripts (team files carry only `SKILL.md` text). Suggested:
- `fleet-enroll-device` (Chief)
- `fleet-create-persona` (Chief)
- `fleet-contact-setup` (device)
- `fleet-start-run` / `fleet-close-run` (Chief)
- `fleet-create-group` (device)
- `fleet-join-late` (device: recap behavior)
- `fleet-reset` (Chief: preview/approve/dispatch; device: execute own items)
- `fleet-run-report` (Chief)

⚠️ Verify how skills reach bots on the installed app: OpenMausBot's skill import, or project skills in each bot's folder (`.claude/skills/`).

---

## 7. Flows

### 7.1 Enroll a new device
1. User: "I connected a new device."
2. Chief → android-core: list connected devices. → fleet MCP `find_new_devices`.
3. For each new device, Chief → android-core: model, installed IM apps, phone number (android-core's extraction function first).
4. Chief shows the user, in human terms: "Found 1 new device: Samsung Galaxy A54, number ending …4417, WhatsApp and Telegram. Add it to the fleet?" (several devices → user picks; "skip" → `ignore_device`).
5. Missing number → device bot reads it from the IM app profile later, or **ask the user**; save it.
6. `enroll_device` → registry entry. Create the device folder from `_template/` (with the serial pinned).
7. Create persona (§7.2). Create the device bot (`create_bot` with persona soul, model, `cwd`).
8. Device bot's first task: take the **snapshot** of existing contacts/groups (`save_snapshot`), confirm IM accounts/profile names.
9. Queue contact setup with the other fleet devices only if the user asks (or as part of a run).

### 7.2 Create a persona
1. Gather facts: profile name in the IM apps, number's country (language hint), anything the user said.
2. Generate a persona distinct from existing ones (`list_personas`).
3. If a required field can't be decided (e.g. language), ask the user.
4. Show a short summary; user accepts or tweaks. `save_persona`.

### 7.3 Contact setup (throttled)
1. Chief decides which pairs need contacts (from snapshot + ledger), queues tasks.
2. Pacer releases tasks at a safe pace (configurable; never "all 50 at once").
3. Device bot adds the contact via android-im using registry data, then `record_created` immediately.
4. Device becomes "ready" for runs with those devices.

### 7.4 Start a run
1. User: e.g. "WhatsApp group with devices 1, 2, 3 for 10 minutes" or "the Samsung and the Pixel chat on Telegram all day."
2. Chief resolves devices (by persona name, model, or "all"), checks readiness (connected, app installed, contacts).
3. `create_run` with end time. **Start with who's ready**; not‑ready devices are added as "waiting" and their setup is queued.
4. For a group: the device bot chosen as creator creates it (`fleet-create-group`), records it in the ledger.
5. Pacer takes over timing.

### 7.5 During a run
Pacer wakes bot → bot reads the named chat → decides (reply / wait / new topic / ignore) → acts in persona → updates memory briefly → turn ends. Repeat until end time.

### 7.6 Late join (waiting device, newly ready, or reconnected)
Chief adds it to the group/chat when ready. Its first wake‑up: sometimes "just got here, what did I miss?" — the pacer wakes **one** member to answer with a short recap — sometimes it just says hi and picks up.

### 7.7 Disconnect / reconnect
Identity is the **serial**. Disconnected → status updated, wake‑ups for it pause, run continues for others. Reconnected → same bot, same persona, same memory; rejoins its active run. A different device on the same cable is a different serial: if not enrolled, it's ignored.

### 7.8 End of run
Pacer stops at end time → Chief closes the run, writes a short summary (who talked, message counts from the call log, problems) and reports to the user.

### 7.9 Reset
- **Standard (default):** for each selected device, `preview_reset` lists only **ledger** items (contacts/groups the system created). Pre‑existing items (snapshot) are never included. Chief shows the preview → user OK → device bots remove their own items → before each removal, verify it still matches the ledger; if changed, **skip and report** → `mark_removed`. Bot memory kept.
- **Clean‑slate:** standard reset + each device bot empties its memory (⚠️ supported route, §5.4).
- Groups: leave/delete only ledger groups, as the app allows.

### 7.10 Retire / replace a device
Chief marks it retired: registry, persona and ledger go to `archive/`; the bot is moved to a "Retired" section (or removed by the user in the UI — the OpenMausBot MCP cannot delete). A replacement phone is enrolled as a new device with a **new persona**.

---

## 8. Configuration — `fleet.config` (filled in Phase 0)
- fleet root path; shared‑memory path
- android-core MCP: name, how it's configured, tool names for: list devices, device details, phone number
- android-im MCP: name, tool names for: send, read chat, list unread (and at what granularity), add/remove contact, create/leave group; how a device serial is passed
- OpenMausBot: install path, port, pairing status for the MCP server (token location, not the token itself)
- models: exact GLM and GLM Flash IDs as OpenMausBot lists them
- LiteLLM address; how the daily token is obtained by Claude Code (for the token check)
- limits: pacer check interval, concurrency cap, min gap per device, contact‑setup pace, log retention days
- section name and bot name prefix

---

## 9. Living with other projects on the same OpenMausBot
- Fleet bots, rooms and routines live in **their own section** with their own Chief; a Chief only manages its own section.
- **Fleet MCPs are attached per bot via `.mcp.json`**, never enabled globally (globally enabled servers reach every bot that hasn't narrowed its list).
- Only enrolled devices are touched; other phones connected for other projects are invisible to the fleet.
- Concurrency cap in the pacer leaves model/GPU capacity for other projects.
- Everything stays under `<fleet-root>`.

---

## 10. Phases and stage gates

Each phase ends with a demo to the user and their OK.

### Phase 0 — Discovery and verification (no building)
1. Inspect the Mac: OpenMausBot install and version, `~/.openmausbot` location (read only), Claude Code/OpenCode setup, LiteLLM access.
2. Find the two MCPs by role; list their tools and parameters; how they target a device; confirm with the user.
3. Test with one connected phone: android-core device list, device details, **phone‑number extraction**; android-im unread reporting granularity.
4. Verify ⚠️ items from §3: Chief `create_bot` with `cwd` (and whether it asks for approval); bots loading `.mcp.json` and `.claude/settings.json` from their folder; tool pre‑approval format; skills route; OpenMausBot MCP pairing locally (without network exposure); how a soul update is applied; how to empty bot memory.
5. Propose `<fleet-root>` location; user confirms. Write `fleet.config` and `README.md`.
**Gate:** user reviews `fleet.config` and the verification results.

### Phase 1 — Foundation
Folder tree; fleet MCP (both modes) with unit tests; call log; `_template/` device folder; `README.md` updated.
**Gate:** fleet MCP tools demonstrated on sample data, then removed.

### Phase 2 — Chief of Staff + enrollment (1 real device)
Chief bot (section, soul, folder, tools). Flows 7.1 and 7.2 on one phone, including snapshot. Ignore list works with a second, non‑fleet phone.
**Gate:** user says "I connected a new device" and the device is enrolled correctly; the non‑fleet phone is untouched.

### Phase 3 — Milestone 1: two devices, one‑to‑one, 10 minutes
Second device enrolled. Contact setup with ledger. Pacer (doorbell + clock + end time + token check + concurrency cap) running as a service. One‑to‑one chat on one app for 10 minutes. Standard reset at the end.
**Gate:** natural‑looking 10‑minute chat; the call log shows only the right devices/apps/chats were touched; reset removed exactly the created contacts.

### Phase 4 — Groups and joining
Three devices: group creation, "start with who's ready", late joiner with recap, one device disconnected and reconnected mid‑run.
**Gate:** all three end up in the group; the reconnected device resumes as the same persona.

### Phase 5 — Full local fleet
All 4–5 devices; two apps at once; different durations for different groups; a long run (e.g. 1–2 hours, then a full day); clean‑slate reset; retire/replace flow.
**Gate:** stable long run; reports make sense; no actions outside run scope.

### Phase 6 — Ready for 25–50
Review throttling, concurrency cap and pacer intervals at scale; batch enrollment (Chief creates max 4 bots per turn); decide whether to switch shared memory to SQLite behind the fleet MCP.
**Gate:** user decides on scale‑up.

### Later (optional, only with the user's OK)
Turn recurring procedures into skills ("Save as skill"); team file export for backup; Bitbucket for `<fleet-root>`.

---

## 11. Testing approach
- **Plain code** (fleet MCP, pacer logic): unit tests.
- **System behavior:** on **real devices only**, phase by phase as above.
- **Assertions on outcomes, not exact text:** right devices/apps/chats in the call log; runs stop at end time; ledger matches what's on the phones; resets remove only ledger items; nothing touches non‑fleet devices or chats.
- **Every test run** writes a short result to `logs/test-runs/` and is followed by a standard reset so the next run starts from the same state.
- Model access (the daily LiteLLM login) must be valid during tests that involve bots.

---

## 12. Risks and mitigations
| Risk | Mitigation |
|---|---|
| IM UI changes / brittle automation | Existing self‑healing in android-im (structure first, GLM Flash vision on failure). Cap retries; report blocks instead of healing through them. |
| Account flagged/banned | Accepted; human‑like pacing; replace device → new persona. |
| Conversation quality drifts over long runs | Short turns, persona in soul, memory for continuity, recap behavior; review transcripts in Phase 5. |
| Wrong device acted on | Serial pinning per bot folder; call log; device bots act only on the chat in the wake note. |
| Expired daily token mid‑run | Pacer token check → pause and notify → resume. |
| Load on shared models/GPUs | Concurrency cap; per‑device minimum gaps. |
| Unread detection not possible via android-im | Notification fallback via ADB (Phase 0 test). |
| OpenMausBot behavior differs from source | Phase 0 verification of every ⚠️ item before building on it. |

---

## 13. Checklist for the user (collected during Phase 0)
- [ ] Confirm the fleet root location
- [ ] Confirm which MCPs are "android-core" and "android-im"
- [ ] Pair the OpenMausBot MCP server once (Claude Code will guide)
- [ ] Confirm GLM / GLM Flash model choices for Chief and device bots
- [ ] Provide any phone number or persona detail the tools couldn't find
- [ ] Approve each stage gate
