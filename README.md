# This was analyzed by claude and listed off here, I couldn't be assed to write this myself but this is egregious so I'm publishing the findings:
#
# What CapCut PC records and sends off — a telemetry catalog

*Evidence-based. Every item below was parsed out of the actual upload payloads the CapCut client
(9.6.0.4097) sent, captured in mitmproxy recordings. Field and event names are the
client's own. Nothing here is guessed.*

## TL;DR

In a short test session the client uploaded **~11 MB** of telemetry — and that's modest. Across the
captured recordings the two biggest channels alone were **~97 MB of behavior events** and **~197 MB of
"APM" logs**. The single largest thing by volume is a log called **`api_all`** that records **every
network request the app makes** — URL, timing, request headers, **cookies**, and response headers
(including auth tokens). There were **32,971** of those entries in one capture.

It is tied to you: every event batch carries your `device_id`, `user_id`, and a stable
`user_unique_id` fingerprint. So this is not anonymous aggregate — it's a per-user, per-device activity
stream.

---

## The four channels it sends to

| Channel | Host (real) | What it is | Volume |
|---|---|---|---|
| **Event logs** | `mcs*` → `/v1/list`, `/v1/json` | Behavior/usage analytics (the big one) — every click, panel, edit, material, search | ~97 MB |
| **APM / crash** | `pc-mon*` → `/monitor_pc/collect/*` | App performance + a full network log (`api_all`), launch/exit, crashes | ~197 MB |
| **Device / security** | `mssdk*` → `/ri/report` | Device-risk SDK (anti-fraud fingerprint) | small |
| **Realtime** | `frontier*` → `/ws/v2` (websocket) | Push channel kept open the whole session | ongoing |

Plus `device_register` at startup, which mints/attaches the device identifiers below.

---

## What normal people would care about

- **It phones home constantly, not just on actions.** Launch, exit, idle performance, and a heartbeat
  websocket all report in. The ~11 MB from "just small tests" is normal.
- **Every action you take in the editor is logged**, individually: opening a panel, clicking a tab,
  adding a track, applying an effect/filter/transition, picking a material, playing the preview,
  zooming the timeline, entering a category. Representative event names:
  `click_material_item`, `enter_edit_page_start`, `add_new_track`, `show_edit_tools`,
  `click_video_function_edit_tab`, `zoom_timeline`, `enter_category`, `tab_show`, `close_function_panel`.
- **What you make is profiled.** Per project it reports draft duration, fps, resolution, track counts,
  and counts of every element type — how many videos, texts, stickers, filters, transitions, beauty
  effects, AI-avatar clips, etc. (`perf_*` and `min_fps_*` families, hundreds of fields).
- **Your hardware is inventoried** (see below) — useful to them for performance, but it's a full spec
  sheet of your PC.
- **It uploads your network activity**, which indirectly reveals everything the app talked to.

## What privacy-conscious people would care about

These are the items that make it *identifying* and *content-bearing*, not just "usage stats":

**Stable identifiers attached to every upload**
- `device_id`, `user_id`, `user_unique_id` (a hashed per-user fingerprint), `user_is_login`, `user_type`
- `install_id` / `app_install_id`, `install_time`, `last_install_time`, `is_new_install`
- `did_pseudonym_status` / `uid_pseudonym_status` + a parallel **"looki"** identity
  (`looki_did`, `looki_uid`, `looki_log_trace_id`) — their pseudonymization layer, which still maps back.

**Hardware fingerprint (uniquely identifies your machine)**
- `disk_serial_num` (your drive's serial number — sent in `web_direct_pc`)
- `hardware_id` (a machine fingerprint — sent at launch in `env_detect_launch`)
- `driver_device_id`, `driver_vendor_id`, `vendor_id`, `driver_name`, `driver_version`, `driver_date`
- GPU/CPU model, `chip_arch`, `cpu_usage`, `gpu_memory`, `memory`, `disk`, `disk_space`
- **Full monitor layout**: `screen_1_width/height/dpi/refresh_rate/manufacturer/x/y` and the same for
  `screen_2_*` — i.e., your exact multi-monitor setup
- `os`, `os_version`, `system_version`, `app_arch`, `device_model`

**Location / network**
- `client_ip` (your IP, in `user_install_info`)
- `x-tt-geo` geo marker and `store_country_code` / `store_region` / `curr_region` (region inference)
- Full network diagnostics: DNS, SSL, QUIC, connection timing, `ifconfig`

**Linked accounts & identity**
- `tiktok_id` and `capcut_id` (captured in `login_status` — links your CapCut use to your TikTok account)
- `account_id`, `account_type`, `oauth_time`

**Your actual content (not just metadata)**
- **`text_content`** — **not** a keylogger (no OS-wide or per-keystroke capture): when you *commit* a
  text/caption layer, the finished text string is attached to the `click_new_text_confirm` event and
  uploaded (confirmed in the capture: a literal string, not a hash). So the on-screen text you place into
  a project leaves the machine as a discrete event, tied to your identity — content telemetry, not
  keystroke logging.
- **`search_keyword` / `keyword`** — everything you search for (materials, fonts, effects).
- **`draft_name` and `draft_path`** — your project names and their **full paths on your disk**
  (reveals your username/folder structure).
- `material_name`, `font`, `tone_picture`, `template_name`, `cover_image_url`.

**Auth tokens and cookies (the worst offender)**
- The `api_all` network log records, for **every request**, the `request_cookie` and
  `request_header_cookie` you sent, plus response headers including `set_cookie`, **`x-tt-token`**,
  `x-tt-token-sign`, and `x-tt-store-sec-uid`. In other words, your **session/auth credentials are
  copied into the uploaded analytics logs** (`commerce_api_monitor` and the APM `api_all` stream).
- It also logs `request_url_no_query`, `urlquery`, `request_host`, `request_body`, `response_header`
  and `error_logid` — effectively a mirror of your whole API traffic.

**AI/credits & commerce**
- `is_vip`, `vip_status`, `vip_type`, `user_subscribe_type`, `credits_use`, `credits_now`,
  `show_free_trial`, `is_trial`, `product_id` — your subscription/credit state and which upsells you saw.

*(The full event-param surface is **1,386 distinct fields** across hundreds of event types; the above are
the categories that matter. The raw lists are in the capture if you ever want the exhaustive dump.)*

---

## What's tied to what

Every behavior event batch ships with the same `header` + `user` block:

```
header: app_id, app_version, app_channel, app_install_id, os_name, os_version, device_model,
        ab_sdk_version (your A/B test bucket assignments), traffic_type
user:   device_id, user_id, user_unique_id, user_is_auth, user_is_login, user_type
```

So any single event — "typed this caption", "searched this word", "opened this project at this path" —
is joinable to your device serial, your account, and your TikTok id. That's the privacy problem in one
sentence: **content + stable identity + hardware fingerprint, in the same payload.**
