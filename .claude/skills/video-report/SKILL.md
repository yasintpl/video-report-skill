---
name: video-report
description: Build a read-only performance card for one published YouTube video (views, retention drop-offs in seconds, traffic sources, thumbnail CTR vs the channel's comparable videos, A/B test status) with the vidIQ MCP tools, and propose 1–3 concrete fixes for the next video. Use when the user asks how a video did, why it flopped, for a 48h or 7-day check, or a video report/card.
argument-hint: "<video ID | YouTube URL | latest>"
---

# Video report

One published video in → one Markdown card out, with numbers, at most 3 findings and 1–3 fixes for the next video.
Talk to the user in their language; the card itself is written in English.

## Setup (once)
- **vidIQ MCP connector** connected and authorized with the Google account that owns or manages the channel
  (tools named `vidiq_*`). Studio analytics need the channel connected in vidIQ.
- **vidIQ Chrome extension** (optional) installed, signed in, and YouTube Studio open on the **same** channel.
  Only CTR and A/B tests need it; without it the card still runs, with those rows marked "no data".
- `config.json` next to this file:
  - `compare_since` — first publish date (YYYY-MM-DD) of the channel's current format. Long-form videos published on
    or after it are the comparison set. `null` = propose a set from titles/tags and ask the user to confirm it.
  - `max_credits_per_card` (default 25), `min_balance` (default 25).
  - `output_dir` — where cards are written, relative to the project root (default `video-reports`).
- Nothing personal is stored in this skill: the channel ID is read at run time, never written here.

## Hard rules
- **Read-only, allowlist.** Only call these vidIQ tools: `vidiq_balance`, `vidiq_user_channels`, `vidiq_user_videos`,
  `vidiq_channel_analytics`, `vidiq_channel_analytics_extended` (reports `thumbnail_ctr` and `ab_tests` only),
  `vidiq_connect_extension`. Never call any other vidIQ tool. No generation tools (thumbnails, titles, video, voice,
  music, images), no plan/billing tools, nothing that writes to YouTube (videos, thumbnails, comments, A/B tests).
- **No invented numbers.** Every number in the card comes from a tool result in this run (or simple arithmetic on one,
  shown as such). `null`, a missing row or a failed call is **"no data"**, never 0. Keep zeros that the tool returns.
- **Credit cap:** at most `max_credits_per_card` (25) per card. Keep a running tally; before each credit call check that
  `spent + cost ≤ cap`, otherwise skip it and say so in the card.
- **Studio lag:** YouTube Studio data arrives 2–3 days late. If the retention curve comes back empty, stop: tell the user,
  write no card, make no further credit calls.
- **Data vs reading:** in the card, measured values are tagged **[data]**, conclusions **[read]**.

## 1. Pre-flight (free calls)
1. `vidiq_balance` → if `totalCredits < min_balance`: stop and tell the user the balance and the reset date.
   Otherwise tell the user the balance before the first credit call.
2. `vidiq_user_channels` → channel ID (`channelId`). If several channels, ask which one. If none, tell the user which
   account vidIQ is connected as and stop.

## 2. Resolve the video (credit call only when needed)
You need: video ID, title, `publishedAt`, duration, and the comparison IDs.
- **Previous card shortcut (free):** if a card for this video already exists in `<output_dir>/` (file name contains the
  video ID), take ID, title, published date, duration and comparison IDs from its frontmatter and skip the listing call.
- Otherwise `vidiq_user_videos` (5 cr): `channelId`, `videoType: ["long"]`, `privacyStatus: ["public"]`, `limit: 15`,
  `order: newest`.
  - Input `latest` → the newest public long-form video. A URL → extract the 11-char ID.
  - Duration from ISO 8601 (`PT6M6S` → 366 s).
  - **Comparison set:** public long-form videos with `publishedAt ≥ compare_since`, excluding the target, max 10.
    If `compare_since` is null, propose the set (same topic/format by title and tags; never the channel's old niche) and
    let the user confirm. State the set and its size `n` in the card; with `n ≤ 2`, say the comparison is thin.

**Checkpoint** from age = today − publish date (UTC):
- age < 2 days → stop, too early for Studio data (no credit calls beyond this point).
- 2 ≤ age < 7 → `48h`.  age ≥ 7 → `7d`.
- **Window:** `startDate` = publish date, `endDate` = yesterday (last full day). Use the same window for every call.
  Write the window and the age in the card: the label is a checkpoint name, the window is what was measured.

## 3. Pull the data (5 credits each, in this order)
| # | Call | Parameters |
|---|---|---|
| A | `vidiq_channel_analytics` | `report: audience_retention`, `filters: video==<ID>`, window. **Empty → stop (Studio lag).** |
| B | `vidiq_channel_analytics` | `dimensions: [video]`, `filters: video==<ID>,<cmp1>,<cmp2>…`, metrics `views, engagedViews, averageViewDuration, averageViewPercentage, subscribersGained`, window |
| C | `vidiq_channel_analytics` | `report: traffic_sources`, `filters: video==<ID>`, window |
| D | `vidiq_channel_analytics_extended` | `report: thumbnail_ctr`, `videoIds: [<ID>, cmp…]`, `startDate` = window start, `endDate` = window end **+1 day** (exclusive) |
| E | `vidiq_channel_analytics_extended` | `report: ab_tests`, `videoIds: [<ID>]` — only if the cap allows |

Budget: listing + A–D = 25. So **E runs only when the listing was skipped** (a previous card exists, e.g. the 7d card
after a 48h card). Otherwise the card says "A/B: not read (credit cap), read on the next card".

Extension errors (D, E):
- `extension_not_connected` → call `vidiq_connect_extension` with the same `report`, tell the user in one sentence,
  and skip E. Retry only when the user says it is connected.
- `channel_mismatch` → tell the user Studio in the browser is on a different account/channel and give the returned link.
  Skip E. Mark CTR/A/B "no data (<errorCode>)".
- `processing` → call again with only the returned `jobId` (no extra credit if the tool says so; count it if it charges).
- Failed calls usually refund. Check `_credits.used` in each result and tally real spend.

## 4. Analyse
- **Totals [data]:** views, engaged views, engaged share = engaged ÷ views, avg view duration (s), avg %, subs gained.
- **Comparison [data]:** same metrics for each comparison video, and their median. Note age differences inside the
  window (a 2-day-old video is not comparable on views; it is on avg % and CTR).
- **Retention:**
  - Seconds = `elapsedVideoTimeRatio × duration`, rounded to whole seconds.
  - Kept at 30 s: interpolate `audienceWatchRatio` linearly between the two points around `30 / duration`.
  - Also report: first point, 50 % of the video, last point, and the second where the curve first falls below 50 %.
  - **Drops:** the 3 largest falls between neighbouring points (percentage points), in seconds. Spikes above 1.0 are
    rewatches, not drops.
  - `relativeRetentionPerformance`: 0.5 = median of similar-length YouTube videos. Summarise by stretch
    (opening ≈ first 30 s, then the rest in bands where it stays above or below 0.5).
  - Chapters: if the user gives chapters (or they are in the video description the user pastes), name the chapter
    each drop or weak stretch falls in; otherwise skip chapter names.
- **Traffic:** per source views, engaged, avg %. Subscriber feed (`SUBSCRIBER`) engaged ÷ views vs suggested
  (`RELATED_VIDEO`) and search (`YT_SEARCH`). A low subscriber-feed ratio means the existing audience is not this
  video's audience.
- **CTR:** target vs comparison videos (impressions + CTR as returned; never recompute CTR from views).
- **A/B:** status, result, each variant's watch-time share. `notEnoughData` / `inconclusive` are reported as such.

## 5. Write the card
Path: `<output_dir>/<YYYY-MM-DD>-<videoID>-<48h|7d>.md` (date = today). Create `<output_dir>/` if needed. If the file
exists, ask before overwriting. Use `card-template.md` in this folder exactly: same headings, same order.
- Max 3 findings, each pointing at a number in the tables.
- 1–3 fixes, each a concrete action for the **next** video (what to change, where, how you would know it worked).
  No vague advice ("improve the hook"); say what to show in which second.
- "Next check": date and the exact metric to look at.
- Credits: list each credit call and its real cost, total, and the balance after.

## 6. Finish
Reply to the user in their language: card path, the 3 key numbers, the fixes in one line each, credits spent.
Do not edit any other file unless the user asks.
