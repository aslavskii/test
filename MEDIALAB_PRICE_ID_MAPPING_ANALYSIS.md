# MediaLab Price ID Mapping Analysis

## Problem Statement

**Customer Issue:**
> "Currently, I can only map them via their name. In case of MBS that doesn't work e.g. SOB Medium."

**Teamlead Comment:**
> "I remember we were also once talking about all of these items that have similar names but just differentiate in either volume or some other metric, but I can't seem to remember what we agreed on with these"

## Root Cause

The MediaLab pipeline currently looks up prices in `PRICE_DIM` **only by SERVICE_NAME**:

```python
# Current implementation in medialab_pipeline.py
query = """
SELECT PRICE_ID
FROM {price_dim_table}
WHERE UPPER(SERVICE_NAME) = UPPER(%s)
AND CATEGORY_ID = %s
AND EFFECTIVE_FROM_TIMESTAMP <= %s
AND (EFFECTIVE_TO_TIMESTAMP IS NULL OR EFFECTIVE_TO_TIMESTAMP >= %s)
ORDER BY EFFECTIVE_FROM_TIMESTAMP DESC
LIMIT 1
"""
```

**Problem:** Multiple entries in `PRICE_DIM` can have the same `SERVICE_NAME` (e.g., "SOB Medium") but different:
- `PRICE_ID` values
- `PRICE` values  
- Effective date ranges
- Other differentiating attributes (volume, packaging, etc.)

When multiple entries match, the query returns only one (the most recent by effective date), which may not be the correct one for the specific order.

## Available Data

From the MediaLab MySQL database (`view_orders`), we have:
- `PRODUCT_ID` - Unique product identifier
- `PRODUCT_NAME` - Product name (e.g., "SOB Medium")
- `PACKAGING_ID` - Unique packaging identifier
- `PACKAGING_NAME` - Packaging name
- `PACKAGING_MULTIPLICATOR` - Packaging multiplier

## Solution Approach

Following the pattern used by the **Proteomics pipeline**, we should:

1. **Use External ID for Mapping**: Store a unique external identifier in `PRICE_DIM.ADDITIONAL_INFORMATION`
2. **Lookup by External ID First**: Update `_get_price_id()` to prioritize lookup by external ID
3. **Fallback to Service Name**: Keep service name lookup as fallback for backward compatibility

### Proposed External ID Format

**Option 1: PRODUCT_ID only**
- Format: `MEDIALAB_PRODUCT_{PRODUCT_ID}`
- Pros: Simple, unique per product
- Cons: Doesn't differentiate packaging variants

**Option 2: PRODUCT_ID + PACKAGING_ID** (RECOMMENDED)
- Format: `MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID}`
- Pros: Unique per product+packaging combination, handles all variants
- Cons: Slightly more complex

**Option 3: Composite key**
- Format: JSON in ADDITIONAL_INFORMATION: `{"product_id": 123, "packaging_id": 456}`
- Pros: Structured, extensible
- Cons: More complex queries, harder to maintain

## Implementation Plan

### Phase 1: Update Price Lookup Logic

1. **Modify `_get_price_id()` method** in `medialab_pipeline.py`:
   - Add parameter for `product_id` and `packaging_id`
   - First try lookup by `ADDITIONAL_INFORMATION` (external ID)
   - Fallback to current service name lookup if external ID not found
   - Update cache key to include external ID

2. **Update `transform()` method**:
   - Pass `PRODUCT_ID` and `PACKAGING_ID` to `_get_price_id()`

### Phase 2: Update Price Creation/Management

**Note:** MediaLab pipeline does NOT auto-create prices (unlike some other pipelines). Prices must be manually created in `PRICE_DIM`.

However, we need to ensure that when prices ARE created (manually or via catalog_prices pipeline), they include the external ID in `ADDITIONAL_INFORMATION`.

**Action Items:**
1. Check `catalog_prices_pipeline.py` - does it create MediaLab prices? If yes, update it to include external ID
2. Document the required format for manual price creation
3. Consider creating a migration script to backfill `ADDITIONAL_INFORMATION` for existing MediaLab prices

### Phase 3: Migration & Backfill

1. **Query existing MediaLab prices** to understand current state
2. **Create mapping** from existing prices to PRODUCT_ID/PACKAGING_ID combinations
3. **Backfill ADDITIONAL_INFORMATION** for existing prices where possible
4. **Document unmapped prices** for manual review

## Code Changes Required

### File: `pipeline_scripts/services/medialab/medialab_pipeline.py`

**Changes:**
1. Update `_get_price_id()` signature and implementation
2. Update `transform()` to pass product/packaging IDs
3. Update cache key format

### File: `pipeline_scripts/services/catalog_prices/catalog_prices_pipeline.py` (if applicable)

**Changes:**
1. When creating MediaLab prices, include external ID in `ADDITIONAL_INFORMATION`
2. Format: `MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID}`

## Testing Strategy

1. **Unit Tests**: Test price lookup with external ID
2. **Integration Tests**: Test with real data from MySQL
3. **Backward Compatibility**: Verify fallback to service name still works
4. **Edge Cases**: 
   - Missing PRODUCT_ID or PACKAGING_ID
   - Multiple prices with same external ID (shouldn't happen, but handle gracefully)
   - Prices without external ID (fallback scenario)

## Current State Analysis

### How MediaLab Prices Are Created

**Via `catalog_prices_pipeline.py`:**
- Fetches from SharePoint table: `VBCF_SERVICE_MEDIA_LAB_CATALOG`
- Has `ID` field (SharePoint list item ID) - stored as `SOURCE_ID` in pipeline
- Currently creates prices in `PRICE_DIM` with:
  - `SERVICE_NAME` (from `LINK_TITLE`)
  - `PRICE` (from `PRICE_1` or `PRICE_2`)
  - `CATEGORY_ID` (MBS Products)
  - **BUT:** Does NOT store `SOURCE_ID` in `ADDITIONAL_INFORMATION`

### Data Flow

1. **SharePoint Catalog** → `catalog_prices_pipeline` → **PRICE_DIM** (creates prices)
2. **MySQL `view_orders`** → `medialab_pipeline` → **BILLING_FACT** (creates billing records)

**Gap:** No direct link between MySQL `PRODUCT_ID`/`PACKAGING_ID` and SharePoint catalog `ID`.

## Questions to Resolve

1. **Does MySQL `view_orders` have a reference to SharePoint catalog ID?**
   - Need to check if `PRODUCT_ID` in MySQL corresponds to SharePoint catalog `ID`
   - Or if there's another field that links them

2. **What does customer mean by "service ID from the datacube"?**
   - Is it the SharePoint catalog `ID`?
   - Or a separate datacube service ID field?
   - Or the MySQL `PRODUCT_ID`?

3. **Should we use PRODUCT_ID alone or PRODUCT_ID + PACKAGING_ID?**
   - Recommendation: PRODUCT_ID + PACKAGING_ID for complete uniqueness
   - But need to verify if PRODUCT_ID alone is sufficient

4. **Migration strategy for existing prices:**
   - Can we automatically map existing prices to PRODUCT_ID/PACKAGING_ID?
   - Or do we need manual mapping?

## Implementation Status

1. ✅ Create analysis document (this file)
2. ✅ Check how MediaLab prices are currently created (via catalog_prices pipeline)
3. ✅ Implement updated `_get_price_id()` method in medialab_pipeline.py
4. ✅ Update price creation logic in catalog_prices_pipeline.py
5. ⏳ Create migration/backfill script for existing prices
6. ⏳ Test with real data
7. ⏳ Document for teamlead and customer

## Implementation Details

### Changes Made

#### 1. `medialab_pipeline.py` - Updated Price Lookup

**File:** `pipeline_scripts/services/medialab/medialab_pipeline.py`

**Changes:**
- Updated `_get_price_id()` method to accept `product_id` and `packaging_id` parameters
- Added external ID lookup using format: `MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID}`
- Falls back to service name lookup for backward compatibility
- Updated `transform()` method to pass `PRODUCT_ID` and `PACKAGING_ID` to price lookup

**External ID Format:**
- `MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID}` - Used by medialab pipeline for precise matching

#### 2. `catalog_prices_pipeline.py` - Store External IDs

**File:** `pipeline_scripts/services/catalog_prices/catalog_prices_pipeline.py`

**Changes:**
- Updated price insertion to store SharePoint catalog ID in `ADDITIONAL_INFORMATION`
- Format: `MEDIALAB_CATALOG_{ID}` (where ID is SharePoint list item ID)
- Updated price lookup/update logic to check by external ID first

**External ID Format:**
- `MEDIALAB_CATALOG_{ID}` - SharePoint catalog ID (for reference)

### Migration Required

**Important:** Existing MediaLab prices in `PRICE_DIM` need to be updated with external IDs:

1. **For prices created from SharePoint catalog:**
   - Already have `MEDIALAB_CATALOG_{ID}` (after catalog_prices pipeline update)
   - Need to add `MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID}` for medialab pipeline lookup

2. **For prices created manually:**
   - Need to add both external ID formats if applicable

**Migration Options:**
- **Option A:** Manual mapping - Update `ADDITIONAL_INFORMATION` with `MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID}` for each price
- **Option B:** Automated mapping - Create script to map SharePoint catalog IDs to PRODUCT_ID + PACKAGING_ID combinations
- **Option C:** Hybrid - Use service name lookup as fallback until mapping is complete

### Testing Checklist

- [ ] Test price lookup with external ID (PRODUCT_ID + PACKAGING_ID)
- [ ] Test fallback to service name lookup
- [ ] Test catalog_prices pipeline creates prices with external ID
- [ ] Test with multiple prices having same SERVICE_NAME
- [ ] Verify backward compatibility (prices without external ID still work)
- [ ] Test with real MySQL data

### Next Steps

1. ⏳ Create migration script to backfill `ADDITIONAL_INFORMATION` for existing prices
2. ⏳ Test with real data from MySQL and SharePoint
3. ⏳ Document solution for teamlead and customer
4. ⏳ Consider creating mapping table if PRODUCT_ID doesn't directly correspond to SharePoint catalog ID
