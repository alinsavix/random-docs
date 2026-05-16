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
| 102 | `min`, `max` | Twitch: subscription (range) |
| 103 | `tiers` | Twitch: sub by tier |
| 104 | `tiers`, `min`, `max`, `subType` | Twitch: sub (detailed) |
| 105 | `tiers`, `min`, `max`, `subType`, `monthsGifted` | Twitch: gift sub |
| 106 | `tiers`, `min`, `max`, `subType` | Twitch: resub? |
| 107 | `min`, `max` | Twitch: raid |
| 108 | — | |
| 111 | — | |
| 112 | `rewardId` | Twitch: channel point redeem (specific reward) |
| 139 | — | |
| 186 | `minutes` | Timer |
| 194 | `rewardType` | Twitch: channel point redeem (by type) |
| 201 | `min`, `max` | Twitch: Bits/cheer |
| 401 | `commandId` | Command |
| 479 | — | |
| 702 | `variables: {string: string}` | Streamer.Bot: startup (with seeded variables) |
| 706 | — | Streamer.Bot: startup |
| 709 | `variableName`, `persisted` | Variable watcher (set) |
| 711 | `variableName`, `persisted` | Variable watcher (clear/change) |
| 1201 | `min`, `max` | |
| 4006 | `min`, `max` | YouTube event |
| 4007 | `min`, `max` | YouTube event |
| 4018 | — | |
| 7001 | `min`, `max` | |
| 10001 | `min`, `max` | |
| 14003 | `eventName`, `obsId` | OBS event |
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
| 30 | OBS: Set Source Visibility | `sceneName`, `sourceName`, `state`: int, `connectionId`: UUID |
| 38 | OBS: Set Browser Source URL | `sceneName`, `sourceName`, `url`, `connectionId`: UUID |
| 46 | OBS: Raw WebSocket Request | `name`, `prefix`, `raw`: JSON string, `resultsToArgs`: bool, `connectionId`: UUID |
| 50 | Twitch: Set Present Viewer | `userLogin` |
| 120 | If / Condition | `input`, `operation`: int, `value`, `autoType`: bool, `subActions`: SubAction[] |
| 121 | Get Variable | `source`: int, `variableName`, `persisted`: bool, `destinationVariable`, `defaultValue` |
| 122 | Set Variable | `destination`: int, `variableName`, `persisted`: bool, `source`: int, `value`, `autoType`: bool |
| 123 | Set Argument | `variableName`, `value`, `autoType`: bool |
| 124 | (Break / Stop?) | — |
| 127 | Switch on Input | `input`, `autoType`: bool, `subActions`: SubAction[] |
| 324 | OBS: Source action | `sceneName`, `sourceName`, `connectionId`: UUID |
| 560 | Twitch: Send Chat Message | `text`, `useBot`: bool, `fallback`: bool, `replyId` |
| 1002 | Sleep / Delay | `value`, `maxValue`: int\|null, `random`: bool |
| 1009 | Comment / Note | `value` (text), `color` |
| 1020 | Log Message | `logLevel`: int, `message` |
| 1024 | Toast Notification | `toastId`, `title`, `text`, `attribution`, `iconPath` |
| 1031 | Signal: Send | `signalName`, `useArge`: bool, `customArgs`: object, `queueSignal`: bool |
| 1032 | Signal: Wait | `signalName`, `overwrite`: bool, `timeout`: int\|null |
| 1034 | Trigger Custom Event | `eventName`, `useArgs`: bool |
| 99900 | Group (named container) | `name`, `color`: string\|null, `random`: bool, `subActions`: SubAction[] |
| 99901 | If — Then block | `random`: bool, `subActions`: SubAction[] |
| 99902 | If — Else block | `random`: bool, `subActions`: SubAction[] |
| 99903 | Switch — Case block | `caseSensitive`: bool, `values`: string[], `random`: bool, `subActions`: SubAction[] |
| 99904 | Switch — Default block | `random`: bool, `subActions`: SubAction[] |
| 99998 | Execute Compiled C# | `executeCodeId`: UUID, `method`, `runOnUiThread`: bool, `saveResultToVariable`: bool, `saveToVariable`: string\|null |
| 99999 | Inline C# Code | `name`, `description`: string\|null, `references`: string[], `byteCode`: base64, `precompile`: bool, `delayStart`: bool, `saveResultToVariable`: bool, `saveToVariable`: string\|null |

## Key structural notes

- `subActions` form a tree via `parentId` — top-level items have `parentId: null`
- Control-flow types (120, 127, 99900–99904) embed children in `subActions` AND those children have `parentId` set (dual representation)
- `%variable%` syntax used throughout for runtime variable substitution
- `byteCode` in type 99999 is base64-encoded C# source
