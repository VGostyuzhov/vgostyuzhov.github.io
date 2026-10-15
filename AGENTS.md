# AGENTS.md - Security Study Guide Site

These shared instructions apply to the entire repository. Keep this file in version control.

## Project Overview

This is a **MkDocs Material** static site deployed via **GitHub Pages**. It serves as an Infrastructure & Cloud Security study guide targeting security engineers preparing for interviews and students building foundational knowledge.

- **Generator**: MkDocs with `material` theme
- **Config**: `mkdocs.yml` (site metadata, nav tree, extensions, plugins)
- **Content**: Markdown files under `docs/`
- **Build**: `mkdocs build` (static output) / `mkdocs serve` (local dev server)

## Content Styles

Two styles coexist. Choose based on topic type.

### Concise / Cheatsheet Style

For **reference pages** - topics with well-defined terms, commands, or configurations.

- Short introduction per section; use this style when the reader needs reference facts rather than a new mechanism
- Bold key terms with brief inline explanation
- Bulleted lists and short code snippets
- Add relevant sources according to External Resources Rules below
- **Reference example**: `docs/linux/file-system.md`

### Detailed / Deep-Dive Style

For **concept pages** - topics requiring attack/defense analysis, protocol flows, or comparative breakdowns.

- Tables comparing options, headers, algorithms, or threat/mitigation pairs
- Longer prose paragraphs when explaining attack chains or protocol mechanics
- Code blocks for config snippets, HTTP headers, or example payloads
- Checklists and numbered steps for workflows
- **Reference example**: `docs/web-security/index.md`

### When to Use Which

| Signal | Style |
|--------|-------|
| Topic is a list of tools, commands, file types, or config options | Concise |
| Topic requires explaining how/why something works or fails | Detailed |
| Content fits naturally in bullet + bold-term format | Concise |
| Content needs tables, attack scenarios, or multi-step flows | Detailed |

Most pages will blend both - concise bullets for definitions, explanations for mechanisms, and tables for comparisons. Choose detailed style for unfamiliar mechanisms, lifecycle decisions, trust boundaries, and attack reasoning even when the terminology could fit in a short table. The reference examples show the range.

## Page Structure Template

```markdown
# Page Title

## Section Name

Short intro paragraph explaining the concept.

**Key Term 1:**

- **Sub-item**: Explanation
- **Sub-item**: Explanation

**Key Term 2:**

- Explanation with `inline code` where appropriate

!!! info "External Resources"
    - [Link Title](https://example.com) (Source Name)
    - [Link Title](https://example.com) (Source Name)
    - [Link Title](https://example.com) (Source Name)

## Next Section

...
```

For concept sections, a useful sequence is: problem and prerequisites, participants and mechanism, concrete example, failure cases and limits, then an optional summary table. This sequence is guidance, not a mandatory set of headings.

Not every page needs every element. Adapt the skeleton to fit the content. The example links illustrate formatting, not a required link count; follow External Resources Rules below.

## Formatting and Extensions

The following MkDocs Material extensions are enabled in `mkdocs.yml`. Use them.

### Admonitions

```markdown
!!! info "External Resources"
    - [Link](https://example.com) (Source)

!!! warning "Common Pitfall"
    Description of the pitfall.

!!! tip "Best Practice"
    Description of the practice.
```

Use `info` for external resource blocks. Use `warning`, `tip`, `note`, `danger` sparingly where they genuinely add value.

### Collapsible Blocks

```markdown
??? note "Click to expand"
    Hidden content here.
```

### Code Highlighting

````markdown
```bash
chmod 755 /etc/config
```

```python
import hashlib
hashlib.sha256(b"data").hexdigest()
```
````

Specify the language for syntax highlighting. Use `bash`, `python`, `yaml`, `json`, `http`, `sql`, etc.

### Inline Code Highlighting

```markdown
The `#!python range()` function generates a sequence.
```

### Content Tabs

```markdown
=== "Linux"

    Linux-specific content here.

=== "macOS"

    macOS-specific content here.
```

### Tables

Standard Markdown tables. Use them for comparisons, attack/defense mappings, and reference data.

## Tone and Style

- Professional, clear, and technically precise
- Write explanatory study material for learners, with concise reference summaries where useful
- No emojis
- No metaphors or analogies
- No filler phrases ("it's important to note that", "in today's landscape")
- Use precise security terminology
- Keep explanations direct - state what something is, what it does, why it matters

## Explanation Quality

Reader understanding takes priority over brevity. Concise style is suitable for reference facts, not a reason to compress an unfamiliar mechanism into a list of control names. The page template is a layout example, not a limit on explanatory depth.

- Define a technical term or acronym on first meaningful use. Explain what it does in this scenario; expanding the acronym alone is insufficient.
- State prerequisites and link to a specific teaching section when needed. External resources supplement the lesson; readers should not need to leave the site to understand its central explanation.
- For a nontrivial protocol or control, explain the actors, information exchanged, state retained, checks performed, and result of failure. Explain why the control addresses the threat and what it leaves unresolved.
- Introduce mechanisms in prose or a worked flow before using tables and checklists to summarize or compare them. Avoid sentences that introduce several unexplained alternatives at once.
- Use concrete examples to explain timing, caching, trust boundaries, and attack/defense behavior. State assumptions and trace cause and effect; distinguish illustrative numbers from standards and defaults.
- Avoid using phrases such as "apply appropriate policy" or "use sufficiently fresh evidence" as substitutes for explaining the actual decision, checks, and timing bound.
- Interview guidance must be supported by teaching material. Add specific study links, and fill prerequisite gaps rather than teaching the entire concept only inside a hidden answer.
- Review concept pages for a learner who knows the stated prerequisites but has not encountered the topic. The reader should be able to explain how and why the mechanism works, not only repeat its terminology.


## Technical Accuracy

- State relevant assumptions, limitations, and platform or version dependencies for security recommendations.
- Distinguish primary mitigations from defense-in-depth controls.
- Verify time-sensitive claims against primary sources and identify dated standards explicitly.
- Do not claim a source was checked unless it was inspected. Report verification limitations.

## External Resources Rules

- For new or substantially revised topic sections, add an `!!! info "External Resources"` block with directly relevant sources.
- Usually 1–3 strong sources are sufficient. Do not add generic homepages or unrelated links to meet a quota.
- Prefer primary documentation, standards, and authoritative security guidance, such as official docs, OWASP, CIS, and NIST. Use secondary sources where they add useful explanation or practical examples.
- Link to the specific page supporting the topic, with a descriptive link title and the source name in parentheses.
- Verify newly added or changed external links when access is available. Avoid dead or paywalled resources; report links that could not be verified.
- Categorize resources when useful, such as Official Docs and Labs.
- Navigation, summary, and exercise sections do not require separate resource blocks where they add no value.
- Do not reformat untouched sections solely to enforce this convention.

## Access Methods

- Use only direct, standard access methods for files, devices, and services.
- If direct access is unavailable, explain the limitation and agree on an approach with the user before proceeding with an alternative.
- Do not use temporary Docker containers, host filesystem mounts, or the Docker API as substitutes for direct NAS access unless the user explicitly requests that method.
- Authorization to complete a task or use Portainer does not authorize access workarounds.

## Workflow Rules

1. **Scope and existing work**: Inspect `git status` before editing. Preserve unrelated changes and untracked files, and keep edits within the requested scope.
2. **Current source**: Read the current page before editing. Expand incomplete content when requested; do not assume pages are empty stubs. Treat `PLAN.md` as historical context and verify its claims against the current repository.
3. **Navigation and links**: When adding, renaming, or removing a page, update `nav:` in `mkdocs.yml` and affected internal links, including section landing pages and incoming links or anchors.
4. **Environment**: Use an isolated Python environment and the repository's declared dependencies. The current CI installation commands are in `.github/workflows/deploy.yml`; no dependency manifest is currently provided. If prerequisites are unavailable, report the limitation.
5. **Build check**: After content or configuration changes, run `mkdocs build --clean`, matching CI, and inspect warnings. Check changed internal links and anchors separately. A successful build does not verify external URLs. Report checks that could not be performed. Instruction-only edits require a diff review; a site build is not necessary.
6. **Local preview**: Use `mkdocs serve` when a local preview is needed.
7. **Generated output**: Do not commit generated `site/` output.
8. **Topic scope**: Keep one topic per page; do not merge unrelated topics into a single file.
9. **Completion report**: Report changed files, checks performed, and unresolved issues.

## File Conventions

- **Filenames**: kebab-case (`file-system.md`, `access-control-systems.md`)
- **Section landing pages**: `index.md` inside each section directory
- **Directory structure**: `docs/<section>/<topic>.md`
- **No nested subdirectories** beyond one level under `docs/`
