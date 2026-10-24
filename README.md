# Study Buddy — laptop dashboard

Plain HTML/CSS/JS. No build step, no backend.

## Run it

```bash
cd study-buddy-web
python3 -m http.server 8000
```

Open **http://localhost:8000** in **Chrome or Edge**. Web Serial and the Idle Detection API only exist in Chromium browsers, and they need a secure origin (localhost counts).

No device? Open **Demo controls** (bottom right) or use the keys:

| Key | Does |
|---|---|
| `Y` | Toggle a YouTube drift |
| `I` | Toggle idle |
| `F` | Toggle Dev (a friend) drifting, which pauses the circle bonus |
| `S` | Speed ×30 |
| `E` | Jump to the end of the session |
| `Space` | Peek at the timer for 5 s |

## Files

| File | What |
|---|---|
| `index.html` | All three views: Setup, Session, Time card |
| `styles.css` | Steampunk theme, tokens at the top match the design canvas |
| `pet.js` | Pet sprite as 16×16 rectangle data plus palettes. **Port this to the firmware** so the device draws the pet in code instead of from images |
| `app.js` | Session engine, rules, Web Serial link, idle detection, helper socket |

## Rules the app enforces

- Break = ⅕ to ⅓ of study time (15 → 3–5, 25 → 5–8, 45 → 9–15, 60 → 12–20). Default 25 / 5.
- Four sessions per set; a 15-minute long rest after the fourth.
- XP: +1 per focused minute × study-circle multiplier (friends in circle, capped at 3×). The bonus pauses whenever anyone in the circle is drifting.
- Under 50% focus: no XP earned and −10 XP, never below the start of the current level. 500 XP per level.
- Hearts: +2 at 80%+, +1 at 60%+, −1 under 50% (0–5).
- A minute is punched as a drift if 15+ seconds of it drifted.
- Accessories unlock at Lv 5 (goggles), 10 (bow tie), 15 (top hat).

## Device protocol (USB serial, 115200 baud, one JSON object per line)

**Laptop → device**

```json
{"t":"config","user":"Sidak","pet":{"name":"Mochi","body":"bot","face":"plain","color":"brass"},"level":3,"xp":340,"hearts":3}
{"t":"start","study":25,"brk":5,"goal":"Problem set 4"}
{"t":"drift","kind":"site","label":"YouTube","active":true}
{"t":"drift","active":false}
{"t":"end","pct":90,"xp":44,"hearts":2,"level":3}
{"t":"break","minutes":5}
{"t":"finished","value":"yes"}
```

**Device → laptop**

```json
{"t":"hello","fw":"0.1"}
{"t":"knob","ev":"press"}
{"t":"knob","ev":"hold"}
{"t":"state","left":522}
{"t":"circle","friends":[{"name":"Ava","focused":true},{"name":"Dev","focused":false}]}
{"t":"pet","hearts":4}
```

- `knob press` → peek at the timer for 5 s. `knob hold` (2 s) → end the session early.
- `state.left` (seconds) lets the device own the countdown; the dashboard follows it.
- `circle` comes from ESP-NOW: the device relays which friends' devices are nearby and focused.
- Lines that aren't JSON are ignored, so `Serial.println` debug output is safe.

## Detecting distracting sites and apps

A web page **can't see other tabs or apps**. The dashboard listens on `ws://localhost:8765` for a helper (a small Chrome extension or Python script) that sends the active window whenever it changes:

```json
{"t":"active","app":"Google Chrome","title":"lofi beats - YouTube","url":"https://www.youtube.com/watch?v=..."}
```

Anything matching the list in `DISTRACTING` (top of `app.js`) starts a screen drift; anything else ends it. Idle (no keyboard or mouse for 60 s) comes from the browser's Idle Detection API; Chrome asks permission when a session begins.
