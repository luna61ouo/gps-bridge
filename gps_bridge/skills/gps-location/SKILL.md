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

## How to query location

**Prefer `gps-geocoder latest` when installed** — it returns a human-readable location (place name or address) in one step. Fall back to `gps-bridge latest` only when:

- `gps-geocoder` is not installed (check with `gps-geocoder --version`)
- The user explicitly asks for raw coordinates
- You need `confirm_mode` / `hint` / `stale` fields for ask-mode handling

### Decision tree

```
User asks about location →
  ├─ gps-geocoder installed? → gps-geocoder latest --name <NAME>
  │    └─ Returns: "你在 {家|公司|路名+門牌|座標}"
  │
  └─ gps-geocoder not installed, or need raw coords / control info →
       gps-bridge latest --name <NAME>
       └─ Returns: {lat, lng, confirm_mode, hint, stale, ...}
```

The `gps-bridge latest` output includes `confirm_mode` and `hint` fields when relevant. **Follow the `hint` if present.** No need to run a separate `status` command.

### Example outputs

**Fresh data (auto mode):**
```json
{
  "name": "Luna",
  "lat": 24.9849,
  "lng": 121.2858,
  "confirm_mode": "auto",
  ...
}
```
→ Use the coordinates directly.

**No data (ask mode):**
```json
{
  "status": "no data",
  "confirm_mode": "ask",
  "hint": "Phone is in ask mode. Run: gps-bridge request --name Luna"
}
```
→ Run the command in `hint`, then tell the user: "已發送位置請求，請在手機上確認。" Wait ~15 seconds, then re-run `gps-bridge latest`.

**Stale data (ask mode):**
```json
{
  "name": "Luna",
  "lat": 24.9849,
  "lng": 121.2858,
  "confirm_mode": "ask",
  "stale": true,
  "hint": "Data is 45 min old. Phone is in ask mode. Run: gps-bridge request --name Luna"
}
```
→ Show the old location with a note that it may be outdated. Run `hint` command to request a fresh fix.

**Deny mode:**
```json
{
  "status": "no data",
  "confirm_mode": "deny",
  "hint": "Phone is set to deny all location requests."
}
```
→ Inform the user that their phone is blocking location sharing.

**No confirm_mode (settings not yet received):**
```json
{"status": "no data"}
```
→ Bridge hasn't received settings yet. Check if `gps-bridge connect` is running and phone tracking is started.

---

## Handling history requests

Check settings first with `gps-bridge status --name <NAME>`:

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

## Extensions: gps-geocoder

`gps-geocoder` turns raw coordinates into a readable location. Check once per session:

```bash
gps-geocoder --version
```

### Commands

| Command | Purpose |
|---------|---------|
| `gps-geocoder latest --name <NAME>` | Latest GPS fix + location name (one-shot) |
| `gps-geocoder history --limit N --name <NAME>` | History with location names |
| `gps-geocoder geocode --lat X --lng Y` | Geocode arbitrary coordinates |
| `gps-geocoder places near --lat X --lng Y` | Nearby saved places |
| `gps-geocoder maps` | List installed regional maps |

### Priority

When reverse geocoding, gps-geocoder returns the **first match** in this order:

1. **Personal place within 50m** → `"家"`, `"公司"` (from `places:at`)
2. **Map data** → `"台北市信義區忠孝東路四段123號"` (from `map:tw`)
3. **Personal place within 500m** (fallback hint) (from `places:near`)
4. **Raw coordinates** (from `none`)

### If not installed

Suggest once: `pip install gps-geocoder`
To install map data: `pip install "gps-geocoder[maps]"` then `gps-geocoder init tw` (or `jp`, `kr`).

---

## Privacy

- Never share raw coordinates in group chats or public channels
- In groups, use only general area names (e.g. "桃園")
- Respect confirm mode settings — if ask/deny, inform user rather than silently failing
