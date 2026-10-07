# Contributing

The plugin runs inside Claude Code. There is no npm runtime dependency or separate translation server.

Use Claude Code 2.1.291 for the current validation baseline:

```bash
claude plugin validate . --strict
claude plugin validate plugins/prompt-translator --strict
claude plugin test plugins/prompt-translator
```

Tests mock the API and do not require an API key or Claude session authentication.

For a live check, load the source plugin and configure a translation provider:

```bash
claude --plugin-dir ./plugins/prompt-translator
```

Do not load another prompt-translation plugin in the same test session. Verify the rewritten model-turn input separately from the translation API response.

Changes should preserve commands, source origin, paths, URLs, inline code, and fenced code. Never submit untranslated source text after a translation failure. Keep credentials, session transcripts, personal directories, and provider response logs out of Git.

The marketplace entry and plugin manifest must use the same name and version. Update both when publishing a release. Contributions are licensed under MIT.
