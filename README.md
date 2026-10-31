# Cricsheet Preprocessing

Parses [Cricsheet](https://cricsheet.org) ball-by-ball JSON files into a clean, analysis-ready table and computes normalized batting and bowling stats. This is the Phase 1 "cold start" ingestion step of the AI Cricket Coaching & Analytics Platform.

## What it does

1. **Parses** every `*.json` match file in a folder into one row per delivery.
2. **Cleans** the data: parses dates, drops rows missing key fields, builds a stable player key, flags legal balls, and tags the match phase.
3. **Aggregates** batting and bowling stats by player, format and phase.
4. **Writes** three Parquet files.

Bad or malformed files are skipped and logged to stderr; the run continues.

## Requirements

- Python 3.10+
- `pandas`, `numpy`, `pyarrow` (for Parquet output)

```bash
pip install pandas numpy pyarrow
```

## Usage

1. Download a JSON zip from Cricsheet (e.g. T20s, ODIs, Tests) and unzip it into a folder, e.g. `cricsheet_json/`.
2. Run:

```bash
python parse_cricsheet.py cricsheet_json
```

If no folder is given, it defaults to `cricsheet_json`.

## Outputs

| File | Contents |
|---|---|
| `balls_clean.parquet` | One row per delivery, cleaned |
| `batting_stats.parquet` | Batting stats per player / format / phase |
| `bowling_stats.parquet` | Bowling stats per player / format / phase |

### Ball-level columns

`match_id`, `date`, `format`, `venue`, `innings`, `team`, `over`, `ball`, `batter`, `batter_id`, `bowler`, `bowler_id`, `runs_bat`, `runs_total`, `is_wide`, `is_noball`, `wicket`, plus derived `batter_key`, `bowler_key`, `legal_ball`, `phase`.

### Stats columns

- **Batting**: `balls`, `runs`, `dismissals`, `strike_rate`, `sr_shrunk`, `average`
- **Bowling**: `balls`, `runs`, `wickets`, `economy`

## Design notes

- **Player identity**: uses Cricsheet's `registry.people` ID when present (`batter_id`, `bowler_id`). When it is missing, `*_key` falls back to a lowercase, punctuation-stripped name. This resolves most "V Kohli" vs "Virat Kohli" cases, but a proper alias table (see the architecture doc, section 1.3) is still recommended for non-Cricsheet sources.
- **Legal balls**: wides and no-balls do not count as balls faced or bowled.
- **Strike rate** is computed from raw counts (`runs / balls x 100`), never by averaging averages.
- **Shrunk strike rate** (`sr_shrunk`) uses Bayesian shrinkage toward a prior strike rate so tiny samples don't produce misleading values:
  `(runs + k * prior_sr / 100) / (balls + k) * 100`, with defaults `k = 30`, `prior_sr = 125`.
- **Economy** is `runs / (legal_balls / 6)`.
- **Missing values** are left as `NaN` (e.g. average with zero dismissals); nothing is imputed silently.

## Known limitations

- **Phases are T20-oriented**: Powerplay is overs 0-5, Middle is 6-14, Death is 15+ (Cricsheet overs are 0-indexed). For ODIs and Tests these buckets aren't meaningful; adjust the `pd.cut` bins in `clean()` or make them format-specific.
- `prior_sr` is a single default; use format- and role-specific priors for better shrinkage.
- Bowler runs use `runs_total` and so include byes and leg-byes. Refine this if you need strict bowler-conceded runs.
- Powerplay overs 0-5 is a coarse default; super-overs and shortened matches are not special-cased.
- Ball speed is not in Cricsheet data; it comes from other sources and should be joined separately, keeping nulls as nulls.

## Licensing

Check Cricsheet's data license and attribution requirements before any public or commercial use.

## Next steps

- Load `balls_clean.parquet` into the `MatchData` / `BallEvent` tables (see the Prisma schema, section 4.1).
- Replace the name fallback with the `PlayerAlias` table and fuzzy matching.
- Schedule the script as a daily job that only processes new match IDs.
