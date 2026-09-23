# Ghidra Plugin for Claude Code

GhidraMCP integration for Claude Code. Provides MCP server configuration and reverse engineering reference skills for working with [GhidraMCP](https://github.com/sadnescity/GhidraMCP) -- a Model Context Protocol server that connects Claude to Ghidra's reverse engineering capabilities.

## What it does

- Configures the GhidraMCP MCP server so Claude Code can call Ghidra tools
- Bundles reference skills for setup, tool usage, and RE workflows
- 50 MCP tools: decompile, disassemble, rename symbols, set types, cross-references, structs, enums, memory access, and more
- Teaches Claude to record what it learns directly in the Ghidra database (meaningful names, prototypes, types, structs/enums, comments) so the decompiled code is readable afterwards

## Installation

```
/plugin marketplace add sadnescity/claude-plugins
/plugin install ghidra@sadnescity-plugins
```

## Prerequisites

- [Ghidra 12.1.x](https://ghidra-sre.org/) (tested on 12.1.4) with the GhidraMCP plugin installed (via **File > Install Extensions**, or unzipped into `<ghidra>/Ghidra/Extensions`)
- On macOS arm64 the official Ghidra release ships no native binaries, so the decompiler will not start: build them once with `cd <ghidra>/support/gradle && gradle buildNatives` (or `./gradlew buildNatives`)
- Java 21+
- Python 3.10+ (or uv/uvx). The bridge uses the `mcp` 2.x SDK and speaks MCP 2026-07-28 as well as older handshake-based revisions
- After a bridge update, run `uvx --refresh --from git+https://github.com/sadnescity/GhidraMCP ghidra-mcp-bridge` once (or `uv cache clean`): uvx caches git installs and keeps running the old version

## Upstream

https://github.com/sadnescity/GhidraMCP
