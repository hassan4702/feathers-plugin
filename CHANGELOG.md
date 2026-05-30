# Changelog

All notable changes to this plugin are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
The authoritative version is the `version` field in `.claude-plugin/plugin.json`.

## [0.2.0] - 2026-05-30

### Added
- **Multi-agent support — one repo now serves Claude Code, OpenAI Codex, and Cursor.**
- Codex: native subagent at `.codex/agents/feathers-expert.toml`, plus a `.codex-plugin/plugin.json` manifest for the Codex Plugin Marketplace.
- Cursor: native `.cursor-plugin/plugin.json` manifest (single-plugin-at-root layout).
- README "Use with Codex and other agents" section, documenting `npx skills add hassan4702/feathers-plugin -a codex` and cross-agent install.

### Changed
- Display name standardized to **"FeathersJS Toolkit"** across the Codex and Cursor manifests.
- The four skills are unchanged and shared across all three agents via the open Agent Skills standard.

## [0.1.1] - 2026-05-29

### Changed
- Add `repository` field and point `homepage` at the plugin's GitHub repo (manifest metadata for public listing).

## [0.1.0] - 2026-05-29

### Added
- Initial release.
- `feathers-service` skill — scaffold the four-file FeathersJS v5 (Dove) service pattern (`.ts` / `.class.ts` / `.schema.ts` / `.shared.ts`) and wire registration.
- `feathers-hooks` skill — write and register around / before / after / error hooks.
- `feathers-schema` skill — TypeBox schemas and the four resolver kinds (result, data, query, external).
- `feathers-review` skill — security-focused review of Feathers services, hooks, and schemas.
- `feathers-expert` agent — multi-file architecture and debugging help.

[0.2.0]: https://github.com/hassan4702/feathers-plugin/releases/tag/v0.2.0
[0.1.1]: https://github.com/hassan4702/feathers-plugin/releases/tag/v0.1.1
[0.1.0]: https://github.com/hassan4702/feathers-plugin/releases/tag/v0.1.0
