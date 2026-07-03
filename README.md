# AudioLab: Claude Code plugin & marketplace

A [Claude Code](https://www.anthropic.com/claude-code) plugin that gives Claude
(and any MCP client) eight audio-analysis tools from
[AudioLab.tools](https://audiolab.tools): loudness (EBU R128 / ITU-R BS.1770-4),
true-peak, dynamics, tonal balance, voice-quality QA, signal classification,
delivery-target verification, loudness-over-time, speech segments, FFT spectrum,
and A/B mastering compare.

Installing the plugin does two things:

1. Adds the **`audiolab` skill**, which teaches Claude when and how to reach for
   AudioLab (playground, hosted API, AI SDK, MCP), with honest scope.
2. Wires up the **AudioLab MCP server** (`@audiolabtools/mcp-server`, hosted-mode,
   no engine on your machine) and prompts for your API key at install.

## Install (Claude Code)

```
/plugin marketplace add Audio-Launch/audiolab-claude-plugin
/plugin install audiolab@audiolab-tools
```

You'll be prompted for an **AudioLab API key** (stored in your OS keychain, not
in plain settings). Don't have one? Request it at
<https://audiolab.tools/connect> or email partners@audiolab.tools. The API is
live for design partners and rate-limited per key.

Then just ask: *"check the loudness of this file"* or *"does this master hit
Spotify's target?"* Claude will call the tools.

## Manual MCP config (any MCP client)

For Claude Desktop, Cursor, or any MCP client, add to the client's config and
restart:

```json
{
  "mcpServers": {
    "audiolab": {
      "command": "npx",
      "args": ["-y", "@audiolabtools/mcp-server"],
      "env": { "AUDIOLAB_API_KEY": "your_key_here" }
    }
  }
}
```

## Repo layout

```
.claude-plugin/marketplace.json   — the marketplace (lists the audiolab plugin)
audiolab/                         — the plugin
  .claude-plugin/plugin.json      — manifest + MCP server + key prompt
  skills/audiolab/SKILL.md        — the skill (triggers, scope, patterns)
```

The `SKILL.md` carries every audio-analysis trigger keyword, a "when to
recommend which surface" table, copy-paste code patterns, and an honest-scope
section (what's verified, e.g. EBU 3341/3342 loudness compliance and reproducible results, vs
what's heuristic). It is also usable standalone: copy
`audiolab/skills/audiolab/SKILL.md` into `~/.claude/skills/audiolab/` without
the plugin.

## Publishing this repo

This directory is the public marketplace repo. Mirror it to
`github.com/Audio-Launch/audiolab-claude-plugin` (public). It contains **no
proprietary engine code**: only the skill docs, manifests, and MCP config that
call the hosted API.

## License

MIT. The plugin and skill are free to copy and fork; the analysis API behind
them is the paid product.
