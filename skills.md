## Skill 1: Routing Number Search
Purpose: Locate FI detail page from routing number.

Input:
- Routing number string

Process:
1. Build query: `routing number <routing_number> site:usbanklocations.com`
2. Open top FI result page on US Bank Locations
3. Extract `bank_name` and optional location context

Output:
- `bank_name`
- FI detail page URL
- source confirmation as `US Bank Locations`

## Skill 2: Website URL Finder
Purpose: Derive `home_url` and `login_url`.

Input:
- FI detail page HTML
- bank name

Process:
1. Parse anchors and collect URL candidates
2. Score login candidates by anchor text:
- `Online Banking`
- `Login`
- `Sign In`
- `Digital Banking`
3. Accept only `http`/`https` URLs
4. If no login URL, attempt common suffixes on home URL:
- `/login`
- `/signin`
- `/online-banking`
- `/digital-banking`

Output:
- `home_url`
- `login_url`
- URL confidence notes (internal use)

## Skill 3: Merge Detection
Purpose: Detect successor FI and compute merge confidence regardless of URL extraction success.

Trigger:
- Run on every lookup after URL extraction completes so merge metadata is always available.

Process:
1. Scan FI page text for:
- `Merged into`
- `Acquired by`
- `Now part of`
- `Succeeded by`
2. Extract successor bank name if present.
3. When legacy FI data is still presented, supplement the page scan by querying the institution plus keywords like `merger`, `acquisition`, and the relevant banking regulator (state banking department and FDIC notices). Include the effective date (e.g., Woodsville Guaranty Savings Bank joined Bar Harbor Bank & Trust on October 14, 2025) when logging a confirmed merge so the routing lookup can shortcut directly to the successor on subsequent requests.
4. Assign normalized confidence scores between 0 and 1 for both `MERGED` and the complementary status (`FOUND` when URLs exist, `NOT_FOUND` otherwise).

5. When a merge is confirmed, collect the merged-into FI's official home and login URLs (via the merged institution's site or regulator filings) so the final response can reference the successor links.

Output:
- `merge_detected`
- `merged_into`
- `merge_confidence` (confidence map with `MERGED` and the relevant counter status)
- `status = MERGED` when confirmed

## Skill 4: Response Composer
Purpose: Return a deterministic JSON response that includes merge confidence.

Rules:
1. If URLs exist, return `status = FOUND` and include `merge_detected` per detection results plus the `merge_confidence` map (e.g., `{"MERGED":0.12,"FOUND":0.88}`).
2. If merge is confirmed and outweighs the URL evidence, return `status = MERGED` along with `merge_confidence` (e.g., `{"MERGED":0.92,"NOT_FOUND":0.08}`).
3. Otherwise return `status = NOT_FOUND` with null URL fields and the confidence map reflecting the balance between `MERGED` and `NOT_FOUND`.
4. Always set `"source": "US Bank Locations"`.
5. When a merge is confirmed, keep the legacy FI's URLs in `legacy_home_url`/`legacy_login_url` and point `home_url`/`login_url` at the merged-into institution so downstream consumers see both sets of links.
