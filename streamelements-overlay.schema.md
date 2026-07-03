# StreamElements Overlay Export Schema

File: `ong-streamelements-mainoverlay.json`
Encoding: UTF-8 (no BOM — unlike the Streamer.bot files)
Companion JSON Schema: `streamelements-overlay.schema.json`
Source: reverse-engineered from a single export — partially unverified

The file is the bundle the StreamElements overlay editor/API delivers: the overlay
itself (widget definitions), plus the channel it belongs to, a session auth token,
the live event counters that feed the widgets, and tipping configuration.

## Root Object

| Field | Type | Description |
|---|---|---|
| `channel` | Channel | The StreamElements channel (account) owning the overlay |
| `overlay` | Overlay | The overlay definition: canvas + widgets |
| `passport` | string | JWT session token — **SECRET**, do not publish |
| `session` | Session | Live/lifetime event counters (the widgets' data feeds) |
| `tipping` | Tipping | Tip currency and profanity/TTS filter config |

## Channel

| Field | Type | Description |
|---|---|---|
| `_id` | ObjectId (24 hex) | SE channel id; referenced by `overlay.channel` |
| `ab` | {`name`, `value`: string}[] | A/B-test / feature-flag assignments |
| `alias` | string | Lowercased channel name |
| `apiToken` | string | Overlay access token — **SECRET**; last path segment of overlay browser-source URLs |
| `avatar` | URL | |
| `broadcasterType` | string | e.g. `partner` |
| `createdAt` | ISO 8601 | |
| `displayName` | string | |
| `provider` | string | Platform, e.g. `twitch` |
| `providerId` | string | Platform-native id (numeric string for Twitch) |
| `username` | string | |

## Overlay

Loaded in OBS as a browser source at
`https://streamelements.com/overlay/<overlay._id>/<channel.apiToken>`.

| Field | Type | Description |
|---|---|---|
| `_id` | ObjectId | Overlay id (first path segment of the overlay URL) |
| `channel` | ObjectId | Owning channel (= `channel._id`) |
| `name` | string | Dashboard display name |
| `type` | string | `regular` |
| `campaign`, `favorite`, `mobile` | bool | |
| `preview` | URL | Thumbnail |
| `createdAt`, `updatedAt` | ISO 8601 | |
| `settings` | object | Canvas: `width`: int, `height`: int, `name`: string (e.g. `1080p`) |
| `widgets` | Widget[] | |

## Widget — base fields (all types)

| Field | Type | Description |
|---|---|---|
| `id` | int | Unique within the overlay |
| `type` | string (enum) | Widget type — see below |
| `name` | string \| null | Editor layer-list name |
| `group` | string \| null | If set: member of the `se-widget-group` whose `variables.uid` matches |
| `version` | number | Widget schema version (1, 2.2, …) |
| `visible`, `locked` | bool | |
| `provider` | string | `twitch` |
| `css` | object | Canvas placement: `width`/`height`/`top`/`left` (number px or CSS string), `z-index`: int, `opacity`: 0–1 |
| `text`, `image`, `video`, `audio` | object | Default element styling blocks (largely identical boilerplate on every widget; `text.value` observed always null) |
| `variables` | object | **Type-specific configuration** — see below |

### Widget types and type-specific fields

| Type | Specific fields | Meaning |
|---|---|---|
| `se-widget-alert-box` | `listeners`, `variables` = AlertBoxVariables | An alert box rendering one or more event feeds |
| `se-widget-group` | `variables` = {`expanded`: bool, `uid`: string} | Editor folder; member widgets reference `uid` via their `group` field |
| `se-widget-subscriber-goal` | `listener`: string (e.g. `subscriber-goal`), `variables` = GoalVariables | Goal progress bar (disabled in this export) |

Other SE widget types certainly exist; only these three are observed.

## `listeners` (alert box only)

Map of event feed → bool: `follower-latest`, `subscriber-latest`, `tip-latest`,
`cheer-latest`, `host-latest`, `raid-latest`, `purchase-latest`.

**This is the master gate.** Every alert box in this export has *all* event types
configured and `enabled: true` under `variables`, but only renders the feeds set
to `true` here. That's how one overlay hosts several specialized boxes
(cheers-only, tips-only, …) without double-firing.

## AlertBoxVariables

| Field | Type |
|---|---|
| `animation` | Animation (widget-level default in/out) |
| `follower`, `subscriber`, `tip`, `cheer`, `host`, `raid`, `merch`, `purchase`, `charityCampaignDonation` | AlertEventConfig (one per event type) |

## AlertEventConfig

Configuration for one event type. **The same shape reappears as `settings` inside
each Variation**, where it overrides this base config for matching events.

| Field | Type | Description |
|---|---|---|
| `enabled` | bool | |
| `duration` | number | Seconds on screen |
| `layout` | string | `column` = text below graphic, `row` = beside, `behind` = text over graphic |
| `minAmount` | number | Minimum amount for the event type to fire at all (bits, currency, viewers…) |
| `showMessage` | bool | Show the user's attached message |
| `enableRandomPick` | bool | When several variations match, pick randomly (weighted by `chance`) instead of first-match |
| `animation` | Animation | Container in/out |
| `audio` | {`src`: URL \| null, `volume`: 0–1} | Alert sound |
| `graphics` | {`type`: `image`\|`video`, `src`: URL \| null, `volume`: 0–1} | Alert visual |
| `text` | AlertText | Headline text |
| `tts` | {`enabled`: bool, `voice`: string, `volume`: 0–1, `minAmount`: number} | Text-to-speech of the message |
| `variations` | Variation[] | Amount-matched overrides — the per-amount custom alert library lives here |
| `enableCustomCss` | bool | |
| `css`, `html`, `js` | string | Custom code for the alert |
| `fields` | string (JSON) | Custom-code editor field definitions (SE boilerplate here) |
| `fieldData` | object | Values for those fields |

### Animation

`in` / `out` (animate.css names: `fadeIn`, `zoomIn`, `slideInLeft`, `none`, …),
`inDuration` / `outDuration` (seconds), optional `text` sub-object with its own
`in`/`out`/`inDuration`/`outDuration` plus `delay` (seconds before text enters)
and `offset` (seconds before the end at which text exits).

### AlertText

| Field | Type | Description |
|---|---|---|
| `message` | string | Template: `{name}`, `{amount}`, `{sender}` placeholders (e.g. `{name} cheered x{amount}`) |
| `animation` | string | Looping emphasis: `pulse`, `bounce`, `none` |
| `enableShadow`, `enableCustomFont`, `enableCustomFontSecondary` | bool | |
| `numberFormat` | string | e.g. `en-EN` |
| `css` | object | CSS-ish bag: `font-*`, `color`, `text-shadow`, `-webkit-text-stroke-*`, margins; nested `highlights` (color of `{name}`/`{amount}` spans) and `message` (styling of the user's message line) |

## Variation

A conditional override of the event's base alert.

| Field | Type | Description |
|---|---|---|
| `name` | string | Editor display name |
| `enabled` | bool | |
| `type` | string | What `requirement` measures: `amount` (bits / currency / months / viewers), `gift` (single gifted sub), `communityGift` (gift-bomb count) |
| `condition` | string | `EXACT` or `ATLEAST` |
| `requirement` | number | The threshold/exact amount |
| `chance` | number 0–100 | Random-pick weight when several variations match and `enableRandomPick` is on |
| `layout` | string | Same values as AlertEventConfig |
| `settings` | AlertEventConfig | Full alert config used when this variation fires (nested `variations` observed always empty) |

Naming convention in this export: `"<amount> <alert name>"`, with markers like
`[disabled for BPM]` on amounts that were migrated to the local OBS alert system
and `[DO NOT ENABLE]` on parked entries. The exact-amount cheer/tip alert
library (~150 entries on the CHEERS box, ~60 on CASH TIPS) lives entirely in
these variation lists.

## GoalVariables (goal widgets)

| Field | Type | Description |
|---|---|---|
| `title` | string | Bar label |
| `goalAmount` | number | |
| `goalListener` | string | Session counter feeding the bar (e.g. `subscriber-goal`) |
| `endDate` | ISO 8601 | |
| `percent` | bool | Show percent instead of count |
| `simpleDesign` | bool | |
| `backgroundColor`, `fillColor` | CSS color | |

## Session

Live/lifetime counters, keyed `<eventType>-<scope>`. Event types observed:
`follower`, `subscriber`, `cheer`, `tip`, `host`, `raid`, `merch`, `purchase`,
`superchat`, `cheerPurchase`, `charityCampaignDonation`, `hypetrain`,
`channel-points`, `community-gift`.

| Key pattern | Shape |
|---|---|
| `*-latest` | {`name`, `amount`?, `message`?, plus specials below} |
| `*-recent` | array of {`name`, `createdAt`, `amount`?, `tier`?}, newest first |
| `*-session`, `*-week`, `*-month`, `*-total` | {`amount`: number} for money-like types, {`count`: int} for follower/subscriber tallies |
| `*-goal`, `*-count` | {`amount`} / {`count`} |
| `*-{session,weekly,monthly,alltime}-top-{donation,donator}` | {`name`, `amount`} |

Specials: `subscriber-latest`/`subscriber-recent[]` add `tier` (`"1000"`, `"2000"`,
`"3000"`, `"prime"`); `subscriber-gifted-latest` adds `sender` + `tier`;
`community-gift-latest` adds `tier`; `channel-points-latest` adds `redemption`
(reward name); `hypetrain-*` use `active`/`level`/`levelChanged`/`type`/`percent`;
`merch-latest`/`purchase-latest` add `items[]` (and `avatar`).

## Tipping

| Field | Type | Description |
|---|---|---|
| `currency` | {`code`, `name`, `symbol`} | ISO 4217, e.g. USD |
| `profanity` | {`default`: bool, `filter`: bool, `mode`: int, `custom`: string[], `replace`: string[]} | Message profanity filtering (mode enum unverified) |
| `ttsFilter` | {`enabled`: bool} | |

## Key structural notes

- **`listeners` gates, `enabled` doesn't**: per-event `enabled` flags are true
  everywhere; whether a box fires for an event type is decided by its
  `listeners` map. Check `listeners` first when asking "which box renders X?"
- Variations and their parent event config share one shape (AlertEventConfig);
  a matching variation's `settings` wholesale replaces the base for that event.
- `channel.apiToken` and `passport` are credentials; the whole file should be
  treated as sensitive.
- The `session` section is a point-in-time snapshot of server-side counters —
  useful context, but it changes constantly and is not configuration.
- `text`/`image`/`video`/`audio` blocks on each widget are editor boilerplate;
  the interesting per-alert styling lives under `variables.<event>` and the
  variations.
