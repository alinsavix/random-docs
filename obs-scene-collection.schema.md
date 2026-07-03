# OBS Studio Scene Collection Schema

File: `<obs install>/config/obs-studio/basic/scenes/<CollectionName>.json`
(portable installs here, e.g. `C:/obs/OBS 30.1.2-1/config/obs-studio/basic/scenes/`)
Encoding: UTF-8 (no BOM)
Companion JSON Schema: `obs-scene-collection.schema.json`
Source: reverse-engineered / partially unverified; validated against the three
collections in `OBS/` (main, projector, Bechstein)

## Root Object

| Field | Type | Description |
|---|---|---|
| `name` | string | Scene collection name |
| `current_scene` | string | Active preview scene |
| `current_program_scene` | string | Active program (output) scene |
| `current_transition` | string | Selected transition name |
| `transition_duration` | int | Default transition duration (ms) |
| `preview_locked` | bool | |
| `scaling_enabled`, `scaling_level`, `scaling_off_x`, `scaling_off_y` | bool/int/number | Preview canvas zoom/pan state |
| `scene_order` | {`name`: string}[] | Scene panel order — includes separator pseudo-scenes (e.g. `-----Main Scenes-----`) |
| `sources` | Source[] | **All** sources *and* scenes (scenes are sources with `id: "scene"`) |
| `groups` | Source[] | Group sources (empty in observed data) |
| `transitions` | Transition[] | Custom transition definitions (built-in Cut/Fade are not stored) |
| `quick_transitions` | QuickTransition[] | Quick-transition buttons |
| `saved_projectors` | object[] | Saved projector window configs |
| `virtual-camera` | object | `type2`: int — 0 = program, 1 = preview, 2 = scene, 3 = source |
| `modules` | object | Per-plugin persistent settings (see Modules) |
| `DesktopAudioDevice1`/`2`, `AuxAudioDevice1`–`4` | Source | **Global audio devices** (Settings → Audio) stored at the collection root, not in `sources[]` — full Source objects (wasapi captures) |

## Source — common fields (sources, scenes, and filters)

| Field | Type | Description |
|---|---|---|
| `prev_ver` | int | OBS version integer that last wrote this source |
| `name` | string | Display name (unique per collection) |
| `uuid` | UUID | Stable id; scene items reference sources by this |
| `id` | string | Source type (see table below) |
| `versioned_id` | string | Type id incl. version suffix (e.g. `color_source_v3`) |
| `settings` | object | Type-specific settings (see per-type tables) |
| `mixers` | int | Audio mixer track bitmask (255 = all, 0 = none) |
| `sync` | int | Audio sync offset (ns) |
| `volume` | number 0–1 | |
| `balance` | number 0–1 | 0.5 = center |
| `enabled`, `muted` | bool | |
| `push-to-mute`, `push-to-talk` (+ `-delay`: ms) | bool/int | |
| `monitoring_type` | int | 0 = none, 1 = monitor only, 2 = monitor and output |
| `deinterlace_mode` | int | 0 = disable, 1 = discard, 2 = blend, 3 = blend-2x, 4 = linear, 5 = linear-2x, 6 = yadif, 7 = yadif-2x |
| `deinterlace_field_order` | int | 0 = top first, 1 = bottom first |
| `flags` | int | Source flags bitmask |
| `hotkeys` | object | Keyed by action name (e.g. `libobs.show_scene_item.<item id>`) → array of key combos |
| `private_settings` | object | Internal/plugin data |
| `filters` | Filter[] | Filters in order (Filter = same common fields, `id` = filter type) |

### Source type ids observed in this setup

| `id` | Meaning |
|---|---|
| `scene` | A scene (settings hold the item list) |
| `ffmpeg_source` | Media source (the workhorse: all alert/animation videos) |
| `image_source`, `color_source`, `text_gdiplus`, `text_ft2_source` | Static image / color / text (GDI+ and FreeType) |
| `browser_source` | Web overlay |
| `vlc_source` | VLC playlist (used for HLS network streams + music playlists) |
| `dshow_input` | DirectShow capture (VR-4HD, Cam Link) |
| `decklink-input` | Blackmagic DeckLink capture |
| `wasapi_input_capture`, `wasapi_output_capture` | Audio device / audio output capture |
| `window_capture`, `monitor_capture`, `game_capture` | Screen captures (game_capture: Keysight on the Bechstein PC) |
| `teleport-source` | Teleport plugin network video (settings: `teleport_list`, e.g. `"bechstein:192.168.1.155"`) |
| `ndi_source` | NDI network video |
| `phandasm_waveform_source` | Waveform audio visualizer plugin |

## Scene settings (`id: "scene"`)

| Field | Type | Description |
|---|---|---|
| `custom_size` | bool | Scene uses custom canvas size |
| `cx`, `cy` | int | Custom size (when `custom_size`) |
| `id_counter` | int | Next scene-item id |
| `items` | SceneItem[] | **Ordered bottom-to-top** (index 0 = bottom layer) |

### SceneItem

| Field | Type | Description |
|---|---|---|
| `name` | string | Name of the referenced source |
| `source_uuid` | UUID | The referenced source's `uuid` |
| `visible`, `locked` | bool | |
| `id` | int | Scene-local item id (matches `libobs.show_scene_item.<id>` hotkeys) |
| `pos`, `scale`, `bounds` | {x, y} | Canvas px / multipliers / bounding box size |
| `rot` | number | Degrees |
| `align` | int | Anchor bitmask: 0 = center, 1 = left, 2 = right, 4 = top, 8 = bottom (5 = top-left, the usual) |
| `bounds_type` | int | 0 = none, 1 = stretch, 2 = scale to inner, 3 = scale to outer, 4 = scale to width, 5 = scale to height, 6 = max size only |
| `bounds_align` | int | Alignment within bounding box (same bitmask) |
| `bounds_crop` | bool | |
| `crop_left/top/right/bottom` | int | Pixels |
| `scale_filter` | string | `disable`, `point`, `bilinear`, `bicubic`, `lanczos`, `area` |
| `blend_method` | string | `default`, `srgb_off` |
| `blend_type` | string | `normal`, `additive`, `subtract`, `screen`, `multiply`, `lighten`, `darken` |
| `show_transition`, `hide_transition` | object | Per-item show/hide transition (`duration`: ms, …) |
| `group_item_backup` | bool | |
| `private_settings` | object | |

## Per-type `settings`

**`ffmpeg_source`** — `local_file`, `is_local_file`: bool, `looping`: bool,
`restart_on_activate`: bool, `close_when_inactive`: bool, `clear_on_media_end`:
bool, `hw_decode`: bool, `speed_percent`: int (100 = normal; **may appear as a
string when written by external tools via obs-websocket** — a literal
`%variable%` string here means the writing tool failed to expand it),
`input` + `input_format` + `reconnect_delay_sec` + `buffering_mb` (network mode).

**`image_source`** — `file`, `linear_alpha`: bool, `unload`: bool.

**`color_source`** — `color`: int (**ABGR packed 32-bit**, as all colors in this
format), `width`, `height`.

**`browser_source`** — `url`, `width`, `height`, `fps` + `fps_custom`: bool,
`css`, `shutdown`: bool (shut down when not visible), `restart_when_active`:
bool, `reroute_audio`: bool, `webpage_control_level`: int 0–5.

**`text_gdiplus`** — `text` or `read_from_file`: bool + `file`; `font`: {`face`,
`style`, `size`, `flags`: bitmask 1 = Bold, 2 = Italic, 4 = Underline,
8 = Strikeout}; `align`: left/center/right, `valign`: top/center/bottom;
`color`, `opacity`; gradient (`gradient*`), background (`bk_*`), outline
(`outline*`) settings; `vertical`, `extents`(+`_cx`/`_cy`/`_wrap`),
`chatlog_lines`, `transform`: int, `antialiasing`: bool.

**`vlc_source`** — `playlist`: [{`value`: path/URL, `selected`, `hidden`}],
`shuffle`: bool, `loop`: bool, `playback_behavior`: `stop_restart` /
`pause_unpause` / `always_play`, `subtitle`: int + `subtitle_enable`,
`network_caching`: int (ms).

**`dshow_input`** — `video_device_id` (+ `last_video_device_id`),
`audio_device_id`, `active`: bool, `res_type`, `resolution`, `frame_interval`,
`video_format`, `color_space`, `color_range`, `buffering`: bool,
`flip_vertically`: bool.

**`wasapi_input_capture` / `wasapi_output_capture`** — `device_id` (Windows
device id — the part that breaks when Windows reshuffles GUIDs; see
`misc/audiounfuck`), `use_device_timing`: bool.

**`window_capture`** — `window`: `Title:Class:Exe` specifier, `priority`: int
(0 = title, 1 = class, 2 = exe), `method`: int (0 = auto, 1 = BitBlt, 2 = WGC),
`cursor`, `compatibility`, `client_area`, `force_sdr`: bool.

**`decklink-input`** — `device_name`, `device_hash`, `mode_id` + `mode_name`,
`video_connection`: int, `audio_connection`: int, `pixel_format`, `color_space`,
`color_range`, `channel_format`, `buffering`: bool, `swap`: bool.

**`phandasm_waveform_source`** — `audio_source` (source name), `width`,
`height`, `display_mode`: string (e.g. `curve`), `render_mode`: string (e.g.
`line`), `pulse_mode`: string (e.g. `peak_magnitude`), `window`: string (FFT
window, e.g. `blackman_harris`), `log_scale` / `mirror_freq_axis` /
`radial_layout` / `invert_direction` / `fast_peaks` / `normalize_volume`: bool,
`deadzone`, `cutoff_low`, `cutoff_high`, `floor`, `ceiling`: number,
`color_base`: int (ABGR).

## Transition

| Field | Type | Description |
|---|---|---|
| `name` | string | |
| `id` | string | Type: `cut_transition`, `fade_transition`, `wipe_transition`, `obs_stinger_transition`, `move_transition` |
| `settings` | object | Type-specific: wipes have `luma_image`, `luma_softness`, `luma_invert`; stingers have `path` (video), `audio_fade_style`, `preload`, `track_matte_enabled`, `track_matte_layout` |

### QuickTransition

`id`: int, `name`: string (transition name), `duration`: int ms,
`fade_to_black`: bool, `hotkeys`: array.

## Modules (`modules`)

Per-plugin persistent state; only the interesting ones:

| Key | Contents |
|---|---|
| `output-timer` | Stream/record countdown timer settings |
| `auto-scene-switcher` | The built-in (legacy) window-title scene switcher: `interval` ms, `switches[]`, `active` |
| `advanced-scene-switcher` | **The big one** — the entire Advanced Scene Switcher plugin state: `macros[]`, `variables[]`, action queues, websocket/Twitch connections, UI state. ~300 KB in the main collection; where most stream automation lives |
| `captions` / `decklink_captions` | Caption source config |
| `source-dock` | Source dock plugin: `docks[]`, `windows[]`, `corner_tl/tr/br/bl`: bool |
| `downstream_keyers` | Downstream Keyer plugin: array of DSKs, each with `name`, `scene` + `scenes[]` (overlay scene), per-DSK transitions, `exclude_scenes[]` |
| `downstream_keyers_channel` | int — OBS output channel the DSK renders on (7 observed) |
| `scripts-tool` | Loaded scripts: [{`path`, `settings`: per-script config}] — Lua/Python automation (media-hide, timecode, countdown, …) |
| `copyTransformHotkey`, `pasteTransformHotkey` | Hotkey bindings (arrays) |

## Key structural notes

- **Scenes are sources**: everything lives in the flat `sources[]` array; scenes
  are entries with `id: "scene"` whose `settings.items[]` reference other
  sources by `source_uuid`. Scene nesting = a scene item pointing at another
  scene source.
- `scene_order` is only the *panel display order* (including the
  `-----separator-----` pseudo-scenes); it doesn't affect rendering.
- Scene items are ordered **bottom-to-top** within `items[]`.
- All colors are **ABGR** packed into 32-bit integers (alpha in the high byte).
- Global audio devices (Settings → Audio) are stored as full Source objects at
  the **collection root** (`DesktopAudioDevice1`, `AuxAudioDevice1`, …), not in
  `sources[]`.
- Third-party plugin state rides along in `modules` — for this setup that means
  the whole Advanced Scene Switcher automation config and the Downstream Keyer
  overlay compositing are *inside the scene collection file*, and move with it.
- Filter types observed here include stock filters (`chroma_key_filter`,
  `color_filter`, `async_delay_filter`, `gpu_delay`, `mask_filter`, audio
  filters) and plugins (`move_source_filter`, `move_audio_value_filter`,
  `motion-filter`, `source_record_filter`, `streamfx-*`, `obs-shaderfilter-plus`,
  `clut_filter`, `vst_filter`).

## Corrections (2026-07, from validating against the three collections in `OBS/`)

- Added the root-level global audio device keys (`DesktopAudioDevice1`,
  `AuxAudioDevice1`, …) — previously the schema rejected the Bechstein
  collection because of them.
- `phandasm_waveform_source` settings `display_mode`, `render_mode`,
  `pulse_mode`, and `window` are **strings** (e.g. `curve`, `line`,
  `peak_magnitude`, `blackman_harris`), not ints/bools as previously assumed.
- `source-dock` corner fields are booleans, not ints.
- `downstream_keyers_channel` is an integer channel number, not an object.
- `ffmpeg_source.speed_percent` can be a string when written via obs-websocket;
  the projector collection contains literal unexpanded
  `"%json.inputSettings.speed_percent%"` strings in ~20 sources — an artifact
  of the Projector Sync Streamer.bot action failing to substitute its variable
  (a bug in the writer, preserved here by obs-data's type-agnostic storage).
