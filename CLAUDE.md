# nb-skills — Claude Code plugin marketplace

This repository is a Claude Code plugin marketplace. Each plugin ships one Agent Skill.

## Repository structure

```
.claude-plugin/marketplace.json          # Lists every plugin: name, source, description, category, tags
plugins/<name>/
  .claude-plugin/plugin.json             # name, version, description, author, license (MIT), homepage, repository, keywords
  skills/<name>/
    SKILL.md                             # Frontmatter (name, description) + skill prompt
    references/                          # Supporting docs, API refs, guides
```

Ship source folders only. Do not commit packaged `.skill` or other zip archives: Anthropic's plugin directory holds
binaries for manual review, and nothing reads them.

## Rules

### Validate before finishing

After any change to a manifest or a skill, run:

```sh
claude plugin validate . --strict
claude plugin validate plugins/<name> --strict
```

To try a change end to end: `claude plugin marketplace add ./`, then `claude plugin install <name>@notabene-skills`
(remove both afterwards).

### Versions

`version` lives in each `plugin.json` only (not in the marketplace entry). Users stay on a version until it is bumped,
so bump it in the same PR as any change users should get: patch for fixes, minor for new content.

### Description length limit

The `description` field in each `SKILL.md` frontmatter **must be under 1024 characters**. After editing a description, verify:

```sh
awk '/^description: >/{found=1; next} found && /^---$/{exit} found{gsub(/^  /,""); printf "%s ", $0}' plugins/<name>/skills/<name>/SKILL.md | wc -c
```

### Skill correctness checks

When reviewing or editing a skill, verify:

1. **Frontmatter is valid** — only `name` and `description` between `---` delimiters (claude.ai and the Skills API
   reject most other keys); `name` is lowercase letters, digits and hyphens, without "claude" or "anthropic"
2. **Description < 1024 chars** — see above; say what the skill does and when to use it, in the third person
3. **References are accurate** — API endpoints, request/response bodies, and field names match the actual
   Notabene/TAP/CAIP documentation. When in doubt, fetch the latest docs from the web.
4. **No stale information** — if external APIs or specs have changed, update the skill to match
5. **Structure** — `SKILL.md` under 500 lines; references one level deep from `SKILL.md`; a `## Contents` list at the
   top of any reference file over 100 lines; every relative link and anchor resolves
6. **Directory limits** — every file under 256 KiB and at most 512 files per plugin

### Commit hygiene

- Keep commits focused: one plugin per commit when possible
- Bump the plugin's `version` when the change matters to its users
