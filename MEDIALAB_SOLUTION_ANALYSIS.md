# MediaLab Price Mapping - Solution Analysis

## Customer Problem

**Customer Statement:**
> "Currently, I can only map them via their name. In case of MBS that doesn't work e.g. SOB Medium."

**Translation:**
- Customer needs to map `PRICE_ID` from `PRICE_DIM` table to `SERVICE_ID` from datacube
- Multiple prices exist with same `SERVICE_NAME` ("SOB Medium") but different:
  - `PRICE_ID` values
  - `PRICE` values
  - Effective date ranges
  - Packaging/volume variants
- Name-based mapping is ambiguous - which `PRICE_ID` corresponds to which datacube `SERVICE_ID`?

## How Current Solution Helps

### Solution Overview: External ID-Based Mapping

The implemented solution uses **external identifiers** stored in `ADDITIONAL_INFORMATION` to create a precise, unambiguous mapping between:
- MySQL `PRODUCT_ID + PACKAGING_ID` (from MediaLab orders)
- `PRICE_ID` (in PRICE_DIM)
- Future: Datacube `SERVICE_ID` (when mapping is established)

### Implementation Details

#### 1. **medialab_pipeline.py** - Price Lookup with External ID

**Before:**
```python
# Ambiguous: Multiple "SOB Medium" entries → which one?
query = "SELECT PRICE_ID FROM PRICE_DIM WHERE SERVICE_NAME = 'SOB Medium'"
# Returns: Random one (most recent by date) - may be wrong!
```

**After:**
```python
# Precise: Uses PRODUCT_ID + PACKAGING_ID for exact match
external_id = f"MEDIALAB_{product_id}_{packaging_id}"
query = "SELECT PRICE_ID FROM PRICE_DIM WHERE ADDITIONAL_INFORMATION = ?"
# Returns: Exact match - unambiguous!
```

**Benefits:**
- ✅ **Unambiguous mapping**: Each `PRODUCT_ID + PACKAGING_ID` maps to exactly one `PRICE_ID`
- ✅ **Backward compatible**: Falls back to service name if external ID not found
- ✅ **Future-proof**: Enables mapping to datacube `SERVICE_ID` when available

#### 2. **catalog_prices_pipeline.py** - Store External IDs

**What it does:**
- Stores SharePoint catalog `ID` in `ADDITIONAL_INFORMATION` when creating prices
- Format: `MEDIALAB_CATALOG_{ID}`
- Enables tracking of which SharePoint catalog item created which price

**Benefits:**
- ✅ **Traceability**: Know which catalog entry created which price
- ✅ **Deduplication**: Prevents creating duplicate prices for same catalog item
- ✅ **Future mapping**: Can link SharePoint catalog → MySQL PRODUCT_ID → PRICE_ID

### How This Solves Customer's Problem

**Scenario:** Customer has datacube with `SERVICE_ID` and needs to find corresponding `PRICE_ID`

**Current State (Before Solution):**
```
Datacube SERVICE_ID: "SVC_12345"
  ↓ (no direct mapping)
PRICE_DIM: Multiple entries with SERVICE_NAME = "SOB Medium"
  - PRICE_ID: 2028, PRICE: 21.12, EFFECTIVE: 2025-08-20
  - PRICE_ID: 2063, PRICE: 21.56, EFFECTIVE: 2025-11-06
  - PRICE_ID: 2441, PRICE: 0.00, EFFECTIVE: 2025-08-26
  ❓ Which PRICE_ID corresponds to SERVICE_ID "SVC_12345"?
```

**With Solution (After Migration):**
```
Datacube SERVICE_ID: "SVC_12345"
  ↓ (mapping table: SERVICE_ID → PRODUCT_ID + PACKAGING_ID)
MySQL: PRODUCT_ID=123, PACKAGING_ID=456
  ↓ (external ID: MEDIALAB_123_456)
PRICE_DIM: ADDITIONAL_INFORMATION = "MEDIALAB_123_456"
  - PRICE_ID: 2028 ✅ (exact match!)
```

**Key Advantage:**
- Customer can create a mapping table: `SERVICE_ID → (PRODUCT_ID, PACKAGING_ID)`
- Then use external ID to find exact `PRICE_ID`
- No ambiguity even when multiple prices share the same name

## Alternative Solutions

### Alternative 1: Merge Similar Records ❌

**Approach:** Combine all "SOB Medium" entries into a single price record

**Implementation:**
- Keep only one active price per `SERVICE_NAME`
- Close/delete duplicate entries
- Use latest price or average price

**Problems:**
- ❌ **Data Loss**: Loses historical price information
- ❌ **Audit Trail**: Can't track price changes over time
- ❌ **Doesn't Solve Problem**: Still can't map to specific `SERVICE_ID` if multiple variants exist
- ❌ **Business Logic**: Different prices may be for different packaging/volumes - merging loses this distinction
- ❌ **Violates Design**: PRICE_DIM is designed to preserve history (never delete, only close)

**Verdict:** Not recommended - loses critical information

---

### Alternative 2: Separate by Names ✅ (Partial Solution)

**Approach:** Make service names unique by adding distinguishing information

**Implementation:**
- Rename "SOB Medium" to:
  - "SOB Medium - 500ml"
  - "SOB Medium - 1L"
  - "SOB Medium - Bottle"
  - "SOB Medium - Bulk"

**Benefits:**
- ✅ **Human-readable**: Easy to understand differences
- ✅ **No schema changes**: Uses existing `SERVICE_NAME` field
- ✅ **Backward compatible**: Existing queries still work

**Problems:**
- ❌ **Manual Work**: Requires renaming all existing prices
- ❌ **Maintenance**: Need to ensure naming consistency
- ❌ **Still Ambiguous**: If two entries have same name+volume but different prices/dates
- ❌ **Doesn't Enable Mapping**: Still can't map to `SERVICE_ID` without additional work
- ❌ **Fragile**: Names can change, breaking mappings

**Verdict:** Good for human readability, but doesn't solve the mapping problem

---

### Alternative 3: Use Composite Key Lookup ✅ (Current Fallback)

**Approach:** Use multiple fields together for disambiguation

**Implementation:**
```python
query = """
SELECT PRICE_ID FROM PRICE_DIM
WHERE SERVICE_NAME = ?
AND CATEGORY_ID = ?
AND PRICE = ?
AND EFFECTIVE_FROM_TIMESTAMP <= ?
AND (EFFECTIVE_TO_TIMESTAMP IS NULL OR EFFECTIVE_TO_TIMESTAMP >= ?)
"""
```

**Benefits:**
- ✅ **More Precise**: Uses name + price + date
- ✅ **No Schema Changes**: Uses existing fields
- ✅ **Works for Most Cases**: If prices are different, this works

**Problems:**
- ❌ **Still Ambiguous**: If two entries have same name + price + overlapping dates
- ❌ **Price Changes**: If price changes, old orders won't match new price
- ❌ **Doesn't Enable Mapping**: Can't map to `SERVICE_ID` without external reference
- ❌ **Complex Logic**: Need to handle date ranges, price changes, etc.

**Verdict:** Good fallback, but not sufficient for precise mapping

---

### Alternative 4: External ID Mapping ✅✅ (Current Solution)

**Approach:** Store unique external identifier in `ADDITIONAL_INFORMATION`

**Implementation:**
- Store `MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID}` in `ADDITIONAL_INFORMATION`
- Lookup by external ID first, fallback to name

**Benefits:**
- ✅ **Unambiguous**: Each external ID maps to exactly one price
- ✅ **Enables Mapping**: Can create `SERVICE_ID → External ID → PRICE_ID` mapping
- ✅ **Preserves History**: Doesn't lose any data
- ✅ **Future-proof**: Can extend to other external systems
- ✅ **Backward Compatible**: Falls back to name lookup if external ID missing
- ✅ **Follows Pattern**: Same approach as Proteomics pipeline

**Problems:**
- ⚠️ **Migration Required**: Existing prices need external IDs added
- ⚠️ **Requires Mapping Table**: Need to map `SERVICE_ID → (PRODUCT_ID, PACKAGING_ID)`

**Verdict:** ✅ **Best Solution** - Solves the problem while preserving data

---

### Alternative 5: Add SERVICE_ID Column to PRICE_DIM ✅ (Future Enhancement)

**Approach:** Add dedicated `SERVICE_ID` column to `PRICE_DIM` table

**Implementation:**
```sql
ALTER TABLE PRICE_DIM ADD COLUMN SERVICE_ID VARCHAR(100);
CREATE INDEX IDX_PRICE_DIM_SERVICE_ID ON PRICE_DIM(SERVICE_ID);
```

**Benefits:**
- ✅ **Direct Mapping**: `SERVICE_ID` → `PRICE_ID` in one query
- ✅ **Clean Schema**: Dedicated field for this purpose
- ✅ **Better Performance**: Indexed lookup
- ✅ **Type Safety**: Explicit column vs. string in `ADDITIONAL_INFORMATION`

**Problems:**
- ❌ **Schema Change**: Requires ALTER TABLE (may need approval)
- ❌ **Migration**: Need to populate `SERVICE_ID` for all existing prices
- ❌ **Not Universal**: Only works if all prices have `SERVICE_ID` (some may not)

**Verdict:** Good future enhancement, but `ADDITIONAL_INFORMATION` is more flexible

---

## Recommendation: Hybrid Approach

**Best Solution:** Combine External ID Mapping (current) + Name Separation (optional)

### Phase 1: External ID Mapping (Current Implementation) ✅
- Use `MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID}` for precise lookup
- Enables `SERVICE_ID → PRICE_ID` mapping
- Preserves all data and history

### Phase 2: Name Enhancement (Optional)
- Update `SERVICE_NAME` to include distinguishing info for human readability
- Example: "SOB Medium (500ml)" instead of just "SOB Medium"
- Makes it easier for users to understand differences
- External ID still used for precise mapping

### Phase 3: Future Enhancement (Optional)
- Add `SERVICE_ID` column to `PRICE_DIM` if needed
- Populate from mapping table
- Provides direct lookup path

## Answer to Your Question

> "Are there any other solution? AFAIK, we can merge similar records or we really have to separate them by names."

**Answer:**

1. **Merging similar records** ❌ - Not recommended because:
   - Loses historical price data
   - Violates audit trail requirements
   - Doesn't solve the mapping problem
   - Different prices may be legitimate (different packaging, volumes, time periods)

2. **Separating by names** ✅ - Good for human readability, but:
   - Doesn't solve the mapping problem alone
   - Requires manual work
   - Can still be ambiguous
   - **Best used together with external ID mapping**

3. **External ID mapping** ✅✅ - **Best solution** because:
   - Solves the mapping problem
   - Preserves all data
   - Enables `SERVICE_ID → PRICE_ID` mapping
   - Backward compatible
   - Follows established patterns (Proteomics pipeline)

**Recommendation:** Keep external ID mapping (current solution) + optionally enhance names for readability. Don't merge records - they likely represent legitimate different prices that should be preserved.
