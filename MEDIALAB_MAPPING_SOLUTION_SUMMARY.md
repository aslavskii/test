# MediaLab Price ID Mapping - Solution Summary

## Problem

Customer reported: "Currently, I can only map them via their name. In case of MBS that doesn't work e.g. SOB Medium."

**Root Cause:** Multiple entries in `PRICE_DIM` have the same `SERVICE_NAME` (e.g., "SOB Medium") but different `PRICE_ID` values, prices, and effective dates. The current lookup only uses service name, which returns ambiguous results.

## Solution Implemented

### 1. External ID-Based Price Lookup

Updated `medialab_pipeline.py` to use external IDs for precise price matching:

- **Primary Strategy:** Look up prices using `PRODUCT_ID + PACKAGING_ID` stored in `ADDITIONAL_INFORMATION`
- **Format:** `MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID}`
- **Fallback:** Service name lookup (for backward compatibility)

### 2. External ID Storage

Updated `catalog_prices_pipeline.py` to store external IDs when creating prices:

- Stores SharePoint catalog ID: `MEDIALAB_CATALOG_{ID}`
- This enables future mapping between SharePoint catalog and MySQL PRODUCT_ID

## Code Changes

### Files Modified

1. **`pipeline_scripts/services/medialab/medialab_pipeline.py`**
   - Updated `_get_price_id()` to accept and use `product_id` and `packaging_id`
   - Added external ID lookup logic
   - Maintains backward compatibility with service name lookup

2. **`pipeline_scripts/services/catalog_prices/catalog_prices_pipeline.py`**
   - Updated to store SharePoint catalog ID in `ADDITIONAL_INFORMATION`
   - Updated price lookup/update logic to use external IDs

## Migration Required

**Important:** Existing MediaLab prices need to be updated with external IDs.

### Current State
- New prices created by `catalog_prices_pipeline` will have `MEDIALAB_CATALOG_{ID}`
- But they still need `MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID}` for the medialab pipeline to use

### Options

**Option 1: Manual Update (Recommended for now)**
- Update `PRICE_DIM.ADDITIONAL_INFORMATION` for existing MediaLab prices
- Format: `MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID}`
- Can be done via SQL UPDATE statements

**Option 2: Automated Mapping Script**
- Create a script to map SharePoint catalog IDs to PRODUCT_ID + PACKAGING_ID
- Requires understanding the relationship between SharePoint catalog and MySQL products

**Option 3: Gradual Migration**
- New prices will work immediately (after catalog_prices update)
- Existing prices will use fallback (service name lookup) until migrated
- Migrate prices as needed

## Testing

Before deploying:

1. **Test price lookup with external ID:**
   ```python
   # Should find price using PRODUCT_ID + PACKAGING_ID
   price_id = pipeline._get_price_id(
       "SOB Medium", 
       "Bottle", 
       datetime.now(),
       product_id=123,
       packaging_id=456
   )
   ```

2. **Test fallback behavior:**
   ```python
   # Should fall back to service name if external ID not found
   price_id = pipeline._get_price_id(
       "SOB Medium", 
       "Bottle", 
       datetime.now()
   )
   ```

3. **Test with real data:**
   - Run medialab pipeline with `--dry-run` flag
   - Verify prices are found correctly
   - Check logs for warnings about missing external IDs

## Questions for Teamlead/Customer

1. **Mapping Relationship:**
   - Does SharePoint catalog `ID` correspond to MySQL `PRODUCT_ID`?
   - Or do we need a separate mapping table?

2. **Migration Priority:**
   - Should we migrate all existing prices immediately?
   - Or can we do it gradually as prices are updated?

3. **External ID Format:**
   - Is `MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID}` the correct format?
   - Or should we use a different identifier?

## Next Steps

1. ✅ Code changes implemented
2. ⏳ Test with real data
3. ⏳ Create migration script (if needed)
4. ⏳ Update documentation
5. ⏳ Deploy to test environment
6. ⏳ Get feedback from teamlead/customer

## Example SQL for Migration

```sql
-- Example: Update existing price with external ID
-- Replace {PRICE_ID}, {PRODUCT_ID}, {PACKAGING_ID} with actual values

UPDATE PRODUCTION.PROPOSED_BILLING.PRICE_DIM
SET ADDITIONAL_INFORMATION = CONCAT('MEDIALAB_', {PRODUCT_ID}, '_', {PACKAGING_ID})
WHERE PRICE_ID = {PRICE_ID}
AND CATEGORY_ID = (SELECT CATEGORY_ID FROM PRODUCTION.PROPOSED_BILLING.CATEGORY_DIM 
                   WHERE CATEGORY_NAME = 'Media Lab');
```

## Notes

- The solution maintains **backward compatibility** - prices without external IDs will still work via service name lookup
- The medialab pipeline will log warnings when it falls back to service name lookup, helping identify prices that need migration
- This follows the same pattern used by the proteomics pipeline (which uses `ADDITIONAL_INFORMATION` for external service IDs)
