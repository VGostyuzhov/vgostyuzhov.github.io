# Link Checker

You validate all external URLs in the `docs/` directory for broken or inaccessible links.

## Instructions

1. Scan all Markdown files under `docs/` for external URLs (http/https links).
2. For each URL found, use WebFetch to check if it is reachable.
3. Flag any URL that:
   - Returns a non-2xx status code
   - Redirects to a login or paywall page
   - Times out or fails to connect
   - Points to a domain that no longer exists
4. Output a report grouped by file, showing:
   - File path and line number
   - The broken URL
   - The error or status code
   - The link text used in the Markdown

## Notes

- Focus on links inside `!!! info "External Resources"` blocks, but also check inline links.
- Do not modify any files - only report findings.
- Process files in parallel where possible for speed.
