## ESPN fantasy ADP API quirks: sort requirement, page-size cap, nested ownership
**Date:** 2026-09-07
**Context:** ESPN lm-api-reads.fantasy.espn.com public default-league endpoint (fantasy-researcher pipeline, Python httpx)
**Tags:** espn, fantasy-football, api, pagination, x-fantasy-filter, schema

### Problem / Observation

Probing ESPN's undocumented public draft-trends endpoint
(`https://lm-api-reads.fantasy.espn.com/apis/v3/games/ffl/seasons/2026/segments/0/leaguedefaults/3?view=kona_player_info`)
produced three surprises: (1) a `400 FILTER_LIMIT_MISSING_SORT` unless the
X-Fantasy-Filter JSON includes a sort key; (2) `limit: 1000` in the filter
returned players with `ownership: null` while smaller limits returned real
ownership/ADP; (3) ownership data is not a top-level property of each player
item — it is nested inside `item["player"]["ownership"]`.

### Resolution / Insight

- The filter must carry a sort, e.g.
  `{"players": {"limit": 100, "offset": 0, "sortPercOwned": {"sortAsc": false, "sortPriority": 1}}}`.
  Sorting by percent-owned keeps ADP players contiguous at the top.
- Bisect the limit: ownership data is served only for limit <= 100. Page with
  `limit: 100` + increasing `offset` instead of one big limit.
- Read ADP via `item["player"]["ownership"]["averageDraftPosition"]`; treat a
  full page with zero ADP-bearing players (or a short page) as the end of
  pagination.
- The endpoint is undocumented: keep all shape assumptions behind one adapter
  with a fail-closed validate() so drift breaks loudly instead of silently.

### Commands / Code

```python
filter = json.dumps({"players": {
    "limit": 100, "offset": offset,
    "sortPercOwned": {"sortAsc": False, "sortPriority": 1},
}})
headers = {"x-fantasy-filter": filter, "accept": "application/json"}
# GET the URL above with those headers; iterate offset in steps of 100.
```
