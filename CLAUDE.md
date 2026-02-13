# Caves Claude Plugin Development

## Publishing Changes

**CRITICAL: When updating the plugin, you MUST update the version in TWO places:**

1. **`.claude-plugin/plugin.json`** - Line 4: `"version": "X.Y.Z"`
2. **`.claude-plugin/marketplace.json`** - Line 13: `"version": "X.Y.Z"`

**Both files must have matching versions** or Claude Code will continue using the old cached version!

### Publishing Workflow

```bash
# 1. Make your changes to skills, MCP config, etc.

# 2. Update BOTH version files:
#    - .claude-plugin/plugin.json (line 4)
#    - .claude-plugin/marketplace.json (line 13)
#    Increment to next version: 0.2.0 → 0.2.1 or 0.3.0

# 3. Commit and push
git add .
git commit -m "Your changes + version bump to X.Y.Z"
git push

# 4. Users can update by:
#    - Clearing cache: rm -rf ~/.claude/plugins/cache/ic-caves
#    - Running: /plugin
#    - Restarting Claude Code
```

### Version Strategy

- **Patch (0.2.0 → 0.2.1)**: Bug fixes, documentation updates
- **Minor (0.2.0 → 0.3.0)**: New skills, new features, skill improvements
- **Major (0.2.0 → 1.0.0)**: Breaking changes, major redesigns

## Plugin Structure

```
caves-claude-plugin/
├── .claude-plugin/
│   ├── plugin.json          # Version MUST match marketplace.json
│   └── marketplace.json     # Version controls what users download
├── .mcp.json                # MCP server configuration
├── skills/
│   ├── caves-knowledge/
│   ├── content-to-caves/
│   ├── cookbook-index-to-caves/
│   ├── cookbook-query/
│   └── perspective-resolver/
└── CLAUDE.md               # This file - development reminders
```

## Testing Locally

To test changes before publishing:

1. Make changes in this directory
2. Clear cache: `rm -rf ~/.claude/plugins/cache/ic-caves`
3. The plugin will reload from GitHub on next use
4. Check skills loaded: `ls ~/.claude/plugins/cache/ic-caves/caves/*/skills/`

## Common Issues

**"My changes aren't showing up"**
→ Did you update **both** version numbers?
→ Did you clear the cache?
→ Did you push to GitHub?

**"Users are still seeing old version"**
→ Check marketplace.json version matches plugin.json
→ Tell them to clear cache: `rm -rf ~/.claude/plugins/cache/ic-caves`
