---
description: Systematically rebrand a project by finding and replacing brand identifiers, names, domains, and related assets throughout the codebase.
scripts:
  sh: scripts/bash/check-prerequisites.sh --json
  ps: scripts/powershell/check-prerequisites.ps1 -Json
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Guide a complete, safe rebrand of the project by:

1. Discovering every occurrence of the current brand identity across the codebase (names, domains, logos, color tokens, taglines, package identifiers, config values, documentation, etc.)
2. Producing a prioritised replacement plan with clear file-level scope
3. Executing replacements incrementally, verifying correctness at each step
4. Reporting what changed and what (if anything) was intentionally left unchanged

## Operating Constraints

- **Surgical replacements only** — do not reformat, refactor, or restructure code beyond the brand change.
- **Preserve behaviour** — symbol renames in source code (class names, function names, variable names, module paths) must be handled carefully to keep the program running correctly. Prefer keeping code identifiers stable unless the user explicitly requests otherwise.
- **Source control safety** — run replacements on a feature branch (or confirm the user is already on one). Never force-push or amend history.
- **User approval gate** — present the complete replacement plan and wait for explicit approval before making any file edits.
- **Incremental verification** — after each file group is updated, check for obvious regressions (broken imports, invalid JSON/YAML, missing references).

## Execution Steps

### 1. Initialise Context

Run `{SCRIPT}` from repo root and parse the JSON output for `FEATURE_DIR` and `AVAILABLE_DOCS`.
All file paths derived from this output must be absolute.
For single quotes in args like "I'm Groot", use escape syntax: e.g. `'I'\''m Groot'` (or double-quote if possible: `"I'm Groot"`).

### 2. Parse Rebrand Request

Extract from `$ARGUMENTS` (or ask if missing):

| Field | Description | Example |
|-------|-------------|---------|
| **From** | Current brand name / domain / identifier | `openclaw.ai` |
| **To** | Replacement brand name / domain / identifier | `mycompany.com` |
| **Scope** | Directories or file globs to search (default: entire repo) | `src/`, `docs/` |
| **Exclude** | Paths to skip (e.g., build artifacts, third-party vendored code) | `node_modules/`, `dist/` |
| **Code identifiers** | Whether to rename code symbols too (default: no) | yes / no |

If `$ARGUMENTS` supplies only partial information (e.g. "rebrand openclaw.ai to mycompany.com"), infer sensible defaults and confirm with the user before proceeding.

### 3. Discover Brand Surface Area

Search the repository for every occurrence of the current brand using case-insensitive matching. Group findings by **category**:

#### A. Textual occurrences

- Exact match (case-sensitive): e.g. `openclaw.ai`, `OpenClaw`, `openclaw`
- Common casing variants: Title Case, UPPER CASE, lower case, camelCase, PascalCase, kebab-case, snake_case
- Domain variants: bare domain, `https://` prefixed, `www.` prefixed, subdomains

For each match record: file path, line number, surrounding context (±2 lines), and category tag.

#### B. Asset / binary references

- Image files containing the brand name in the filename (e.g. `openclaw-logo.png`, `openclaw-icon.svg`)
- CSS/SCSS color tokens or variables derived from the brand (e.g. `--openclaw-primary`)
- Font files or embed URLs referencing the brand

#### C. Configuration & metadata

- `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `pom.xml`, etc. — name, homepage, repository fields
- Environment files (`.env`, `.env.example`) — variable names or values
- CI/CD config (`.github/`, `Dockerfile`, etc.) — registry names, image tags, secret names
- Manifest files (`manifest.json`, `Info.plist`, `AndroidManifest.xml`) — app name, bundle id
- `README.md`, `CHANGELOG.md`, `LICENSE` — project name and URLs

#### D. Code identifiers *(only if user opted in)*

- Class names, function names, module/package names, constants that contain the brand string
- Import paths and module references

### 4. Build Replacement Plan

Produce a structured plan table sorted by risk (lowest → highest):

| # | File(s) | Category | Occurrences | Current value | Replacement | Risk | Notes |
|---|---------|----------|-------------|---------------|-------------|------|-------|

**Risk levels:**

- **Low** — Documentation, comments, string literals in config/metadata
- **Medium** — Asset file renames, URL strings in source code, env var names
- **High** — Package/module names, import paths, code symbol renames

Flag any occurrence that **should not be replaced** (e.g., third-party attribution, legal notices, vendored code) with a "SKIP — reason" note.

Summarise total counts: files affected, lines affected, binary assets to rename.

### 5. Request Approval

Present the full replacement plan and ask:

> "The rebrand plan above covers N files and M total occurrences. Binary assets (if any) will be renamed. Code identifiers will [be renamed / remain unchanged].
>
> Shall I proceed? Reply **yes** to apply all changes, **no** to cancel, or tell me which items to skip."

Do **not** edit any file until the user approves.

### 6. Execute Replacements (Incrementally)

Process categories in order: Documentation → Configuration → Assets → Source code.

For each group:

1. Apply all textual replacements in that group.
2. For binary asset renames, rename the files and update every textual reference to the old filename.
3. Validate:
   - JSON/YAML/TOML files parse without error after edits.
   - No import or `require` statements reference the old name.
   - No broken internal links in Markdown files.
4. Report a brief summary: "Group [name] — N files updated, N occurrences replaced."

If any validation step fails, **pause**, report the error, and ask the user how to proceed before continuing.

### 7. Post-Replacement Checklist

After all replacements, verify:

- [ ] Old brand string no longer appears in tracked source files (run a final search and report residual hits)
- [ ] Package manifests reflect the new name and any related fields (homepage, repository URL)
- [ ] Internal cross-references (links in docs, import aliases) resolve correctly
- [ ] Any lock files (`package-lock.json`, `yarn.lock`, `Cargo.lock`, etc.) are noted as needing regeneration — instruct the user to run the appropriate install command
- [ ] CI/CD pipelines reference the new image/registry name where applicable
- [ ] Binary assets have been renamed and all references updated

### 8. Report

Output a final summary:

- Total files modified
- Total occurrences replaced
- Files / occurrences intentionally skipped (with reasons)
- Residual hits (old brand still present) with file:line references
- Recommended follow-up actions (e.g., regenerate lock files, update DNS records, update third-party integrations)

## Important Notes

- If the project uses a **monorepo**, apply the scope per sub-package to avoid accidental cross-package renames.
- For **binary image assets** (PNG, JPEG, ICO, etc.) that contain the old logo, flag them for manual replacement — automated text replacement cannot update pixel content.
- If the rebrand involves a **domain change**, remind the user to update DNS records, SSL certificates, OAuth redirect URIs, and any hard-coded CORS allow-lists after the code changes are complete.
- Respect `.gitignore` — do not modify files that are intentionally excluded from version control.
