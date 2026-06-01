# rpg-app

Next.js full-stack app for tracking, browsing, and running analytics against TTRPG characters across multiple systems and sources. Characters are cached locally in PostgreSQL with extra metadata. The app links back to characters and campaigns on their source sites rather than replicating full character sheets.

## Systems Supported

| System | Version | Source |
|---|---|---|
| D&D 5e | 2014 rules | D&D Beyond |
| D&D 5.5e | 2024 rules (One D&D) | D&D Beyond |
| Daggerheart | 1.x | Manual / future source |

Designed to be extended to additional systems and sources without schema changes.

## Architecture

- **Framework:** Next.js (App Router, TypeScript)
- **UI:** AWS Cloudscape Design System
- **Database:** PostgreSQL via CloudNativePG (CNPG) operator — already deployed on pibernetes
- **Credentials:** 1Password Kubernetes Operator → K8s Secret → env vars
- **Deployment:** Kubernetes (multi-arch: `linux/amd64` + `linux/arm64`)
- **No nodeSelector** — multi-arch image runs on any node

## Sync Architecture

Sync lives **outside this repo**. The app exposes an upsert API; any external process that speaks HTTP can populate it. The K8s deployment never contacts D&D Beyond or any other external source.

### Sync Contract

**No authentication required.** Single-user personal app, private deployment (ingress disabled by default, Tailscale for remote access).

Upsert on `(source, source_id)` — preserves manually-set fields (`labels`, `currently_playing`) on conflict.

**`POST /api/characters`**
```json
{
  "name": "Arannis",
  "system": "5.5e",
  "source": "dndbeyond",
  "source_id": "12345678",
  "source_url": "https://www.dndbeyond.com/characters/12345678",
  "level": 8,
  "class": "Warlock",
  "subclass": "The Archfey",
  "species": "Elf",
  "background": "Sage",
  "raw_data": { }
}
```

**`POST /api/campaigns`** — same upsert pattern.

All fields except `name`, `system`, and `source` are optional. Omit `source_id` for manual entries with no external ID.

## URLs

| Environment | URL |
|---|---|
| Production (public) | `https://rpg.salvo.services` |
| Production (Tailscale) | `https://rpg.tail5017c.ts.net` |
| Local dev | `http://localhost:3000` |

## Database Architecture

PostgreSQL is used for all data storage, managed by the **CloudNativePG (CNPG)** operator already running on pibernetes. Character JSON from external sources is deeply nested and schema-variable per system — stored in full in a `raw_data JSONB` column. Fields needed for filtering, display, and analytics are **promoted to real indexed columns**.

> **Infrastructure decision — CNPG:** This app uses a CNPG `Cluster` CRD rather than a raw StatefulSet or self-managed PostgreSQL. CNPG provides replication, automated backups, and proper lifecycle management at no extra infra cost given it's already deployed on pibernetes. The `DATABASE_URL` connection string format is identical to standard PostgreSQL — no app-level changes needed. **If ever making this repo public or deploying elsewhere:** replace `helm/rpg-app/templates/cnpg-cluster.yaml` with a standard StatefulSet or point `DATABASE_URL` at an external PostgreSQL instance, and remove the CNPG CRD dependency.

### `characters` table

| Column | Type | Description |
|---|---|---|
| `id` | `uuid` PK | Internal ID |
| `name` | `text` | Character name |
| `system` | `text` | `5e`, `5.5e`, `daggerheart` |
| `source` | `text` | `dndbeyond`, `manual`, etc. |
| `source_id` | `text` | External ID at source (nullable) |
| `source_url` | `text` | Direct link to character on source site |
| `portrait_url` | `text` | Character portrait/avatar image URL (nullable) |
| `campaign_id` | `uuid` FK | References `campaigns.id` — set when campaign is tracked in this tool (nullable) |
| `external_campaign_id` | `text` | Source's campaign ID when campaign is NOT in this tool (nullable) |
| `external_campaign_name` | `text` | Source's campaign name when campaign is NOT in this tool (nullable) |
| `currently_playing` | `boolean` | Active/favorite flag |
| `labels` | `text[]` | Freeform tags |
| `level` | `integer` | Current level or tier (indexed) |
| `class` | `text` | Primary class name (indexed) |
| `subclass` | `text` | Subclass name (nullable, indexed) |
| `species` | `text` | Race/species/ancestry (nullable, indexed) |
| `background` | `text` | Background/community (nullable, indexed) |
| `raw_data` | `jsonb` | Full character JSON from source — no schema constraint |
| `last_synced_at` | `timestamptz` | Last sync from source (nullable) |
| `created_at` | `timestamptz` | |
| `updated_at` | `timestamptz` | |

**Indexes:** `level`, `class`, `subclass`, `species`, `background`, `currently_playing`, `campaign_id`, `source`, `system`. GIN index on `labels`.

> `campaign_id` and `external_campaign_*` are mutually exclusive in practice. Sync logic: match character's source campaign ID against tracked campaigns by `source_id` → hit sets `campaign_id`; miss sets `external_campaign_id` + `external_campaign_name`. UI: FK present → internal CampaignBadge with link; external fields only → name + link out to source site; neither → no campaign shown.

> `portrait_url` for D&D Beyond characters points to their publicly accessible CDN. No auth required to load in browser. Images are cached by the browser after first load — no repeated requests to D&D Beyond. If URLs ever become auth-gated, caching to a PV is a future sync-side enhancement — no app schema change needed.

> The former `system_data` column is replaced by `raw_data`. All system-specific fields live there. Analytics queries operate only on the promoted columns above.

### Example `raw_data` content

```json
// D&D 5e / 5.5e character from D&D Beyond
{
  "multiclass": [{ "class": "Warlock", "level": 3 }, { "class": "Sorcerer", "level": 5 }],
  "ability_scores": { "str": 10, "dex": 14, "con": 13, "int": 8, "wis": 12, "cha": 18 },
  "hp": { "current": 54, "max": 54, "temp": 0 },
  "ac": 15,
  "proficiency_bonus": 4,
  "spells": [],
  "features": [],
  "inventory": []
}

// Daggerheart
{
  "tier": 2,
  "community": "Highborne",
  "hope": 3,
  "evasion": 12,
  "damage_thresholds": { "major": 10, "severe": 22 }
}
```

### `campaigns` table

| Column | Type | Description |
|---|---|---|
| `id` | `uuid` PK | Internal ID |
| `name` | `text` | Campaign name |
| `system` | `text` | `5e`, `5.5e`, `daggerheart` |
| `source` | `text` | `dndbeyond`, `manual`, etc. |
| `source_id` | `text` | External campaign ID (nullable) |
| `source_url` | `text` | Link to campaign on source site (nullable) |
| `dm` | `text` | DM/GM name (nullable) |
| `is_dm` | `boolean` | True if you are the DM/GM for this campaign |
| `vtt` | `text` | VTT platform name, e.g. `foundry`, `roll20`, `owlbear`, `none` (nullable) |
| `vtt_url` | `text` | Direct link to the VTT session/game — used for the Play button (nullable) |
| `active` | `boolean` | Is campaign currently running |
| `created_at` | `timestamptz` | |
| `updated_at` | `timestamptz` | |

> `is_dm` distinguishes campaigns you run from campaigns you play in. Relevant for display ("My Campaigns" vs. "Playing In") and future features.

> `vtt_url` is the link surfaced as the **Play** button on character cards. Walk: character → `campaign_id` → `vtt_url` → open. If `campaign_id` is null (character in an untracked campaign), no Play button is shown.

## Navigation Structure

Cloudscape `AppLayout` with:

- **Top navigation (`TopNavigation`):** d20 icon + "RPG App" title. Use Font Awesome's `fa-dice-d20` glyph via `@fortawesome/react-fontawesome` + `@fortawesome/free-solid-svg-icons`. Color: dragon red `#C41E3A`. Cloudscape's built-in icon set does not include a d20.

- **Favicon:** `app/icon.svg` — the Font Awesome `fa-dice-d20` SVG path rendered in dragon red `#C41E3A`. Next.js App Router picks up `app/icon.svg` automatically as the favicon. No separate `favicon.ico` needed for modern browsers.

- **Side navigation (`SideNavigation`):** Module-level nav items:
  - **Characters** → `/characters`
  - **Campaigns** → `/campaigns`

Analytics is not a primary nav item in v1 — surfaced within the Characters module as a dashboard section below the browser.

## Directory Structure

```
rpg-app/
  app/
    api/
      health/
        route.ts              # GET /api/health → { status: "ok" }
      characters/
        route.ts              # GET /api/characters, POST /api/characters
        [id]/
          route.ts            # GET, PUT, DELETE /api/characters/[id]
      campaigns/
        route.ts              # GET /api/campaigns, POST /api/campaigns
        [id]/
          route.ts            # GET, PUT /api/campaigns/[id]
      analytics/
        route.ts              # GET /api/analytics
    characters/
      page.tsx                # Characters module page
    campaigns/
      page.tsx                # Campaigns module page
    layout.tsx                # AppLayout shell with TopNav + SideNav
    page.tsx                  # Redirect → /characters
  components/
    icons/
      D20Icon.tsx             # Custom d20 SVG icon component
    characters/
      CurrentlyPlaying.tsx    # Hero section — active characters strip
      CharacterBrowser.tsx    # Filterable/searchable character grid
      CharacterCard.tsx       # Single character card (used in both views)
      CharacterDetail.tsx     # Side panel — metadata, labels, links, sync status
      LabelEditor.tsx         # Add/remove labels on a character
    campaigns/
      CampaignBrowser.tsx     # Campaign list/grid
      CampaignDetail.tsx      # Campaign side panel with linked characters
      CampaignBadge.tsx       # Inline campaign chip with link (used in CharacterCard)
    analytics/
      AnalyticsDashboard.tsx  # Charts and stat widgets
  lib/
    db.ts                     # PostgreSQL client (node-postgres / pg)
    queries/
      characters.ts           # Character CRUD + filter queries
      campaigns.ts            # Campaign CRUD queries
      analytics.ts            # Aggregate queries against promoted columns
  migrations/
    001_initial.sql           # characters + campaigns tables
  helm/
    rpg-app/
      Chart.yaml
      values.yaml
      _helpers.tpl
      templates/
        deployment.yaml
        service.yaml
        ingress.yaml          # disabled by default; host: rpg.salvo.services
        cnpg-cluster.yaml     # CNPG Cluster CRD — remove if using external PostgreSQL
  k8s/
    onepassword-item.yaml     # syncs DB credentials from 1Password
  .vscode/
    launch.json
    tasks.json                # pre-launch: op inject -i .env.op -o .env.local
  .env.op
  .gitignore                  # excludes .env.local
  .dockerignore
  Dockerfile
  next.config.ts              # output: 'standalone'
  package.json                # name: rpg-app
  tsconfig.json
  SPEC.md
```

## API Endpoints

### Characters

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/characters` | List all characters. Query params: `system`, `currently_playing`, `label`, `campaign_id`, `source` |
| `POST` | `/api/characters` | Create a character (manual or sync) |
| `GET` | `/api/characters/[id]` | Get single character (includes `raw_data`) |
| `PUT` | `/api/characters/[id]` | Update metadata, labels, currently_playing |
| `DELETE` | `/api/characters/[id]` | Delete character |

### Campaigns

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/campaigns` | List all campaigns |
| `POST` | `/api/campaigns` | Create a campaign |
| `GET` | `/api/campaigns/[id]` | Get campaign + associated characters |
| `PUT` | `/api/campaigns/[id]` | Update campaign metadata |

### Analytics

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/analytics` | Aggregates: class distribution, system breakdown, level spread, label frequency, active vs. retired |
| `GET` | `/api/health` | `{ status: "ok" }` |

## UI Views

### Module: Characters (`/characters`)

#### Currently Playing — hero section at top of page
- Cloudscape `Cards` in a horizontal strip
- Filtered to `currently_playing = true`
- Each card: name, system badge, class + subclass, level, campaign link, source link
- One-click toggle to remove from currently playing

#### Character Browser
- Cloudscape `Table` or `Cards` (user-toggleable)
- Filter bar: system, source, label, campaign, currently_playing
- Search by name
- Each row/card: name, system, source, class/subclass, level, species, campaign badge, labels, currently_playing toggle, source link
- Click → `CharacterDetail` side panel

#### Character Detail (side panel)
- Cloudscape `ColumnLayout` — promoted metadata fields
- `LabelEditor` — add/remove freeform labels
- Currently Playing toggle
- Source link (opens character on D&D Beyond or source site)
- Campaign link (via `CampaignBadge`)
- Last synced timestamp
- `raw_data` shown as a collapsible raw JSON view (for debugging/reference)

#### Analytics Dashboard (below browser, collapsed by default)
- Cloudscape `ExpandableSection` wrapping stat widgets
- Class distribution (bar chart)
- System breakdown (pie/donut)
- Level spread (histogram)
- Label frequency (bar chart)

### Module: Campaigns (`/campaigns`)

#### Campaign Browser
- Cloudscape `Table` or `Cards`
- Filter bar: system, source, active/retired, **owned** (`is_dm = true` → "Running" / `is_dm = false` → "Playing In")
- Default grouping: **Running** campaigns first, then **Playing In**
- Each row/card: name, system, DM (if playing in), VTT badge, active status, source link, character count, Play button (if `vtt_url` set)

#### Campaign Detail (side panel)
- Campaign metadata
- Linked characters list (names, class, level, currently playing flag)
- Source link

## Environment Variables

| Variable | Description |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string |

Stored in 1Password, synced via the Operator. Add a new item to the `pibernetes` vault (e.g. `rpg-app-db`) with a `connection-string` field.

```yaml
# k8s/onepassword-item.yaml
apiVersion: onepassword.com/v1
kind: OnePasswordItem
metadata:
  name: rpg-app-db
  namespace: default
spec:
  itemPath: "vaults/pibernetes/items/rpg-app-db"
```

```yaml
# In Deployment env:
- name: DATABASE_URL
  valueFrom:
    secretKeyRef:
      name: rpg-app-db
      key: connection-string
```

## Local Development

```sh
# .env.op (committed)
DATABASE_URL=op://pibernetes/rpg-app-db/connection-string
```

VS Code pre-launch task:
```sh
op inject -i .env.op -o .env.local
```

For local dev, `DATABASE_URL` can point to a local PostgreSQL instance or a port-forwarded cluster DB (`kubectl port-forward svc/postgres 5432:5432`).

## Helm Chart

`helm/rpg-app/` — single chart. Does not manage PostgreSQL directly; database is provided by the CNPG operator already running on pibernetes.

Key `values.yaml` defaults:
```yaml
image:
  repository: ghcr.io/buzzsurfr/rpg-app
  tag: latest

ingress:
  enabled: false
  host: rpg.salvo.services

# No nodeSelector — multi-arch image handles scheduling

cnpg:
  cluster:
    name: rpg-app-db
    instances: 1        # increase for HA
    storageSize: 5Gi
```

The chart includes `helm/rpg-app/templates/cnpg-cluster.yaml` — a CNPG `Cluster` resource. The CNPG operator must be installed on the target cluster. `DATABASE_URL` is populated from the CNPG-managed credential secret.

> **Portability note:** If deploying to a cluster without CNPG, remove `cnpg-cluster.yaml` and point `DATABASE_URL` at an external PostgreSQL instance. No app code changes required.

Liveness and readiness probes wired to `GET /api/health`.

## Dockerfile

Multi-stage, multi-arch (`linux/amd64` + `linux/arm64`):

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

Build and push:
```sh
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t ghcr.io/buzzsurfr/rpg-app:latest .
```

Requires `output: 'standalone'` in `next.config.ts`.

## npm Packages

```
@cloudscape-design/components
@cloudscape-design/global-styles
@fortawesome/react-fontawesome
@fortawesome/free-solid-svg-icons
@fortawesome/fontawesome-svg-core
pg
@types/pg
uuid
@types/uuid
```

## Error Response Format

```json
{ "error": "description" }
```

## Open Questions for Claude Code

1. **CNPG cluster name** — confirm the CNPG `Cluster` name and namespace to use for `rpg-app-db`. Default to `rpg-app-db` in `default` namespace unless told otherwise.
2. **Daggerheart** — no sync source. Characters entered manually via API. Schema supports it with no changes needed.
3. **Display name** — repo and package are `rpg-app`; the UI shows "RPG App" in `TopNavigation`. Update if a different display name is preferred.
4. **Portrait caching** — `portrait_url` stored as-is from the sync source. If D&D Beyond CDN URLs ever require auth, a future enhancement would cache images to a PV. No action needed now.

## First Task for Claude Code

1. Scaffold full directory structure per the layout above
2. `migrations/001_initial.sql` — `characters` and `campaigns` tables with all indexes
3. `lib/db.ts` — PostgreSQL client using `DATABASE_URL`
4. `lib/queries/characters.ts`, `campaigns.ts`, `analytics.ts`
5. All API routes — `/api/health`, `/api/characters` (with upsert logic on `source`+`source_id`), `/api/campaigns`, `/api/analytics`
6. Font Awesome `fa-dice-d20` icon in dragon red (`#C41E3A`) wired into `TopNavigation`; same SVG path exported as `app/icon.svg` for the favicon
7. `app/layout.tsx` — Cloudscape `AppLayout` with `TopNavigation` (d20 icon + title) and `SideNavigation` (Characters, Campaigns)
8. `app/page.tsx` — redirect to `/characters`
9. `app/characters/page.tsx` with `CurrentlyPlaying`, `CharacterBrowser`, `CharacterDetail`, `AnalyticsDashboard`
10. `app/campaigns/page.tsx` with `CampaignBrowser`, `CampaignDetail`
11. All components under `components/characters/`, `components/campaigns/`, `components/analytics/`
12. `k8s/onepassword-item.yaml`
13. `helm/rpg-app/` — full chart including optional PostgreSQL StatefulSet
14. `.env.op`, `.vscode/launch.json`, `.vscode/tasks.json`
15. `Dockerfile` — multi-stage, multi-arch
16. `.gitignore`, `.dockerignore`
