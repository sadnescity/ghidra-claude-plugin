---
description: "GhidraMCP reverse engineering workflows: function analysis, cross-reference tracing, symbol annotation, multi-instance management, struct/enum creation. Use when planning or executing reverse engineering tasks with Ghidra."
---

# GhidraMCP Reverse Engineering Workflows

## Single Instance (Default)

The simplest setup: one Ghidra CodeBrowser window with one program loaded.

1. Start Ghidra, open a program in CodeBrowser
2. Enable GhidraMCPPlugin (File > Configure > Developer)
3. The MCP bridge auto-starts via the `.mcp.json` configuration
4. All tools automatically target the single running instance

No instance management needed. Just start calling tools directly.

## Multi-Instance Workflow

When analyzing multiple related binaries (e.g., a main executable and its DLLs, or a bootloader and firmware):

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

## Tips

- **Start broad, then narrow down**: use `list_functions()` or `search_functions_by_name()` to orient, then decompile specific targets.
- **Rename early and often**: even tentative names like `maybe_init_network` are better than `FUN_00401000`. You can always rename again.
- **Use bookmarks** to track progress across analysis sessions. Categories help organize findings (e.g., "Crypto", "Network", "Parsing").
- **Paginate large results**: for binaries with thousands of functions, use `offset` and `limit` to avoid timeouts.
- **Decompile after annotation**: each rename, retype, or struct application improves the decompiler output. Re-read after changes.
