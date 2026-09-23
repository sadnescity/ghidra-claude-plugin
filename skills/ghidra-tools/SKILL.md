---
name: ghidra-tools
description: "GhidraMCP MCP tool reference: 50 tools for binary analysis — decompile, disassemble, list functions/classes/imports/exports, rename symbols, set types, cross-references, structs, enums, bookmarks, memory access. Use when working with Ghidra MCP tools."
---

# GhidraMCP Tool Reference

50 MCP tools for binary analysis via Ghidra. When loaded from this plugin, Claude Code exposes them as `mcp__plugin_ghidra_ghidra-mcp__<tool>`.

## Instance Management (3)

The bridge never picks a target by itself: call `use_program` or `use_instance` before any other tool, or every call is refused.

| Tool | Parameters | Description |
|------|-----------|-------------|
| `list_instances()` | — | Discover all active Ghidra instances. Returns port, program name, project name. Shows `[ACTIVE]` for the current target. |
| `use_instance(port)` | `port: int` | Target the program open on that port. The bridge remembers the program, not the port, and follows it if it moves. |
| `use_program(name)` | `name: str` | Target by program name — the exact name `list_instances()` shows (the program's file name in the project); no partial match. Preferred: ports change between restarts. Refused if the name is open on several ports. |

## Function Analysis (10)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `list_methods(offset, limit)` | `offset=0, limit=100` | List function names with pagination. Use for incremental browsing of large binaries. |
| `list_functions()` | — | List ALL functions at once (no pagination). May be slow on large binaries. |
| `decompile_function(name)` | `name: str` | Decompile function by exact name. Returns C-like pseudocode. |
| `decompile_function_by_address(address)` | `address: str` (hex, e.g. `"0x401000"`) | Decompile function containing the given address. Returns C-like pseudocode. |
| `disassemble_function(address)` | `address: str` (hex) | Get assembly listing for function at address. |
| `get_function_by_address(address)` | `address: str` (hex) | Get the function whose **entry point** is exactly at address: name, signature, entry, body range. An address inside a function returns "No function found". |
| `get_current_function()` | — | Get function at the current cursor position in Ghidra UI. |
| `get_callee(address)` | `address: str` (hex) | List all functions called by the function at the given address. |
| `create_function(address, name)` | `address: str` (hex, or several comma-separated), `name=""` (single address only) | Create a function, disassembling first if needed. For code auto-analysis missed (jump table targets, hand-written asm, RAM dumps). Returns a per-address report. |
| `delete_function(address)` | `address: str` (hex, or several comma-separated) | Delete the function starting at the address, leaving the instructions disassembled. An address inside a function is reported, not deleted. |

## Symbol Listing (5)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `list_classes(offset, limit)` | `offset=0, limit=100` | List namespace/class names with pagination. |
| `list_segments(offset, limit)` | `offset=0, limit=100` | List memory blocks (name, start, end). |
| `list_imports(offset, limit)` | `offset=0, limit=100` | List imported symbols (external functions/data). |
| `list_exports(offset, limit)` | `offset=0, limit=100` | List exported functions/symbols. |
| `list_namespaces(offset, limit)` | `offset=0, limit=100` | List all non-global namespaces. |

## Search (3)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `search_functions_by_name(query, offset, limit)` | `query: str, offset=0, limit=100` | Substring search across all function names. Case-insensitive. |
| `search_bytes(bytes_hex, offset, limit)` | `bytes_hex: str, offset=0, limit=100` | Search for byte sequence. Accepts `"DEADBEEF"` or `"DE AD BE EF"`. Returns matching addresses. |
| `list_strings(offset, limit, filter)` | `offset=0, limit=2000, filter=None` | List defined strings with addresses. Optional case-insensitive substring filter. |

## Renaming / Modification (6)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `rename_function(old_name, new_name)` | `old_name: str, new_name: str` | Rename function by its current name. |
| `rename_function_by_address(function_address, new_name)` | `function_address: str (hex), new_name: str` | Rename function at address. |
| `rename_variable(function_name, old_name, new_name)` | `function_name: str, old_name: str, new_name: str` | Rename a local variable within a function. |
| `rename_data(address, new_name)` | `address: str (hex), new_name: str` | Rename a data label at address. |
| `set_function_prototype(function_address, prototype)` | `function_address: str (hex), prototype: str` | Set function signature. Example: `"int parse_header(FILE* f, int size)"` |
| `set_local_variable_type(function_address, variable_name, new_type)` | `function_address: str (hex), variable_name: str, new_type: str` | Set a local variable's type within a function. |

Use these continuously during analysis: whenever you identify what a function, variable, parameter, global, struct or enum is for, record it in the database (rename, retype, comment) so the decompiled code explains itself. Conventions (`maybe_`/`unk_` for uncertain names, not renaming library symbols, propagation, verification) are in the **ghidra-workflows** skill, "Core Rule: Record Knowledge in the Database".

## Cross-References (3)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `get_xrefs_to(address, offset, limit)` | `address: str (hex), offset=0, limit=100` | All references TO the given address (callers, data refs). |
| `get_xrefs_from(address, offset, limit)` | `address: str (hex), offset=0, limit=100` | References FROM the instruction/data at exactly that address (calls, jumps, data refs). For everything a function calls, use `get_callee`. |
| `get_function_xrefs(name, offset, limit)` | `name: str, offset=0, limit=100` | All references to a function by name. |

## Memory and Data (5)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `list_data_items(offset, limit)` | `offset=0, limit=100` | List defined data labels with addresses and values. |
| `get_data_by_label(label)` | `label: str` | Get data info by exact label name (address, type, value). |
| `get_bytes(address, size)` | `address: str (hex), size=1` | Read raw bytes at address. Returns hexdump format. |
| `set_bytes(address, bytes_hex)` | `address: str (hex), bytes_hex: str` | Write bytes at address (existing mapped memory only). Space-separated hex: `"90 90 90 90"`. Clears the code units in the written range and re-disassembles from the address, so defined data there is lost. |
| `set_global_data_type(address, data_type, length, clear_mode)` | `address: str (hex), data_type: str, length=-1, clear_mode="CHECK_FOR_SPACE"` | Apply a data type at address. Use for arrays: `set_global_data_type(addr, "char", 64)`. Clear modes: `CHECK_FOR_SPACE` (default), `CLEAR_SINGLE_DATA`, `CLEAR_ALL_UNDEFINED_CONFLICT_DATA`, `CLEAR_ALL_DEFAULT_CONFLICT_DATA`, `CLEAR_ALL_CONFLICT_DATA`. |

## Commenting (2)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `set_decompiler_comment(address, comment)` | `address: str (hex), comment: str` | Set comment visible in the decompiler/pseudocode view. |
| `set_disassembly_comment(address, comment)` | `address: str (hex), comment: str` | Set comment visible in the disassembly/listing view. |

## Structs (5)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `create_struct(name, category, size, members)` | `name: str, category=None, size=0, members=None` | Create a new structure type. Members format: `[{"name": "field", "type": "int", "offset": 0, "comment": "optional"}]`. If size is 0 and members are provided, size is auto-calculated. |
| `get_struct(name, category)` | `name: str, category=None` | Get struct definition as JSON (fields, offsets, types, sizes). |
| `add_struct_members(struct_name, members, category)` | `struct_name: str, members: list, category=None` | Add members to existing struct. Same member format as `create_struct`. |
| `remove_struct_members(struct_name, members, category)` | `struct_name: str, members: list[str], category=None` | Remove members by field name. |
| `clear_struct(struct_name, category)` | `struct_name: str, category=None` | Remove all members from a struct, leaving it empty. |

## Enums (4)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `create_enum(name, category, size, values)` | `name: str, category=None, size=4, values=None` | Create a new enum type. Values format: `[{"name": "VAL_A", "value": 0, "comment": "optional"}]`. |
| `get_enum(name, category)` | `name: str, category=None` | Get enum definition as JSON (names, values, comments). |
| `add_enum_values(enum_name, values, category)` | `enum_name: str, values: list, category=None` | Add values to existing enum. Same format as `create_enum`. |
| `remove_enum_values(enum_name, values, category)` | `enum_name: str, values: list[str], category=None` | Remove enum values by name. |

## Classes (2)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `add_class_members(class_name, members, parent_namespace)` | `class_name: str, members: list, parent_namespace=None` | Add members to a class namespace. |
| `remove_class_members(class_name, members, parent_namespace)` | `class_name: str, members: list[str], parent_namespace=None` | Remove members from a class by name. |

## Annotation (1)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `add_bookmark(address, category, comment, type)` | `address: str (hex), category: str, comment: str, type="Note"` | Create a bookmark at address. Types: `Note`, `Info`, `Warning`, `Error`, `Analysis`. |

## Selection (1)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `get_current_address()` | — | Get the address at the current cursor position in Ghidra UI. |

## Pagination Notes

Most listing tools support `offset` and `limit` parameters for pagination:

- `offset` — number of items to skip (default: 0)
- `limit` — maximum items to return (default: 100, strings default: 2000)

For large binaries, paginate through results incrementally to avoid timeouts:

```
list_methods(offset=0, limit=100)    # first 100
list_methods(offset=100, limit=100)  # next 100
```

## Which Program a Call Acts On

Every tool acts on **one program**: the one the bridge is targeting (`use_program` / `use_instance`), which is the program in the **active tab** of that instance's CodeBrowser window. There is no cross-program listing and no `PROGRAM::address` prefix.

- To work on another program, open it in its own CodeBrowser window (it gets its own port) and switch with `use_program(name)`.
- If one window holds several programs as tabs, the instance serves whichever tab is active. Switching tabs changes what that port holds, and the bridge refuses every call (it checks the program's identity each time) until the tab is switched back or the new program is chosen with `use_program` / `use_instance`.

## Address Format

Address strings are parsed by the target program's own Ghidra address factory:

- Plain hex, with or without `0x`: `"0x80010000"`, `"80010000"` — the program's **default** address space (e.g. `ram`).
- `SPACE:offset` for another space: `"ram:80010000"`.
- **Overlay spaces**: `"OVL::80010000"` (the form Ghidra prints) or `"OVL:80010000"`, where `OVL` is the overlay block/space name. A plain address never resolves into an overlay, even when the overlay covers the same range — it always means the default space.

Addresses returned by the tools use the same notation (overlay addresses already carry `OVL::`), so they can be passed back unchanged. A string that does not parse (unknown space name, bad hex) is not guessed at: the tool returns an error or an empty result. Listing tools cover the whole program, overlay spaces included.
