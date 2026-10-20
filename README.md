# Mike Alfred Sentiment Tracker

A static, shareable dashboard that tracks daily sentiment about **Mike Alfred** on X (Twitter).
Each day gets a **negative score from 0 to 100** (higher means more negative). The dashboard charts
that score with a 7-day trailing average, shows the latest day's post mix, and lists recent days.

The page is pure HTML/JS with no backend and no build step. It reads `data/sentiment.json` at load
time, so updating that one file updates the dashboard.

## Files

```
Mike Alfred Sentiment/
├── index.html            The dashboard (Chart.js 4.4.1 from cdnjs)
├── data/
│   ├── sentiment.json    Daily scores. The Grok Bot writes this file.
│   └── prices.json       IREN / CIFR price overlay. PLACEHOLDER values for now.
├── .nojekyll             Tells GitHub Pages to serve files as-is
└── README.md
```

## Data format: `data/sentiment.json`

```json
{
  "last_updated": "2026-10-20T13:00:00Z",
  "scores": [
    {
      "date": "2026-10-20",
      "negative_count": 19,
      "positive_count": 4,
      "neutral_count": 9,
      "negative_score": 72,
      "engagement_weighted_negative": 64.1,
      "notes": "Elevated negative sentiment"
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `last_updated` | ISO 8601 UTC timestamp | When the file was last written |
| `date` | `YYYY-MM-DD` | The day the score covers |
| `negative_count` | integer | Posts classified as negative |
| `positive_count` | integer | Posts classified as positive |
| `neutral_count` | integer | Posts classified as neutral |
| `negative_score` | number, 0 to 100 | **Primary score.** Higher means more negative. This is the main chart series. |
| `engagement_weighted_negative` | number, 0 to 100, optional | Same score weighted by likes, reposts and replies |
| `notes` | string, optional | Short note for the day |

Score bands used for colour and labels: **Low** below 30, **Moderate** 30 to 59, **Elevated Negative** 60 and above.

## How the Grok Bot should update the data

The Grok Bot runs once a day and updates `data/sentiment.json`:

1. Read the existing file.
2. Append one object for the new day to `scores`. If an entry for that `date` already exists, replace it rather than adding a duplicate.
3. Set `last_updated` to the current UTC time in ISO format.
4. Write valid JSON back (UTF-8, no trailing commas) and commit it to the branch GitHub Pages serves.

Order of `scores` does not matter (the page sorts by date), but keeping it oldest-first makes diffs readable.
Keep every past day: the chart and the 7-day average use the full history.

If the bot pushes through the GitHub API, the commit is the whole deployment. Pages republishes
within a minute or two, and the page fetches the file with `cache: "no-store"`.

## Price overlay: `data/prices.json`

The IREN / CIFR chart is a **placeholder**. The values are made up for layout and are marked
`"is_placeholder": true`, which shows a warning flag on the chart. To use real data, put daily
closing prices under `series.IREN` and `series.CIFR` keyed by date, and set `is_placeholder` to
`false`. Leave weekends and holidays out; the chart shows them as gaps. Prices are indexed to 100
on the first date in the sentiment period and drawn on their own chart, not a second axis on the
sentiment chart.

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

- Current score with qualitative label, change versus the previous day, and position on a 0 to 100 band meter
- Line chart with 7-day trailing average (toggle) and Low / Moderate / Elevated background bands
- Toggle between raw `negative_score` and `engagement_weighted_negative`
- Latest day post mix (negative, neutral, positive)
- Recent activity table (last 10 days)
- Export full data set as CSV
- Automatic light / dark mode with a manual toggle; responsive down to phone width
