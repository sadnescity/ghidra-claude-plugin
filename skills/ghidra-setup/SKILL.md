---
name: ghidra-setup
description: "GhidraMCP setup and configuration: Ghidra plugin installation, connecting Claude Code (the bundled MCP server over stdio), multi-instance and multi-program targeting, base address and overlay setup, plugin options. Use when setting up or troubleshooting GhidraMCP."
---

# GhidraMCP Setup and Configuration

## Prerequisites

- **Ghidra 12.2** (requires **Java 25**)
- A JDK 21+ on the `PATH` for the plugin's MCP server (Ghidra's JDK 25 is fine)
- Nothing else: there is no Python bridge and no SDK.

## Architecture

```
Claude Code ──stdio─▶ java -jar server/ghidra-mcp-bridge.jar   (MCP 2026-07-28; a child process of Claude Code)
                         │
                         │  confirms the chosen program, then forwards the call
                         ▼
     Ghidra CodeBrowser endpoints 127.0.0.1:8080, 8081, ...  (one per CodeBrowser)
                         │
                         ▼
     the program's database (any program open in that CodeBrowser)
```

- Each CodeBrowser with **GhidraMCPPlugin** enabled serves its open programs on the first free loopback port from 8080.
- Tools are declared once, in the Java plugin; the MCP schema is generated from them.

## Ghidra Plugin Installation

1. Get the release `.zip` built for your Ghidra version (`ghidra_12.2_PUBLIC_..._GhidraMCP.zip`), or build it (`gradle` in the GhidraMCP repository with `GHIDRA_INSTALL_DIR` set)
2. Ghidra: **File > Install Extensions**, `+`, select the zip, restart Ghidra (or unzip it into `<ghidra>/Ghidra/Extensions`)
3. Open a program in **CodeBrowser**; enable **GhidraMCPPlugin** in **File > Configure > Developer** (Ghidra may offer it in a "New Plugins Found" dialog)
4. The Ghidra console logs `GhidraMCP serving this tool's programs on 127.0.0.1:8080`

**macOS arm64:** the official release ships no native binaries for this platform, so the decompiler will not start (decompile tools fail). Build them once:

```bash
cd <ghidra>/support/gradle && gradle buildNatives   # or ./gradlew buildNatives
```

### Plugin Options

In CodeBrowser: **Edit > Tool Options > GhidraMCP HTTP Server**

- **Server Port** — first port tried for this CodeBrowser's endpoint (default 8080; the next free one up to +99 is used)
- **Decompile Timeout** — seconds (default 60)

Everything listens on 127.0.0.1 only; requests from web pages (`Origin` header, foreign `Host`) are refused.

## Connecting Claude Code

This plugin's `.mcp.json` runs the MCP server as Claude Code's child process:

```json
{ "mcpServers": { "ghidra-mcp": { "command": "java",
  "args": ["-jar", "${CLAUDE_PLUGIN_ROOT}/server/ghidra-mcp-bridge.jar"] } } }
```

`ghidra-mcp-bridge.jar` is the MCP server of the GhidraMCP project (no Ghidra classes; built by `gradle bridgeJar` in the GhidraMCP repository). It speaks MCP 2026-07-28 (stateless; Claude Code picks it after a `server/discover` probe) and still answers clients that open with `initialize` (2025-11-25 and the two before). It keeps no state: it looks the named program up on every port at each call. So:

- **Start order does not matter.** With Ghidra closed the tools are listed anyway (from the catalog bundled in the jar) and calls answer `[NO_GHIDRA]`; open Ghidra and the next call finds it.
- Ghidra can be closed and reopened while Claude Code runs; the program is found on whatever port it comes back on.
- If the jar and the installed extension come from different builds, the first call that reaches Ghidra switches to the extension's tool list and Claude Code reloads it (`notifications/tools/list_changed` on its `subscriptions/listen` stream). Keep both from the same build anyway.

Options (append to `args`): `--discovery-base-port 8080`, `--discovery-range 100`, `--ghidra-timeout 120`, `--version`.

There is no HTTP MCP endpoint: MCP over HTTP is meant for remote servers, and a local one would be one more open port to protect.

## Targeting: Instances and Programs

Any number of Ghidra instances, each with any number of programs open (tabs or windows). Ports follow boot order.

**Every tool takes a required `program` argument**, exactly as `list_instances()` shows it:

- `list_instances()` — every instance and every open program in it, each with its `program="..."` value: the program's name, or `project:/path` when the same name is open in more than one project
- `decompile_function(program="SLUS_012.57", name="main")` — every other tool names its program the same way

Nothing is remembered between calls and nothing is chosen for you, not even when a single program is open: a call without `program` is refused with `[MISSING_PARAMETER]`. This is what MCP 2026-07-28 requires (no state carried from earlier requests), and it is also safer: one MCP server process serves the main conversation and its subagents at once, so a remembered choice made by one of them would silently redirect the others.

The `program` value names a database (its file id), not a port. At each call it is looked up on every port: if Ghidra restarted on another port it is found there; if it is closed the call fails with `[PROGRAM_NOT_FOUND]` and nothing is sent; if the value matches two open programs it fails with `[AMBIGUOUS]` and lists the values that tell them apart. If a Ghidra port does not answer in time the call fails with `[BUSY]` rather than risk the program being open there too. Several programs of one project, and one program open in two CodeBrowser windows of the same Ghidra, are fine; only two copies of one project open in two Ghidra processes cannot be told apart: close one. The instance itself also refuses (`[NOT_HERE]`) a call for a database it does not have open. The analyst switching tabs changes nothing.

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

**Strategy A — Separate programs**:
- Import each overlay as a separate program with its correct base address
- Open them in one CodeBrowser (as tabs) or several; each call names the one it works on with `program="<name>"`
- Each overlay gets independent analysis; references between programs are not followed

**Strategy B — Overlay memory blocks inside one program**:
- Add each module to the main program as an **overlay** memory block (**Window > Memory Map > Add Block**, tick **Overlay**, or `create_memory_block(name, address, file_path=..., overlay=true)`); Ghidra creates an address space per overlay, so modules sharing the same RAM range do not collide
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

## Plugin Internals

- `tools/` — the tools: methods annotated `@Tool`, parameters `@P`. The annotation is the whole definition; the MCP schema is generated from it.
- `core/` — `Ctx` (address parsing, function lookup, type resolution, transactions on the Swing thread named after the Claude Code session, a cached decompiler per program), `InstanceServer` (the per-CodeBrowser endpoint), `ToolRegistry`.
- `mcp/` — the MCP server: protocol (dual-era), router (discovery, resolving `program`), stdio transport. No Ghidra code; built separately as `ghidra-mcp-bridge.jar` (the plugin's `server/`), not part of the extension.

## Troubleshooting

### Plugin not loading
- Ghidra 12.2 needs Java 25: `java -version`
- Check the Ghidra console after enabling the plugin in **File > Configure > Developer**

### Claude Code shows the server as failed
- `java -version` must print 21 or newer; run `java -jar <plugin>/server/ghidra-mcp-bridge.jar --version`

### `[NO_GHIDRA]` / no instances discovered
- Start Ghidra, open a program in the CodeBrowser and enable the plugin
- `curl http://127.0.0.1:8080/instances` should list the open programs
- The port range scanned is 8080-8179

### `[MISSING_PARAMETER]` about `program`
- Every tool needs `program="..."`, as `list_instances()` shows it

### `[PROGRAM_NOT_FOUND]` / `[NOT_HERE]`
- The program was closed, or its Ghidra exited, or the value is misspelt. `list_instances()` shows what is open

### Decompiler does not start (macOS arm64)
- Build the natives: `cd <ghidra>/support/gradle && gradle buildNatives`
