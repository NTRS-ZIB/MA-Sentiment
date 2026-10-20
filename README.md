# Mike Alfred Sentiment Tracker

A static, shareable dashboard that tracks daily sentiment about **Mike Alfred** on X (Twitter).
Each day gets a **negative score from 0 to 100**: the share of that day's classified posts that are
negative. The dashboard charts that score with a 7-day trailing average, shows the latest day's post
mix, and lists recent days. IREN and CIFR closing prices are charted underneath for context.

Live at https://ntrs-zib.github.io/MA-Sentiment/

The page is pure HTML/JS with no backend and no build step. It reads `data/sentiment.json` and
`data/prices.json` at load time, so updating those files updates the dashboard.

## Files

```
Mike Alfred Sentiment/
├── index.html            The dashboard (Chart.js 4.4.1 from cdnjs)
├── data/
│   ├── sentiment.json    Daily scores. The Grok Bot writes this file.
│   └── prices.json       IREN / CIFR daily closes (a snapshot; see below)
├── .nojekyll             Tells GitHub Pages to serve files as-is
├── .gitignore            Keeps the local backup/ folder out of git
└── README.md
```

## Data format: `data/sentiment.json`

A real row (Oct 19, 2026):

```json
{
  "last_updated": "2026-10-20T13:26:56-04:00",
  "scores": [
    {
      "date": "2026-10-19",
      "negative_count": 78,
      "positive_count": 15,
      "neutral_count": 217,
      "negative_score": 25,
      "engagement_weighted_negative": 54.9,
      "classified_posts": 310,
      "total_matching_posts": 374,
      "unique_negative_accounts": 55,
      "unique_accounts": 224,
      "negative_accounts_score": 25,
      "top_account_negative_posts": 13,
      "full_capture": true,
      "partial_day": false,
      "notes": "Labelled by reading each post (sentiment toward Mike Alfred personally). Full capture: all 312 matching posts paged (16 pages, recency, 25/page) until no next_token; 310 classified after dropping 2 spam/dup; X counts reports 374 for the day, +62 vs paged (17%) (X counts includes posts that search paging did not return, e.g. since-deleted posts or protected/suspended accounts; cause not verified)"
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `last_updated` | ISO 8601 timestamp with an explicit offset (`Z` or `±hh:mm`) | When the file was last written |
| `date` | `YYYY-MM-DD`, zero-padded | The day the score covers, midnight to midnight US Eastern |
| `negative_count` | integer | Posts classified as negative (toward Mike Alfred personally) |
| `positive_count` | integer | Posts classified as positive |
| `neutral_count` | integer | Posts classified as neutral |
| `classified_posts` | integer | Posts actually classified, after spam and duplicates are dropped. Equals the three counts added together. |
| `negative_score` | integer, 0 to 100, **required** (`null` only when `classified_posts` is 0) | **Primary score.** `round(100 × negative_count ÷ classified_posts)`. This is the main chart series and the only one the bands apply to. |
| `engagement_weighted_negative` | number, 0 to 100, one decimal | Same score weighted by likes, reposts and replies |
| `full_capture` | JSON boolean, **required on every row** | `true` if every post X search returned for the day was classified; `false` if only a sample was |
| `partial_day` | JSON boolean | `true` if the day was not over when it was scored |
| `total_matching_posts` | integer or `null` | X's own count of matching posts for the day. X only provides it for the last 7 days, so older days are `null`. It is higher than what search returns: on Oct 14–19, 2026 search returned 62–91% of X's figure (most of the gap on Oct 15: 222 of 356). The day notes give the shortfall as a share of X's count (9–38%). The likely cause is posts search no longer returns, such as deleted posts or protected or suspended accounts, but that is not verified. It is context, not the coverage denominator. |
| `unique_negative_accounts` | integer | Distinct accounts behind the negative posts |
| `top_account_negative_posts` | integer | Most negative posts from any one account that day |
| `unique_accounts` | integer | Distinct accounts posting that day |
| `negative_accounts_score` | integer, 0 to 100 | `round(100 × unique_negative_accounts ÷ unique_accounts)`. The dashboard shows a "By account" toggle as soon as any row has it. |
| `notes` | string | Free-text note on how the day was captured |

Numbers must be JSON numbers and flags JSON booleans, not strings. The page tolerates `"12"` and
`"true"`, but don't rely on it. Any other fields the bot adds are kept and included in the CSV export.

### Row rules

- **One row for every calendar day**, with no skipped dates. On a day with no matching posts, still
  write the row with all counts `0`, `classified_posts: 0` and `negative_score: null`. The page shows it
  as "No posts" and leaves it out of the averages, rather than scoring it as a calm day. If a day is
  missing, the dashboard shows it as a gap labelled "No data".
- **Optional fields go on every row or on none.** For example, add `negative_accounts_score` to the
  whole history in one go. If only some days have it, the "By account" line shows gaps.
- **If `full_capture` is missing or not true/false**, the page cannot tell whether the day was sampled.
  It draws the day solid like a full-capture day, gives it no coverage label, starts a separate 7-day
  average for it, and counts it in a notice at the top of the page. While any day is full capture, the
  High figure uses only those days. If no row has a valid flag, the notice says so and High falls back
  to all finished days, labelled "no full-capture days".
- A row with an unreadable date is left out, and so is a row dated after today (US Eastern; `last_updated`
  is converted to Eastern time when it has an offset), or one or
  two older rows stranded more than 31 days from the rest (usually a mistyped year). The newest one or
  two rows are always kept when they fall within two days of today or of `last_updated`, so the first
  days after a long pause show straight away. The page names any rows it leaves out in a notice.
- Runs of up to 30 missing days are filled with "No data" days. A run of 31 or more is shown as a single break, marked
  in the table and named in the notice; lines and the 7-day average never join across it.
- A score outside 0–100 (`negative_score`, `engagement_weighted_negative` or `negative_accounts_score`)
  is blanked: the day stays in the table with its other figures, the chart leaves a gap, and a notice
  appears. The CSV export still writes the value as found in the file.
- If two rows share a date, the page keeps one: a row with a score written (even one outside 0–100,
  which is then blanked and reported) or marked as no posts comes first, then full capture over sampled,
  then a finished day over a partial one, then the later row in the file.

### Score bands

Low below 15, Moderate 15 to 24, Elevated 25 and above, on the raw `negative_score` only.
They were set on Oct 20, 2026 from the full-capture days of Sep 4 to Oct 19, after the Grok Bot
relabelled them by reading each post:

- Moderate = the median of finished full-capture days (13) plus one, rounded up to a multiple of 5 → 15.
- Elevated = the 90th percentile (nearest rank) of the 39 days before the Oct 13 run (24) plus one,
  rounded up to a multiple of 5 → 25. The run is left out because it is what the bands are meant to flag.
- Result: before Oct 13, 29 days Low, 7 Moderate, 3 Elevated; Oct 13 to 19, 6 of 7 days Elevated.

The bands live in two constants at the top of the script in `index.html` (`BAND_MODERATE`,
`BAND_ELEVATED`), with `METER_MAX` (50) setting the right end of the headline meter. The repo owner
re-applies the rule above after the Jul 22 – Sep 3 re-capture lands (it roughly doubles the
full-capture history) and whenever the way posts are labelled changes. The page footer shows the
current median.

Sampled days and unfinished days get no band label (one post moves a sampled score 2–5 points), and
the engagement-weighted and by-account views show no bands because the bands were set from the raw
score only. When one account wrote at least a quarter of a day's negative posts (and at least 3), the page
shows what the score would be without it, and the table marks and lists days where that changes the band.

The chart's scale is set from finished full-capture days, so sampled spikes don't flatten the current
data; any value above the top of the chart is marked with ▲, with its number where there is room (the
tooltip and table always have it).

### Sampled vs full-capture days

Days with `full_capture: false` classified a sample of 20 to 50 posts, mostly X's top-ranked results.
On a few days the bot fell back to recent posts; each day's `notes` says so. These days were also
labelled by an earlier keyword method, which reads much lower than reading each post, so they are not
comparable with full-capture days until the bot re-captures them. The chart draws sampled
days dashed on a shaded background, with the 7-day average dashed there too. The average is computed
separately for each kind of day so the two are never blended. It also skips partial days and averages
fewer than seven days at the start of a run (the tooltip shows how many). High uses finished full-capture days
when there are any; otherwise all finished days, labelled "no full-capture days".

## How the Grok Bot should update the data

The morning run scores **yesterday** (midnight to midnight US Eastern), so each new row is a finished day.

1. Read the existing file.
2. For each day being written, replace the row with that `date` if one exists; otherwise add it. This
   applies to daily updates, re-captures and backfills alike. A re-captured day replaces its old
   sampled row and sets `full_capture: true`.
3. If a day is scored before it ends, set `partial_day: true`, and rewrite that date with final numbers
   and `partial_day: false` on the next run. The page shows a partial day as "Day not finished" with no
   band and leaves it out of the averages, the High figure and the chart scale, but its score still
   appears as the headline number, so avoid it where possible.
4. Leave `total_matching_posts` as `null` for days older than X's 7-day count window.
5. Set `last_updated` to the current time as an ISO 8601 timestamp with an explicit offset.
6. Write valid JSON back (UTF-8, no trailing commas) and commit it to the branch GitHub Pages serves.
7. Only change files under `data/`. Dashboard code changes go through the repo owner.

Order of `scores` does not matter (the page sorts by date), but keeping it oldest-first makes diffs
readable. Keep every past day: the chart and the 7-day average use the full history.

If the bot pushes through the GitHub API, the commit is the whole deployment. Pages republishes
within a minute or two, and the page fetches both files with `cache: "no-store"`.

## Price overlay: `data/prices.json`

Daily split-adjusted closing prices for IREN and CIFR:
`series.<TICKER>["YYYY-MM-DD"] = close`, trading days only, with `"is_placeholder": false` and a
top-level `note` saying where the prices came from. The first two tickers in `series` are charted.

**Nobody updates this file yet.** It is a one-off snapshot through Oct 19, 2026. Either refresh it by
hand or add a step to the Grok Bot: append the previous session's close for each ticker and update the
retrieval date in `note`. The chart legend shows "Closes through <date>", the last close drawn on the chart, and warns when prices
are more than 4 days behind the sentiment data.

Prices are indexed to 100 on the first trading day shown and drawn on their own chart, not a second
axis on the sentiment chart. The tooltip shows the actual close beside the index. Weekends and
holidays are absent; the line is joined across them (but not across holes longer than 5 days) and
hovering snaps to the nearest trading day. The legend warns about any ticker more than 4 days behind.
If `is_placeholder` is `true`, the chart shows a warning flag.

## Run locally

Browsers block `fetch` from `file://`, so serve the folder over HTTP:

```bash
python -m http.server 8000
```

Then open http://localhost:8000.

## Deploy to GitHub Pages

1. Create a GitHub repository and push the contents of this folder to its root.
2. In the repo, go to **Settings → Pages**, choose **Deploy from a branch**, branch `main`, folder `/ (root)`.
3. The site is published at `https://<user>.github.io/<repo>/`. All paths in `index.html` are relative, so it works under a repo sub-path.

`data/sentiment.json` is a normal static file in the repo, so it is publicly readable by the dashboard
and by anyone who visits the URL.

## Features

- Latest score with its band, change versus the previous day, and position on a 0 to 50+ meter coloured by the Low / Moderate / Elevated bands (grey on days with no band; scores above 50 pin at the right end)
- Coverage line (full capture or sampled, posts classified, X's count), a "Day not finished" badge in place of the band on unfinished days, a warning when the data is more than two days old, and a note when one account wrote a large share of the day's negative posts, with the score without it
- Line chart with 7-day trailing average (toggle), bands on the raw score, sampled days dashed on a shaded background, and off-scale markers; missing days show as gaps
- Toggle between raw `negative_score`, `engagement_weighted_negative`, and (when present) `negative_accounts_score`
- Latest day post mix with distinct negative accounts and the top single account; 7-day average, High (full-capture days when there are any) and engagement-weighted figures
- IREN / CIFR price chart, indexed, with actual closes in the tooltip and a "closes through" date
- Recent activity table (last 10 days) with band labels, single-account flags, coverage and expandable notes
- Export the full data set as CSV, every field included
- Automatic light / dark mode with a manual toggle; prints in light colours; responsive down to phone width
