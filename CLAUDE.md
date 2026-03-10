# Claude Instructions

## Repository structure

This is a monorepo for Claude Code plugins. Each plugin lives in its own directory under `plugins/`:

```
plugins/{plugin-name}/
├── .claude-plugin/plugin.json   ← plugin manifest and version
└── skills/{plugin-name}/
    └── SKILL.md
.claude-plugin/marketplace.json  ← marketplace catalog (no plugin files here)
```

## Releases

Releases are managed by [Release Please](https://github.com/googleapis/release-please) using conventional commits. Always use conventional commits with the plugin name as the scope:

```
feat(playwright-test): add screenshot comparison support
fix(playwright-test): handle missing playwright config
docs(playwright-test): update API reference
```

- `feat(plugin-name):` → minor version bump
- `fix(plugin-name):` → patch version bump
- `feat(plugin-name)!:` or `BREAKING CHANGE:` → major version bump
- `chore:`, `docs:` → no version bump

When commits are merged to `main`, Release Please opens a release PR that bumps the version in `plugins/{plugin-name}/.claude-plugin/plugin.json` and `marketplace.json`. Merging that PR creates a tag `{plugin-name}-v{version}` and triggers a post-release workflow that pins `marketplace.json` `source.ref` to that tag — ensuring users are always on a frozen snapshot.

**Never manually edit** `.release-please-manifest.json` after initial setup; Release Please owns it.

## Adding a new plugin

1. Create the plugin directory:
   ```
   plugins/new-plugin/.claude-plugin/plugin.json
   plugins/new-plugin/skills/new-plugin/SKILL.md
   plugins/new-plugin/version.txt          ← required by simple release type; content: 0.0.0
   ```
2. Add a package block to `release-please-config.json`:
   ```json
   "plugins/new-plugin": {
     "release-type": "simple",
     "component": "new-plugin",
     "tag-component": "new-plugin",
     "changelog-path": "CHANGELOG.md",
     "extra-files": [
       { "type": "json", "path": ".claude-plugin/plugin.json", "jsonpath": "$.version" },
       { "type": "json", "path": "/.claude-plugin/marketplace.json", "jsonpath": "$.plugins[?(@.name=='new-plugin')].version" }
     ]
   }
   ```
3. Add `"plugins/new-plugin": "1.0.0"` to `.release-please-manifest.json`
4. Add the plugin entry to `.claude-plugin/marketplace.json` with a `git-subdir` source:
   ```json
   {
     "name": "new-plugin",
     "source": {
       "source": "git-subdir",
       "url": "https://github.com/zizzfizzix/playwright-skill.git",
       "path": "plugins/new-plugin",
       "ref": "new-plugin-v1.0.0"
     },
     "version": "1.0.0"
   }
   ```
