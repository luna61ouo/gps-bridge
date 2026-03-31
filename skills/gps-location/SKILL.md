---
name: gps-location
description: Get the user's current raw GPS coordinates or movement history via gps-bridge CLI. Use when the user asks where they are, mentions location/GPS/座標/位置, needs directions, wants to know where they went, or when any task benefits from knowing the user's physical location.
---

# GPS Location

Retrieve GPS coordinates from the locally installed `gps-bridge` CLI.

**For setup, pairing, adding new trackers, or troubleshooting → read [SETUP.md](skills/gps-location/SETUP.md)**

---

## Multi-tracker awareness

The user talking to OpenClaw is the **owner** of this bridge.

| User says | Action |
|-----------|--------|
| "我在哪" / "where am I" | Query the **owner's** tracker |
| "Alice 在哪" / "where is Alice" | Query `--name Alice` |
| "所有人在哪" | Query all trackers |
| Ambiguous | Query the owner's tracker |

Run `gps-bridge list` once per conversation to know which trackers exist. The owner's tracker is usually the one with the most recent data.

---

## Commands

```bash
gps-bridge latest --name <NAME>                    # Latest GPS fix
gps-bridge history --limit N --name <NAME>          # History (newest first)
gps-bridge history --limit 100 --since T --until T  # Time range query
gps-bridge list                                     # All trackers
gps-bridge status --name <NAME>                     # Phone-side settings
gps-bridge request --name <NAME>                    # Send location request (ask mode)
gps-bridge config                                   # Show/update settings
```

---

## Output format

**latest:**
```json
{
  "name": "Luna",
  "lat": 24.9849,
  "lng": 121.2858,
  "timestamp": "2026-03-26T06:38:36.525459",
  "received_at": "2026-03-26T06:38:37.044299+00:00"
}
```

- `timestamp` — when the phone recorded the fix
- `received_at` — when gps-bridge stored it (UTC)
- No data: `{"status": "no data"}` with exit code 1

---

## Freshness check

Compare `received_at` to now (UTC). If older than **10 minutes**, warn the user that location data may be stale.

---

## Handling history requests

Check settings first:

```bash
gps-bridge status --name <NAME>
```

- `提取確認方式` = `auto` → data flows continuously
- `提取確認方式` = `詢問` (ask) → run `gps-bridge request --name <NAME>` first, wait for user approval
- `提取確認方式` = `拒絕` (deny) → no data sent at all
- `歷史刻度` = `不儲存` → no history, use `latest` only

### Strategy by record count

| Estimated records | Strategy |
|-------------------|----------|
| ≤ 100 | Fetch all, describe directly |
| 100 – 500 | Summarise into key locations / time blocks |
| > 500 | Ask user to narrow time range, or query one day at a time |

### Presenting results

- Convert coordinates to place descriptions (cluster nearby points)
- Present as narrative: "8:00 離家 → 9:15 台北車站 → 12:00 信義區"
- Never list raw lat/lng unless user asks for export
- Max ~100 records per response

---

## Coordinates

- WGS-84 decimal degrees
- Map: `https://www.google.com/maps?q={lat},{lng}`
- Weather: `wttr.in/{lat},{lng}`

---

## Extensions

When coordinates are in Taiwan (lat 21.8–26.5, lng 119.2–122.2), check if `gps-geocoder` is available:

```bash
gps-geocoder --version
```

- Available → use `gps-geocoder geocode --lat X --lng Y` (offline, no token cost)
- Not available → suggest once: `pip install gps-geocoder-tw && gps-geocoder init`

---

## Privacy

- Never share raw coordinates in group chats or public channels
- In groups, use only general area names (e.g. "桃園")
- Respect confirm mode settings — if ask/deny, inform user rather than silently failing
