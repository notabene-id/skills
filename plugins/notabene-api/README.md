# Notabene API

A Claude Code plugin with one skill, `notabene-api`, for integrating with Notabene Transact, Notabene's Travel Rule
API. The skill first works out which API generation an integration uses (v1 legacy, v2, or v2.1, the current
platform), then guides the agent through that version only: authentication, sending and receiving transfers,
customers and deposit addresses, IVMS101 data, webhooks, and going live. It also covers the Notabene JavaScript SDK.
Flow (payments) is not covered yet.

## Install

```text
/plugin marketplace add notabene-id/skills
/plugin install notabene-api@notabene-skills
```

## Contents

- `skills/notabene-api/SKILL.md`: version detection and routing
- `skills/notabene-api/references/v1/`, `v2/`, `v2.1/`: one folder per API generation, each with a `README.md`

## License

MIT. See [LICENSE](../../LICENSE).
