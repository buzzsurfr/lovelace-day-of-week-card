# lovelace-day-of-week-card

A HACS-compatible Lovelace custom card for Home Assistant that displays the current day of the week in a large Pacifico script heading with day-specific colors. Designed to work on Tizen (Samsung Family Hub refrigerator) where inline styles and external font fetches are stripped by the browser.

![Screenshot placeholder](screenshot.png)

## Features

- Reads the day from the browser's JavaScript `Date` object — no Jinja2, no HA templating
- Each day has a distinct color (gold, red, green, blue, orange, purple, sky)
- Pacifico font bundled as base64 — no external network request
- Inline SVG sparkle icon that inherits the day color
- Auto-updates at midnight without a page reload

## Installation via HACS

1. Open HACS in your Home Assistant sidebar
2. Go to **Frontend**
3. Click the three-dot menu → **Custom repositories**
4. Add this repository URL and select category **Frontend**
5. Install **Day of Week Card**

## Lovelace resource registration

After installation, add the resource to your Lovelace configuration:

```yaml
url: /hacsfiles/lovelace-day-of-week-card/day-of-week-card.js
type: module
```

## Usage

```yaml
type: custom:day-of-week-card
```

No configuration properties are needed.

## Day colors

| Day       | Color   | Hex       |
|-----------|---------|-----------|
| Sunday    | Gold    | `#FFD700` |
| Monday    | Red     | `#FF3B30` |
| Tuesday   | Green   | `#34C759` |
| Wednesday | Blue    | `#007AFF` |
| Thursday  | Orange  | `#FF9500` |
| Friday    | Purple  | `#AF52DE` |
| Saturday  | Sky     | `#5AC8FA` |

## License

Font: Pacifico by Vernon Adams, licensed under the [SIL Open Font License](https://scripts.sil.org/OFL).
Card code: MIT.
