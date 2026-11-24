# Chunkify skills

Agent skills and the Claude plugin for [Chunkify](https://chunkify.dev), the video processing platform.

## chunkify

The Chunkify skill guides your coding agent through adding video uploads, on-demand streaming, playback, and playback analytics to an application. The agent inspects your code and follows its existing framework, data models, and UI conventions. It also works when you start a new application.

You can use the skill to:

- add browser uploads and playback with Chunkify Assets;
- ingest videos from your server with the official SDKs;
- add playback analytics to an existing player;
- migrate Jobs API code to Transcodings, Thumbnails, and Assets;
- manage Project resources and query video performance through MCP.

## Install the Claude plugin

The Chunkify plugin bundles the skill and the Chunkify MCP server. In Claude Code, run:

```
/plugin install chunkify --marketplace chunkifydev/skills
```

On Claude Code versions before 2.1.275, add the marketplace first:

```
/plugin marketplace add chunkifydev/skills
/plugin install chunkify@chunkify
```

Run `/mcp`, choose the Chunkify server, sign in to Chunkify, and select the Projects Claude can access. Then ask Claude to use the Chunkify skill, or start your request with `/chunkify:chunkify`.

## Install the skill in other agents

```sh
npx skills add chunkifydev/skills
```

The command works with Claude Code, Codex, Cursor, and other agents that support Agent Skills.

Then ask your agent to use the Chunkify skill. In Claude Code, start your request with `/chunkify`. In Codex, mention `$chunkify`.

### Connect the Chunkify MCP

Connect your agent to `https://api.chunkify.dev/mcp` using OAuth. Sign in and select the Projects it can access. The skill uses this connection to inspect your Project, manage tokens and analytics connections, and query video performance.

## Documentation

- [Build with agents](https://chunkify.dev/docs/integration/chunkify-skill)
- [Chunkify documentation](https://chunkify.dev/docs)
- [OpenAPI schema](https://chunkify.dev/docs/openapi.json)
- [Privacy policy](https://chunkify.dev/docs/legal/privacy-policy)

## Contributing

The files in this repository are synced from Chunkify's main repository on each release, so changes made here are overwritten. To report a problem or suggest a change, [open an issue](https://github.com/chunkifydev/skills/issues).

## License

[MIT](LICENSE)
