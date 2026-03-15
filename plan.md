# TCGCSV Market Price Integration Plan

## Overview
Replace manual market price entry with auto-fetched prices from TCGCSV (which mirrors TCGPlayer data). Users link each product to a TCGPlayer product via a search/browse UI, then prices auto-refresh.

## Database Changes (Supabase)

Add 3 columns to `products` table:
- `tcgplayer_product_id` (integer, nullable) — the TCGPlayer product ID
- `tcgplayer_group_id` (integer, nullable) — the TCGPlayer group ID (needed to fetch prices by group)
- `price_updated_at` (text/timestamp, nullable) — when the market price was last fetched

No new tables needed. `market_price` column stays — it gets auto-updated for linked products.

## TCGCSV API Endpoints We'll Use

1. **Groups (sets):** `GET https://tcgcsv.com/tcgplayer/3/groups` — list all Pokemon sets
2. **Products by group:** `GET https://tcgcsv.com/tcgplayer/3/{groupId}/products` — all products in a set
3. **Prices by group:** `GET https://tcgcsv.com/tcgplayer/3/{groupId}/prices` — prices for all products in a set

## Implementation Steps

### Step 1: Database Migration
- Add columns via Supabase SQL editor or migration
- Update `productToDb()` and `productFromDb()` mappers to include new fields

### Step 2: TCGPlayer Link Modal
- New modal: "Link to TCGPlayer"
- Two-step flow:
  1. Search/select a Pokemon set (group) from cached group list
  2. Browse products within that set, filtered to sealed products
- User clicks a product to link it
- Saves `tcgplayer_product_id` and `tcgplayer_group_id` to the product
- Button added to product form: "Link to TCGPlayer" (shows current link status)

### Step 3: Price Fetching Logic
- `fetchPricesForGroup(groupId)` — calls TCGCSV prices endpoint, returns map of productId → price
- `refreshMarketPrices()` — for all linked products, group by `tcgplayer_group_id`, fetch prices per group, update `market_price` and `price_updated_at`
- On app load (`onAuthenticated`): if any product's `price_updated_at` is >24h old (or null), trigger refresh

### Step 4: UI Updates
- **Product form:** Replace manual "Market Price ($)" input with:
  - If linked: read-only price display + "View on TCGPlayer" link + "Unlink" button
  - If not linked: "Link to TCGPlayer" button + manual input as fallback
- **Products table:** Show price source indicator (auto vs manual)
- **Dashboard/Holdings:** No changes needed (already uses `marketPrice` field)
- Add "Refresh Prices" button in app header or product section

### Step 5: TCGPlayer URL Generation
- TCGPlayer product URLs follow pattern: `https://www.tcgplayer.com/product/{productId}`
- Display as clickable "View on TCGPlayer" link for price verification

## Caching Strategy
- Cache the groups list in localStorage (sets don't change often)
- Products list per group: cache in localStorage with 24h expiry
- Prices: fetch fresh each time, no client cache (the whole point is current prices)

## Error Handling
- If TCGCSV is down: keep existing `market_price`, show "Last updated: {date}" with a stale indicator
- If a linked product's price isn't found: keep existing price, log warning
- Network errors: show toast, don't clear prices

## Files Changed
- `index.html` — all changes in this single file (HTML modal + CSS + JS)

## Scope/Effort
- ~200-300 lines of new code
- 1 database migration (3 columns)
- No new dependencies
