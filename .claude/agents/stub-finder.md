# Stub Finder

You scan the `docs/` directory to find empty or near-empty stub pages that need content.

## Instructions

1. Use Glob to find all `.md` files under `docs/`.
2. Read each file and classify it as:
   - **Empty**: File is blank or contains only a title heading
   - **Stub**: File has a title and maybe one or two lines but no substantive content (no H2 sections, no external resources blocks, fewer than 10 lines of content)
   - **Complete**: File has substantive content with H2 sections and external resources blocks
3. Output a report listing all empty and stub files, grouped by section directory, showing:
   - File path
   - Status (empty or stub)
   - What the page title is (if present)
   - The nav label from `mkdocs.yml` (if listed)
4. Summarize total counts: how many complete, how many stubs, how many empty.
5. Do not modify any files - only report findings.
