# 🛒 Smart Grocery Optimizer

> **A fully offline Progressive Web App (PWA) that lives on your phone like a native app — no App Store, no subscription, no ads, no data sent to any server.**

Compare grocery store prices, scan shopping lists by photo or email, track 4 weeks of purchase history, and automatically split your cart across stores to maximize savings.

**Live app → [https://roopeshaithal.github.io/smart-grocery](https://roopeshaithal.github.io/smart-grocery)**

---

## 📋 Table of Contents

1. [How It Works](#how-it-works)
2. [File Structure](#file-structure)
3. [How Offline Works](#how-offline-works)
4. [Where Data Lives](#where-data-lives)
5. [Install on Your Phone](#install-on-your-phone)
6. [How to Use — Every Feature](#how-to-use)
7. [Store Coverage](#store-coverage)
8. [Adding or Editing Stores](#adding-or-editing-stores)
9. [Photo Scanning Setup](#photo-scanning-setup)
10. [Deploy to GitHub Pages](#deploy-to-github-pages)
11. [Local Development](#local-development)
12. [Tech Stack](#tech-stack)
13. [Roadmap](#roadmap)
14. [Contact](#contact)

---

## How It Works

The app is built around three ideas:

**1. Offline-first** — Everything is cached on your device after the first visit. The service worker intercepts every network request and serves files from the local cache. Your grocery list, history, and settings all live in IndexedDB — a built-in browser database with up to 2 GB capacity. No internet is ever required after the initial load.

**2. Data separated from code** — Store names, prices, and grocery history are stored in JSON files (`stores.json`, `seed.json`), not hardcoded into the app. This means you can update store data, add new stores, or change category prices by editing a single JSON file — no code changes needed.

**3. Smart price routing** — Each store in `stores.json` has a price multiplier per category (e.g. Patel Brothers is 0.55× on spices vs 1.0× baseline). The app uses these multipliers to estimate your total at each store and to automatically route each item to the cheapest store in the Split & Save view.

### Architecture at a glance

```
GitHub Pages (server — internet needed only once)
├── index.html      ← React app shell
├── sw.js           ← Service worker (caching brain)
├── manifest.json   ← PWA install config
├── stores.json     ← Store database (names, prices, addresses)
└── seed.json       ← Category definitions + 4-week history seed

Your Phone (always available, no internet needed)
├── Cache API       ← Stores all 5 files above after first visit
└── IndexedDB       ← Your live data
    ├── groceryList     ← Current shopping list
    ├── history         ← All past scanned/added weeks
    ├── settings        ← Location, radius, API key, preferred stores
    └── customStores    ← Any stores you've added manually
```

### Data flow on first launch

```
1. Phone opens app → browser fetches index.html from GitHub Pages
2. sw.js registers → downloads all 5 files into Cache API
3. App starts → fetches stores.json + seed.json (now from cache)
4. IndexedDB is empty → app seeds it from seed.json (history + categories)
5. App renders with your pre-loaded 4-week history
```

### Data flow on every subsequent launch (fully offline)

```
1. Phone opens app → service worker intercepts
2. Returns index.html from Cache API (no network call)
3. App starts → reads stores.json from Cache API (no network call)
4. App reads your list, history, settings from IndexedDB (no network call)
5. App is fully ready — 100% offline
```

---

## File Structure

```
smart-grocery/
├── index.html       Main React PWA (all UI, logic, offline boot)
├── sw.js            Service worker (cache management, offline routing)
├── manifest.json    Web app manifest (PWA install metadata)
├── stores.json      Store database — edit this to add/change stores
├── seed.json        Category config + initial 4-week history
└── README.md        This file
```

### What each file does

| File | Purpose | Edit when… |
|------|---------|------------|
| `index.html` | Full app — React UI, IndexedDB logic, scan modal, all tabs | Changing app behaviour or UI |
| `sw.js` | Intercepts requests, serves from cache offline | Bumping cache version after major update |
| `manifest.json` | Makes the app installable on home screen | Changing app name, icon, or shortcuts |
| `stores.json` | All store data — names, addresses, price multipliers | Adding a store, updating prices |
| `seed.json` | Category colours, base prices, initial history | Adding a category, changing base prices |

---

## How Offline Works

The offline capability is powered by two browser technologies working together:

### Service Worker (`sw.js`)

A service worker is a background script that the browser runs separately from the app. It acts as a proxy between your app and the network.

```
Without service worker:
  App → Network request → Server → Response → App

With service worker:
  App → Network request → Service Worker
                              ↓
                         Check cache
                         ↓          ↓
                     Found          Not found
                         ↓          ↓
                    Return cache  Fetch network
                                  → Cache it
                                  → Return it
```

On first visit, the service worker pre-caches all 5 files. On every visit after, it intercepts the request for `index.html` and returns it instantly from cache — no server involved. The same happens for `stores.json`, `seed.json`, and the React library files.

**The only requests not intercepted** are calls to `api.anthropic.com` (photo scanning) — these always go to the network because they need live AI inference.

### Cache versioning

The service worker uses a version string (`smart-grocery-v5`). When you push an update to GitHub, bump the version number in `sw.js`. The next time a user opens the app with internet, the new service worker activates, downloads the fresh files, and deletes the old cache automatically.

```javascript
// sw.js — change this when you push a major update
const CACHE = "smart-grocery-v5";  // ← bump to v6, v7 etc.
```

### IndexedDB

IndexedDB is a full structured database built into every browser. It stores your personal data permanently on device — nothing is lost when you close the app. It has four object stores:

| Store | Contents | Key |
|-------|---------|-----|
| `groceryList` | Your current shopping list items | item `id` |
| `history` | All past weeks (scanned or added) | week `id` |
| `settings` | Location, radius, API key, preferred stores | setting key |
| `customStores` | Stores you've added manually | store `id` |

Every change you make — adding an item, checking something off, changing a quantity — is written to IndexedDB immediately. There is no "save" button. Everything is auto-saved.

### Persistent storage

By default, browsers can clear cached data when device storage is low. To prevent this, the app offers a **Request persistent storage** button in Settings. This tells the browser to protect your data from auto-deletion. On Android this is granted automatically; on iOS (Safari) it is granted when you add the app to your home screen.

---

## Where Data Lives

```
Device Storage
│
├── Cache API (~5 MB)
│   ├── index.html          (app shell, ~120 KB)
│   ├── sw.js               (service worker, ~3 KB)
│   ├── manifest.json       (PWA config, ~1 KB)
│   ├── stores.json         (store database, ~8 KB)
│   ├── seed.json           (categories + history, ~5 KB)
│   └── React + Babel       (libraries, ~4 MB)
│
└── IndexedDB (~10–50 KB, grows with your history)
    ├── groceryList         Your current list
    ├── history             All past shopping weeks
    ├── settings            Your preferences
    └── customStores        Any stores you've added
```

**Nothing leaves your device** except:
- Photo scanning → image sent to Anthropic API for AI reading (requires your API key)
- No analytics, no tracking, no ads, no accounts

**To see your storage usage:** Settings tab → Storage Info card shows KB used and % of quota.

**To protect your data:** Settings → Request persistent storage.

**To wipe everything:** Settings → Danger Zone → Clear all app data.

---

## Install on Your Phone

### iPhone (Safari) — Recommended ⭐

1. Open **[https://roopeshaithal.github.io/smart-grocery](https://roopeshaithal.github.io/smart-grocery)** in **Safari** (must be Safari, not Chrome)
2. Tap the **Share** button (the box with an arrow, at the bottom of the screen)
3. Scroll down and tap **"Add to Home Screen"**
4. Rename it **Smart Grocery** if you like
5. Tap **Add**
6. The app icon now sits on your home screen — tap it to launch fullscreen with no browser chrome

> After adding to home screen, iOS grants persistent storage automatically — your data is protected.

### Android (Chrome)

1. Open the link in **Chrome**
2. A banner appears at the bottom: **"Add Smart Grocery to Home Screen"** — tap **Install**
3. Or tap the **3-dot menu → Add to Home Screen → Add**
4. The app icon appears on your home screen

### Any browser (without installing)

Just open the URL. The app works fully in any modern browser tab — you just won't get the fullscreen native feel or the install prompt.

---

## How to Use

### First launch

When you first open the app, the grocery list is pre-seeded from a sample 4-week history so you can explore immediately. All data is editable. The first load requires internet — every launch after works fully offline.

---

### 🛒 My List tab

Your active shopping list for the current trip.

#### Adding items

| Method | How |
|--------|-----|
| Type manually | Enter item name, pick category, set quantity, tap **Add** or press Enter |
| Quick scan | Tap **+ Scan** (top right) to open the scan panel |
| Paste a list | Tap **+ Scan → ✏️ Type** and paste any format |
| Photo | Tap **+ Scan → 📷 Photo** and photograph your handwritten list |
| Email | Tap **+ Scan → 📧 Email** and paste email body |

#### Managing items

| Action | How |
|--------|-----|
| Check off | Tap the checkbox on the left |
| Remove checked | Red **"Remove X checked items"** button appears at top |
| Change quantity | Tap − or + on each item row |
| Change category | Tap the faint category label on the right |
| Delete item | Tap × on the far right |
| See frequency | Items bought 2+ times show a badge like **4x** |

The **best estimated total** and cheapest store are shown at the bottom.

---

### 📷 Scan — Add items 3 ways

Tap **+ Scan** in the top-right corner.

#### ✏️ Type / Paste

Paste or type your list in any format:

```
Milk
2x Eggs
Chicken breast
Basmati rice 2
Garam masala, cumin
Dish soap
```

Supported formats:
- One item per line
- `2x Item` or `Item x2` for quantities
- Comma-separated items on one line
- Mix of all the above

Tap **Parse list** → review detected items → fix categories if needed → tap **✓ Add X items to my list**.

#### 📷 Photo / Camera

1. Tap **Upload** to pick from your gallery or **Camera** to take a photo
2. The image is sent to Claude AI (requires your API key — see Settings)
3. AI reads the image and extracts all grocery items
4. Review the list, fix any categories, confirm

Works with: handwritten lists, printed receipts, fridge notes, sticky notes, store circulars.

#### 📧 From Email

1. Open an email with a grocery list (from family, a meal plan, an order confirmation)
2. Copy the full email body text
3. Paste it into the text area
4. Tap **Extract items** — the app strips bullet points and list formatting automatically

All three scan modes show a **preview** before adding. You can remove individual items or correct their categories before confirming.

---

### 🏪 Compare tab

Ranks all your preferred stores from cheapest to most expensive for your exact current list.

- **Best price** store highlighted with a coloured border
- Price bar shows relative cost visually
- % difference vs cheapest shown for each store
- Category strength badges show what each store specialises in
- Addresses shown for custom stores you've added

---

### ✂️ Split & Save tab

Automatically routes each **category** to whichever preferred store prices it lowest — not just one cheapest store overall, but the best store per type of item.

Example with a mixed list:

| Category | Best store | Why |
|----------|-----------|-----|
| Spices, rice/grains | Patel Brothers | 45–50% of regular price |
| Produce, dairy | ALDI | 85–90% |
| Pantry, beverages | Walmart | 85% |
| Meat | Food Lion | 90% |
| Household, bulk | Costco / Sam's Club | 70–75% |

The **Optimized total** at the bottom shows your savings vs shopping at a single expensive store.

---

### 📅 History tab

Tracks all past shopping lists — both seeded from the initial 4 weeks and everything you've scanned since.

| Action | How |
|--------|-----|
| View most-bought items | Shown as amber frequency pills at the top |
| Re-add a past week | Tap **+ Add to list** on any week card |
| Expand a week | Tap the week header to see all items |
| Delete individual items | Tap **Edit** → checkboxes appear → select items → **Delete selected** |
| Delete a whole week | Tap **Edit** → tap **Delete** next to the week |

---

### ❓ Help tab

In-app guide covering:
- Quick start (5 steps)
- What each tab does
- How each scan mode works
- Store tips for Hanover, MD
- Data privacy explanation
- FAQ
- Contact button

---

### ⚙️ Settings tab

#### Location
- Type your city, zip code, or full address
- Tap **📍 GPS** to auto-detect your coordinates
- Drag the **Radius** slider (2–25 miles)
- All settings auto-save to IndexedDB immediately

#### Preferred Stores
Toggle which stores appear in Compare and Split & Save. Organised by type:

| Type | Stores |
|------|--------|
| 🛒 Mainstream | ALDI, Walmart, Food Lion, Safeway, Trader Joe's, Whole Foods |
| 🏪 Warehouse | Costco, Sam's Club |
| 🇮🇳 Indian | Patel Brothers, Tara Vani |
| 🇰🇷 Korean | Lotte Plaza, H Mart |
| 🌏 Asian | Grand Mart |
| 📍 Custom | Any store you add |

#### Add a Custom Store

1. Tap **+ Add a store not in the list**
2. Enter the store **name** (e.g. Bharat Bazaar)
3. Enter the **address** (e.g. 123 Main St, Hanover MD)
4. Enter **best categories** comma-separated (e.g. `spices, rice/grains, produce`)
5. Tap **Add store**

The store is saved to IndexedDB and added to your preferred list immediately. It appears in Compare and Split & Save right away.

#### Anthropic API Key (for photo scanning)

1. Go to [console.anthropic.com](https://console.anthropic.com)
2. Create a free account → click **API Keys** → **Create Key**
3. Copy the key (starts with `sk-ant-…`)
4. Paste it in Settings → tap **Save**

The key is stored in IndexedDB on your device only — never sent anywhere except directly to Anthropic when you use photo scanning.

#### Storage Info

Shows:
- **Used** — how many KB the app is using
- **Quota** — how much storage your browser allows (typically 500 MB–2 GB)
- **Used %** — bar turns red above 80%
- **Request persistent storage** button — prevents auto-deletion

#### Danger Zone

**Clear all app data** — wipes all lists, history, settings, and custom stores. Resets to factory state. Cannot be undone.

---

## Store Coverage

### Built-in stores (Hanover, MD area)

| Store | Type | Best for | Membership |
|-------|------|---------|------------|
| ALDI | Mainstream Budget | Produce, dairy, pantry | None. Closed Sundays |
| Walmart | Mainstream Budget | Pantry, beverages, household | None |
| Food Lion | Mainstream Mid | Meat, dairy, bakery | Loyalty card recommended |
| Safeway | Mainstream Mid | Deli, bakery, meat | Use Safeway app for coupons |
| Trader Joe's | Mainstream Mid | Frozen, snacks, produce | None |
| Whole Foods | Mainstream Premium | Organic, produce, meat | Prime for 10% off |
| Costco | Warehouse Budget | Bulk meat, pantry, household | ~$65/yr |
| Sam's Club | Warehouse Budget | Bulk beverages, household | ~$50/yr |
| Patel Brothers | Indian Budget | Spices, rice/grains, Indian pantry | None |
| Tara Vani | Indian Budget | South Indian groceries, snacks | None |
| Lotte Plaza | Korean Mid | Asian produce, frozen, snacks | None |
| H Mart | Korean Mid | Seafood, Asian produce, frozen | None |
| Grand Mart | Asian Budget | Pan-Asian produce, rice, spices | None |

### Adding stores

**One-off custom store** → use the Settings tab in the app (see above).

**Permanent addition** → edit `stores.json` on GitHub (see below).

---

## Adding or Editing Stores

All store data lives in `stores.json`. You never need to touch `index.html` to change store information.

### JSON structure for each store

```json
{
  "id": "mystore",
  "name": "My Store Name",
  "flag": "🛒",
  "tier": 1,
  "type": "mainstream",
  "strengths": ["produce", "dairy"],
  "address": "123 Main St, Hanover MD",
  "notes": "Short tip shown in the Help tab",
  "mult": {
    "dairy":       1.0,
    "produce":     0.85,
    "pantry":      0.9,
    "meat":        0.95,
    "bakery":      1.0,
    "beverages":   0.95,
    "frozen":      0.9,
    "household":   1.0,
    "spices":      0.7,
    "rice/grains": 0.7,
    "snacks":      0.9
  }
}
```

### Field reference

| Field | Values | Meaning |
|-------|--------|---------|
| `id` | Short unique string | Used internally — no spaces |
| `flag` | Any emoji | Shown next to store name |
| `tier` | `1`, `2`, `3` | Budget, Mid-range, Premium |
| `type` | `mainstream` `warehouse` `indian` `korean` `asian` `custom` | Groups stores in Settings |
| `strengths` | Array of category names | Shown as badges in Compare |
| `mult` | 0.5–1.5 per category | Price multiplier vs baseline |

### Price multiplier guide

| Mult value | Meaning |
|-----------|---------|
| `0.5–0.6` | 40–50% cheaper (e.g. ethnic stores on their specialty items) |
| `0.7–0.8` | 20–30% cheaper (warehouse bulk) |
| `0.85–0.95` | 5–15% cheaper (budget mainstream) |
| `1.0` | Baseline price |
| `1.1–1.2` | 10–20% more expensive (mid-tier) |
| `1.3–1.5` | 30–50% more expensive (premium) |

### Example: adding a new Indian store

```json
{
  "id": "apnaBazaar",
  "name": "Apna Bazaar",
  "flag": "🇮🇳",
  "tier": 1,
  "type": "indian",
  "strengths": ["spices", "rice/grains", "produce"],
  "address": "456 Route 1, Glen Burnie MD",
  "notes": "Good South Indian snacks and frozen section.",
  "mult": {
    "dairy": 0.88,
    "produce": 0.85,
    "pantry": 0.78,
    "meat": 0.95,
    "bakery": 0.88,
    "beverages": 0.88,
    "frozen": 0.88,
    "household": 0.95,
    "spices": 0.52,
    "rice/grains": 0.52,
    "snacks": 0.68
  }
}
```

After saving and pushing to GitHub, the app picks up the change on next open with internet.

---

## Photo Scanning Setup

Photo scanning uses Claude AI to read images of grocery lists.

### Get your free API key

1. Go to **[console.anthropic.com](https://console.anthropic.com)**
2. Sign up for a free account (no credit card needed to start)
3. Click **API Keys** in the left sidebar
4. Click **Create Key** → copy the key (starts with `sk-ant-…`)
5. Open the app → Settings → paste the key → tap **Save**

### How it works

```
You tap Camera → photo taken on your phone
  ↓
Image converted to base64
  ↓
Sent to Anthropic API with prompt:
  "Extract all grocery items from this image.
   Return ONLY JSON: [{"name":"Milk","qty":1}]"
  ↓
Claude reads the image → returns structured JSON
  ↓
App parses JSON → shows preview list
  ↓
You review, fix categories if needed → confirm
  ↓
Items added to your list + saved to IndexedDB
```

### What it can read

- Handwritten grocery lists (even messy handwriting)
- Printed receipts
- Sticky notes on the fridge
- Typed lists on paper
- Store circulars and flyers
- Screenshots of lists from other apps

### Cost

New Anthropic accounts get free credits. Scanning one grocery list image uses approximately $0.002–0.005 of credits — enough for thousands of scans before you'd ever need to add payment.

---

## Deploy to GitHub Pages

### First time setup

1. Fork or create `github.com/RoopeshAithal/smart-grocery`
2. Upload all 5 files: `index.html`, `sw.js`, `manifest.json`, `stores.json`, `seed.json`
3. Go to **Settings → Pages**
4. Under "Branch" select **main** and folder **/ (root)**
5. Click **Save**
6. Wait ~60 seconds → live at `https://roopeshaithal.github.io/smart-grocery`

### Pushing updates

```bash
git add .
git commit -m "Describe your change"
git push origin main
# GitHub Pages auto-deploys in ~60 seconds
```

### After a major update — clear phone cache

When you push a new version with a bumped service worker cache name:

**iPhone Safari:**
Settings → Safari → Clear History and Website Data → reopen app

**Android Chrome:**
Chrome → Settings → Privacy → Clear browsing data → reload app

---

## Local Development

No build step, no npm, no node_modules. It's a single HTML file with external CDN scripts.

```bash
# Clone
git clone https://github.com/RoopeshAithal/smart-grocery.git
cd smart-grocery

# You MUST use a local server (service workers don't work on file://)
# Option 1 — Python
python3 -m http.server 8080

# Option 2 — Node
npx serve .

# Option 3 — VS Code Live Server extension
# Right-click index.html → Open with Live Server

# Open in browser
open http://localhost:8080
```

### Making changes

| Change type | File to edit |
|-------------|-------------|
| Add a store | `stores.json` |
| Change base prices or category colours | `seed.json` |
| Add a UI feature | `index.html` |
| Change offline caching | `sw.js` (bump cache version!) |
| Change app name or icon | `manifest.json` |

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| UI | React 18 (CDN, no build) | Component model, hooks |
| Local DB | IndexedDB | Persistent, structured, ~2 GB, offline |
| Offline | Service Worker + Cache API | File caching, network interception |
| Install | Web App Manifest | Home screen icon, fullscreen |
| AI scanning | Anthropic Claude API | Vision model reads handwritten lists |
| Hosting | GitHub Pages | Free, static, auto-deploy |
| Store data | JSON files | Editable without code changes |
| Build tools | None | Zero dependencies, no npm |

### Storage capacity on real devices

| Device | Typical IndexedDB quota |
|--------|------------------------|
| iPhone (Safari) | 500 MB – 1 GB |
| Android (Chrome) | 60% of free disk space |
| Desktop Chrome | 60% of free disk space |

The app currently uses under 1 MB of storage for a full year of weekly history.

---

## Roadmap

- [ ] Barcode scanner — scan product barcodes to add items
- [ ] Kroger API integration — real weekly sale prices for Food Lion and Safeway
- [ ] Walmart API integration — live item prices
- [ ] Family sharing — sync list between phones (using a shared PIN)
- [ ] Budget tracker — set a weekly budget, track estimated spend
- [ ] Recipe import — paste a recipe URL, ingredients added to list
- [ ] Deal alerts — notify when a watched item goes on sale
- [ ] Export list — share as text or PDF

---

## Contact

Found a bug? Want a new store added? Have a feature idea?

**Email:** [roopeshaithal@gmail.com](mailto:roopeshaithal@gmail.com?subject=Smart%20Grocery%20App)

---

## License

MIT — free to use, fork, and modify.

---

*Built with ❤️ using React + Claude AI · Version 4.1 · Optimized for Hanover, MD*
