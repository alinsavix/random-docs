# Streamer.Bot actions.json Schema

File: `C:\OBS\streamer.bot-1.0.4-dev\data\actions.json`
Encoding: UTF-8 with BOM (open with `utf-8-sig`)
Format version: 23

## Root Object

| Field | Type | Description |
|---|---|---|
| `version` | int | File format version (currently 23) |
| `t` | ISO 8601 string | Last-saved timestamp |
| `blocking` | bool | Default queue blocking mode |
| `groups` | string[] | Ordered list of action group names |
| `collapsedGroups` | string[] | Group names collapsed in the UI |
| `queues` | Queue[] | Named execution queues |
| `actions` | Action[] | All actions |

## Queue

| Field | Type |
|---|---|
| `id` | UUID |
| `name` | string |
| `blocking` | bool |

## Action

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | |
| `name` | string | |
| `group` | string | Must be in root `groups` |
| `queue` | UUID | References a queue `id` |
| `enabled` | bool | |
| `alwaysRun` | bool | |
| `randomAction` | bool | Pick one subAction at random |
| `concurrent` | bool | |
| `excludeFromHistory` | bool | |
| `excludeFromPending` | bool | |
| `triggers` | Trigger[] | |
| `subActions` | SubAction[] | |
| `collapsedGroups` | string[]? | SubAction group IDs collapsed in the UI |

## Trigger — base fields

| Field | Type |
|---|---|
| `id` | UUID |
| `type` | int (enum) |
| `enabled` | bool |
| `exclusions` | array (always `[]` in observed data) |

### Trigger types and extra fields

| Type | Extra fields | Likely meaning |
|---|---|---|
| 101 | — | Twitch: chat message |
| 102 | `min`, `max` | Twitch: cheer — bits amount range (populates `%bits%`) |
| 103 | `tiers` | Twitch: sub by tier |
| 104 | `tiers`, `min`, `max`, `subType` | Twitch: sub (detailed) |
| 105 | `tiers`, `min`, `max`, `subType`, `monthsGifted` | Twitch: gift sub |
| 106 | `tiers`, `min`, `max`, `subType` | Twitch: gift bomb — community gifts; `min`/`max` = gift count (populates `%gifts%`) |
| 107 | `min`, `max` | Twitch: raid |
| 108 | — | |
| 111 | — | |
| 112 | `rewardId` | Twitch: channel point redeem (specific reward) |
| 139 | — | |
| 186 | `minutes` | Timer |
| 194 | `rewardType` | Twitch: channel point redeem (by type) |
| 201 | `min`, `max` | Donation/tip — currency amount (fractional values like 7.99 observed, so not bits) |
| 401 | `commandId` | Command |
| 479 | — | |
| 701 | `timerId` | Timed action tick (fires the named timer from settings.json `timedActions`) |
| 702 | `variables: {string: string}` | Streamer.Bot: startup (with seeded variables) |
| 706 | — | Streamer.Bot: startup |
| 709 | `variableName`, `persisted` | Variable watcher (set) |
| 711 | `variableName`, `persisted` | Variable watcher (clear/change) |
| 1201 | `min`, `max` | Donation/tip from a second platform — currency amount (always paired with a matching type 201 trigger in observed data) |
| 4006 | `min`, `max` | YouTube event |
| 4007 | `min`, `max` | YouTube event |
| 4018 | — | |
| 7001 | `min`, `max` | |
| 10001 | `min`, `max` | |
| 14003 | `eventName`, `obsId` | OBS event |
| 14004 | `sceneName`, `obsId`: UUID\|null | OBS: scene changed (provides `%obs.sceneName%`; null `obsId` = default connection) |
| 14005 | `obsId`: UUID\|null | OBS: connected? (observed used for run-on-OBS-connect startup work) |
| 14009 | `vendorName`, `eventName`, `obsId` | OBS vendor event |
| 18002 | `name`, `eventName` | WebSocket event |
| 24006 | `min`, `max` | |
| 28003 | `min`, `max` | |

## SubAction — base fields (all types)

| Field | Type |
|---|---|
| `id` | UUID |
| `type` | int (enum) |
| `enabled` | bool |
| `index` | int (display order) |
| `weight` | float (for random selection) |
| `parentId` | UUID \| null (null = top-level) |

### SubAction types and extra fields

| Type | Name | Extra fields |
|---|---|---|
| 4 | Run Action | `actionId`: UUID, `runImmedately`: bool |
| 21 | (unknown, no-op?) | — |
| 30 | OBS: Set Source Visibility | `sceneName`, `sourceName`, `state`: int (0 = visible, 1 = hidden; only these observed), `connectionId`: UUID |
| 35 | OBS: Set Source Filter State | `sceneName`, `sourceName`, `filterName`, `state`: int (0 = enabled, 1 = disabled), `connectionId`: UUID |
| 38 | OBS: Set Browser Source URL | `sceneName`, `sourceName`, `url`, `connectionId`: UUID |
| 39 | OBS: Set GDI Text | `sceneName`, `sourceName`, `text`, `connectionId`: UUID |
| 46 | OBS: Raw WebSocket Request | `name`, `prefix`, `raw`: JSON string, `resultsToArgs`: bool, `connectionId`: UUID |
| 50 | Twitch: Set Present Viewer | `userLogin` |
| 120 | If / Condition | `input`, `operation`: int, `value`, `autoType`: bool, `subActions`: SubAction[] |
| 121 | Get Variable | `source`: int, `variableName`, `persisted`: bool, `destinationVariable`, `defaultValue` |
| 122 | Set Variable | `destination`: int, `variableName`, `persisted`: bool, `source`: int, `value`, `autoType`: bool |
| 123 | Set Argument | `variableName`, `value`, `autoType`: bool |
| 124 | (Break / Stop?) | — |
| 127 | Switch on Input | `input`, `autoType`: bool, `subActions`: SubAction[] |
| 128 | If / Condition (variant of 120 with case option) | `input`, `operation`: int, `value`, `autoType`: bool, `ignoreCase`: bool, `subActions`: SubAction[] |
| 324 | OBS: Source action | `sceneName`, `sourceName`, `connectionId`: UUID |
| 326 | OBS: random scene item? (exact semantics unverified) | `sceneName`, `connectionId`: UUID |
| 560 | Twitch: Send Chat Message | `text`, `useBot`: bool, `fallback`: bool, `replyId` |
| 1001 | Action Queue: pause/resume/clear | `actionQueueId`: UUID, `state`: int, `clear`: bool |
| 1002 | Sleep / Delay | `value`: string (ms), `maxValue`: string\|null (upper bound when `random` is true), `random`: bool |
| 1008 | Color picker → variable | `red`, `green`, `blue`, `alpha`: int, `random`: bool, `variableName` (exposes sub-fields like `%<var>.html%`) |
| 1009 | Comment / Note | `value` (text), `color` |
| 1020 | Log Message | `logLevel`: int, `message` |
| 1021 | Read file → variable | `file`, `count`, `variableName`, `parseVariables`: bool, `autoType`: bool (also sets `%fileFound%`) |
| 1024 | Toast Notification | `toastId`, `title`, `text`, `attribution`: string\|null, `iconPath` |
| 1031 | Signal: Send | `signalName`, `useArge`: bool, `customArgs`: object, `queueSignal`: bool |
| 1032 | Signal: Wait | `signalName`, `overwrite`: bool, `timeout`: int\|null |
| 1034 | Trigger Custom Event | `eventName`, `useArgs`: bool |
| 99900 | Group (named container) | `name`, `color`: string\|null, `random`: bool, `subActions`: SubAction[] |
| 99901 | If — Then block | `random`: bool, `subActions`: SubAction[] |
| 99902 | If — Else block | `random`: bool, `subActions`: SubAction[] |
| 99903 | Switch — Case block | `caseSensitive`: bool, `values`: string[], `random`: bool, `subActions`: SubAction[] |
| 99904 | Switch — Default block | `random`: bool, `subActions`: SubAction[] |
| 99998 | Execute Compiled C# | `executeCodeId`: UUID, `method`, `runOnUiThread`: bool, `saveResultToVariable`: bool, `saveToVariable`: string\|null |
| 99999 | Inline C# Code | `name`: string\|null, `description`: string\|null, `references`: string[], `byteCode`: base64, `precompile`: bool, `delayStart`: bool, `saveResultToVariable`: bool, `saveToVariable`: string\|null |

## Key structural notes

- `subActions` form a tree: control-flow types (120, 127, 128, 99900–99904) embed
  their children in their own nested `subActions` array. Children carry `parentId`
  (pointing at the container) but do **not** also appear in the action's top-level
  `subActions` list — every top-level entry has `parentId: null`.
- `%variable%` syntax used throughout for runtime variable substitution
- `byteCode` in type 99999 is base64-encoded C# source

## Corrections (2026-07, from cross-referencing Jon's configs)

- Type 102 was previously labeled "subscription (range)" but is the **cheer**
  trigger: actions triggered by it switch on `%bits%`, and its exact amounts
  match the StreamElements cheer-variation amounts.
- Type 106 was previously "resub?" but is the **gift bomb** trigger: both
  observed uses are gift-sub actions, one reading `%gifts%`.
- Type 201 was previously "Bits/cheer" but carries fractional `min`/`max`
  (dollar values); it and 1201 are the two donation/tip platform triggers
  (which platform is which is still unverified).
- State polarity is consistent across OBS sub-actions: **0 = on/visible/enabled,
  1 = off/hidden/disabled** (verified for types 30 and 35).
- The earlier "dual representation" note was wrong: nested children exist only
  inside their container's `subActions`, never duplicated at the top level.
- Newly documented from observed data: trigger types 701, 14004, 14005 and
  sub-action types 35, 39, 128, 326, 1001, 1008, 1021 (see tables above);
  type 99999 `name` can be null.
- Trigger `tiers` is an integer bitmask (16 in all observed data), not an
  array as previously assumed; trigger `obsId` can be null (= default OBS
  connection).
