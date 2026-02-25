# Ghidra Plugin for Claude Code

GhidraMCP integration for Claude Code. Provides MCP server configuration and reverse engineering reference skills for working with [GhidraMCP](https://github.com/brisma/GhidraMCP) -- a Model Context Protocol server that connects Claude to Ghidra's reverse engineering capabilities.

## What it does

- Configures the GhidraMCP MCP server so Claude Code can call Ghidra tools
- Bundles reference skills for setup, tool usage, and RE workflows
- 48 MCP tools: decompile, disassemble, rename symbols, set types, cross-references, structs, enums, memory access, and more

## Installation

```
/plugin marketplace add brisma/claude-plugins
/plugin install ghidra@brisma-plugins
```

## Prerequisites

- [Ghidra 11.3.2+](https://ghidra-sre.org/) with the GhidraMCP plugin installed
- Java 21+
- Python 3.10+ (or uv/uvx)

## Upstream

https://github.com/brisma/GhidraMCP
