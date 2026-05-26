# Chill Out Games 2026

## 1. About

This is a static leaderboard for Chill Out Games 2026, hosted on GitHub Pages and updated every Thursday via GitHub Actions from a Google Sheets source. The site has no backend. GitHub Pages serves `index.html`, and all leaderboard data is loaded from `data.json`.

## 2. Google Sheets Setup

The score data lives in one Google Spreadsheet with multiple sheets, also called tabs. Keep the sheet names and columns exactly as listed below so the GitHub Actions workflow can merge everything into `data.json`.

### Sheet: `Roster`

Columns: `team_id`, `team_name`, `division`, `captain`, `athlete_name`

Use one row per athlete. Set `captain` to `TRUE` for the team captain and `FALSE` for everyone else.

```csv
team_id,team_name,division,captain,athlete_name
team_001,Tinikling Titans,RX,TRUE,Miguel Santos
team_001,Tinikling Titans,RX,FALSE,Paolo Reyes
team_001,Tinikling Titans,RX,FALSE,Bianca Cruz
team_009,Scaled Silogs,Scaled,TRUE,Ana Lopez
```

### Sheet: `Swaps`

Columns: `wod`, `team_id`, `swapped_out`, `swapped_in`

Use one row per swap event. This is only used for WOD 3 and WOD 4. The `captain` column in `Roster` is used by the workflow to prevent the captain from ever appearing as `swapped_out`.

```csv
wod,team_id,swapped_out,swapped_in
3,team_001,Bianca Cruz,Jessa Lim
4,team_003,Rafa Mendoza,Enzo Ramos
```

### Sheets: `WOD_1`, `WOD_2`, `WOD_3`, `WOD_4`

Columns: `team_id`, `score`, `rank`, `points`

Use one row per team. Score format depends on workout type, such as `"13:42"` for For Time, `"141 reps"` for AMRAP, or `"565 lb"` for Max Load. Leave rows blank or omit rows for unreleased WODs.

```csv
team_id,score,rank,points
team_001,12:58,2,2
team_002,13:21,3,3
team_003,14:02,5,5
```

### Sheet: `WOD_Info`

Columns: `wod_id`, `name`, `release_date`, `released`, `format`, `description`

Use one row per WOD. Set `released` to `TRUE` or `FALSE`.

```csv
wod_id,name,release_date,released,format,description
1,WOD 1,2026-06-19,TRUE,For Time,21-15-9 thrusters and pull-ups
4,WOD 4,2026-07-10,FALSE,TBA,TBA
```

## 3. How the GitHub Actions Workflow Works

- Runs every Thursday at a set time in UTC.
- Fetches each sheet as a CSV using the Google Sheets publish-to-web CSV URL.
- Merges all sheets into a single `data.json`.
- Commits and pushes `data.json` to the repo.
- The Pages deployment workflow publishes the current static files to GitHub Pages automatically.

Make the Google Sheet public with View access and publish each sheet to the web as CSV to get the URL.

## 4. How to Get the Google Sheets CSV URLs

1. Open the Google Spreadsheet.
2. Go to File -> Share -> Publish to web.
3. Select one sheet, such as `Roster`.
4. Choose CSV.
5. Copy the published URL.
6. Repeat for every sheet.

CSV URL format:

```text
https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID/gviz/tq?tqx=out:csv&sheet=SHEET_NAME
```

## 5. GitHub Actions Workflow Setup

Create `.github/workflows/update-leaderboard.yml` with this content:

```yaml
name: Update Leaderboard Data

on:
  schedule:
    - cron: "0 10 * * 4"
  workflow_dispatch:

permissions:
  contents: write

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Build data.json from Google Sheets CSV
        env:
          SHEET_ID: ${{ secrets.GOOGLE_SHEET_ID }}
        run: |
          python - <<'PY'
          import csv
          import io
          import json
          import os
          import urllib.parse
          import urllib.request
          from collections import defaultdict

          SHEET_ID = os.environ["SHEET_ID"]
          SHEETS = ["Roster", "Swaps", "WOD_Info", "WOD_1", "WOD_2", "WOD_3", "WOD_4"]

          def fetch_csv(sheet_name):
              encoded = urllib.parse.quote(sheet_name)
              url = f"https://docs.google.com/spreadsheets/d/{SHEET_ID}/gviz/tq?tqx=out:csv&sheet={encoded}"
              with urllib.request.urlopen(url) as response:
                  text = response.read().decode("utf-8-sig")
              return list(csv.DictReader(io.StringIO(text)))

          tables = {name: fetch_csv(name) for name in SHEETS}

          teams = {}
          roster_by_team = defaultdict(list)
          captains = {}

          for row in tables["Roster"]:
              team_id = row["team_id"].strip()
              athlete = row["athlete_name"].strip()
              teams.setdefault(team_id, {
                  "id": team_id,
                  "name": row["team_name"].strip(),
                  "division": row["division"].strip(),
                  "captain": "",
                  "roster": {},
                  "swaps": [],
                  "scores": [],
                  "total_points": 0,
                  "overall_rank": None,
              })
              roster_by_team[team_id].append(athlete)
              if row["captain"].strip().upper() == "TRUE":
                  teams[team_id]["captain"] = athlete
                  captains[team_id] = athlete

          swaps_by_team = defaultdict(list)
          for row in tables["Swaps"]:
              if not row.get("team_id"):
                  continue
              team_id = row["team_id"].strip()
              wod = int(row["wod"])
              swapped_out = row["swapped_out"].strip()
              if swapped_out == captains.get(team_id):
                  raise ValueError(f"Captain cannot be swapped out: {team_id} {swapped_out}")
              swap = {
                  "wod": wod,
                  "swapped_out": swapped_out,
                  "swapped_in": row["swapped_in"].strip(),
                  "swapped_in_from_team": "External Roster",
              }
              swaps_by_team[team_id].append(swap)
              teams[team_id]["swaps"].append(swap)

          for team_id, team in teams.items():
              current = list(roster_by_team[team_id])
              for wod in range(1, 5):
                  for swap in sorted(swaps_by_team[team_id], key=lambda item: item["wod"]):
                      if swap["wod"] == wod:
                          current = [athlete for athlete in current if athlete != swap["swapped_out"]]
                          if swap["swapped_in"] not in current:
                              current.append(swap["swapped_in"])
                  if team["captain"] not in current:
                      raise ValueError(f"Captain missing from roster: {team_id}")
                  team["roster"][f"wod{wod}"] = list(current)

          wods = []
          released_wods = []
          for row in tables["WOD_Info"]:
              wod_id = int(row["wod_id"])
              released = row["released"].strip().upper() == "TRUE"
              if released:
                  released_wods.append(wod_id)
              wods.append({
                  "id": wod_id,
                  "name": row["name"].strip(),
                  "release_date": row["release_date"].strip(),
                  "released": released,
                  "format": row["format"].strip(),
                  "description": row["description"].strip(),
              })

          for wod in range(1, 5):
              rows = {row["team_id"].strip(): row for row in tables[f"WOD_{wod}"] if row.get("team_id")}
              for team_id, team in teams.items():
                  row = rows.get(team_id, {})
                  score = row.get("score", "").strip() or None
                  rank = int(row["rank"]) if row.get("rank", "").strip() else None
                  points = int(row["points"]) if row.get("points", "").strip() else None
                  team["scores"].append({"wod": wod, "score": score, "rank": rank, "points": points})

          divisions = sorted({team["division"] for team in teams.values()})
          for team in teams.values():
              team["total_points"] = sum(
                  score["points"] for score in team["scores"]
                  if score["wod"] in released_wods and score["points"] is not None
              )

          recent_wod = max(released_wods) if released_wods else 0
          for division in divisions:
              division_teams = [team for team in teams.values() if team["division"] == division]
              division_teams.sort(key=lambda team: (
                  team["total_points"],
                  next((score["rank"] for score in team["scores"] if score["wod"] == recent_wod), 999),
                  team["name"],
              ))
              for index, team in enumerate(division_teams, start=1):
                  team["overall_rank"] = index

          output = {
              "competition": "Chill Out Games 2026",
              "next_wod_release": next((f"{wod['release_date']}T00:00:00" for wod in wods if not wod["released"]), ""),
              "divisions": divisions,
              "wods": sorted(wods, key=lambda wod: wod["id"]),
              "teams": sorted(teams.values(), key=lambda team: team["id"]),
          }

          with open("data.json", "w", encoding="utf-8") as file:
              json.dump(output, file, indent=2)
              file.write("\n")
          PY

      - name: Commit updated data
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add data.json
          git diff --cached --quiet || git commit -m "chore: update leaderboard data [skip ci]"
          git push
```

Add your spreadsheet ID as a repository secret named `GOOGLE_SHEET_ID`.

Create `.github/workflows/deploy-pages.yml` with this content:

```yaml
name: Deploy GitHub Pages

on:
  push:
    branches:
      - main
  workflow_dispatch:
  workflow_run:
    workflows:
      - Update Leaderboard Data
    types:
      - completed

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  deploy:
    if: github.event_name != 'workflow_run' || github.event.workflow_run.conclusion == 'success'
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          ref: main

      - name: Configure Pages
        uses: actions/configure-pages@v5

      - name: Upload static site
        uses: actions/upload-pages-artifact@v3
        with:
          path: .

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

In the GitHub repository settings, go to Settings -> Pages and set Build and deployment Source to `GitHub Actions`.

## 6. Manual Update

To update data manually, open the GitHub repository, go to Actions, select `Update Leaderboard Data`, click `Run workflow`, and confirm. The `workflow_dispatch` trigger runs the same update job outside the Thursday schedule.

To redeploy the current static files manually, select `Deploy GitHub Pages`, click `Run workflow`, and confirm.

## 7. File Structure

```text
/
├── index.html
├── data.json
├── README.md
└── .github/
    └── workflows/
        ├── update-leaderboard.yml
        └── deploy-pages.yml
```
