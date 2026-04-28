# Depop Resale Business Tracker

An Excel workbook for tracking sourcing, inventory, listings, and sales for a Depop resale business — designed as both a functional business tool and a demonstration of relational data modeling in Excel.

## The Problem

I run a clothing resale business on Depop. I source inventory from thrift warehouses that sell by the pound. These are clothes that would otherwise end up in a landfill, so reselling them keeps them in circulation a little longer. I sort through what I bring home and list items for sale on Depop. Before building this workbook, I had no structured way to answer basic business questions:

- Are my sourcing trips actually profitable?
- Which brands and categories sell best, and which ones just sit?
- How often am I discounting items to make a sale, and by how much?
- What does each item truly cost me after accounting for clothes I keep for myself, items I have to discard because I missed a flaw, and pieces that need repair?

Depop provides a CSV export of sales data, but it has a critical gap: it only captures the final sale price, not the original listing price. So if I drop an item from $25 to $18 to get it to sell, that pricing history is lost. I needed a system that captured what Depop doesn't.

## The Approach

I designed a relational data model with four entities, each in its own tab:

**Trip → Item → Listing → Sale**

Each entity exists at a different stage of the business and captures different types of information:

- **Trips** are logged in the car and at home after sorting. They capture spending, timing, and how many items I brought back. I also track how busy the warehouse was, so I can eventually identify the best days and times to go when there's less competition for the good finds.
- **Items** are logged during sorting. Every item gets tracked — keeps, sells, and discards — not just the ones I plan to list.
- **Listings** are logged at my desk while I simultaneously create the Depop listing. This is where I capture the original asking price that Depop doesn't preserve.
- **Sales** come from Depop's CSV export via a Power Query import pipeline. This is platform-generated data, not manual entry.

### Why separate entities?

An item is a physical thing with a brand, size, and category. A listing is a marketing event with a price, condition rating, and package size. Separating them means I can track what happens when I relist an item at a different price or with a different description — the item stays the same, but the listing changes. The sale is a third event entirely, owned by the platform's export data.

## Key Design Decisions

**Cost Per Sellable Item** — Total trip spend is divided only by the number of items I plan to sell, not the total number of items I brought home. When I keep something for myself or discard an item because I missed a stain or a hole, those items still cost money. The sellable items have to cover that cost too. Dividing by sellable items only gives me a true cost floor for pricing each listing.

**Tracking every item, not just sellable ones** — I log keeps and discards alongside sellable items because it reveals sourcing patterns. If I'm consistently keeping 70% of what I buy, that changes the math on whether a trip was worth it. It also shows which brands and sizes I gravitate toward personally versus what I actually sell.

**Original Price capture** — This is the primary reason the Listings tab exists. Without it, I can't calculate how often I'm discounting, how much margin I'm losing, or which categories sell at full price versus which ones always require a markdown.

**Composite key matching (Listing → Sale)** — Depop's CSV export doesn't include a listing ID, so I built a composite key system to connect sales back to listings. The Listings tab has helper columns that pull Brand and Category from the Items tab via VLOOKUP, then combine them with the listing date using TEXT() into a Match_Key. The Sales tab uses INDEX/MATCH against this key to look up the corresponding Listing_ID. This works because of a workflow rule: I never list two items with the same brand and category on the same day.

**Sales tab mirrors the Depop CSV export** — Columns A–AE on the Sales tab match the Depop export column order exactly. This means I can paste an entire export directly into the tab without rearranging columns. Calculated columns (Days to Sell, Allocated Shipping, Allocated Fees, Listing ID) sit after the export columns in AF–AI so they never interfere with the paste.

**Bundle fee allocation** — Depop exports bundles as one row per item but dumps all shared costs (shipping, fees) onto the first row. I split shipping evenly across bundle items (it's a fixed package cost) and split Depop fees proportionally by item price (since fees are percentage-based). This gives an accurate per-item profit even within bundles.

## Power Query Import Pipeline

Rather than pasting raw CSV data and manually cleaning it, I built a Power Query workflow that standardizes the Depop export on import:

- **Date conversion** — Depop exports dates as text strings (e.g., "04/10/2026"). The query converts Date of sale, Date of listing, Estimated payout date, and Payout arrival date to proper Excel date values, which enables chronological sorting and direct date arithmetic in formulas.
- **Currency cleanup** — Depop exports monetary values with dollar signs and dashes (e.g., "$16.00", "-"). The query strips these characters and converts all money columns to Currency type, so formulas can operate on them directly without SUBSTITUTE() wrappers.
- **Null value handling** — The export uses "N/A" and `="-"` for empty fields. The query replaces these with actual blanks so they don't interfere with calculations or filtering.

The query loads to a staging sheet, where I review the cleaned data before pasting it into the Sales tab. This two-step approach keeps the Sales tab as the single source of truth while Power Query handles the data transformation.

## Workbook Structure

| Tab | Purpose | Key Fields |
|-----|---------|------------|
| **Trips** | Sourcing trip logging | Date, Total Spent, Time Spent, Arrival Time, Busy level, Item counts (kept/sell/discard), Cost Per Sellable Item, Validation check |
| **Items** | Full inventory of every item sourced | SKU (auto-generated), Trip ID, Brand, Size, Gender, Category, Status (Keep/Sell/Discard) |
| **Listings** | Active Depop listing details | Listing ID (auto-generated), SKU, Listing Date, Original Price, Condition, Package Size, Brand Lookup, Category Lookup, Match Key |
| **Sales** | Depop CSV export + calculated fields | All 31 Depop export columns (A–AE), Days to Sell, Allocated Shipping, Allocated Fees, Listing ID |

## Formulas & Techniques

- **INDEX/MATCH with composite keys** for connecting sales to listings across tabs when no shared ID exists — the match key concatenates TEXT(date) + Brand + Category to create a unique identifier
- **VLOOKUP** for pulling Brand and Category from the Items tab into the Listings tab via SKU
- **Auto-generated IDs** using ROW(), TEXT(), and COUNTIF — SKUs encode the Trip ID and item sequence (e.g., T003I012 = Trip 3, Item 12 from that trip)
- **Data validation dropdowns** on categorical fields (Status, Category, Condition, Busy, Package Size, Gender) to keep data entry clean and consistent
- **Conditional validation** with a helper column that checks Total Items = Kept + Sell + Discard on every trip
- **Power Query** for automated CSV import with type conversion, currency cleanup, and null handling
- **IFERROR wrapping** on all lookup and calculation formulas to handle missing data gracefully

## What's Included

- `Depop_Tracking_Workbook.xlsx` — The complete workbook with sample data (real business data has been replaced for privacy)

## Future Enhancements

- Refine analytical questions into specific, calculated metrics (profit per item, sourcing accuracy trends, category performance)
- Build a dashboard for visualizing sourcing and sales performance over time
