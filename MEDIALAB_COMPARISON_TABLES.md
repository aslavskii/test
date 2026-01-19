# MediaLab Price Records - Solution Comparison

This document shows real MediaLab records from Snowflake and how they would look with each solution approach.

## Current State (Original)

| PRICE_ID | SERVICE_NAME | PRICE | UNIT_TYPE | EFFECTIVE_FROM | EFFECTIVE_TO | ADDITIONAL_INFO |
|----------|--------------|-------|-----------|----------------|--------------|-----------------|
| 7208 | 0,5M EDTA | 18.41 | item | 2025-12-03 | OPEN | (empty) |
| 4794 | 0,5M EDTA | 18.41 | ml | 2025-11-06 | OPEN | (empty) |
| 3815 | 0,5M EDTA | 0.00 | Bottle 500ml | 2025-10-28 | OPEN | (empty) |
| 2467 | 0,5M EDTA | 18.41 | ml | 2025-08-26 | OPEN | (empty) |
| 2079 | 0,5M EDTA | 18.41 | ml | 2025-08-20 | 2025-11-06 | (empty) |
| 3828 | 0,7x PBS Biooptics | 0.00 | Bottle 2L | 2025-10-28 | OPEN | (empty) |
| 7524 | 0.5 Gamborg, MES medium | 45.67 | item | 2025-12-03 | OPEN | (empty) |
| 4738 | 0.5 Gamborg, MES medium | 45.67 | ml | 2025-11-06 | OPEN | (empty) |
| 4214 | 0.5 Gamborg, MES medium | 0.00 | Bottle 500ml | 2025-10-28 | OPEN | (empty) |
| 2417 | 0.5 Gamborg, MES medium | 45.67 | ml | 2025-08-26 | OPEN | (empty) |
| 2134 | 0.5 Gamborg, MES medium | 45.67 | ml | 2025-08-20 | 2025-11-06 | (empty) |
| 12405 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 21.76 | item | 2026-01-18 | OPEN | (empty) |
| 12404 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 5.44 | item | 2026-01-18 | 2026-01-18 | (empty) |
| 11906 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 5.44 | item | 2026-01-11 | 2026-01-11 | (empty) |
| 11908 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 21.76 | item | 2026-01-11 | 2026-01-18 | (empty) |
| 11306 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 5.44 | item | 2026-01-04 | 2026-01-04 | (empty) |
| 11405 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 21.76 | item | 2026-01-04 | 2026-01-11 | (empty) |
| 10508 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 21.76 | item | 2025-12-28 | 2026-01-04 | (empty) |
| 10306 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 5.44 | item | 2025-12-28 | 2025-12-28 | (empty) |
| 9805 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 5.44 | item | 2025-12-21 | 2025-12-21 | (empty) |

**Problem:** Multiple records with same `SERVICE_NAME` but different `PRICE_ID`, `PRICE`, `UNIT_TYPE`, and dates. Cannot map `SERVICE_ID` → `PRICE_ID` unambiguously.

---

## Solution 1: External ID Mapping

**Approach:** Store `MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID}` in `ADDITIONAL_INFORMATION` column.

| PRICE_ID | SERVICE_NAME | PRICE | UNIT_TYPE | EFFECTIVE_FROM | EFFECTIVE_TO | ADDITIONAL_INFO | Lookup Method |
|----------|--------------|-------|-----------|----------------|--------------|-----------------|---------------|
| 7208 | 0,5M EDTA | 18.41 | item | 2025-12-03 | OPEN | **MEDIALAB_1008_208** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1008_208'` |
| 4794 | 0,5M EDTA | 18.41 | ml | 2025-11-06 | OPEN | **MEDIALAB_1094_244** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1094_244'` |
| 3815 | 0,5M EDTA | 0.00 | Bottle 500ml | 2025-10-28 | OPEN | **MEDIALAB_1015_215** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1015_215'` |
| 2467 | 0,5M EDTA | 18.41 | ml | 2025-08-26 | OPEN | **MEDIALAB_1067_217** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1067_217'` |
| 2079 | 0,5M EDTA | 18.41 | ml | 2025-08-20 | 2025-11-06 | **MEDIALAB_1079_229** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1079_229'` |
| 3828 | 0,7x PBS Biooptics | 0.00 | Bottle 2L | 2025-10-28 | OPEN | **MEDIALAB_1028_228** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1028_228'` |
| 7524 | 0.5 Gamborg, MES medium | 45.67 | item | 2025-12-03 | OPEN | **MEDIALAB_1024_224** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1024_224'` |
| 4738 | 0.5 Gamborg, MES medium | 45.67 | ml | 2025-11-06 | OPEN | **MEDIALAB_1038_238** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1038_238'` |
| 4214 | 0.5 Gamborg, MES medium | 0.00 | Bottle 500ml | 2025-10-28 | OPEN | **MEDIALAB_1014_214** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1014_214'` |
| 2417 | 0.5 Gamborg, MES medium | 45.67 | ml | 2025-08-26 | OPEN | **MEDIALAB_1017_217** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1017_217'` |
| 2134 | 0.5 Gamborg, MES medium | 45.67 | ml | 2025-08-20 | 2025-11-06 | **MEDIALAB_1034_234** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1034_234'` |
| 12405 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 21.76 | item | 2026-01-18 | OPEN | **MEDIALAB_1005_205** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1005_205'` |
| 12404 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 5.44 | item | 2026-01-18 | 2026-01-18 | **MEDIALAB_1004_204** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1004_204'` |
| 11906 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 5.44 | item | 2026-01-11 | 2026-01-11 | **MEDIALAB_1006_206** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1006_206'` |
| 11908 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 21.76 | item | 2026-01-11 | 2026-01-18 | **MEDIALAB_1008_208** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1008_208'` |
| 11306 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 5.44 | item | 2026-01-04 | 2026-01-04 | **MEDIALAB_1006_206** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1006_206'` |
| 11405 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 21.76 | item | 2026-01-04 | 2026-01-11 | **MEDIALAB_1005_205** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1005_205'` |
| 10508 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 21.76 | item | 2025-12-28 | 2026-01-04 | **MEDIALAB_1008_208** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1008_208'` |
| 10306 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 5.44 | item | 2025-12-28 | 2025-12-28 | **MEDIALAB_1006_206** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1006_206'` |
| 9805 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | 5.44 | item | 2025-12-21 | 2025-12-21 | **MEDIALAB_1005_205** | `ADDITIONAL_INFORMATION = 'MEDIALAB_1005_205'` |

**Mapping Flow:**
```
Datacube SERVICE_ID 
  → (PRODUCT_ID, PACKAGING_ID) [from mapping table]
  → MEDIALAB_{PRODUCT_ID}_{PACKAGING_ID} [external ID]
  → PRICE_ID [unambiguous lookup]
```

**Example:**
- Customer has `SERVICE_ID = "SVC_12345"` in datacube
- Mapping table: `SVC_12345 → (PRODUCT_ID=1008, PACKAGING_ID=208)`
- Lookup: `ADDITIONAL_INFORMATION = 'MEDIALAB_1008_208'` → `PRICE_ID = 7208` ✅

**Benefits:**
- ✅ **Unambiguous**: Each external ID maps to exactly one PRICE_ID
- ✅ **Enables mapping**: SERVICE_ID → PRICE_ID via external ID
- ✅ **Preserves data**: All historical records kept
- ✅ **Backward compatible**: Falls back to name lookup if external ID missing

**Migration Required:**
- Update `ADDITIONAL_INFORMATION` for all MediaLab prices
- Create mapping table: `SERVICE_ID → (PRODUCT_ID, PACKAGING_ID)`

---

## Solution 2: Separate by Names

**Approach:** Enhance `SERVICE_NAME` to include distinguishing information (unit, price, date).

| PRICE_ID | SERVICE_NAME (Original) | SERVICE_NAME (Enhanced) | PRICE | UNIT_TYPE | EFFECTIVE_FROM | EFFECTIVE_TO | Lookup Method |
|----------|------------------------|-------------------------|-------|-----------|----------------|--------------|---------------|
| 7208 | 0,5M EDTA | **0,5M EDTA (item) - €18.41 [2025-12]** | 18.41 | item | 2025-12-03 | OPEN | `SERVICE_NAME = '0,5M EDTA (item) - €18.41 [2025-12]'` |
| 4794 | 0,5M EDTA | **0,5M EDTA (ml) - €18.41 [2025-11]** | 18.41 | ml | 2025-11-06 | OPEN | `SERVICE_NAME = '0,5M EDTA (ml) - €18.41 [2025-11]'` |
| 3815 | 0,5M EDTA | **0,5M EDTA (Bottle 500ml) [2025-10]** | 0.00 | Bottle 500ml | 2025-10-28 | OPEN | `SERVICE_NAME = '0,5M EDTA (Bottle 500ml) [2025-10]'` |
| 2467 | 0,5M EDTA | **0,5M EDTA (ml) - €18.41 [2025-08]** | 18.41 | ml | 2025-08-26 | OPEN | `SERVICE_NAME = '0,5M EDTA (ml) - €18.41 [2025-08]'` |
| 2079 | 0,5M EDTA | **0,5M EDTA (ml) - €18.41 [2025-08] (closed)** | 18.41 | ml | 2025-08-20 | 2025-11-06 | `SERVICE_NAME = '0,5M EDTA (ml) - €18.41 [2025-08] (closed)'` |
| 3828 | 0,7x PBS Biooptics | **0,7x PBS Biooptics (Bottle 2L) [2025-10]** | 0.00 | Bottle 2L | 2025-10-28 | OPEN | `SERVICE_NAME = '0,7x PBS Biooptics (Bottle 2L) [2025-10]'` |
| 7524 | 0.5 Gamborg, MES medium | **0.5 Gamborg, MES medium (item) - €45.67 [2025-12]** | 45.67 | item | 2025-12-03 | OPEN | `SERVICE_NAME = '0.5 Gamborg, MES medium (item) - €45.67 [2025-12]'` |
| 4738 | 0.5 Gamborg, MES medium | **0.5 Gamborg, MES medium (ml) - €45.67 [2025-11]** | 45.67 | ml | 2025-11-06 | OPEN | `SERVICE_NAME = '0.5 Gamborg, MES medium (ml) - €45.67 [2025-11]'` |
| 4214 | 0.5 Gamborg, MES medium | **0.5 Gamborg, MES medium (Bottle 500ml) [2025-10]** | 0.00 | Bottle 500ml | 2025-10-28 | OPEN | `SERVICE_NAME = '0.5 Gamborg, MES medium (Bottle 500ml) [2025-10]'` |
| 2417 | 0.5 Gamborg, MES medium | **0.5 Gamborg, MES medium (ml) - €45.67 [2025-08]** | 45.67 | ml | 2025-08-26 | OPEN | `SERVICE_NAME = '0.5 Gamborg, MES medium (ml) - €45.67 [2025-08]'` |
| 2134 | 0.5 Gamborg, MES medium | **0.5 Gamborg, MES medium (ml) - €45.67 [2025-08] (closed)** | 45.67 | ml | 2025-08-20 | 2025-11-06 | `SERVICE_NAME = '0.5 Gamborg, MES medium (ml) - €45.67 [2025-08] (closed)'` |
| 12405 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | **0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €21.76 [2026-01]** | 21.76 | item | 2026-01-18 | OPEN | `SERVICE_NAME = '0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €21.76 [2026-01]'` |
| 12404 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | **0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €5.44 [2026-01]** | 5.44 | item | 2026-01-18 | 2026-01-18 | `SERVICE_NAME = '0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €5.44 [2026-01]'` |
| 11906 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | **0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €5.44 [2026-01]** | 5.44 | item | 2026-01-11 | 2026-01-11 | `SERVICE_NAME = '0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €5.44 [2026-01]'` ⚠️ |
| 11908 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | **0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €21.76 [2026-01]** | 21.76 | item | 2026-01-11 | 2026-01-18 | `SERVICE_NAME = '0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €21.76 [2026-01]'` |
| 11306 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | **0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €5.44 [2026-01]** | 5.44 | item | 2026-01-04 | 2026-01-04 | `SERVICE_NAME = '0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €5.44 [2026-01]'` ⚠️ |
| 11405 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | **0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €21.76 [2026-01]** | 21.76 | item | 2026-01-04 | 2026-01-11 | `SERVICE_NAME = '0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €21.76 [2026-01]'` |
| 10508 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | **0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €21.76 [2025-12]** | 21.76 | item | 2025-12-28 | 2026-01-04 | `SERVICE_NAME = '0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €21.76 [2025-12]'` |
| 10306 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | **0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €5.44 [2025-12]** | 5.44 | item | 2025-12-28 | 2025-12-28 | `SERVICE_NAME = '0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €5.44 [2025-12]'` |
| 9805 | 0.5 Gamborg, MES, 0.8% agar for Marchantia | **0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €5.44 [2025-12]** | 5.44 | item | 2025-12-21 | 2025-12-21 | `SERVICE_NAME = '0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €5.44 [2025-12]'` ⚠️ |

**⚠️ Ambiguity Issues:**
- PRICE_ID 12404, 11906, 11306, 9805 all have same enhanced name: `'0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €5.44 [2026-01]'` or `[2025-12]`
- PRICE_ID 12405, 11908, 11405, 10508 all have same enhanced name: `'0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €21.76 [2026-01]'` or `[2025-12]`
- Still ambiguous! Need to add more distinguishing info (like exact timestamp or PRICE_ID in name)

**Benefits:**
- ✅ **Human-readable**: Easy to understand differences
- ✅ **No schema changes**: Uses existing `SERVICE_NAME` field
- ✅ **Visual clarity**: Can see unit, price, date in name

**Problems:**
- ❌ **Still ambiguous**: Multiple records can have same enhanced name (see ⚠️ above)
- ❌ **Doesn't solve mapping**: Cannot map `SERVICE_ID → PRICE_ID` without additional work
- ❌ **Fragile**: Names can change, breaking lookups
- ❌ **Manual work**: Need to rename all prices
- ❌ **Long names**: Some names become very long and unwieldy

---

## Side-by-Side Comparison

### Example: "0,5M EDTA" (5 variants)

| Approach | PRICE_ID | Lookup Method | Result |
|----------|----------|---------------|--------|
| **Original** | 7208, 4794, 3815, 2467, 2079 | `SERVICE_NAME = '0,5M EDTA'` | ❌ Returns random one (ambiguous) |
| **Solution 1: External ID** | 7208 | `ADDITIONAL_INFORMATION = 'MEDIALAB_1008_208'` | ✅ Exact match |
| **Solution 1: External ID** | 4794 | `ADDITIONAL_INFORMATION = 'MEDIALAB_1094_244'` | ✅ Exact match |
| **Solution 1: External ID** | 3815 | `ADDITIONAL_INFORMATION = 'MEDIALAB_1015_215'` | ✅ Exact match |
| **Solution 2: Enhanced Name** | 7208 | `SERVICE_NAME = '0,5M EDTA (item) - €18.41 [2025-12]'` | ✅ Unique (for this case) |
| **Solution 2: Enhanced Name** | 4794 | `SERVICE_NAME = '0,5M EDTA (ml) - €18.41 [2025-11]'` | ✅ Unique (for this case) |
| **Solution 2: Enhanced Name** | 3815 | `SERVICE_NAME = '0,5M EDTA (Bottle 500ml) [2025-10]'` | ✅ Unique (for this case) |

### Example: "0.5 Gamborg, MES, 0.8% agar for Marchantia" (9 variants)

| Approach | PRICE_ID | Lookup Method | Result |
|----------|----------|---------------|--------|
| **Original** | 12405, 12404, 11906, 11908, 11306, 11405, 10508, 10306, 9805 | `SERVICE_NAME = '0.5 Gamborg, MES, 0.8% agar for Marchantia'` | ❌ Returns random one (ambiguous) |
| **Solution 1: External ID** | 12405 | `ADDITIONAL_INFORMATION = 'MEDIALAB_1005_205'` | ✅ Exact match |
| **Solution 1: External ID** | 12404 | `ADDITIONAL_INFORMATION = 'MEDIALAB_1004_204'` | ✅ Exact match |
| **Solution 1: External ID** | 11906 | `ADDITIONAL_INFORMATION = 'MEDIALAB_1006_206'` | ✅ Exact match |
| **Solution 2: Enhanced Name** | 12405 | `SERVICE_NAME = '0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €21.76 [2026-01]'` | ⚠️ **AMBIGUOUS** - Same as 11908, 11405 |
| **Solution 2: Enhanced Name** | 12404 | `SERVICE_NAME = '0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €5.44 [2026-01]'` | ⚠️ **AMBIGUOUS** - Same as 11906, 11306 |
| **Solution 2: Enhanced Name** | 11906 | `SERVICE_NAME = '0.5 Gamborg, MES, 0.8% agar for Marchantia (item) - €5.44 [2026-01]'` | ⚠️ **AMBIGUOUS** - Same as 12404, 11306 |

---

## Summary & Recommendation

### Solution 1: External ID Mapping ✅ **RECOMMENDED**

**Pros:**
- ✅ **100% Unambiguous**: Each external ID maps to exactly one PRICE_ID
- ✅ **Enables SERVICE_ID mapping**: Can create mapping table `SERVICE_ID → (PRODUCT_ID, PACKAGING_ID) → PRICE_ID`
- ✅ **Preserves all data**: No information loss
- ✅ **Backward compatible**: Falls back to name lookup if external ID missing
- ✅ **Future-proof**: Can extend to other external systems

**Cons:**
- ⚠️ Requires migration: Update `ADDITIONAL_INFORMATION` for existing prices
- ⚠️ Requires mapping table: `SERVICE_ID → (PRODUCT_ID, PACKAGING_ID)`

**Verdict:** ✅ **Best solution** - Solves the problem completely

---

### Solution 2: Separate by Names ⚠️ **PARTIAL SOLUTION**

**Pros:**
- ✅ Human-readable
- ✅ No schema changes
- ✅ Easy to understand visually

**Cons:**
- ❌ **Still ambiguous**: Multiple records can have same enhanced name (see examples above)
- ❌ **Doesn't solve SERVICE_ID mapping**: Cannot map `SERVICE_ID → PRICE_ID` without additional work
- ❌ **Fragile**: Names can change, breaking lookups
- ❌ **Manual work**: Need to rename all prices
- ❌ **Long names**: Some become unwieldy

**Verdict:** ⚠️ **Not sufficient alone** - Good for readability, but doesn't solve the mapping problem

---

## Final Recommendation

**Use Solution 1 (External ID Mapping) as primary approach.**

**Optionally enhance names (Solution 2) for human readability**, but rely on external IDs for precise mapping.

**Why?**
1. Solution 1 solves the customer's problem: Enables `SERVICE_ID → PRICE_ID` mapping
2. Solution 1 is unambiguous: No ambiguity even with identical names
3. Solution 2 still has ambiguity issues (see examples above)
4. Solution 2 doesn't enable SERVICE_ID mapping without additional work

**Migration Path:**
1. Implement Solution 1 (External ID Mapping) - **Required**
2. Optionally enhance names for readability - **Nice to have**
3. Create mapping table: `SERVICE_ID → (PRODUCT_ID, PACKAGING_ID)` - **Required for customer**
