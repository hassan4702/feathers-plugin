# Changelog

All notable changes to this plugin are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
The authoritative version is the `version` field in `.claude-plugin/plugin.json`.

## [0.1.0] - 2026-05-29

### Added
- Initial release.
- `feathers-service` skill — scaffold the four-file FeathersJS v5 (Dove) service pattern (`.ts` / `.class.ts` / `.schema.ts` / `.shared.ts`) and wire registration.
- `feathers-hooks` skill — write and register around / before / after / error hooks.
- `feathers-schema` skill — TypeBox schemas and the four resolver kinds (result, data, query, external).
- `feathers-review` skill — security-focused review of Feathers services, hooks, and schemas.
- `feathers-expert` agent — multi-file architecture and debugging help.

[0.1.0]: https://github.com/hassan4702/feathers-plugin/releases/tag/v0.1.0
