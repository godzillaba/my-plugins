# CLAUDE.md

## Versioning

Every plugin has its version in two places that must stay in sync:
- `.claude-plugin/marketplace.json` → the plugin's entry in the `plugins` array
- `plugins/<name>/.claude-plugin/plugin.json` → `version`

When bumping a plugin's version, update both files.
