# Routing Number Web Search Agent

This project defines a runtime-only routing lookup agent that searches `https://www.usbanklocations.com/` and returns FI website details without storing local FI data.

## Goal
Given a routing number, return one structured JSON response containing:
- `bank_name`
- `home_url`
- `login_url`
- `merge_confidence` (confidence scores for `MERGED` vs `FOUND/NOT_FOUND`)
- merge status fields (`merge_detected`, `merged_into`) when required
- when a merge is confirmed, also include `legacy_home_url`/`legacy_login_url` to preserve the legacy FI links while pointing `home_url`/`login_url` at the successor institution.

## Required Runtime Flow
1. Accept routing number input from user.
2. Search using: `routing number <routing_number> site:usbanklocations.com`.
3. Open FI result page and extract:
- bank name
- city/state (optional for internal validation)
- home URL
- login URL
4. Run merge detection on the FI page text regardless of whether URLs were extracted, so the lookup records both the merge verdict and a paired confidence score before returning output. When a merge is confirmed, resolve the merged-into FI's official home and login URLs, include them as `home_url`/`login_url`, and retain the legacy FI's links under `legacy_home_url`/`legacy_login_url`.
5. Return final JSON.

## Status Rules
- `FOUND`: at least one trusted URL identified (prefer both URLs).
- `MERGED`: both URLs missing and merge/acquisition phrase found.
- `NOT_FOUND`: no FI result or insufficient data.

## URL Extraction Rules
- Parse anchor tags first.
- Prioritize anchor text containing:
- `Online Banking`
- `Login`
- `Sign In`
- `Digital Banking`
- Accept only `http` or `https` links.
- If login link is absent, try common patterns against home URL:
- `/login`
- `/signin`
- `/online-banking`
- `/digital-banking`

## Merge Detection Rules
- Run on every lookup so the final JSON can report both the highest-confidence `MERGED` finding and its complement (e.g., `FOUND` or `NOT_FOUND`).
Detect merge indicators in page text:
- `Merged into`
- `Acquired by`
- `Now part of`
- `Succeeded by`

## Output Example
```json
{
  "routing_number": "021000021",
  "bank_name": "JPMorgan Chase Bank",
  "status": "FOUND",
  "home_url": "https://www.chase.com",
  "login_url": "https://secure.chase.com",
  "merge_confidence": {
    "MERGED": 0.01,
    "FOUND": 0.99
  },
  "merge_detected": false,
  "merged_into": null,
  "source": "US Bank Locations"
}
```

When a merge is confirmed, the JSON might look like:
```json
{
  "routing_number": "111314575",
  "bank_name": "Vista Bank",
  "status": "MERGED",
  "legacy_home_url": "https://www.vistabank.com/",
  "legacy_login_url": "https://online.myvista.bank/",
  "home_url": "https://www.nbhbank.com/",
  "login_url": "https://www.nbhbank.com/personal-banking/",
  "merge_confidence": {
    "MERGED": 0.92,
    "FOUND": 0.08
  },
  "merge_detected": true,
  "merged_into": "NBH Bank (National Bank Holdings Corporation)",
  "source": "US Bank Locations"
}
```

## Confidence reporting
- Always include a `merge_confidence` map in the JSON response even when URLs exist. Populate `MERGED` with the probability that a merger/acquisition occurred and use the complementary key (`FOUND` when URLs are present or `NOT_FOUND` when no FI data was located) to represent the counterfactual. Aim for values between 0 and 1 that sum to 1 so consumers can see exactly how confident the agent is about both outcomes.

## Constraints
- No local storage, cache, or FI registry.
- No broad crawling across unrelated sites.
- No Playwright or Selenium.
- No API or UI service layer.
- Keep execution lightweight and on-demand.

## Merger vigilance & verification plan
1. Run the merge detection flow on every lookup (scan the FI page plus targeted web queries for phrases such as `Merged into`, `Acquired by`, or `Now part of` paired with the institution) so the JSON output always reports both the confidence-weighted `MERGED` reading and its complement. Log the discovery and, whenever possible, capture the effective date of the merger (the Bar Harbor Bank & Trust conversion of Woodsville Guaranty Savings Bank completed on October 14, 2025, for example) before returning a `MERGED` status.
2. Supplement the runtime scan with at least one follow-up query per session that targets the institution's name plus `merger`, `acquisition`, or the applicable banking regulator (state banking department, FDIC, etc.) to ensure newly announced consolidations are detected even when the FI details page still shows the legacy routing number. Include confidence percentages for both `MERGED` and `FOUND/NOT_FOUND` angles in the log entry.
3. When a merger is confirmed, document the outcome in the project notes (the README itself or a designated changelog) so future requests for the same routing number can shortcut to the merged bank without re-running the full search, and refresh that document whenever new official announcements or regulator filings appear.
