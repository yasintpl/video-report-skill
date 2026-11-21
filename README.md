# video-report — a Claude Code skill

A Claude Code skill that reads one published YouTube video and tells you what to fix in the next one.
It only reads. It never changes anything on YouTube.

## What it does

One video in, one card out.

You type `/video-report latest`. The skill pulls the video's numbers through vidIQ and writes a Markdown card:

- the numbers, next to your other comparable videos
- the retention curve in seconds: where people leave
- where your viewers came from
- thumbnail click rate and A/B test status (if the vidIQ extension is connected)
- at most 3 findings and 1–3 fixes for your next video
- what to check next time, and how many credits the card cost

An example card summary (**made-up example numbers, not real data**):

```
Video card · My example video · 7d
Views 1,234 [data] · comparison median 1,000
Still watching at 30 s: 55 % [data]
Half the viewers gone by second 42 [data]
Relative retention in the first 30 s: 0.35 (below the 0.5 median) [data]
Finding: the opening loses people faster than similar videos [read]
Fix: show the result by second 5, no setup sentence first [read]
Credits: 20 of 25
```

## What it never does

- It only reads. It never writes to YouTube: no video, title, thumbnail, comment or A/B test changes.
- It never invents numbers. Missing data says "no data", never 0.
- It never spends more than 25 vidIQ credits on one card.
- It only calls the vidIQ tools on its allowlist (listed in `SKILL.md`). No generation tools, no plan or billing tools.

## Requirements

- **Claude Code.** This is paid: you need a Claude Pro or Max plan, or API usage.
- **A free vidIQ account**, with the vidIQ connector added to Claude Code and signed in with the Google account of your channel.
- **The vidIQ Chrome extension** (optional). Only the thumbnail click rate and A/B test rows need it.
  Without it, those rows say "no data" and the rest of the card still works.
- **You own or manage the channel.** YouTube only shares these analytics with the channel's owner or managers.

## Costs

- The skill is free.
- The vidIQ Free plan gives a monthly credit allowance. Most data calls cost 5 credits each.
  One card spends at most 25 credits. At the time of writing, the Free plan had 150 credits a month, so about 6 cards.
- Nothing is bought automatically. If your credits run out, the skill stops and tells you.
- vidIQ can change its prices, credit costs or MCP access at any time. Check vidIQ's current plans before you rely on this.

## Install

1. Copy the folder `.claude/skills/video-report/` from this repo into your project's `.claude/skills/` folder.
   Or copy it into `~/.claude/skills/` to use it in every project.
2. Add the vidIQ connector to Claude Code and sign in with your channel's Google account.
3. Open `config.json` in the skill folder:
   - `compare_since`: the first publish date (`YYYY-MM-DD`) of your current video format. Videos from that date on are
     the comparison set. Leave it `null` and the skill proposes a set and asks you to confirm.
   - `output_dir`: where the cards go, relative to the folder you run Claude Code in. Default: `video-reports`.
   - `max_credits_per_card` and `min_balance`: both 25 by default.

## Use

```
/video-report latest
/video-report <video ID or YouTube URL>
```

You can also just ask: "how did my last video do?"

The skill picks a checkpoint from the video's age:

- Under 2 days old: it stops. YouTube Studio data is not ready yet.
- 2 to 6 days old: a **48h** card.
- 7 days or older: a **7d** card.

The window is always the publish day to yesterday.
Cards are saved as `<output_dir>/<date>-<videoID>-<48h|7d>.md`.
A second card for the same video reads the first card for free, so it has room for the A/B test.

## Reading the card

- **[data]** is a number from YouTube, or simple arithmetic on one. **[read]** is the skill's interpretation.
- **Relative retention:** YouTube compares your curve with videos of similar length.
  0.5 is the median. Below 0.5, people leave faster than on similar videos. Above 0.5, they stay longer.
- **"no data" is not 0.** 0 means YouTube measured nothing happening. "no data" means the number could not be read.

## Troubleshooting

- **`channel_mismatch`:** YouTube Studio in your browser is open on a different account or channel.
  Switch Studio to the channel's account, then run the card again.
- **Empty retention curve:** YouTube Studio data arrives 2–3 days late. The skill stops after one call. Try again later.
- **`extension_not_connected`:** the vidIQ Chrome extension is not installed or not signed in.
  The skill shows how to connect it. The rest of the card still works.
- **Low balance:** with fewer than 25 credits, the skill stops before spending anything and tells you when credits reset.

## License

MIT. See `LICENSE`.
