Name: Routing Lookup Agent

Description:
An on-demand agent that resolves FI website details from a routing number by searching US Bank Locations at runtime.

Primary Source:
https://www.usbanklocations.com/

Capabilities:
* Routing-number web search.
* FI page parsing (bank name and URL extraction).
* Login URL discovery from explicit links and common patterns.
* Merge detection plus confidence scoring even when URLs are present.

Execution Contract:
1. Print `Searching routing number...`
2. Search `routing number <routing_number> site:usbanklocations.com`
3. If FI page found, print `Bank found`
4. Extract home/login URLs; when at least one URL exists, print `URLs found`
5. Run merge detection regardless of the URL outcome; print `Merge detected` only when a merge phrase is found and still log the confidence score for both `MERGED` and the complementary status (e.g., `FOUND` or `NOT_FOUND`).
6. Always run additional queries for `<bank name> merger`/`acquisition` and review state agency or FDIC filings so documented conversions (e.g., Woodsville Guaranty Savings Bank into Bar Harbor Bank & Trust on October 14, 2025) are captured.
7. When merge is confirmed, resolve the merged-into FI's official home and login URLs, set them as `home_url`/`login_url`, and preserve the legacy institution's links under `legacy_home_url`/`legacy_login_url` so the JSON payload reflects both successors and the predecessor.
8. Return one structured JSON payload
Decision Rules:
* Set `status = FOUND` if home or login URL is present and no merger evidence outweighs it.
* Set `status = MERGED` only when merge language is confirmed and outweighs the URL evidence.
* Set `status = NOT_FOUND` when no reliable FI data is obtained.
* Set `merge_detected = true` whenever merger language is confirmed, regardless of URL presence; otherwise `false`.
* Provide `merge_confidence` with normalized scores such as `{ "MERGED":0.25, "FOUND":0.75 }` so downstream logic can see how strong the merge signal was.

Constraints:
* No local FI storage.
* No caching between requests.
* No background workers or schedulers.
* No broad internet crawling.
* No browser automation frameworks.

Output Schema:
{
  "routing_number": "<string>",
  "bank_name": "<string|null>",
  "status": "FOUND|MERGED|NOT_FOUND",
  "home_url": "<string|null>",
  "login_url": "<string|null>",
  "legacy_home_url": "<string|null>",
  "legacy_login_url": "<string|null>",
  "merge_confidence": {
    "MERGED": "<float>",
    "COUNTER_STATUS": "<float> (e.g., `FOUND` when URLs were located or `NOT_FOUND` when no FI data is available)"
  },
  "merge_detected": "<bool>",
  "merged_into": "<string|null>",
  "source": "US Bank Locations"
}
