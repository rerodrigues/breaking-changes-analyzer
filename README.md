# breaking-changes-analyzer

Agent skill that analyzes an NPM package upgrade between two versions and outputs:

- Breaking changes, non-breaking changes and new features, in one section from the current to the target version (one section per segment only when several current versions exist in a monorepo)
- Compatibility blockers (engines, peer ranges, tools that cap the package), shown only when something blocks
- Dependency updates: companion dependencies that must move with the package, at compatible versions
- One migration plan per current version
- A prompt to proceed with the upgrade and/or save the report to a file

Works in regular repos and npm-workspaces monorepos.

## Install

Distributed via the [`skills`](https://github.com/vercel-labs/skills) CLI, which installs from a git repository into the agents you pick.

```bash
npx skills add rerodrigues/breaking-changes-analyzer
```

## Usage

```text
/breaking-changes-analyzer [local-package] <npm-package> [min-version] [max-version]
```

Tokens are classified by shape: semver-looking values are versions, everything else is a package name.

### Regular repo

| Command | min | max |
| --- | --- | --- |
| `/breaking-changes-analyzer chalk` | version in `package.json` | latest on npm |
| `/breaking-changes-analyzer chalk 5.3.0` | version in `package.json` | `5.3.0` |
| `/breaking-changes-analyzer chalk 4.1.2 5.3.0` | `4.1.2` | `5.3.0` |

With 0 or 1 versions, the run exits if the package is not in `package.json`. With both versions it runs regardless. A `local-package` name is not needed here; if given, it is ignored.

### Monorepo (root `package.json` has `workspaces`)

| Command | Scope |
| --- | --- |
| `/breaking-changes-analyzer chalk [versions]` | The root manifest and all workspace packages. Warns about cost first and asks to continue. Manifests that do not use the package are skipped |
| `/breaking-changes-analyzer @scope/app chalk [versions]` | Only the `@scope/app` workspace |
| `/breaking-changes-analyzer chalk 4.1.2 5.3.0` | One analysis with the given versions, no local lookup |

If `package.json` files are attached to the conversation, those are used as the local manifests instead of discovering workspaces.

## How it works

- **Local version:** read from `package.json` only (lower bound of the range). `node_modules` and lockfiles are ignored. A `peerDependencies` range is never treated as an installed version. Odd ranges reduce as follows: `||` unions use the lowest alternative, hyphen ranges use the first version, and `*` or `latest` count as no version.
- **Cooldown:** if the project sets a minimum release age (for example npm `min-release-age` in `.npmrc`, or pnpm `minimumReleaseAge`), versions inside it are not eligible unless the package is on the exclusion list. When the max is `latest` (or omitted), it resolves to the newest eligible version and the report header says so. When you pass an explicit max that is inside the cooldown, the skill asks whether to keep it (the install may be refused) or use the newest eligible version.
- **Version groups (monorepo):** manifests are grouped by current version. The version range is cut into segments only at each local version and at the target, so a single group always gets one section. Each segment is analyzed once and shared by the groups that cross it.
- **Release data:** GitHub Releases, `CHANGELOG.md`, tags, linked migration guides and the official migration guide of the package (docs site, `MIGRATION.md`, `UPGRADING.md`), used as a helper that complements the changelog analysis when building the migration plan. Only versions published on npm are used. If a source is unreachable the report says so.
- **Breaking vs. not:** requirements the local environment already satisfies (for example a minimum Node version) are not reported as breaking.
- **Compatibility blockers:** shown only when something blocks. Four checks: target `engines` vs local Node, target peers vs local packages, local peer ranges that exclude the target, and direct dependencies whose peer range excludes the target. The last check runs one `npm view` per direct dependency, so it is slow in big monorepos.
- **Dependency updates:** dependencies flagged by the peer check, plus those the changelog or official guide says need a new version. Each goes to the newest version whose peer range includes the target and that respects the cooldown. A dependency with no compatible version is reported as a blocker instead. Nothing else is bumped.
- **Report:** a header with "Found in x of y packages in the workspace" (monorepo, `y` includes the root manifest), a version-groups table, the compatibility blockers, the dependency updates, per-segment sections (Breaking Changes, Non-Breaking Changes / Attention Required, New Features & Highlights), and one migration plan per version group, titled `[{current} -> {target}]`.
- **Analysis is read-only and does not scan your code.** Migration steps stay generic.
- **Close-out:** the skill asks whether to proceed and whether to save the report to `docs/upgrades/<pkg-slug>-<from>-to-<to>.md`.
- **On proceed:** it bumps the affected manifests to the resolved target and the companion dependencies (keeping each range style), installs once with the package manager detected from the lockfile without bypassing the cooldown or exclusion rules, reads the repo to apply the migration plan, and runs the typecheck, lint and test scripts.
- **The skill never commits or pushes.**

## Requirements

- `npm` (version lookups) and Node.js
- `gh` (optional, used for GitHub release notes; falls back to fetching the web page)
- An agent that can run shell commands, read files and fetch URLs. The frontmatter `allowed-tools` and `argument-hint` keys are used by agents that support them and ignored by the rest. Agents without an interactive question tool get the questions in chat.

## Credits

Created by [Renato Rodrigues](https://github.com/rerodrigues) (`@rerodrigues`). Version 1.0.0.

Issues and contributions: <https://github.com/rerodrigues/breaking-changes-analyzer>

## License

[MIT](LICENSE) (c) 2026 Renato Rodrigues
