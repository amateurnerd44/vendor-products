# BoardGameGeek (BGG) Integration Guide

## Overview

BoardGameGeek (BGG) is the world's largest board game database and community. This guide explains how to integrate BGG data to enrich product information with game mechanics, categories, complexity ratings, and more.

---

## Why BGG Integration?

### Benefits ✅

- **Rich game data** - Mechanics, categories, player counts, play time
- **Complexity ratings** - BGG weight (1.0-5.0 scale)
- **Community ratings** - Average ratings and rankings
- **Images** - High-quality game images
- **FREE API** - No cost for API access
- **Authoritative source** - Most comprehensive board game database

### Use Cases

1. **Product Enrichment** - Fill missing product details
2. **Search & Filtering** - Enable filtering by mechanics, complexity
3. **Recommendations** - Suggest similar games
4. **Validation** - Verify product information accuracy
5. **SEO** - Improve search visibility with rich metadata

---

## BGG API Overview

### API Endpoints

**Thing API** (Game Details):
```
https://boardgamegeek.com/xmlapi2/thing?id={bgg_id}&stats=1
```

**Search API** (Find Games):
```
https://boardgamegeek.com/xmlapi2/search?query={game_name}&type=boardgame
```

**Collection API** (User Collections):
```
https://boardgamegeek.com/xmlapi2/collection?username={username}
```

### API Characteristics

- **Format**: XML (not JSON)
- **Rate Limit**: ~2 requests/second (unofficial, be respectful)
- **Cost**: FREE
- **Authentication**: None required
- **Caching**: Highly recommended (data changes infrequently)

---

## Data Mapping

### BGG Fields → Our Schema

| BGG Field | Our Field | Notes |
|-----------|-----------|-------|
| `<name type="primary">` | `Name` | Validate/enrich product name |
| `<description>` | `Copy_html` | Rich game description |
| `<minplayers>` + `<maxplayers>` | `Player_Count_text` | e.g., "2-4" |
| `<playingtime>` | `Play_Time` | e.g., "30 minutes" |
| `<minage>` | `Ages_text` | e.g., "8+" |
| `<statistics><averageweight>` | `BGG_weight` | Complexity rating (1.0-5.0) |
| `<link type="boardgamemechanic">` | `Mechanics_list` | Comma-separated |
| `<link type="boardgamecategory">` | `Category_list` | Comma-separated |
| `<image>` | External storage | Product images |
| `<yearpublished>` | `metadata` | Publication year |

---

## Implementation

### Phase 2.5: BGG Enrichment Workflow

#### Step 1: Identify Board Games

```javascript
// n8n workflow: Find products that need BGG enrichment
const boardGames = $('Data Table').all().filter(product => {
  // Has board game indicators
  const isBoardGame = 
    product.Category_list?.includes('Board Game') ||
    product.Category_list?.includes('Card Game') ||
    product.Player_Count_text !== null;
  
  // Missing BGG data
  const needsEnrichment = !product.BGG_weight;
  
  return isBoardGame && needsEnrichment;
});
```

#### Step 2: Search BGG by Product Name

```javascript
// Search BGG for game
const searchUrl = `https://boardgamegeek.com/xmlapi2/search?query=${encodeURIComponent(product.Name)}&type=boardgame`;

const searchResponse = await $('HTTP Request').get(searchUrl);
const searchXML = parseXML(searchResponse);

// Get first result's BGG ID
const bggId = searchXML.items.item[0]?.id;
```

#### Step 3: Fetch Game Details

```javascript
// Get detailed game info
const detailsUrl = `https://boardgamegeek.com/xmlapi2/thing?id=${bggId}&stats=1`;

const detailsResponse = await $('HTTP Request').get(detailsUrl);
const gameXML = parseXML(detailsResponse);
```

#### Step 4: Parse XML and Extract Data

```javascript
// Parse BGG XML response
const game = gameXML.items.item;

const bggData = {
  BGG_weight: parseFloat(game.statistics.ratings.averageweight),
  Player_Count_text: `${game.minplayers}-${game.maxplayers}`,
  Play_Time: `${game.playingtime} minutes`,
  Ages_text: `${game.minage}+`,
  Mechanics_list: game.link
    .filter(l => l.type === 'boardgamemechanic')
    .map(l => l.value)
    .join(', '),
  Category_list: game.link
    .filter(l => l.type === 'boardgamecategory')
    .map(l => l.value)
    .join(', ')
};
```

#### Step 5: Update Product Record

```javascript
// Update product with BGG data
$('Data Table').update({
  SKU: product.SKU,
  ...bggData,
  updatedAt: new Date().toISOString()
});
```

---

## n8n Workflow Example

### Complete BGG Enrichment Workflow

```
[Cron Trigger: Daily 3 AM]
    ↓
[Get Products Needing BGG Data]
    ↓
[Loop Over Products]
    ↓
[Search BGG by Name]
    ↓
[Parse Search Results]
    ↓
[Get BGG ID]
    ↓
[Fetch Game Details]
    ↓
[Parse Game XML]
    ↓
[Extract BGG Data]
    ↓
[Update Product Record]
    ↓
[Wait 500ms] ← Rate limiting
    ↓
[Next Product]
```

### Node Configuration

**HTTP Request Node (Search)**:
- Method: GET
- URL: `https://boardgamegeek.com/xmlapi2/search`
- Query Parameters:
  - `query`: `{{$json.Name}}`
  - `type`: `boardgame`
- Response Format: XML

**HTTP Request Node (Details)**:
- Method: GET
- URL: `https://boardgamegeek.com/xmlapi2/thing`
- Query Parameters:
  - `id`: `{{$json.bgg_id}}`
  - `stats`: `1`
- Response Format: XML

**Code Node (Parse XML)**:
```javascript
// Parse XML to JSON
const xml2js = require('xml2js');
const parser = new xml2js.Parser();

const result = await parser.parseStringPromise($input.first().json.body);
return [{ json: result }];
```

**Wait Node**:
- Wait Time: 500ms
- Purpose: Respect BGG rate limits

---

## XML Parsing

### Sample BGG XML Response

```xml
<?xml version="1.0" encoding="utf-8"?>
<items termsofuse="https://boardgamegeek.com/xmlapi/termsofuse">
  <item type="boardgame" id="13">
    <name type="primary" value="Catan"/>
    <description>In Catan, players try to be the dominant force...</description>
    <yearpublished value="1995"/>
    <minplayers value="3"/>
    <maxplayers value="4"/>
    <playingtime value="120"/>
    <minplaytime value="60"/>
    <maxplaytime value="120"/>
    <minage value="10"/>
    
    <link type="boardgamecategory" id="1021" value="Economic"/>
    <link type="boardgamecategory" id="1026" value="Negotiation"/>
    <link type="boardgamemechanic" id="2072" value="Dice Rolling"/>
    <link type="boardgamemechanic" id="2041" value="Hand Management"/>
    
    <statistics>
      <ratings>
        <average value="7.12345"/>
        <averageweight value="2.3214"/>
      </ratings>
    </statistics>
  </item>
</items>
```

### Parsing with xml2js

```javascript
const xml2js = require('xml2js');
const parser = new xml2js.Parser({ explicitArray: false });

const result = await parser.parseStringPromise(xmlString);
const game = result.items.item;

// Access fields
const name = game.name.$.value;
const minPlayers = game.minplayers.$.value;
const weight = game.statistics.ratings.averageweight.$.value;

// Get mechanics
const mechanics = Array.isArray(game.link) 
  ? game.link.filter(l => l.$.type === 'boardgamemechanic')
  : [game.link].filter(l => l.$.type === 'boardgamemechanic');
```

---

## Rate Limiting

### BGG Rate Limit Guidelines

- **Unofficial limit**: ~2 requests/second
- **Be respectful**: BGG is a community resource
- **Use caching**: Don't re-fetch data unnecessarily
- **Implement delays**: Wait 500ms between requests

### Rate Limiting Implementation

```javascript
// In n8n workflow
const RATE_LIMIT_DELAY = 500; // milliseconds

for (const product of products) {
  // Fetch BGG data
  await enrichWithBGG(product);
  
  // Wait before next request
  await new Promise(resolve => setTimeout(resolve, RATE_LIMIT_DELAY));
}
```

---

## Caching Strategy

### Why Cache BGG Data?

- **BGG data rarely changes** - Game mechanics don't change
- **Reduce API calls** - Respect BGG's resources
- **Faster enrichment** - No need to re-fetch
- **Reliability** - Less dependent on BGG availability

### Caching Implementation

**Option 1: Cache in Product Record**
```javascript
// Store BGG data directly in product
// Already cached by nature of storage
// No additional caching needed
```

**Option 2: Separate BGG Cache Table**
```javascript
// Create bgg_cache table
{
  bgg_id: 13,
  name: "Catan",
  weight: 2.32,
  mechanics: "Dice Rolling, Hand Management",
  categories: "Economic, Negotiation",
  cached_at: "2026-03-19T10:00:00Z"
}

// Check cache before API call
const cached = $('Data Table').get('bgg_cache', { bgg_id: bggId });
if (cached && isFresh(cached.cached_at)) {
  return cached;
}
```

**Option 3: TTL-based Caching**
```javascript
// Only re-fetch if data is older than 30 days
const CACHE_TTL_DAYS = 30;

const needsRefresh = (product) => {
  if (!product.BGG_weight) return true;
  
  const lastUpdate = new Date(product.updatedAt);
  const now = new Date();
  const daysSinceUpdate = (now - lastUpdate) / (1000 * 60 * 60 * 24);
  
  return daysSinceUpdate > CACHE_TTL_DAYS;
};
```

---

## Error Handling

### Common Issues

#### Issue: Game Not Found in BGG

```javascript
// Handle missing games
if (!searchXML.items.item) {
  console.log(`Game not found in BGG: ${product.Name}`);
  // Flag for manual review
  $('Data Table').update({
    SKU: product.SKU,
    metadata: { ...product.metadata, bgg_not_found: true }
  });
  return;
}
```

#### Issue: Multiple Search Results

```javascript
// Handle ambiguous searches
if (searchXML.items.item.length > 1) {
  // Use first result or implement fuzzy matching
  const bestMatch = searchXML.items.item[0];
  console.log(`Multiple BGG results for ${product.Name}, using: ${bestMatch.name}`);
}
```

#### Issue: XML Parsing Error

```javascript
try {
  const gameXML = parseXML(response);
} catch (error) {
  console.error(`XML parsing error for ${product.Name}:`, error);
  // Skip this product, try again later
  return;
}
```

#### Issue: Rate Limit Exceeded

```javascript
// Detect rate limiting (HTTP 429 or slow responses)
if (response.status === 429) {
  console.log('BGG rate limit hit, waiting 60 seconds...');
  await new Promise(resolve => setTimeout(resolve, 60000));
  // Retry request
}
```

---

## Data Quality

### Validation

```javascript
// Validate BGG data before saving
const isValidBGGData = (data) => {
  // Weight should be 1.0-5.0
  if (data.BGG_weight && (data.BGG_weight < 1.0 || data.BGG_weight > 5.0)) {
    return false;
  }
  
  // Player count should be reasonable
  if (data.Player_Count_text && !data.Player_Count_text.match(/^\d+-?\d*$/)) {
    return false;
  }
  
  return true;
};
```

### Conflict Resolution

```javascript
// Handle conflicts between vendor data and BGG data
const resolveConflicts = (product, bggData) => {
  return {
    // Prefer vendor data for pricing
    MSRP: product.MSRP,
    Cost: product.Cost,
    
    // Prefer BGG data for game-specific fields
    BGG_weight: bggData.BGG_weight,
    Mechanics_list: bggData.Mechanics_list,
    
    // Merge categories
    Category_list: mergeLists(product.Category_list, bggData.Category_list),
    
    // Use BGG if vendor data is missing
    Player_Count_text: product.Player_Count_text || bggData.Player_Count_text,
    Play_Time: product.Play_Time || bggData.Play_Time,
    Ages_text: product.Ages_text || bggData.Ages_text
  };
};
```

---

## Advanced Features

### BGG ID Storage

Store BGG ID for future reference:

```javascript
// Add BGG ID to metadata
$('Data Table').update({
  SKU: product.SKU,
  metadata: {
    ...product.metadata,
    bgg_id: bggId,
    bgg_url: `https://boardgamegeek.com/boardgame/${bggId}`
  }
});
```

### BGG Images

```javascript
// Extract image URLs from BGG
const imageUrl = game.image;
const thumbnailUrl = game.thumbnail;

// Store in product (if needed)
// Note: Consider copyright and BGG terms of use
```

### BGG Ratings

```javascript
// Extract community ratings
const ratings = {
  average: parseFloat(game.statistics.ratings.average.$.value),
  usersrated: parseInt(game.statistics.ratings.usersrated.$.value),
  rank: parseInt(game.statistics.ratings.ranks.rank.$.value)
};

// Store in metadata
$('Data Table').update({
  SKU: product.SKU,
  metadata: {
    ...product.metadata,
    bgg_rating: ratings.average,
    bgg_rank: ratings.rank
  }
});
```

---

## Monitoring

### Track BGG Enrichment

```javascript
// Count products with BGG data
const withBGG = $('Data Table').all().filter(p => p.BGG_weight).length;
const total = $('Data Table').all().length;
const enrichmentRate = (withBGG / total * 100).toFixed(1);

console.log(`BGG Enrichment: ${withBGG}/${total} (${enrichmentRate}%)`);
```

### Alert on Failures

```javascript
// Track failed BGG lookups
const failedLookups = [];

products.forEach(product => {
  if (product.metadata?.bgg_not_found) {
    failedLookups.push(product.Name);
  }
});

if (failedLookups.length > 0) {
  // Send alert
  sendEmail({
    subject: 'BGG Enrichment Failures',
    body: `Failed to find ${failedLookups.length} games in BGG:\n${failedLookups.join('\n')}`
  });
}
```

---

## Best Practices

### ✅ DO

- **Cache BGG data** aggressively (30+ days)
- **Respect rate limits** (500ms between requests)
- **Handle errors gracefully** (missing games, parsing errors)
- **Validate data** before saving
- **Store BGG ID** for future reference
- **Run enrichment off-peak** (overnight)
- **Monitor success rate** and failures

### ❌ DON'T

- **Don't hammer BGG API** with rapid requests
- **Don't re-fetch** data unnecessarily
- **Don't ignore rate limits** (be a good citizen)
- **Don't assume** all games are in BGG
- **Don't overwrite** good vendor data with BGG data
- **Don't forget** to handle XML parsing errors

---

## Future Enhancements

### Potential Improvements

1. **Fuzzy Matching** - Better game name matching
2. **Manual BGG ID Entry** - Allow manual BGG ID specification
3. **BGG Collection Sync** - Import user's BGG collection
4. **Expansion Detection** - Identify game expansions
5. **BGG Hot List** - Track trending games
6. **User Ratings** - Display BGG community ratings

---

## Related Documentation

- [Product Data Schema](../schemas/product_data.md)
- [Vendor Management](./VENDOR_MANAGEMENT.md)
- [Setup Guide](../../SETUP_GUIDE.md)
- [Implementation Plan](../../IMPLEMENTATION_PLAN.md)

---

## External Resources

- [BGG XML API Documentation](https://boardgamegeek.com/wiki/page/BGG_XML_API2)
- [BGG Terms of Use](https://boardgamegeek.com/xmlapi/termsofuse)
- [xml2js NPM Package](https://www.npmjs.com/package/xml2js)

---

**Document Version**: 1.0  
**Last Updated**: 2026-03-19  
**Author**: Codegen AI

