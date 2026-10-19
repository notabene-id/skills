# nb-skills

Claude Code plugins for working with Notabene APIs and related standards.

Each plugin adds one skill that gives your coding agent working knowledge of a domain: integrating with Notabene
Transact (Travel Rule), the Transaction Authorization Protocol (TAP), or Chain Agnostic Improvement Proposals (CAIPs).
The agent loads a skill only when a task needs it.

## Plugins

| Plugin | What it covers |
|-------|-------------|
| [notabene-api](./plugins/notabene-api) | Integration guide for Notabene Transact (Travel Rule) across API v1 (legacy), v2 and v2.1 (current platform), plus the Notabene JavaScript SDK |
| [tap-protocol-expert](./plugins/tap-protocol-expert) | Deep expertise on the Transaction Authorization Protocol (TAP) and TAIPs |
| [caip-standards-expert](./plugins/caip-standards-expert) | Expert on Chain Agnostic Improvement Proposals (CAIPs) and the CASA namespace registry |

## Installation

In Claude Code, add this repository as a plugin marketplace, then install the plugins you need:

```text
/plugin marketplace add notabene-id/skills
/plugin install notabene-api@notabene-skills
/plugin install tap-protocol-expert@notabene-skills
/plugin install caip-standards-expert@notabene-skills
```

Other agents that read Agent Skills can use the skill folders directly, for example with
`npx skills add notabene-id/skills`. Each skill lives at `plugins/<name>/skills/<name>/`.

## Repository layout

```text
.claude-plugin/marketplace.json        # The marketplace: lists every plugin
plugins/<name>/.claude-plugin/plugin.json
plugins/<name>/skills/<name>/SKILL.md  # The skill: when to use it, and where to read next
plugins/<name>/skills/<name>/references/
```

## License

This project is licensed under the [MIT License](./LICENSE).

## Resources

- [Notabene Developer Experience Portal](https://devx.notabene.id/)
- [TAP Protocol](https://github.com/TransactionAuthorizationProtocol/TAIPs)
- [CAIP Standards](https://github.com/ChainAgnostic/CAIPs)
