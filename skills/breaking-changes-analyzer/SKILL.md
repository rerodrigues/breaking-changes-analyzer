---
name: breaking-changes-analyzer
description: Analyze an NPM package upgrade between two versions and report breaking changes, non-breaking changes, new features, engine/peer compatibility and a migration plan. Use when the user wants to upgrade an npm dependency, asks what breaks between versions, or invokes /breaking-changes-analyzer. Works in regular repos and npm-workspaces monorepos.
license: MIT
metadata:
  author: Renato Rodrigues (rerodrigues)
  version: "1.0.0"
  repository: https://github.com/rerodrigues/breaking-changes-analyzer
argument-hint: "[local-package] <npm-package> [min-version] [max-version]"
allowed-tools: Read, Glob, Grep, WebFetch, AskUserQuestion, Bash(npm view:*), Bash(gh release:*), Bash(gh api:*), Bash(node --version), Bash(node -e:*), Bash(find:*)
---

# NPM Package Upgrade and Compatibility Analyzer

Analyze the transition between two versions of one NPM package. Identify breaking changes, changes that need attention, new features, engine/peer incompatibilities, and produce a practical migration plan.

Tone: technical, precise, objective, instructional. Concise, no filler. Focus on preventing bugs in production.

The analysis is strictly read-only. Modify nothing until the user confirms in the final step.

Arguments: `$ARGUMENTS` (if the agent does not substitute it, use the text the user typed after the skill name)

## 1. Parse arguments

Classify each token by shape:

- **Version**: matches `^v?\d+(\.\d+){0,2}(-[\w.]+)?$`
- **Name**: anything else

Rules:

- More than 2 names or more than 2 versions: print the usage below and stop.
- Versions: none = derive both; 1 = `max-version`; 2 = `min-version` then `max-version`.
- Names: 1 = `{npm-package}`. In a monorepo, 2 = `{local-package}` then `{npm-package}`. In a regular repo, 2 names is a usage error.
- No `{npm-package}`: print usage and stop.

Usage:

```text
/breaking-changes-analyzer [local-package] <npm-package> [min-version] [max-version]
```

## 2. Detect repo type and local manifests

- Read the root `package.json`. A `workspaces` key (array or `{ "packages": [...] }`) means monorepo; otherwise regular repo.
- If the user attached or referenced `package.json` files in the conversation, use exactly those as the local manifests and skip discovery.
- Monorepo discovery: expand the `workspaces` globs and find the member `package.json` files. Recursive patterns like `shop/**/*` often match nothing in a glob search; in that case run `find <top-level workspace dirs> -name package.json -not -path "*/node_modules/*" -not -path "*/dist/*"` and keep only manifests that have a `name`. The root `package.json` is a candidate manifest too, so include it.
- `{local-package}` (workspace package name or directory) restricts the run to that single manifest. If it matches no workspace, say so and stop.
- The manifests selected here are the **candidate manifests**. The **in-scope manifests** are the candidate manifests that list `{npm-package}` in `dependencies`, `devDependencies`, `optionalDependencies` or `peerDependencies`. Every later step that says "in-scope manifests" means exactly this set.

## 3. Resolve versions

Local version comes from `package.json` only. Do not read `node_modules` or lockfiles.

- Search `dependencies`, `devDependencies`, `optionalDependencies`. A `peerDependencies` range is not an installed version: never use it as a local version. A manifest that lists the package only as a peer counts as having no local version (see below), but its range is still checked in section 6.
- Reduce the range to its lower bound: `^4.1.0` and `~4.1.0` become `4.1.0`; `>=2 <4` becomes `2.0.0`; `4` becomes `4.0.0`. For `||` unions use the lowest alternative (`^3 || ^4` becomes `3.0.0`); for hyphen ranges use the first version (`1.2.3 - 2.0.0` becomes `1.2.3`). Ranges with no version (`*`, `x`, empty, `latest` and other dist-tags) have no lower bound: treat them as no local version. Ignore `workspace:`, `file:`, `link:`, git URLs and `npm:` aliases.
- Default `max-version`: the `latest` dist-tag (`npm view <pkg> dist-tags --json`).
- Validate that min and max exist (`npm view <pkg> versions --json`). If the package does not exist on npm, say so and stop.
- Work only with versions that are published on npm. Never mention versions that were skipped or never published.
- If min >= max, report that there is nothing to upgrade and stop.

Missing local version:

| Case | Behavior |
| --- | --- |
| 0 or 1 versions given, regular repo | Say the package is not in `package.json` and stop |
| 0 or 1 versions given, monorepo | Skip that workspace. In the "Found in {x} of {y}" line, `x` is the number of in-scope manifests with a local version and `y` is the number of candidate manifests |
| 2 versions given (explicit min and max) | Local version is not needed. Run one analysis with the given versions |

## 4. Monorepo cost guard

When in a monorepo with no `{local-package}` and no explicit min/max:

1. Scan the candidate manifests (cheap) and compute: the in-scope manifests, and the distinct local versions (version groups).
2. Warn that analyzing every workspace can be lengthy and costly (release notes per segment, plus one `npm view` per direct dependency of the in-scope manifests for the peer check), and encourage re-running with a `{local-package}`.
3. Ask the user whether to continue or cancel (use the agent's interactive question tool if it has one, otherwise ask in chat and wait). Cancel stops the run.
4. Build the segment list (section 7) from all version groups and analyze each segment once, never once per group.

## 5. Gather data

For each segment (section 7):

- `npm view <pkg> repository.url deprecated` plus `engines` and `peerDependencies` of the segment's start and end versions.
- Official sources: GitHub Releases (`gh release list` / `gh release view`, or `gh api`), `CHANGELOG.md`, tags, commits or associated pull requests. Use WebFetch when `gh` is unavailable or the repo is not on GitHub.
- Release bodies are often only a link (blog post, migration guide, docs). Follow those links with the agent's web-fetch tool and read the linked content.
- If a source is unreachable or the changelog is missing, say so in the report. Never invent changes.
- Check every breaking-change candidate against the local context. A candidate that is only a runtime or toolchain requirement (e.g. minimum Node version) which the local context already satisfies is not a breaking change: omit it. Local Node version, in order: `engines.node` of the local or root manifest (use its lower bound), `.nvmrc` / `.node-version`, then `node --version`.

## 6. Peer/engine check

Report only what blocks or breaks. If every check passes, omit the whole Compatibility block from the report; do not list passing checks, and do not add any line saying that no blockers were found or why.

Checks, per version group (local manifests differ):

1. Target `engines.node` against the local Node version (section 5).
2. Target `peerDependencies` against the versions of those peer packages declared in the in-scope manifest.
3. Local peer ranges on the package: if an in-scope manifest lists the package in `peerDependencies` and the range does not include the max version, flag it (the range needs widening).
4. Tools that cap the package: collect the distinct direct `dependencies` and `devDependencies` (name and lower-bound version) of all in-scope manifests, skipping workspace-internal packages. Reduce each dependency range to its lower bound as in section 3 and skip dependencies that have none (`workspace:`, `file:`, `link:`, git, tags, `*`). Skip `{npm-package}` itself. Run `npm view <dep>@<version> peerDependencies --json` for each in one loop and flag every dep whose `peerDependencies` range for the package excludes the max version. Only peer ranges count; a nested dependency range is not a conflict. Do not scan manifests outside the in-scope set: they do not use the package.

For each flagged requirement, name the version that raised it when known. Workspaces in one group with identical results are listed together.

## 7. Segments

Build segments once for the whole run, then reuse them:

1. List the published stable versions between the lowest local version and max (`npm view <pkg> versions --json`; keep prereleases only if max is one).
2. Cut points: every distinct local version; the last published version of each major line below max's major (for `0.x`, of each minor line); and max.
3. A segment runs from one cut point (exclusive) to the next (inclusive). Because cuts sit at the end of each major line, a segment never spans two majors and a major hop is never separated from its own patch releases. Analyze each segment once.
4. A version group applies every segment from its local version up to max. Merge adjacent segments in the report when they have nothing worth listing.

Example: groups 5.4.3 and 6.0.14, max 7.0.4. Cut points 5.4.3, 6.0.14 (local version and last 6.x), 7.0.4. Segments: 5.4.3 -> 6.0.14 (applies to the 5.4.3 group), 6.0.14 -> 7.0.4 (applies to both).

Rendering:

- Single group and a single segment: flat format, no segment heading.
- Otherwise: one section per segment, each with its own Breaking, Non-Breaking, and New Features lists, followed by the group table (section 8).

## 8. Report

Present the result in this format. Keep descriptions brief. Omit empty sub-sections (Breaking, Non-Breaking, New Features) but keep a segment's own heading even when only one sub-section remains.

```markdown
# {package} - Upgrade from {lowest_version_from} to {version_to}

Found in {x} of {y} packages in the workspace  <!-- monorepo only; y counts the root manifest too -->

## Version groups                <!-- only when more than one group -->
| Current | Workspaces | Segments to apply |
| --- | --- | --- |
| {version} | {n}: {list} | {segment}, {segment} |

## Compatibility blockers       <!-- omit the whole block when nothing is flagged -->
- [{group or workspaces}] {what blocks}: {local state} vs {required}, {fix}

## {vA} -> {vB}                  <!-- one per segment; omit heading for single segment -->
Applies to: {groups}             <!-- only when more than one group -->

### Breaking Changes
- {critical change that breaks backwards compatibility}

### Non-Breaking Changes / Attention Required
- {change that needs code or config modification but does not break directly}

### New Features & Highlights
- {feature}: {benefit}

## Upgrade and Migration Plan

### [{group_version} -> {version_to}]    <!-- one per version group; single group: no sub-heading -->
1. {refactoring step to resolve a breaking change or adopt a new practice}
```

Single group: heading is `{package} - Upgrade from {version_from} to {version_to}`, and the group table and segment tags are omitted. Regular repo: always this case.

Migration plan rules:

- Omit trivial steps such as running `npm install`.
- Step-by-step, focused on the refactoring required for the breaking changes and on adopting advantageous new features.
- Keep steps generic ("check scripts that parse `pm2 ls` output"). Do not scan the codebase for usage during the analysis, and do not name repo files you have not been told about.
- One self-contained plan per version group, titled `[{group_version} -> {version_to}]` and ordered from the group's own version upward. Build it from the findings of the segments that group applies. Do not tag steps by segment and do not make groups read each other's plans.
- A step needed by several groups is repeated in each plan.

## 9. Close-out

Ask the user two questions (interactive question tool if available, otherwise in chat):

1. **Proceed with the upgrade?** (Yes / No)
2. **Save report to file?** (Yes / No)

If saving: write the report markdown to `docs/upgrades/{pkg-slug}-{lowest_from}-to-{to}.md` (slugify scoped names, `@scope/name` becomes `scope-name`), create the directory if missing, and print the path.

If proceeding:

1. Update the version in the affected manifests only (every version group), keeping each manifest's existing range style (pinned stays pinned, `^` stays `^`). Apply to each workspace only the plan of its own version group.
2. Install with the package manager indicated by the lockfile (`package-lock.json` = npm, `pnpm-lock.yaml` = pnpm, `yarn.lock` = yarn). If install fails, stop and tell the user. Do not try to work around it.
3. Apply the migration plan to the code. This is where the repo is read for how it uses the package (imports, config files, scripts); the analysis phase never does that.
4. Run the project's typecheck, lint and test scripts when they exist. Fix failures caused by the upgrade; report any you cannot fix.

Never commit or push.
