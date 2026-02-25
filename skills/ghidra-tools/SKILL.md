---
description: "GhidraMCP MCP tool reference: 48 tools for binary analysis — decompile, disassemble, list functions/classes/imports/exports, rename symbols, set types, cross-references, structs, enums, bookmarks, memory access. Use when working with Ghidra MCP tools."
---

# GhidraMCP Tool Reference

48 MCP tools for binary analysis via Ghidra. All tools are prefixed with `mcp__ghidra-mcp__` in the MCP protocol.

## Instance Management (2)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `list_instances()` | — | Discover all active Ghidra instances. Returns port, program name, project name. Shows `[ACTIVE]` for the current target. |
| `use_instance(port)` | `port: int` | Switch active instance. All subsequent tool calls target this instance. |

## Function Analysis (8)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `list_methods(offset, limit)` | `offset=0, limit=100` | List function names with pagination. Use for incremental browsing of large binaries. |
| `list_functions()` | — | List ALL functions at once (no pagination). May be slow on large binaries. |
| `decompile_function(name)` | `name: str` | Decompile function by exact name. Returns C-like pseudocode. |
| `decompile_function_by_address(address)` | `address: str` (hex, e.g. `"0x401000"`) | Decompile function containing the given address. Returns C-like pseudocode. |
| `disassemble_function(address)` | `address: str` (hex) | Get assembly listing for function at address. |
| `get_function_by_address(address)` | `address: str` (hex) | Get function metadata: name, entry point, size, signature. |
| `get_current_function()` | — | Get function at the current cursor position in Ghidra UI. |
| `get_callee(address)` | `address: str` (hex) | List all functions called by the function at the given address. |

## Symbol Listing (5)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `list_classes(offset, limit)` | `offset=0, limit=100` | List namespace/class names with pagination. |
| `list_segments(offset, limit)` | `offset=0, limit=100` | List memory segments (name, start, end, permissions). |
| `list_imports(offset, limit)` | `offset=0, limit=100` | List imported symbols (external functions/data). |
| `list_exports(offset, limit)` | `offset=0, limit=100` | List exported functions/symbols. |
| `list_namespaces(offset, limit)` | `offset=0, limit=100` | List all non-global namespaces. |

## Search (3)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `search_functions_by_name(query, offset, limit)` | `query: str, offset=0, limit=100` | Substring search across all function names. Case-insensitive. |
| `search_bytes(bytes_hex, offset, limit)` | `bytes_hex: str, offset=0, limit=100` | Search for byte sequence. Accepts `"DEADBEEF"` or `"DE AD BE EF"`. Returns matching addresses. |
| `list_strings(offset, limit, filter)` | `offset=0, limit=2000, filter=None` | List defined strings with addresses. Optional substring filter. |

## Renaming / Modification (6)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `rename_function(old_name, new_name)` | `old_name: str, new_name: str` | Rename function by its current name. |
| `rename_function_by_address(function_address, new_name)` | `function_address: str (hex), new_name: str` | Rename function at address. |
| `rename_variable(function_name, old_name, new_name)` | `function_name: str, old_name: str, new_name: str` | Rename a local variable within a function. |
| `rename_data(address, new_name)` | `address: str (hex), new_name: str` | Rename a data label at address. |
| `set_function_prototype(function_address, prototype)` | `function_address: str (hex), prototype: str` | Set function signature. Example: `"int parse_header(FILE* f, int size)"` |
| `set_local_variable_type(function_address, variable_name, new_type)` | `function_address: str (hex), variable_name: str, new_type: str` | Set a local variable's type within a function. |

## Cross-References (3)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `get_xrefs_to(address, offset, limit)` | `address: str (hex), offset=0, limit=100` | All references TO the given address (callers, data refs). |
| `get_xrefs_from(address, offset, limit)` | `address: str (hex), offset=0, limit=100` | All references FROM the given address (callees, data refs). |
| `get_function_xrefs(name, offset, limit)` | `name: str, offset=0, limit=100` | All references to a function by name. |

## Memory and Data (5)

| Tool | Parameters | Description |
|------|-----------|-------------|
| `list_data_items(offset, limit)` | `offset=0, limit=100` | List defined data labels with addresses and values. |
| `get_data_by_label(label)` | `label: str` | Get data info by exact label name (address, type, value). |
| `get_bytes(address, size)` | `address: str (hex), size=1` | Read raw bytes at address. Returns hexdump format. |
| `set_bytes(address, bytes_hex)` | `address: str (hex), bytes_hex: str` | Write bytes at address. Space-separated hex: `"90 90 90 90"`. |
| `set_global_data_type(address, data_type, length, clear_mode)` | `address: str (hex), data_type: str, length=-1, clear_mode="CHECK_FOR_SPACE"` | Apply a data type at address. Use for arrays: `set_global_data_type(addr, "char", 64)`. Clear modes: `CHECK_FOR_SPACE`, `CLEAR_SINGLE_DATA`, `CLEAR_ALL_CONFLICT_DATA`. |

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

## Address Format

All address parameters accept hex strings: `"0x401000"`, `"0x00401000"`, `"401000"`. The leading `0x` prefix is recommended for clarity.
