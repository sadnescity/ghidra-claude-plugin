---
description: "GhidraMCP setup and configuration: Ghidra plugin installation, Python bridge startup, multi-instance auto-discovery, MCP client configuration, transport options. Use when setting up or troubleshooting GhidraMCP."
---

# GhidraMCP Setup and Configuration

## Prerequisites

- **Ghidra 11.3.2+** (National Security Agency reverse engineering framework)
- **Java 21+** (required by Ghidra 11.x)
- **Python 3.10+** (for the MCP bridge)
- **uv** (recommended) or pip for Python dependency management

## Architecture

```
MCP Client (Claude Code)
    │
    │  stdio / SSE / streamable-http
    ▼
Python Bridge (ghidra-mcp-bridge)
    │
    │  HTTP REST
    ▼
Ghidra Plugin (GhidraMCPPlugin)
    │
    │  Ghidra Flat API
    ▼
Ghidra Program Database
```

The system has two components:

1. **Java Plugin** — runs inside Ghidra, exposes an HTTP API on a configurable port (default 8080)
2. **Python Bridge** — translates MCP protocol to HTTP calls against the Java plugin

## Ghidra Plugin Installation

1. Download the latest release `.zip` from the GhidraMCP repository
2. Open Ghidra
3. Go to **File > Install Extensions**
4. Click the `+` button, select the downloaded `.zip`
5. Restart Ghidra
6. Open a program in **CodeBrowser**
7. Go to **File > Configure > Developer**
8. Enable **GhidraMCPPlugin**
9. The plugin starts an HTTP server (check Ghidra console for the port message)

### Plugin Configuration

In CodeBrowser: **Edit > Tool Options > GhidraMCP**

- **Server Port** — HTTP port (default: 8080). Change if another service uses 8080.
- **Server Address** — bind address (default: localhost). Use 0.0.0.0 for remote access.

## Python Bridge

### Option A: uvx (recommended, no install needed)

```bash
uvx --from git+https://github.com/brisma/GhidraMCP ghidra-mcp-bridge
```

### Option B: pip install

```bash
git clone https://github.com/brisma/GhidraMCP.git
cd GhidraMCP
pip install -r requirements.txt
python -m ghidra_mcp_bridge
```

### Bridge CLI Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `--ghidra-server` | `http://localhost:8080` | Ghidra plugin HTTP URL |
| `--mcp-host` | `localhost` | MCP server bind address |
| `--mcp-port` | `0` (auto) | MCP server port (stdio ignores this) |
| `--transport` | `stdio` | Transport: `stdio`, `sse`, or `streamable-http` |
| `--ghidra-timeout` | `30` | HTTP request timeout in seconds |
| `--discovery-base-port` | `8080` | Start of auto-discovery port range |
| `--discovery-range` | `100` | Number of ports to scan (8080-8179) |

## Multi-Instance Auto-Discovery

When multiple Ghidra CodeBrowser windows are open, each gets its own HTTP port:

- First window: port 8080
- Second window: port 8081
- Third window: port 8082
- ...up to port 8179

The bridge auto-discovers all active instances by scanning the port range every 30 seconds. If only one instance is running, it is automatically selected as the active target. Use `list_instances()` and `use_instance(port)` to manage multiple instances.

## Java Plugin Internals

The Java plugin uses **handler auto-discovery via reflection**. Request handlers are organized into subpackages:

- `get/` — read-only queries (list functions, decompile, get bytes, etc.)
- `set/` — modifications (rename, set type, write bytes, etc.)
- `act/` — actions (create struct, add bookmark, etc.)
- `comment/` — comment operations (decompiler, disassembly comments)
- `search/` — search operations (functions, bytes, strings)

New handlers are automatically registered when added to the correct subpackage.

## Troubleshooting

### Plugin not loading
- Verify Java 21+ is installed: `java -version`
- Check Ghidra console for errors after enabling the plugin
- Ensure the plugin is enabled in **File > Configure > Developer**

### Port conflicts
- If port 8080 is in use, the plugin will fail to start
- Change the port in **Edit > Tool Options > GhidraMCP > Server Port**
- Or stop the conflicting service

### Bridge cannot connect
- Verify the Ghidra plugin is running: `curl http://localhost:8080/methods` should return JSON
- Check firewall rules if using remote access
- Increase timeout with `--ghidra-timeout` for slow operations (large binaries)

### No instances discovered
- Ensure at least one CodeBrowser window has the plugin enabled
- Check that the port range is correct (default 8080-8179)
- Try connecting directly: `--ghidra-server http://localhost:PORT`
