# learn-claude-code

This repository stores a local research copy of `@anthropic-ai/claude-code@2.1.88`, including:

1. **npm-original/**
   - Original npm tarball and key extracted package artifacts
   - `cli.js.map` used for source reconstruction

2. **source/**
   - Reconstructed source tree from `cli.js.map`
   - Focused on Claude Code's own source (third-party `node_modules` removed)

3. **doc/**
   - Analysis reports, module mapping, structure summary, and sensitive-surface scans

## Layout

- `npm-original/` — raw package artifacts
- `source/` — reconstructed source code
- `doc/` — analysis documents
- `README.md` — overview

## Notes

- Package: `@anthropic-ai/claude-code`
- Version: `2.1.88`
- Source reconstruction is based on `sources` + `sourcesContent` embedded in `cli.js.map`
- This repo is organized for inspection and learning
