# FoodSwipe

Pantry, crop, and garden accessory tracker for **Philinity** — Phil & Trinity Sigler's Boise foothills food resilience system.

Part of the Philinity app suite alongside [Command Center](https://siglerventures.github.io/philinity), [Task Board](https://siglerventures.github.io/taskboard), and [Eats](https://siglerventures.github.io/eats).

**Live:** [siglerventures.github.io/foodswipe](https://siglerventures.github.io/foodswipe) · [foodswipe.net](https://foodswipe.net)

---

## What it does

Single-pane inventory for everything food-related in the Philinity system:

- **Pantry** — canned goods, grains, oils, emergency supplies with purchase/expiration dates and shelf life
- **Crops** — plants and seeds with variety, planting location, sublocation (e.g. Plant Tower Tier 2 → Outdoor Garden Bed), and Boise 6b/7a planting windows
- **Accessories** — grow bags, soils, pots, amendments, and what they're used for

Location-aware — answers questions like *"which row are my Syrah grapes in?"* or *"where's the German Extra Hardy garlic stored?"*.

---

## Features

- 🔐 Google Sign-In with UID whitelist (Phil & Trinity only)
- 🔍 Unified filtering — type, status, category, full-text search
- 📊 Stats bar — total items, expiring &lt; 6 months, need-to-buy count
- 🤖 AI chat seeded with full inventory + Phil's food project instruction set (Boise zone, closed-loop principles, seasonality, hard constraints)
- 📷 Photo upload + camera capture, stored in Firebase Storage
- ✏️ Full CRUD with 20+ fields per item
- 🧠 AI can `add`, `update`, or run multiple `actions` — JSON parsed via brace-counting extractor so nested `data:{}` blocks work

---

## Tech stack

- Single-file static HTML — no build step
- Firebase Realtime Database (path: `foodswipe`)
- Firebase Storage (photos at `foodswipe/{id}/photo-*.jpg`)
- Firebase Auth (Google provider)
- Anthropic Claude Sonnet 4 via the shared `askAI` Cloud Function
- Playfair Display + DM Sans typography; sage-green / terracotta dark palette

---

## Setup

### 1. Cloud Function

Add `foodswipe` to `KEY_MAP` in the `askAI` function (`functions/index.js`):

```js
const KEY_MAP = {
  'eats':      'config/Philinity-Eats',
  'cc':        'config/Philinity-CC',
  'veritas':   'config/veritas_key',
  'taskboard': 'config/Taskboard',
  'foodswipe': 'config/Philinity-FoodSwipe',
};
```

Deploy:

```bash
firebase deploy --only functions:askAI
```

### 2. Firebase Realtime DB

Store the Anthropic API key at path `config/Philinity-FoodSwipe` (string value).

Add to the database rules:

```json
"foodswipe": {
  ".read": "auth != null",
  ".write": "auth != null"
}
```

### 3. Storage rules

Add to `storage.rules`:

```
match /foodswipe/{allPaths=**} {
  allow read, write: if request.auth != null;
}
```

### 4. Seed data

Import `foodswipe-seed.json` via Firebase Console → Realtime Database → ⋮ → Import JSON. Creates the `/foodswipe` node with 61 items (36 pantry · 13 crops · 12 accessories).

### 5. GitHub Pages

Enable Pages on `main` branch, root directory. Site publishes at `siglerventures.github.io/foodswipe`.

### 6. Custom domain (optional)

Add a `CNAME` file containing `foodswipe.net` and point a DNS CNAME record at `siglerventures.github.io`.

---

## AI command schema

The AI can issue `add`, `update`, or batched `actions` commands returned as JSON in its reply. The client extracts them via brace-counting (handles nested objects).

### Add pantry item

```json
{"action":"add","data":{
  "type":"pantry",
  "name":"Quinoa",
  "category":"Grains & Starches",
  "categoryEmoji":"🌾",
  "location":"Pantry",
  "quantity":"5 lbs",
  "price":12.99,
  "shelfLifeYears":5,
  "purchaseDate":"2026-04-23",
  "expirationDate":"2031-04-23",
  "status":"have",
  "vendor":"Costco",
  "notes":"Rinsed / tri-color"
}}
```

### Add crop

```json
{"action":"add","data":{
  "type":"crop",
  "name":"Syrah Grapes",
  "category":"Outdoor",
  "categoryEmoji":"🍇",
  "variety":"Syrah (clone 877)",
  "location":"Outdoor Garden Bed - Row 2",
  "sublocation":"South-facing hillside",
  "status":"planted-outdoor",
  "purpose":"Wine production, long-term perennial",
  "vendor":"Edwards Nursery",
  "plantingWindow":"Late Mar – early Apr",
  "plantedDate":"2026-04-10",
  "notes":"Trellis needed within 18 months"
}}
```

### Update

```json
{"action":"update","id":"crop-037-garlic","data":{
  "status":"planted-outdoor",
  "plantedDate":"2026-04-15"
}}
```

### Multiple actions

```json
{"actions":[
  {"action":"update","id":"pantry-006-skippy","data":{"status":"need"}},
  {"action":"add","data":{"type":"pantry","name":"Almond butter","category":"Protein","status":"have"}}
]}
```

---

## Data schema

Every item shares a common schema; type-specific fields are nullable.

| Field | Type | Notes |
|---|---|---|
| `id` | string | `{type}-{timestamp36}-{slug}` |
| `type` | `pantry` \| `crop` \| `accessory` | |
| `name` | string | Required |
| `category` | string | Free-form |
| `categoryEmoji` | string | Display emoji |
| `variety` | string | Crop-specific |
| `location` | string | Primary storage/planting location |
| `sublocation` | string | Row, tier, bin, final destination |
| `quantity` | string | Free-form ("6 cans", "115 lbs frozen") |
| `price` | number | Dollars |
| `status` | `have` \| `need` \| `dormant` \| `not-planted` \| `planted-indoor` \| `planted-outdoor` | |
| `vendor` | string | Where purchased |
| `purchaseDate` | `YYYY-MM-DD` | |
| `expirationDate` | `YYYY-MM-DD` | Pantry |
| `plantedDate` | `YYYY-MM-DD` | Crop |
| `shelfLifeYears` | number | Pantry |
| `plantingWindow` | string | Crop (e.g. "Late Mar – early Apr") |
| `purpose` | string | Why it's grown/bought |
| `usedFor` | string | Accessory-specific |
| `url` | string | Product or reference link |
| `notes` | string | Free-form |
| `photos` | string[] | Firebase Storage URLs |
| `imageUrl` | string | Primary display photo |
| `tags` | string[] | Future search enhancement |
| `createdAt` | number | Unix ms |
| `lastUpdated` | number | Unix ms |

---

## Design principles

FoodSwipe inherits Phil's food project guidance:

- **Design around existing inventory first** — acknowledge what's there before recommending new
- **Boise-specific** — USDA zone 6b/7a, salt aerosols in pool room, no tropical outdoor species
- **Prefer perennials over annuals**; low-water, high-yield; nutrient & calorie density
- **Closed-loop** — propagation, division, and seed-saving over buying
- **Function over aesthetics** — food resilience, not hobby gardening

---

## Security

- Firebase Auth required for all reads and writes
- UID whitelist enforced at auth gate *and* Cloud Function layer
- API key never exposed to the client — requests proxied through `askAI` function with Firebase ID token verification
- CSP locked down to Firebase / Anthropic Cloud Function endpoints only

---

## Version history

- **v1.0** (2026-04-23) — Initial release. 61 seeded items, unified grid view, AI chat, photo upload, full CRUD.

---

© 2026 Sigler Ventures · Part of the Philinity app suite
