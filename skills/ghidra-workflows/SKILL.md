---
description: "GhidraMCP reverse engineering workflows: function analysis, cross-reference tracing, symbol annotation, multi-instance management, struct/enum creation. Core rule: record every finding in the Ghidra database (rename functions/variables/data, set prototypes and types, add comments) so the code reads on its own. Use when planning or executing reverse engineering tasks with Ghidra."
---

# GhidraMCP Reverse Engineering Workflows

## Before You Start

**Always call `list_instances()` first** before any RE work. This discovers all running Ghidra CodeBrowser instances and shows which one is active. Without this step, you may target the wrong program or miss available instances entirely.

## Core Rule: Record Knowledge in the Database

**Every time you work out what something is for, write it into the Ghidra program — don't keep it only in the chat.** Applies to every workflow below. The goal: a person (or a later session) reads the decompiler output and understands it without redoing your analysis — no `FUN_80012345`, `param_1`, `DAT_8001f000` left behind for things you've understood.

| You understood… | Record it with |
|---|---|
| a function's purpose | `rename_function_by_address` (or `rename_function`) |
| its signature / parameter meanings | `set_function_prototype` — names and types params in one call |
| a local (or parameter) | `rename_variable`, then `set_local_variable_type` |
| a global / data label | `rename_data`, then `set_global_data_type` |
| a memory layout / set of constants | `create_struct` / `create_enum`, then apply the type to data, locals or prototypes |
| non-obvious logic (bit tricks, magic numbers, hardware access, side effects) | `set_decompiler_comment` / `set_disassembly_comment` at that instruction |

**When:** as soon as confidence is reasonable — don't wait for certainty. Mark guesses consistently: `maybe_` prefix for a plausible purpose (`maybe_load_save_slot`), `unk_` for "something, role unclear" (`unk_flags_8001f000`), or a comment stating the doubt. Revise the name when evidence confirms or refutes it, and drop the prefix then.

**Naming:**
- Descriptive `snake_case` verbs for functions (`decompress_lz_block`), nouns for data (`sprite_table`, `player_hp`). If the program already has a naming convention (imported symbols, earlier annotations, SDK names), match it instead.
- Keep addresses, register names, hardware details and units in comments, not in names — unless the address is the only distinguishing fact (then `unk_…_<addr>` is fine).
- Don't rename symbols that already carry real names: imports/exports, library or SDK functions identified by signatures/FID, symbols loaded from a symbol file. Rename a library function only if you've verified it's misidentified, and say why in a comment.

**Propagate:** after naming a function, look at its callers (`get_xrefs_to`) and callees (`get_callee`) — the new name and prototype often reveal what they do; name them too. Typed prototypes and struct pointers flow into every caller's decompilation, so prefer fixing the type at the source (prototype, global type) over patching each use site.

**Verify:** re-decompile (or `get_function_by_address` / `get_data_by_label`) after changes to confirm the rename or type took effect — a failed call or a name collision can leave the old name silently. Variable names can change after a retype/re-decompile; re-read before the next `rename_variable`.

**Summarise:** put a 1–3 line comment at the function's entry address with `set_decompiler_comment` — what it does, key inputs/outputs, side effects, and anything still uncertain.

## Single Instance (Default)

The simplest setup: one Ghidra CodeBrowser window with one program loaded.

1. Start Ghidra, open a program in CodeBrowser
2. Enable GhidraMCPPlugin (File > Configure > Developer)
3. The MCP bridge auto-starts via the `.mcp.json` configuration
4. Verify with `list_instances()` — confirms the program name and port
5. All tools automatically target the single running instance

## Multi-Instance Workflow

When analyzing multiple related binaries (e.g., a main executable and its DLLs, or a bootloader and firmware) in **separate CodeBrowser windows**:

1. Open multiple programs in separate CodeBrowser windows
2. Each window gets its own HTTP port (8080, 8081, 8082, ...)
3. Discover instances:
   ```
   list_instances()
   → port=8080, program=main.exe, project=MyProject [ACTIVE]
   → port=8081, program=helper.dll, project=MyProject
   ```
4. Switch between them as needed:
   ```
   use_instance(8081)  # now targeting helper.dll
   decompile_function("DllMain")
   use_instance(8080)  # back to main.exe
   ```

## Multi-Program Workflow (Single Window)

When a single CodeBrowser window contains multiple programs (e.g., a main executable with overlays or shared libraries loaded into the same project):

- **Listing tools** return results from all programs, prefixed with the program name
- **Address-based tools** target the active program by default
- Use the `PROGRAM::address` prefix to target other programs:

```
# Active program (e.g., main.exe) — plain address
decompile_function_by_address("0x00401000")
get_bytes("0x00401000", 16)

# Other programs in the same window — PROGRAM::address prefix
decompile_function_by_address("helper.dll::00401000")
get_xrefs_to("overlay.bin::00042090")
get_bytes("module.so::0x1000", 32)
```

This is common when analyzing systems with dynamically-loaded overlays, where the main executable and its modules coexist in the same Ghidra project.

## Function Analysis Workflow

Systematic approach to understanding an unknown function:

1. **Identify the target**
   ```
   search_functions_by_name("parse")     # find by name pattern
   get_current_function()                 # or use Ghidra cursor position
   get_function_by_address("0x401000")    # or by known address
   ```

2. **Read the code**
   ```
   decompile_function_by_address("0x401000")  # C pseudocode
   disassemble_function("0x401000")            # assembly listing
   ```

3. **Trace references**
   ```
   get_xrefs_to("0x401000")    # who calls this function?
   get_callee("0x401000")      # what does this function call?
   ```

4. **Annotate**
   ```
   rename_function_by_address("0x401000", "parse_packet_header")
   set_function_prototype("0x401000", "int parse_packet_header(uint8_t* buf, int len)")
   set_decompiler_comment("0x401020", "Validate magic bytes before parsing fields")
   ```

5. **Annotate local variables**
   ```
   rename_variable("parse_packet_header", "local_10", "packet_size")
   set_local_variable_type("0x401000", "packet_size", "uint32_t")
   ```

6. **Re-read** the decompiled output — it should now be much clearer with proper names and types.

## Data Annotation Workflow

When you find interesting data (strings, tables, buffers):

1. **Find strings of interest**
   ```
   list_strings(filter="error")
   list_strings(filter="config")
   ```

2. **Trace who uses the string**
   ```
   get_xrefs_to("0x4050A0")  # address of the string
   ```

3. **Label and type the data**
   ```
   rename_data("0x4050A0", "error_msg_invalid_header")
   set_global_data_type("0x4050A0", "char", 32)  # char[32]
   ```

4. **For data tables**, combine with struct types:
   ```
   set_global_data_type("0x405100", "ConfigEntry", 10)  # array of 10
   ```

## Type Creation Workflow

Creating structs and enums to improve decompiler output:

### Struct Creation

1. **Analyze raw bytes** to determine field layout:
   ```
   get_bytes("0x405000", 32)  # read raw data
   ```

2. **Create the struct**:
   ```
   create_struct("PacketHeader", members=[
     {"name": "magic",    "type": "uint",   "offset": 0,  "comment": "Should be 0xDEADBEEF"},
     {"name": "version",  "type": "ushort", "offset": 4},
     {"name": "flags",    "type": "ushort", "offset": 6},
     {"name": "length",   "type": "uint",   "offset": 8},
     {"name": "checksum", "type": "uint",   "offset": 12}
   ])
   ```

3. **Apply to data**:
   ```
   set_global_data_type("0x405000", "PacketHeader")
   ```

4. **Verify** — decompile functions that reference this address; fields should now show named access.

5. **Iterate** — add or modify members as understanding improves:
   ```
   add_struct_members("PacketHeader", [
     {"name": "payload_offset", "type": "uint", "offset": 16}
   ])
   ```

### Enum Creation

1. **Identify magic values or flags** in decompiled code

2. **Create the enum**:
   ```
   create_enum("PacketType", size=4, values=[
     {"name": "PKT_HANDSHAKE", "value": 1, "comment": "Initial handshake"},
     {"name": "PKT_DATA",      "value": 2, "comment": "Data transfer"},
     {"name": "PKT_ACK",       "value": 3, "comment": "Acknowledgment"},
     {"name": "PKT_FIN",       "value": 0xFF, "comment": "Connection close"}
   ])
   ```

3. **Apply to struct fields or variables**:
   ```
   set_local_variable_type("0x401000", "type_field", "PacketType")
   ```

## Byte Pattern Search Workflow

Finding specific data patterns across the binary:

1. **Search for known signatures**:
   ```
   search_bytes("89 50 4E 47")  # PNG signature
   search_bytes("FF D8 FF E0")  # JPEG signature
   search_bytes("50 4B 03 04")  # ZIP local file header
   ```

2. **Verify context** around each match:
   ```
   get_bytes("0x450000", 64)  # read surrounding bytes
   ```

3. **Bookmark findings** for later analysis:
   ```
   add_bookmark("0x450000", "Embedded Media", "PNG image data", "Info")
   add_bookmark("0x460000", "Embedded Media", "JPEG image data", "Info")
   ```

## Cross-Reference Tracing Workflow

Understanding how data flows through a program:

1. **Start from a known point** (string, function, data address):
   ```
   list_strings(filter="password")
   # found at 0x4080C0
   ```

2. **Find all code that references it**:
   ```
   get_xrefs_to("0x4080C0")
   # → 0x402100 in check_auth()
   # → 0x403500 in log_attempt()
   ```

3. **Analyze each referencing function**:
   ```
   decompile_function("check_auth")
   get_xrefs_to("0x402000")  # who calls check_auth?
   ```

4. **Follow the chain upward** until you reach the entry points (main, event handlers, etc.)

5. **Document the call chain** with comments:
   ```
   set_decompiler_comment("0x402100", "References password string for validation")
   set_disassembly_comment("0x402100", "XREF: password check")
   ```

## Combined Static + Runtime Analysis

Ghidra excels at static analysis, but some questions can only be answered at runtime. When a runtime debugger with MCP support is available (e.g., DuckStation MCP for PSX, or any debugger exposing an MCP interface), you can combine both approaches:

### Static → Runtime

Use Ghidra to identify targets, then verify at runtime:

1. **Find a function in Ghidra** — decompile and understand its logic
2. **Set a breakpoint in the debugger** at the function's entry address
3. **Trigger the code path** in the running program
4. **Read registers and memory** to confirm your static analysis interpretation
5. **Update Ghidra annotations** with confirmed runtime behavior

### Runtime → Static

Use runtime observations to guide Ghidra analysis:

1. **Observe unexpected behavior** at runtime (wrong output, crash, etc.)
2. **Find the responsible code** — use memory write breakpoints or memory scanning to locate the address being modified
3. **Look up the address in Ghidra** — decompile the function containing that address
4. **Trace the call chain** with `get_xrefs_to` to understand how the code is reached
5. **Annotate** the function with findings from both static and runtime analysis

### Common Patterns

| Goal | Runtime Step | Ghidra Step |
|------|-------------|-------------|
| Find what writes to address X | Set write breakpoint at X, read PC on hit | Decompile function at PC, trace callers |
| Understand unknown variable | Memory scan: initial → change value → refine | Once address found, find xrefs in Ghidra |
| Verify a patch | Read memory / disassemble at patched address | Compare with expected assembly from Ghidra |
| Map data structures | Snapshot memory before/after action, diff | Apply discovered struct layout in Ghidra |

### Tips for Combined Analysis

- **Keep address spaces aligned**: ensure Ghidra's base address matches the runtime load address so addresses correspond directly.
- **Use bookmarks** to mark addresses you've verified at runtime vs. those still unconfirmed.
- **Document runtime findings** with `set_decompiler_comment` so the knowledge persists in the Ghidra project.

## Tips

- **Start broad, then narrow down**: use `list_functions()` or `search_functions_by_name()` to orient, then decompile specific targets.
- **Rename early and often**: see the Core Rule above — tentative names like `maybe_init_network` beat `FUN_00401000`, and you can always rename again.
- **Use bookmarks** to track progress across analysis sessions. Categories help organize findings (e.g., "Crypto", "Network", "Parsing").
- **Paginate large results**: for binaries with thousands of functions, use `offset` and `limit` to avoid timeouts.
- **Decompile after annotation**: each rename, retype, or struct application improves the decompiler output. Re-read after changes.
- **Create functions manually**: if Ghidra didn't auto-detect a function (grayed-out code in the decompiler), right-click the entry point in the listing view and select **Create Function** (or press **F**). This is common with RAM dumps or hand-written assembly. Once created, the function can be decompiled and renamed normally.
- **Set register values for better decompilation**: some architectures use a global pointer register (e.g., `gp` on MIPS). If Ghidra can't resolve global accesses, right-click the function entry point → **Set Register Values** (CTRL+R) and provide the known register value. This disambiguates offsets and dramatically improves the decompiled output.
- **Iterate on parameter and variable types**: changing `int param_1` to `char *param_1` or a custom struct pointer in `set_function_prototype` can transform unreadable pointer arithmetic into clean field accesses. Each type refinement improves the decompiler output — re-read after changes and refine further.
