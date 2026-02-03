# Caves Plugin for Claude Code

Integrate Caves knowledge graph with Claude Code through the Caves Oracle MCP server.

## Features
- 🔗 Connect concepts and build knowledge graphs
- 📚 Convert cookbook indexes to navigable structures
- 🔍 Search and navigate your Caves perspectives
- 🎯 Automatic skill invocation for cookbook processing
- ☁️ Cloud-based MCP server (no local installation required)

## Prerequisites
- Caves account (sign up at https://itscaves.com)

## Installation

### Quick Install (Recommended)

```bash
# Add the ic-caves marketplace
/plugin marketplace add ic-caves/caves-claude-plugin

# Install the plugin
/plugin install caves@ic-caves-caves-claude-plugin
```

### Manual Install

```bash
# Clone the repository
git clone https://github.com/ic-caves/caves-claude-plugin.git

# Add as a local marketplace
/plugin marketplace add ./caves-claude-plugin

# Install the plugin
/plugin install caves@caves-claude-plugin
```

### Verification

Check that the plugin loaded successfully:
```bash
/plugin
# Look for "caves" in the Installed tab
```

### Authenticate with Caves
On first use, Claude Code will prompt you to authenticate:
1. A browser window will open to itscaves.com
2. Log in with your Caves account
3. Authorize Claude Code to access your Caves data
4. Return to Claude Code - authentication is complete!

## Verification

Test the installation:
```
# In Claude Code, try:
"Show me my Caves perspectives"
```

You should see your perspectives listed.

## Skills Included

### cookbook-index-to-caves
Automatically converts cookbook index pages to Caves knowledge graph connections.

**Usage:**
- Upload cookbook index images
- Mention "convert to Caves" or "organize recipe index in Caves"
- The skill will guide you through the process

## MCP Tools Available

The Caves Oracle MCP server provides these tools:

- `caves__search_caves` - Search for tags/caves by query
- `caves__my_shadows` - Get your perspectives (PIDs) with descriptions
- `caves__connect_caves` - Create connections between caves/tags
- `caves__disconnect_caves` - Remove connections between caves
- `caves__get_subcaves` - Get child caves for a parent cave
- `caves__my_shadow_caves` - Get complete graph for a perspective
- `caves__getPidsForTags` - Find PIDs that have connected to specific tags
- `caves__getPidsForConnection` - Find PIDs that agree/disagree on a connection
- `caves__createPerspective` - Create a new perspective with a username
- `caves__read_ipfs` - Read IPFS content (images/text) from caves

## Configuration

### Authentication
No manual configuration required! The plugin uses OAuth for authentication:
- First use prompts browser-based login
- Credentials securely stored by Claude Code
- Re-authentication only needed if token expires

### Caves URL Schema
Access caves directly via URLs:
```
https://itscaves.com/c/{url-encoded-cave-name}
```

Examples:
- https://itscaves.com/c/smoke%20and%20pickles
- https://itscaves.com/c/edward%20lee
- https://itscaves.com/c/kimchi

## Troubleshooting

### "caves-mcp MCP server not found"
- Restart Claude Code to load the plugin
- Check that the plugin is installed in `~/.claude/plugins/caves`
- Verify `.mcp.json` file exists in the plugin directory

### "Authentication failed" or "Login required"
- Click the authentication prompt when it appears
- Complete the OAuth flow in your browser
- If browser doesn't open, check for popup blockers
- Try logging out and back in at itscaves.com

### "Connection timeout"
- Check your internet connection
- Verify https://oracle.itscaves.com is accessible
- Try again in a few moments (server may be updating)

## Usage Examples

### View Your Perspectives
```
"Show me all my Caves perspectives"
```

### Search for Caves
```
"Search Caves for 'recipe' tags"
```

### Create Connections
```
"Connect 'kimchi' to 'fermented foods' in my Caves perspective"
```

### Convert Cookbook Index
```
[Upload cookbook index images]
"Convert this cookbook index to Caves using my 'cookbooks' perspective"
```

## Documentation

- Caves Website: https://itscaves.com
- Discord Community: https://discord.com/invite/8bW2xwv7kc
- MCP Server: https://oracle.itscaves.com/sse
- **Caves Documentation**: https://github.com/ic-caves/caves-docs - Technical documentation and guides for the Caves knowledge graph system
- **.ic File Format**: https://github.com/ic-caves/ic-docs - Documentation for the .ic file format (underlying data structure)

## License
MIT

## Contributing
Issues and PRs welcome at https://github.com/ic-caves/caves-claude-plugin
