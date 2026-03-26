---
name: gps-location
description: Get the user's current GPS location via gps-bridge CLI. Use when the user asks where they are, asks for nearby places, mentions location/GPS/座標/位置, needs directions, or when any task benefits from knowing the user's physical location (e.g. weather with no city specified, finding nearby restaurants, travel time estimates). Also use when checking if location data is available or stale.
---

# GPS Location

Retrieve the user's real-time GPS coordinates from the locally installed `gps-bridge` CLI.

## Commands

```bash
# Latest fix (single JSON object)
gps-bridge latest

# Recent history (array, newest first)
gps-bridge history --limit N
```

## Output format

```json
{
  "lat": 24.9849,
  "lng": 121.2858,
  "timestamp": "2026-03-26T06:38:36.525459",
  "received_at": "2026-03-26T06:38:37.044299+00:00"
}
```

- `timestamp` — when the phone recorded the fix (device clock)
- `received_at` — when gps-bridge stored it (UTC)
- If no data exists, output is `{"status": "no data"}` with exit code 1.

## Freshness check

Compare `received_at` to now (UTC). If older than **10 minutes**, warn the user that location data may be stale (phone might be off or out of range).

## Using the coordinates

- For reverse geocoding or place lookup, pass lat/lng to a web search like `"restaurants near 24.9849,121.2858"`.
- For weather queries without a specified city, use the coordinates with the weather skill or wttr.in (`wttr.in/24.9849,121.2858`).
- For map links, use: `https://www.google.com/maps?q={lat},{lng}`
- Coordinates are in WGS-84 decimal degrees.

## Privacy

GPS data is sensitive. Never share raw coordinates in group chats or public channels. When reporting location in groups, use only a general area name (e.g. "桃園" not exact lat/lng).
