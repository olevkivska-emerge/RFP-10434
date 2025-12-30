# INS-20 Address Normalization Test Results

## Overview

We tested the address normalization and deduplication feature for Emerge Insights, which uses Google Places API to normalize location data and deduplicate addresses across shipments.

**Test Date:** December 29, 2024  
**Environment:** INT  
**Total Test Cases:** 40 loads (80 stops)

---

## What We Tested

### 1. Address Variation Deduplication

The core functionality tested was whether different representations of the same physical address correctly resolve to a single normalized location. This included:

- **Street suffix variations**: Road/Rd, Avenue/Ave/Av, Street/St, Drive/Dr, Boulevard/Blvd, Parkway/Pkwy, Lane/Ln, Circle/Cir
- **Directional variations**: North/N, South/S, West/W
- **Suite/unit variations**: Suite/Ste, case sensitivity
- **Address2 exclusion**: Dock numbers and building identifiers should NOT affect deduplication

### 2. Canonical Naming

When multiple shipments reference the same physical location with different user-provided names, the first name submitted should become the "canonical" name stored in the normalized location record.

### 3. Graceful Failure Handling

When Google Places cannot geocode an address, the system should still create the load but flag the location for review (null coordinates).

### 4. User Input Preservation

Original user-provided values (location_code, name, address2) should be preserved on the stop record even when the address is normalized.

---

## Results Summary

### ✅ All Deduplication Tests Passed

| Test Group | Variations Tested | Result |
|------------|-------------------|--------|
| Tracy, CA (OAK4) | Road/Rd, North/N, address2 variations | 8 stops → 1 location ✅ |
| Avenel, NJ (EWR6) | Avenue/Ave/Av | 4 stops → 1 location ✅ |
| Newark, CA (OAK5) | Street/St | 2 stops → 1 location ✅ |
| Patterson, CA (OAK3) | Drive/Dr | 2 stops → 1 location ✅ |
| Nashville, TN | Boulevard/Blvd | 2 stops → 1 location ✅ |
| Orlando, FL | Parkway/Pkwy | 2 stops → 1 location ✅ |
| Opelousas, LA | Lane/Ln | 2 stops → 1 location ✅ |
| Grove City, OH | Circle/Cir | 2 stops → 1 location ✅ |
| Phoenix, AZ (24th St) | South/S, Street/St | 5 stops → 1 location ✅ |
| Phoenix, AZ (Buckeye) | West/W, Road/Rd | 2 stops → 1 location ✅ |
| Bellevue, WA | Suite/Ste, case sensitivity | 3 stops → 1 location ✅ |

**Overall:** 80 stops deduplicated to 44 unique locations (1.8x deduplication ratio)

---

## Unexpected Behavior: Google's Geocoding Resilience

### What We Expected

We designed three test cases (TC31-33) to verify "graceful failure" behavior when the system receives invalid or incomplete addresses:

| Test Case | Input Address | Expected Outcome |
|-----------|---------------|------------------|
| TC31 | 123 Fake Street, Nowhere, XX 00000 | Null coordinates, flagged for review |
| TC32 | PO Box 80387, Seattle, WA 98108 | Null coordinates (PO Boxes can't be geocoded) |
| TC33 | Main Street, Anytown, CA 90210 | Null coordinates (missing street number) |

### What Actually Happened

Google Places successfully geocoded all three "invalid" addresses:

| Test Case | Input | Google's Resolution | Coordinates |
|-----------|-------|---------------------|-------------|
| TC31 | 123 Fake Street, Nowhere, XX | Found a location in Colorado | (38.79, -106.53) |
| TC32 | PO Box 80387, Seattle, WA | Resolved to 4244 University Way NE, Seattle | (47.66, -122.31) |
| TC33 | Main Street, Anytown, CA | Found a Main Street in Los Angeles area | (33.93, -118.27) |

### Why This Happened

Google Places API is designed to be extremely resilient and will attempt to find the "best match" even for ambiguous or incorrect input:

1. **TC31 (Fake Address)**: Google ignored the invalid state code "XX" and found a "123 Fake Street" somewhere in the US (apparently one exists in Colorado).

2. **TC32 (PO Box)**: Rather than failing, Google resolved the PO Box to the physical location of the post office or a nearby address in Seattle.

3. **TC33 (No Street Number)**: Google found a "Main Street" in California and returned coordinates for it, even without a specific street number.

### Implications

This behavior is actually **positive for data quality** in most real-world scenarios:
- Typos and minor errors in addresses will still geocode successfully
- The system won't fail on edge cases

However, it means:
- We cannot rely on null coordinates to identify "bad" addresses
- A separate validation layer may be needed if we want to flag suspicious geocoding results
- The 11-meter coordinate proximity check (Tier 3 deduplication) becomes more important since Google might resolve ambiguous addresses to unexpected locations

### Recommendation

If detecting invalid addresses is a requirement, consider:
1. Checking the Google Places `types` array in the response for indicators like `street_address` vs `route` vs `locality`
2. Comparing the input address to the normalized output - large differences might indicate a "best guess" resolution
3. Flagging addresses where the input state/city doesn't match the resolved location

---

## Conclusion

The address normalization feature is working correctly for its primary purpose: deduplicating equivalent addresses with different formatting. Google Places handles edge cases more gracefully than anticipated, which is generally beneficial but means we cannot use geocoding failure as a validation mechanism.
