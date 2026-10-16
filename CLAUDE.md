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

### notabene-api specifics

- `SKILL.md` is a router: the version-detection table plus rules that hold for every version. Version-specific detail
  goes in `references/<v1|v2|v2.1>/`, each routed by its own `README.md`. Never mix calls or field names across versions.
- Flow (payments) is out of scope for now: no Flow endpoints, `flow.*` events, or Flow fields.
- `references/v2.1/` is a copy of notabene-id/node's `plugins/notabene/skills/notabene-api/guides/`. Re-sync it when
  the node's guides change: strip dashboard-only markup (`<!-- slot:… -->`, `<!-- environments -->`,
  `<!-- sandbox -->`/`<!-- production -->`, `transfer-chain` fences, `tab=` labels), leave out `coding-agents.md`,
  and update `references/v2.1/README.md`. Keep these deliberate corrections, which upstream does not have yet:
  - `wallets.md`: a `rejected` address confirmation refuses later transfers only when just a counterparty claimed the
    address; if you registered, confirmed or used it, the rejection answers that transfer only.
  - `webhooks.md`: the revert statuses list all six (`RevertRequested` … `RevertedIncompleteAuthorization`);
    `requires_action` is only on `GET /api/v1/transfers/{id}`; errors include `400` as well as `422`.
  - `transact-send.md`: requirement `type` is PascalCase while the resolve path takes kebab-case.
  - `customers.md`: the customer id is always required on a transfer, an inline profile only alongside it (the
    node's OpenAPI says otherwise; its code agrees with the guide).
- `references/v1/` is written from the node's typed v1 client (`go/pkg/clients/nb/legacy/node/`); `references/v2/`
  can be checked against the node's V2 route declarations (`go/services/node/internal/api/oldnode/`).
- Facts that could not be confirmed are marked as such in the guides (`references/v1/caveats.md`, the v2 test host).
  Replace the marker once a fact is confirmed; never state a guess as fact.

### Keep this file current

Update this `CLAUDE.md` in the same PR whenever a change alters how the repo works: its layout, the manifests, the
validation steps, a plugin's conventions or sources, or anything a future session would otherwise have to rediscover.

### Commit hygiene

- Keep commits focused: one plugin per commit when possible
- Bump the plugin's `version` when the change matters to its users
