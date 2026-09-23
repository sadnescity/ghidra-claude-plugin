---
name: ghidra-setup
description: "GhidraMCP setup and configuration: Ghidra plugin installation, Python bridge startup, multi-instance discovery and target selection, base address and overlay setup, MCP client configuration, transport options. Use when setting up or troubleshooting GhidraMCP."
---

# GhidraMCP Setup and Configuration

## Prerequisites

- **Ghidra 12.1.x** (tested on 12.1.4)
- **Java 21+** (required by Ghidra 12.x)
- **Python 3.10+** (for the MCP bridge; it uses the `mcp` 2.x SDK and speaks MCP 2026-07-28 plus the older handshake-based revisions)
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

1. Download the release `.zip` built for your Ghidra version (e.g. `ghidra_12.1.4_PUBLIC_..._GhidraMCP.zip`) from the GhidraMCP repository
2. Open Ghidra
3. Go to **File > Install Extensions**
4. Click the `+` button, select the downloaded `.zip`
5. Restart Ghidra
6. Open a program in **CodeBrowser**
7. Go to **File > Configure > Developer**
8. Enable **GhidraMCPPlugin**
9. The plugin starts an HTTP server (check Ghidra console for the port message)

Alternatively, unzip the extension into `<ghidra>/Ghidra/Extensions` and restart Ghidra.

**macOS arm64:** the official release ships no native binaries for this platform, so the decompiler will not start (decompile tools fail). Build them once:

```bash
cd <ghidra>/support/gradle && gradle buildNatives   # or ./gradlew buildNatives
```

### Plugin Configuration

In CodeBrowser: **Edit > Tool Options > GhidraMCP HTTP Server**

- **Server Port** — first HTTP port to try (default: 8080). If it is taken, the plugin uses the next free one (up to +99).
- **Server Address** — bind address (default: 127.0.0.1). Use 0.0.0.0 for remote access.
- **Decompile Timeout** — seconds (default: 30).

## Python Bridge

### Option A: uvx (recommended, no install needed)

```bash
uvx --from git+https://github.com/sadnescity/GhidraMCP ghidra-mcp-bridge
```

This is what the plugin's `.mcp.json` runs. uvx caches git installs, so after the bridge is updated upstream force a refresh once with `uvx --refresh --from git+https://github.com/sadnescity/GhidraMCP ghidra-mcp-bridge` (or `uv cache clean`), then restart Claude Code.

### Option B: pip install

```bash
git clone https://github.com/sadnescity/GhidraMCP.git
cd GhidraMCP
pip install -r requirements.txt
python bridge_mcp_ghidra.py
```

### Bridge CLI Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `--ghidra-server` | none | Pin the bridge to the program open at this URL (e.g. `http://127.0.0.1:8080/`). Without it, nothing is targeted until `use_program`/`use_instance` |
| `--mcp-host` | `127.0.0.1` | MCP server bind address (`sse`/`streamable-http` only) |
| `--mcp-port` | `8081` | MCP server port (`sse`/`streamable-http` only; `/mcp` endpoint) |
| `--transport` | `stdio` | Transport: `stdio`, `streamable-http`, or `sse` (deprecated) |
| `--ghidra-timeout` | `30` | HTTP request timeout in seconds |
| `--discovery-base-port` | `8080` | Start of auto-discovery port range |
| `--discovery-range` | `100` | Number of ports to scan (8080-8179) |

## Multi-Instance Discovery and Target Selection

Each CodeBrowser window with GhidraMCPPlugin enabled runs its own HTTP server, on the next free port starting at 8080 (8080, 8081, 8082, ... up to 8179). Which program lands on which port depends on boot order. An instance serves only the program in its window's **active tab**; there is no way to address another program through it.

The bridge **never selects a target on its own**, not even when only one instance is running. Until a target is chosen, every tool call fails with "No Ghidra instance has been chosen" and nothing is sent. Choose once per session:

- `list_instances()` — scans the whole port range on demand (no background polling) and lists port, program and project; the chosen one is marked `[ACTIVE]`
- `use_program(name)` — target by program name, exactly as `list_instances()` shows it (preferred: ports change between restarts)
- `use_instance(port)` — target whatever program is open on that port
- `--ghidra-server URL` at startup — pin to the program found at that URL

The bridge remembers the **program**, not the port. On every call it checks that the port still holds that program; if the program moved to another port it follows it, and if the port now holds a different program (or the program is open on several ports) the call is refused rather than sent to the wrong database. Switching the active tab in a CodeBrowser window counts as "a different program": calls are refused until you switch back or choose again. The bridge also sends the program's database file id with each call, and the plugin itself rejects (HTTP 409) a request meant for a different database.

## Base Address and Overlay Configuration

Many binaries are loaded at a specific base address in memory (not 0x0). Ghidra needs to know this address for correct analysis — especially for resolving absolute pointers, cross-references, and call targets.

### Setting the Base Address

When importing a binary in Ghidra:

1. **File > Import File** — select the binary
2. In the import dialog, click **Options...**
3. Set **Base Address** to the actual load address (e.g., `0x80010000` for a PSX executable, `0x10000000` for a MIPS firmware)
4. Ghidra will rebase all addresses accordingly

For already-imported programs: **Window > Memory Map > Set Image Base** (home icon).

### Working with Overlays

Overlays are code/data modules loaded dynamically at specific memory addresses. Common examples:
- Game overlays loaded into a fixed RAM region
- DLLs/shared libraries loaded at their preferred base
- Firmware modules mapped at specific peripheral addresses

**Strategy A — Separate programs, one CodeBrowser window each**:
- Import each overlay as a separate program with its correct base address
- Open each in its own CodeBrowser window (each gets its own port) and switch with `use_program(name)` (or `use_instance(port)`)
- Each overlay gets independent analysis; references between programs are not followed

**Strategy B — Overlay memory blocks inside one program**:
- Add each module to the main program as an **overlay** memory block (**Window > Memory Map > Add Block**, tick **Overlay**); Ghidra creates an address space per overlay, so modules sharing the same RAM range do not collide
- MCP tools address them with Ghidra's own syntax: `"OVL::80100000"` (or `"OVL:80100000"`), where `OVL` is the overlay name; a plain address always means the default space
- Listing tools cover the whole program, overlays included, and print overlay addresses with the `OVL::` prefix
- Useful when you need to cross-reference between overlays and the main executable

### Why Base Addresses Matter for MCP

GhidraMCP addresses correspond to Ghidra's internal address space. If the base address is wrong:
- `get_xrefs_to` won't find cross-references that use absolute addresses
- `decompile_function_by_address` may target the wrong function
- Struct/data annotations will be at incorrect offsets
- Addresses from a runtime debugger won't match Ghidra addresses

Always verify that Ghidra's base address matches the actual load address at runtime.

### Importing RAM Dumps

Instead of analyzing an executable file, you can import a raw memory dump captured from a running process or emulator. This captures the full runtime state including dynamically loaded code and patched memory.

1. **File > Import File** — select the raw dump file
2. Set **Format** to **Raw Binary**
3. Select the correct **Language** (processor architecture and endianness)
4. Set **Base Address** to the dump's memory origin (e.g., `0x80000000` for PSX RAM)
5. Click **Options...** and confirm the block size matches the dump

**Analysis tip**: After import, enable **Aggressive Instruction Finder (Prototype)** in the analysis options (**Analysis > Auto Analyze > Analyzers**). This helps Ghidra discover functions in raw dumps where entry points aren't obvious — especially useful for memory dumps where code boundaries aren't marked by standard executable headers. It can be slow on large dumps but significantly improves function detection.

## Java Plugin Internals

The Java plugin uses **handler auto-discovery via reflection**. Request handlers are organized into subpackages:

- `get/` — read-only queries (list functions, decompile, get bytes, etc.)
- `set/` — modifications (rename, set type, write bytes, etc.)
- `act/` — actions (create struct, add bookmark, etc.)
- `comment/` — comment operations (decompiler, disassembly comments)
- `search/` — search operations (functions, bytes); strings are listed by `get/`

New handlers are automatically registered when added to the correct subpackage.

## Troubleshooting

### Plugin not loading
- Verify Java 21+ is installed: `java -version`
- Check Ghidra console for errors after enabling the plugin
- Ensure the plugin is enabled in **File > Configure > Developer**

### Port conflicts
- If port 8080 is in use, the plugin takes the next free port (the Ghidra console logs which one)
- Change the starting port in **Edit > Tool Options > GhidraMCP HTTP Server > Server Port**
- With `--transport streamable-http`, the bridge's default `--mcp-port 8081` can collide with a second Ghidra instance: pick another port

### Bridge cannot connect
- Verify the Ghidra plugin is running: `curl http://127.0.0.1:8080/instances` should return JSON naming the program
- Check firewall rules if using remote access
- Increase timeout with `--ghidra-timeout` for slow operations (large binaries)

### No instances discovered
- Ensure at least one CodeBrowser window has the plugin enabled
- Check that the port range is correct (default 8080-8179)
- Try pinning directly: `--ghidra-server http://127.0.0.1:PORT/`

### "No Ghidra instance has been chosen"
- Expected until you call `use_program(name)` or `use_instance(port)` (see above)

### Decompiler does not start (macOS arm64)
- Build the natives: `cd <ghidra>/support/gradle && gradle buildNatives`

### Bridge behaves like an old version
- uvx is serving a cached git install: run it once with `--refresh`, or `uv cache clean`
