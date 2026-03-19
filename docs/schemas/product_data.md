# Product Data Table Schema

## Overview
The `product_data` table stores all collected product information from vendors. This schema is specifically designed for board game and toy products, with fields for game mechanics, player counts, complexity ratings, and BoardGameGeek (BGG) integration.

**Note:** This is the ACTUAL schema currently implemented in n8n, not a theoretical design.

## Table Name
`Products List` (n8n Data Table)

## Schema Definition (31 Fields)

| Field Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `id` | Number | PRIMARY KEY, AUTO_INCREMENT | Unique internal identifier |
| `UPC` | Text | NULLABLE | Universal Product Code (12-digit barcode) |
| `EAN` | Text | NULLABLE | European Article Number (13-digit barcode) |
| `SKU` | Text | UNIQUE, NOT NULL | Stock Keeping Unit - vendor's product identifier |
| `MSRP` | Number | NULLABLE | Manufacturer's Suggested Retail Price |
| `MAP` | Number | NULLABLE | Minimum Advertised Price |
| `Cost` | Number | NULLABLE | Wholesale cost from vendor |
| `Case` | Number | NULLABLE | Number of units per case |
| `MOQ` | Number | NULLABLE | Minimum Order Quantity |
| `Increments_Above_MOQ` | Number | NULLABLE | Order increment above MOQ (e.g., order in multiples of 6) |
| `Weight_lbs` | Number | NULLABLE | Product weight in pounds |
| `Length_in` | Number | NULLABLE | Package length in inches |
| `Width_in` | Number | NULLABLE | Package width in inches |
| `Height_in` | Number | NULLABLE | Package height in inches |
| `Shopify_ID` | Text | NULLABLE | Shopify product ID for integration |
| `ClickUp_ID` | Text | NULLABLE | ClickUp task/item ID for project management |
| `Name` | Text | NOT NULL | Product name/title |
| `Copy_html` | Long Text | NULLABLE | Product description in HTML format |
| `Tagline` | Text | NULLABLE | Short marketing tagline or subtitle |
| `Ages_text` | Text | NULLABLE | Recommended age range (e.g., "8+", "10-14") |
| `keywords_list` | Text | NULLABLE | Comma-separated keywords for search/SEO |
| `Player_Count_text` | Text | NULLABLE | Number of players (e.g., "2-4", "1-6") |
| `Piece_Count_text` | Text | NULLABLE | Number of game pieces/components |
| `Play_Time` | Text | NULLABLE | Average play time (e.g., "30 minutes", "45-60 min") |
| `BGG_weight` | Number | NULLABLE | BoardGameGeek complexity rating (1.0-5.0) |
| `Mechanics_list` | Text | NULLABLE | Comma-separated game mechanics (from BGG) |
| `Rules_url` | Text | NULLABLE | URL to rulebook PDF or rules page |
| `Category_list` | Text | NULLABLE | Comma-separated product categories |
| `Shopify_Vendor_Id` | Text | NULLABLE | Shopify vendor ID (replaces separate vendor_config table) |
| `createdAt` | DateTime | NOT NULL, AUTO | Timestamp when record was created |
| `updatedAt` | DateTime | NOT NULL, AUTO | Timestamp when record was last updated |

---

## Field Details

### Core Identifiers

#### id
- **Type**: Auto-incrementing number
- **Purpose**: Internal database primary key
- **Auto-generated**: Yes
- **Usage**: Internal references only, not exposed to users

#### UPC
- **Format**: 12-digit numeric string
- **Purpose**: Universal Product Code for retail scanning
- **Example**: "123456789125"
- **Nullable**: Yes (not all products have UPCs)
- **Validation**: Should be 12 digits if provided

#### EAN
- **Format**: 13-digit numeric string
- **Purpose**: European Article Number (international barcode)
- **Example**: "1234567891234"
- **Nullable**: Yes
- **Note**: Often used for international products

#### SKU
- **Format**: Alphanumeric string, vendor-specific
- **Purpose**: Primary product identifier for ordering and inventory
- **Examples**: "ST1234", "ACME-WIDGET-001", "BGG-12345"
- **Required**: Yes (cannot be null)
- **Uniqueness**: Should be unique across all products
- **Usage**: Primary lookup field for order fulfillment

---

### Pricing Fields

#### MSRP
- **Type**: Decimal number (currency)
- **Purpose**: Manufacturer's Suggested Retail Price
- **Example**: 10.99
- **Nullable**: Yes
- **Usage**: Reference pricing, margin calculations

#### MAP
- **Type**: Decimal number (currency)
- **Purpose**: Minimum Advertised Price (vendor restriction)
- **Example**: 10.99
- **Nullable**: Yes
- **Important**: Must not advertise below this price if set

#### Cost
- **Type**: Decimal number (currency)
- **Purpose**: Wholesale cost from vendor
- **Example**: 5.50
- **Nullable**: Yes
- **Confidential**: Should not be exposed to customers

---

### Inventory & Logistics

#### Case
- **Type**: Integer
- **Purpose**: Number of units per case pack
- **Example**: 6 (product ships in cases of 6)
- **Nullable**: Yes
- **Usage**: Inventory management, shipping calculations

#### MOQ
- **Type**: Integer
- **Purpose**: Minimum Order Quantity from vendor
- **Example**: 6 (must order at least 6 units)
- **Nullable**: Yes
- **Usage**: Order validation, vendor compliance

#### Increments_Above_MOQ
- **Type**: Integer
- **Purpose**: Order increment above MOQ
- **Example**: 6 (after MOQ, order in multiples of 6)
- **Nullable**: Yes
- **Usage**: Order quantity validation

#### Weight_lbs
- **Type**: Decimal number
- **Purpose**: Product weight in pounds
- **Example**: 2.3
- **Nullable**: Yes
- **Usage**: Shipping cost calculations

#### Length_in, Width_in, Height_in
- **Type**: Decimal numbers
- **Purpose**: Package dimensions in inches
- **Examples**: 5, 5, 2 (5"x5"x2" box)
- **Nullable**: Yes
- **Usage**: Shipping calculations, warehouse storage

---

### Integration IDs

#### Shopify_ID
- **Type**: Text (numeric string)
- **Purpose**: Shopify product ID for sync
- **Example**: "1535781596463"
- **Nullable**: Yes
- **Usage**: Bi-directional sync with Shopify store

#### ClickUp_ID
- **Type**: Text
- **Purpose**: ClickUp task/item ID for project management
- **Example**: "EM-4567"
- **Nullable**: Yes
- **Usage**: Link products to ClickUp tasks/projects

#### Shopify_Vendor_Id
- **Type**: Text (numeric string)
- **Purpose**: Links product to Shopify vendor
- **Nullable**: Yes
- **Important**: Replaces need for separate vendor_config table
- **Usage**: Vendor filtering, vendor-specific operations

---

### Product Information

#### Name
- **Type**: Text
- **Purpose**: Product name/title
- **Example**: "Catan Board Game"
- **Required**: Yes (cannot be null)
- **Usage**: Display, search, ordering

#### Copy_html
- **Type**: Long text (HTML)
- **Purpose**: Full product description with HTML formatting
- **Example**: `<p>Marketing Info</p><ul><li>Feature 1</li></ul>`
- **Nullable**: Yes
- **Format**: HTML markup allowed
- **Usage**: Product pages, marketing materials

#### Tagline
- **Type**: Text
- **Purpose**: Short marketing tagline or subtitle
- **Example**: "The classic game of settlement and trade"
- **Nullable**: Yes
- **Length**: Typically 50-100 characters
- **Usage**: Product listings, search results

---

### Board Game Specific Fields

#### Ages_text
- **Type**: Text
- **Purpose**: Recommended age range
- **Examples**: "8+", "10-14", "Ages 12 and up"
- **Nullable**: Yes
- **Format**: Free text (not strictly numeric)
- **Usage**: Product filtering, compliance

#### Player_Count_text
- **Type**: Text
- **Purpose**: Number of players supported
- **Examples**: "2-4", "1-6", "2-4 players"
- **Nullable**: Yes
- **Format**: Free text (allows ranges)
- **Usage**: Product filtering, game selection

#### Piece_Count_text
- **Type**: Text
- **Purpose**: Number of game pieces/components
- **Example**: "58", "200+ pieces"
- **Nullable**: Yes
- **Usage**: Product details, complexity indicator

#### Play_Time
- **Type**: Text
- **Purpose**: Average game duration
- **Examples**: "30 minutes", "45-60 min", "1-2 hours"
- **Nullable**: Yes
- **Format**: Free text (allows ranges)
- **Usage**: Product filtering, game selection

#### BGG_weight
- **Type**: Decimal number (1.0-5.0)
- **Purpose**: BoardGameGeek complexity/weight rating
- **Example**: 1.6 (light-medium complexity)
- **Nullable**: Yes
- **Scale**: 1.0 (light) to 5.0 (heavy/complex)
- **Source**: BoardGameGeek API
- **Usage**: Complexity filtering, recommendations

#### Mechanics_list
- **Type**: Text (comma-separated)
- **Purpose**: Game mechanics from BoardGameGeek
- **Examples**: "Deck Building, Worker Placement", "Dice Rolling, Set Collection"
- **Nullable**: Yes
- **Format**: Comma-separated list
- **Source**: BoardGameGeek API
- **Usage**: Product filtering, recommendations

#### Category_list
- **Type**: Text (comma-separated)
- **Purpose**: Product categories/genres
- **Examples**: "Strategy Games, Family Games", "Card Games, Party Games"
- **Nullable**: Yes
- **Format**: Comma-separated list
- **Usage**: Product organization, navigation

#### Rules_url
- **Type**: Text (URL)
- **Purpose**: Link to rulebook PDF or rules page
- **Example**: "https://vendor.com/rules/product-rules.pdf"
- **Nullable**: Yes
- **Validation**: Should be valid URL if provided
- **Usage**: Customer support, product information

---

### Search & Discovery

#### keywords_list
- **Type**: Text (comma-separated)
- **Purpose**: Keywords for search and SEO
- **Example**: "strategy, board game, family, medieval, trading"
- **Nullable**: Yes
- **Format**: Comma-separated list
- **Usage**: Search indexing, SEO optimization

---

### System Fields

#### createdAt
- **Type**: DateTime (ISO 8601)
- **Purpose**: Record creation timestamp
- **Example**: "2026-03-19T15:34:41.255Z"
- **Required**: Yes
- **Auto-generated**: Yes (on insert)
- **Usage**: Audit trail, data freshness tracking

#### updatedAt
- **Type**: DateTime (ISO 8601)
- **Purpose**: Last update timestamp
- **Example**: "2026-03-19T15:39:07.609Z"
- **Required**: Yes
- **Auto-updated**: Yes (on every update)
- **Usage**: Cache invalidation, sync tracking

---

## Data Source Mapping

### Tier 1: Vendor-Supplied Data (Drive/Dropbox/Sheets)
Vendors may provide data in various formats. Map their fields to this schema:

**Common Vendor Fields → Our Schema:**
- Product Name → `Name`
- Description → `Copy_html`
- Price → `MSRP`
- Wholesale Price → `Cost`
- SKU/Item # → `SKU`
- Barcode → `UPC` or `EAN`
- Weight → `Weight_lbs`
- Dimensions → `Length_in`, `Width_in`, `Height_in`
- Min Order → `MOQ`
- Case Pack → `Case`

### Tier 2: Shopify Data
Shopify provides structured data via JSON API:

**Shopify Fields → Our Schema:**
- `product.id` → `Shopify_ID`
- `product.title` → `Name`
- `product.body_html` → `Copy_html`
- `product.vendor` → `Shopify_Vendor_Id`
- `variant.sku` → `SKU`
- `variant.price` → `MSRP`
- `variant.barcode` → `UPC` or `EAN`
- `variant.weight` → `Weight_lbs`
- `product.tags` → `keywords_list`, `Category_list`

### Tier 3: Zyte + AI Scraping
Use Gemini AI to extract data from vendor websites:

**AI Extraction Targets:**
- Product title → `Name`
- Description → `Copy_html`
- Price → `MSRP`
- SKU/Item number → `SKU`
- Specifications → Parse into relevant fields
- Images → Store URLs separately (not in this table)

### Tier 4: BoardGameGeek API (Enrichment)
Enrich board game products with BGG data:

**BGG XML API → Our Schema:**
- `<item><name>` → Validate `Name`
- `<item><description>` → Enrich `Copy_html`
- `<item><minplayers>` + `<maxplayers>` → `Player_Count_text`
- `<item><playingtime>` → `Play_Time`
- `<item><minage>` → `Ages_text`
- `<statistics><averageweight>` → `BGG_weight`
- `<link type="boardgamemechanic">` → `Mechanics_list`
- `<link type="boardgamecategory">` → `Category_list`

---

## Validation Rules

### Required Fields
- `SKU` - Must not be empty
- `Name` - Must not be empty
- `createdAt` - Auto-generated
- `updatedAt` - Auto-generated

### Recommended Fields
- At least one of: `UPC`, `EAN`, or `Shopify_ID`
- At least one of: `MSRP`, `MAP`, or `Cost`
- `Shopify_Vendor_Id` for vendor tracking

### Data Quality Checks
- `UPC` should be 12 digits if provided
- `EAN` should be 13 digits if provided
- `MSRP` >= `MAP` >= `Cost` (if all provided)
- `MOQ` <= `Case` (typically)
- `BGG_weight` should be 1.0-5.0 if provided
- URLs (`Rules_url`) should be valid if provided

---

## Usage Examples

### Sample Product Record
```csv
id,UPC,EAN,SKU,MSRP,MAP,Cost,Case,MOQ,Increments_Above_MOQ,Weight_lbs,Length_in,Width_in,Height_in,Shopify_ID,ClickUp_ID,Name,Copy_html,Tagline,Ages_text,keywords_list,Player_Count_text,Piece_Count_text,Play_Time,BGG_weight,Mechanics_list,Rules_url,Category_list,Shopify_Vendor_Id,createdAt,updatedAt
1,1234567891253,,ST1234,10.99,10.99,5.5,6,6,6,2.3,5,5,2,1535781596463,EM-4567,Test Product,Marketing Info,Short tag line,8+,,2-4,58,30 minutes,1.6,,,,,2026-03-19T15:34:41.255Z,2026-03-19T15:39:07.609Z
```

### Inserting a New Product (n8n)
```javascript
$('Data Table').insert({
  SKU: 'CATAN-BASE-001',
  Name: 'Catan Board Game',
  Copy_html: '<p>The classic game of settlement and trade</p>',
  Tagline: 'Build, trade, and settle the island of Catan',
  MSRP: 44.99,
  MAP: 39.99,
  Cost: 22.50,
  Case: 6,
  MOQ: 6,
  Weight_lbs: 2.8,
  Ages_text: '10+',
  Player_Count_text: '3-4',
  Play_Time: '60-120 minutes',
  BGG_weight: 2.3,
  Mechanics_list: 'Dice Rolling, Hand Management, Trading',
  Category_list: 'Strategy Games, Family Games',
  Shopify_Vendor_Id: '12345678',
  createdAt: new Date().toISOString(),
  updatedAt: new Date().toISOString()
});
```

### Updating a Product
```javascript
$('Data Table').update({
  SKU: 'CATAN-BASE-001',
  MSRP: 49.99,  // Price increase
  updatedAt: new Date().toISOString()
});
```

### Querying Products
```javascript
// Get all products from a specific vendor
const vendorProducts = $('Data Table').all().filter(p => 
  p.Shopify_Vendor_Id === '12345678'
);

// Get products by complexity
const lightGames = $('Data Table').all().filter(p => 
  p.BGG_weight && p.BGG_weight < 2.0
);

// Get products needing BGG enrichment
const needsBGG = $('Data Table').all().filter(p => 
  !p.BGG_weight && p.Category_list && p.Category_list.includes('Board Game')
);
```

---

## BoardGameGeek Integration

### BGG API Overview
- **Endpoint**: `https://boardgamegeek.com/xmlapi2/thing?id={bgg_id}`
- **Format**: XML
- **Rate Limit**: ~2 requests/second (unofficial)
- **Cost**: FREE

### BGG Enrichment Workflow
1. **Identify board games** in product catalog
2. **Search BGG** by product name or UPC
3. **Extract BGG ID** from search results
4. **Fetch game details** via BGG API
5. **Parse XML** and extract relevant fields
6. **Update product record** with BGG data
7. **Cache BGG data** to avoid repeated API calls

### BGG Data Mapping
```javascript
// Example BGG XML parsing
const bggData = parseXML(response);
const product = {
  BGG_weight: bggData.statistics.averageweight,
  Mechanics_list: bggData.mechanics.join(', '),
  Category_list: bggData.categories.join(', '),
  Player_Count_text: `${bggData.minplayers}-${bggData.maxplayers}`,
  Play_Time: `${bggData.playingtime} minutes`,
  Ages_text: `${bggData.minage}+`
};
```

---

## Vendor Management

### Using Shopify Vendors
Instead of a separate `vendor_config` table, this system uses Shopify's built-in vendor management:

**Advantages:**
- ✅ No duplicate vendor data
- ✅ Shopify handles vendor relationships
- ✅ Simpler data model
- ✅ Automatic sync with Shopify

**Vendor Operations:**
```javascript
// Get all products from a vendor
const vendorProducts = $('Data Table').all().filter(p => 
  p.Shopify_Vendor_Id === vendorId
);

// Get unique vendors
const vendors = [...new Set(
  $('Data Table').all().map(p => p.Shopify_Vendor_Id)
)];
```

---

## Migration Notes

### Importing Existing Data
If you have existing product data:

1. **Export to CSV** with matching column headers
2. **Map fields** to this schema
3. **Validate data** (required fields, data types)
4. **Import to n8n** Data Table
5. **Verify import** with sample queries

### Data Cleanup
- Remove duplicate SKUs
- Standardize text formats (Ages, Player Count, Play Time)
- Validate numeric fields (prices, dimensions)
- Ensure Shopify_Vendor_Id is populated

---

## Performance Considerations

### Indexing
Recommended indexes for fast queries:
- `SKU` (primary key, automatic)
- `Shopify_ID` (for Shopify sync)
- `Shopify_Vendor_Id` (for vendor filtering)
- `updatedAt` (for finding stale data)

### Caching
- Cache BGG data aggressively (rarely changes)
- Cache vendor product lists (refresh weekly)
- Cache Shopify sync data (refresh on webhook)

---

## Change Log

| Version | Date | Changes |
|---------|------|---------|
| 2.0 | 2026-03-19 | Updated to reflect actual 31-field schema in use |
| 1.0 | 2026-03-19 | Initial theoretical schema (9 fields) |

---

**Related Documentation:**
- [Vendor Management via Shopify](../guides/shopify_vendor_management.md)
- [BoardGameGeek Integration](../guides/bgg_integration.md)
- [Setup Guide](../../SETUP_GUIDE.md)
- [Implementation Plan](../../IMPLEMENTATION_PLAN.md)

