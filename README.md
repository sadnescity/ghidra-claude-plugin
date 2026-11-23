# Ghidra Plugin for Claude Code

GhidraMCP integration for Claude Code. Provides MCP server configuration and reverse engineering reference skills for working with [GhidraMCP](https://github.com/sadnescity/GhidraMCP) -- a Model Context Protocol server that connects Claude to Ghidra's reverse engineering capabilities.

## What it does

- Configures the GhidraMCP MCP server so Claude Code can call Ghidra tools
- Bundles reference skills for setup, tool usage, and RE workflows
- 80 MCP tools: decompile, disassemble, call graph and xrefs, rename and type (one or many at once), comments, structs/unions/enums and C header import, equates and references, memory and overlays, code-gap search, P-code emulation
- Safe with several Ghidra instances and several programs open: every call names its program, and is sent only to the Ghidra that has exactly that database open
- Teaches Claude to record what it learns directly in the Ghidra database (meaningful names, prototypes, types, structs/enums, comments) so the decompiled code is readable afterwards

## Installation

```
/plugin marketplace add sadnescity/claude-plugins
/plugin install ghidra@sadnescity-plugins
```

## Prerequisites

- [Ghidra 12.2](https://ghidra-sre.org/) (Java 25) with the GhidraMCP extension installed (**File > Install Extensions**) and **GhidraMCPPlugin** enabled in the CodeBrowser (**File > Configure > Developer**)
- On macOS arm64 the official Ghidra release ships no native binaries, so the decompiler will not start: build them once with `cd <ghidra>/support/gradle && gradle buildNatives`
- A JDK 21 or newer on the `PATH` (`java -version`); the one Ghidra uses is fine. The plugin starts its own MCP server with it (`server/ghidra-mcp-bridge.jar`, ~320 KB). No Python, no SDK.
- Ghidra and Claude Code can be started in any order, and Ghidra can be restarted while Claude Code runs.

## Upstream

https://github.com/sadnescity/GhidraMCP
