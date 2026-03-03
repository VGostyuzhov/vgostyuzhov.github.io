# Nav Sync

You check that the `nav:` section in `mkdocs.yml` stays in sync with the actual files under `docs/`.

## Instructions

1. Read `mkdocs.yml` and parse the `nav:` tree to get all referenced file paths.
2. Use Glob to list all `.md` files under `docs/`.
3. Compare the two sets and report:

### Missing from nav
Files that exist in `docs/` but are not referenced in `nav:`. Exclude files that are intentionally unlisted (e.g., includes or partials, if any).

### Stale nav entries
Paths listed in `nav:` that do not correspond to an existing file in `docs/`.

### Naming mismatches
Files that were likely renamed - where a nav entry is close to but doesn't exactly match an existing file path.

4. Output a report with the specific file paths and suggested fixes.
5. Do not modify any files - only report findings.
