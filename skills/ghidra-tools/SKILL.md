---
name: ghidra-tools
description: "GhidraMCP MCP tool reference: 80 tools for binary analysis — find the open programs, decompile, disassemble, list and search, call graph and xrefs, rename and type (one or many at once), comments of every kind, structs/unions/enums and C header import, equates and references, memory read/patch/overlays, code-gap search, P-code emulation. Use when working with Ghidra MCP tools."
---

# GhidraMCP Tool Reference

80 MCP tools. When loaded from this plugin, Claude Code exposes them as `mcp__plugin_ghidra_ghidra-mcp__<tool>`. Parameters shown as `name=default` are optional, `name?` optional without default.

## The program (1)

| Tool | Description |
|------|-------------|
| `list_instances()` | Every running Ghidra instance and every program open in each (in front or not), with the `program="..."` value that names it: its name, or `project:/path` when the name is open in more than one project. |

**Every other tool takes a required `program` argument** — the value `list_instances()` shows — before the parameters listed below (which omit it). Nothing is remembered between calls and nothing is chosen for you, even with a single program open.

## Errors

A failed call is an MCP error whose text starts with a code, then says what to do:
- Finding the program (nothing was sent): `[NO_GHIDRA]`, `[PROGRAM_NOT_FOUND]`, `[AMBIGUOUS]` (a `program` value or a name matching more than one thing), `[BUSY]` (a Ghidra port did not answer in time, so the program might also be open there — retry).
- Reaching Ghidra: `[NOT_HERE]` (the instance no longer has that program open), `[CANCELLED]` (the call was cancelled; Ghidra stops it too, or skips it if it had not started), `[UNREACHABLE]`, `[UNAVAILABLE]` (the CodeBrowser is closing), `[TIMEOUT]` (Ghidra was asked to stop the call: reads and emulation stop, a write already running may still complete — check before repeating it), `[OLD_PLUGIN]` / `[BAD_RESPONSE]` (update the Ghidra extension).
- The call itself: `[MISSING_PARAMETER]` (also for a missing `program`), `[INVALID_PARAMETER]` (also for a misspelt parameter name), `[BAD_ADDRESS]`, `[FUNCTION_NOT_FOUND]`, `[NOT_FOUND]` (e.g. an unknown data type — never replaced by `int`), `[CONFLICT]` (name taken, field overlap), `[DECOMPILE_FAILED]`, `[UNSUPPORTED]`, `[FAILED]`, `[INTERNAL_ERROR]` (a bug: report it).
A write that fails is rolled back entirely: batch tools (`set_variables`, `set_comments`, `create_label` with a list, struct members) apply all or nothing.

## Addresses

- Hex, with or without `0x`: `"0x80010000"`, `"80010000"` — the program's default space.
- Overlays: `"OVL::80100000"` (also `"OVL:80100000"`). A plain address never means an overlay.
- Where an address is expected, a **function or label name** also works (`"main"`, `"g_party"`) if it names exactly one place; otherwise `[AMBIGUOUS]` lists them.
- Addresses in results use the same notation and can be passed back unchanged.

## Pagination

List tools take `offset`/`limit` and end with a footer whenever there is more:
`-- items 1-1000 of 2400; next page: offset=1000 --`. No footer means you have everything.

## Program and view (5)

| Tool | Description |
|------|-------------|
| `get_program_info()` | Identify the target program: name, project path, format, processor/language, compiler spec, endianness, pointer size, image base, address range, memory blocks, function count, entry points, unsaved changes. Call it first to confirm which binary you are working on. |
| `get_current_address()` | Get the address the analyst has selected in the Ghidra listing (cursor position). |
| `get_current_function()` | Get the function containing the analyst's cursor in the Ghidra listing. |
| `goto_address(address)` | Move the analyst's Ghidra listing and decompiler view to an address (or a function/label name), bringing the target program to the front. Use it to show the analyst what you are talking about. |
| `save_program()` *(writes)* | Save the program's changes to the project, as File > Save does. Changes made through these tools are otherwise unsaved until the analyst saves. |

## Listings (10)

| Tool | Description |
|------|-------------|
| `list_functions(offset=0, limit=1000, filter?)` | List functions as 'name @ address', in address order, optionally filtered by a case-insensitive substring of the name. Paged: a footer gives the total and the next offset. |
| `list_methods(offset=0, limit=100)` | List function names only, in address order (paged). list_functions also gives addresses. |
| `search_functions_by_name(query, offset=0, limit=100)` | Find functions whose name contains a substring (case-insensitive), as 'name @ address'. |
| `list_classes(offset=0, limit=100)` | List class namespaces (fully qualified), paged. |
| `list_namespaces(offset=0, limit=100)` | List non-global namespaces (fully qualified), paged. |
| `list_segments(offset=0, limit=100)` | List memory blocks: name, range, size, permissions (r/w/x, v=volatile), initialized or not, overlay or not. |
| `list_imports(offset=0, limit=100)` | List imported (external) symbols with their library. |
| `list_exports(offset=0, limit=100)` | List exported symbols / entry points as 'name @ address'. |
| `list_data_items(offset=0, limit=100)` | List defined data: address, label, type, value. |
| `list_strings(offset=0, limit=2000, filter?)` | List defined strings as 'address: "text"', optionally filtered by a case-insensitive substring of the text. |

## Functions (12)

| Tool | Description |
|------|-------------|
| `decompile_function(name)` | Decompile a function by name and return its C pseudocode. Fails (AMBIGUOUS) if several functions share the name; then use decompile_function_by_address. |
| `decompile_function_by_address(address)` | Decompile the function at, or containing, an address (or a function/label name) and return its C pseudocode. |
| `disassemble_function(address, offset=0, limit=2000)` | Disassemble a function's body: one line per instruction, 'address: bytes  instruction ; comment', labels on their own line. Paged. |
| `get_listing(address, count=40)` | Show the listing from an address onward, whether or not it belongs to a function: instructions, defined data and undefined bytes, with labels and comments. Use it for code that is not (yet) a function and for data tables. |
| `get_function_by_address(address)` | Describe the function at, or containing, an address: name, entry, signature, calling convention, body range and size, parameters, flags (thunk, no-return, external), plate comment. |
| `create_function(address, name="")` *(writes)* | Create a function at one or more addresses, disassembling first if the bytes are undefined. Auto-analysis leaves code that nothing references statically (jump-table targets, hand-written assembly, overlays) as undefined bytes. Refuses an address inside an existing function, since that would split it. Returns a per-address report and counts. |
| `delete_function(address)` *(writes)* | Delete the function defined at one or more addresses (the inverse of create_function). Removes the definition (name, signature, locals) and leaves the instructions. Only an address a function STARTS at is deleted; an address inside one is reported and left alone. |
| `disassemble(address, follow_flow=true)` *(writes)* | Disassemble undefined bytes starting at an address, following the flow (calls, branches) as Ghidra's D key does, without creating a function. Reports how many instructions now exist. |
| `rename_function(old_name, new_name)` *(writes)* | Rename a function given its current name. Prefer rename_function_by_address: names can be ambiguous. |
| `rename_function_by_address(function_address, new_name)` *(writes)* | Rename the function at, or containing, an address. |
| `set_function_prototype(function_address, prototype)` *(writes)* | Set a function's signature from a C prototype, e.g. 'int damage_actor(Actor *a, int amount)' or 'void __stdcall f(void)'. Types must exist (create them first). The name in the prototype renames the function. |
| `set_function_no_return(function_address, no_return=true)` *(writes)* | Mark (or unmark) a function as never returning, e.g. an error handler or an infinite loop. Wrongly unmarked, such calls make the decompiler run into the following function. |

## Parameters and locals (4)

| Tool | Description |
|------|-------------|
| `list_variables(function_address)` | List a function's parameters and local variables as the decompiler shows them: name, type, storage, and whether the name/type was set by a user. Use it before rename_variable / set_local_variable_type / set_variables. |
| `rename_variable(function_name?, old_name, new_name, function_address?)` *(writes)* | Rename a parameter or local variable of a function, as shown in the decompiler. Give the function by address (preferred) or name. |
| `set_local_variable_type(function_address, variable_name, new_type)` *(writes)* | Set the type of a parameter or local variable, as shown in the decompiler (e.g. 'Actor *', 'short', 'uint[4]'). |
| `set_variables(function_address, variables)` *(writes)* | Rename and/or retype several variables of one function in a single decompile and a single transaction (all or nothing). Each entry: {"name": current name, "new_name": optional, "type": optional C type}. Far cheaper than one call per variable when annotating a whole function. |

## Cross-references and call graph (5)

| Tool | Description |
|------|-------------|
| `get_xrefs_to(address, offset=0, limit=100)` | List references TO an address (or symbol): 'from-address (in function) TYPE'. Paged. |
| `get_xrefs_from(address, offset=0, limit=100)` | List references FROM an instruction or data address: 'to-address (name) TYPE'. Paged. |
| `get_function_xrefs(name?, offset=0, limit=100, address?)` | List references to a function's entry point, by name or (preferably) address. Paged. |
| `get_function_calls(address, direction="both", offset=0, limit=200)` | One level of the call graph around a function. Callers: call sites that reach it, with the calling function. Callees: every call made from its body, including calls through pointers and unresolved indirect calls (marked indirect), with thunks resolved. offset/limit apply to each list. |
| `get_callee(address)` | List the distinct functions called by the function at, or containing, an address, as 'name @ address' (external functions marked). |

## Comments (5)

| Tool | Description |
|------|-------------|
| `set_decompiler_comment(address, comment)` *(writes)* | Set the comment the decompiler shows above the statement at an address (a PRE comment). An empty comment removes it. |
| `set_disassembly_comment(address, comment)` *(writes)* | Set the end-of-line comment shown in the disassembly at an address (an EOL comment). An empty comment removes it. |
| `set_comment(address, comment, type="pre")` *(writes)* | Set a listing comment of any kind at any address (code or data). type: plate (a header box above; at a function entry it is the function's header comment, shown by the decompiler too), pre (above the line; shown in the decompiler), eol (end of line), post (below the line), repeatable (shown at every reference). An empty comment removes it. |
| `set_comments(comments)` *(writes)* | Set many comments in one transaction (all or nothing). Each entry: {"address", "comment", "type"?: plate\|pre\|eol\|post\|repeatable (default pre)}. Use it to annotate a whole function at once. |
| `get_comments(address, scope="address", offset=0, limit=500)` | Read comments. With scope=address: every kind of comment at that address. With scope=function: every comment inside the function containing the address, in address order. |

## Labels, data names, bookmarks (6)

| Tool | Description |
|------|-------------|
| `rename_data(address, new_name)` *(writes)* | Name the data (or any location) at an address: renames its primary label, or creates one if it has none. Typically used for globals such as 'g_gold'. |
| `create_label(address?, name?, labels?, make_primary=true)` *(writes)* | Create labels (extra names at addresses, e.g. jump targets or table entries). One: address+name. Many in one transaction: labels=[{address, name}, ...]. The new label becomes primary when make_primary is true. |
| `delete_label(address, name)` *(writes)* | Delete a label at an address by name (function names are not labels: use delete_function). |
| `get_data_by_label(label)` | Look up a label: '<label> -> <address> : <type> <value>' for each place it names. |
| `add_bookmark(address, category, comment, type="Note")` *(writes)* | Create a bookmark at an address (replacing one of the same type and category there). |
| `list_bookmarks(type?, offset=0, limit=200)` | List bookmarks: 'address type/category: comment', optionally only one type. |

## Data types (18)

| Tool | Description |
|------|-------------|
| `create_struct(name, category?, size=0, members?)` *(writes)* | Create a structure, optionally with its members. With size > 0 the struct starts as that many undefined bytes and members with an offset fill them in place. |
| `add_struct_members(struct_name, members, category?)` *(writes)* | Add members to an existing structure (in one transaction). Members with an offset fill undefined bytes there; others are appended. |
| `remove_struct_members(struct_name, members, category?)` *(writes)* | Remove members from a structure by name. Their bytes become undefined; later fields keep their offsets. |
| `clear_struct(struct_name, category?)` *(writes)* | Remove every member of a structure, keeping its name (and size 0). |
| `modify_struct_field(struct_name, field, new_name?, new_type?, comment?, category?)` *(writes)* | Change one field of a structure or union in place: rename it, retype it (at the same offset; a larger type takes undefined bytes after it, and fails if a defined field is in the way), and/or set its comment. |
| `get_struct(name, category?)` | Show a structure (or union) as C, with every field's offset, type, name and comment, and the undefined gaps between them. |
| `create_union(name, members, category?)` *(writes)* | Create a union with its members. |
| `create_enum(name, category?, size=4, values?)` *(writes)* | Create an enum, optionally with values. |
| `add_enum_values(enum_name, values, category?)` *(writes)* | Add values to an existing enum. |
| `remove_enum_values(enum_name, values, category?)` *(writes)* | Remove values from an enum by name. |
| `get_enum(name, category?)` | Show an enum: its values (hex and decimal) and comments. |
| `create_typedef(name, base_type, category?)` *(writes)* | Create a typedef: a new name for an existing type, e.g. name='ActorPtr' base_type='Actor *'. |
| `import_c_types(source)` *(writes)* | Parse C declarations (structs, unions, enums, typedefs, function pointer typedefs) with Ghidra's C parser and add them to the program. The fastest way to define many types, or types that refer to each other. Pass the source TEXT, not a path. Reports the types added or changed and any parser messages. |
| `list_data_types(filter?, category?, kind="any", offset=0, limit=200)` | List the program's data types as 'path  kind  size', optionally filtered by a substring of the name, a category prefix and/or a kind. |
| `rename_data_type(name, new_name, category?)` *(writes)* | Rename a data type in place; every use of it follows. |
| `delete_data_type(name, category?)` *(writes)* | Delete a data type. Data and variables using it become undefined, so check get_xrefs/usage first. |
| `set_global_data_type(address, data_type, length=-1, clear_mode="CHECK_FOR_SPACE")` *(writes)* | Apply a data type at an address (define a global): e.g. 'int', 'Actor', 'char[16]', 'Actor[8]', 'string' (length found from the terminating zero). clear_mode decides what happens to data already there. |
| `clear_data(address, length=1)` *(writes)* | Undefine the data (or instructions) covering an address range, making the bytes undefined again. |

## C++ classes (3)

| Tool | Description |
|------|-------------|
| `create_class(name, parent_namespace?, members?)` *(writes)* | Create a C++ class: a class namespace and its structure (in /classes/...), optionally with data members. |
| `add_class_members(class_name, members, parent_namespace?)` *(writes)* | Add data members to a class's structure. |
| `remove_class_members(class_name, members, parent_namespace?)` *(writes)* | Remove data members from a class's structure by name. |

## Memory (4)

| Tool | Description |
|------|-------------|
| `read_memory(address, length=256)` | Read program memory as a hex dump with ASCII, 16 bytes per line. Bytes that are not mapped or not initialized show as '??' and are listed after the dump, so a gap can never pass for zeros. |
| `get_bytes(address, size=1)` | Read raw bytes as a hex dump (same as read_memory; kept for existing callers). |
| `set_bytes(address, bytes_hex)` *(writes)* | Write bytes into program memory (patch). Instructions overlapping the range are cleared and the range is disassembled again; defined data keeps its type with the new value. |
| `create_memory_block(name, address, length=0, file_path?, file_offset=0, overlay=false, permissions="rwx", initialized=true)` *(writes)* | Add a memory block, e.g. to load an overlay or a data file into the program. Contents come from a file on disk (file_path, file_offset, length) or are zero-filled/uninitialized. overlay=true creates an overlay block that can share addresses with existing memory (PSX/PS2 overlays). |

## Search (3)

| Tool | Description |
|------|-------------|
| `search_bytes(bytes_hex, offset=0, limit=100)` | Search memory for a byte pattern. Wildcards: '??' for any byte, '?' for any nibble (e.g. '3c 08 80 ?? 25 08' or '3c0880??'). Lists each hit with the function containing it. Paged. |
| `search_instructions(pattern, function_address?, offset=0, limit=200)` | Search instructions by a regular expression, matched against both the raw text ('jal 0x80012345') and the listing text with names ('jal clamp_int'), e.g. 'lui .*0x1f80' (hardware register access), '^jal 0x80012345', '^syscall'. Optionally only inside one function. Paged. |
| `find_code_gaps(min_length=16, offset=0, limit=200)` | Find stretches of executable memory that no function covers: undefined bytes, and instructions outside any function. These are where analysis missed code (jump-table targets, functions only reached through pointers). Ranges shorter than min_length are skipped; all-zero ranges are reported as padding. |

## References and equates (3)

| Tool | Description |
|------|-------------|
| `add_reference(from_address, to_address, ref_type="DATA", operand_index=-1)` *(writes)* | Add a reference from an instruction or data address to a target the analyzer could not infer: a pointer computed at run time, a jump-table entry, a callback stored in a table. Makes the target show up in xrefs and, for calls/jumps, in the call graph. |
| `remove_reference(from_address, to_address)` *(writes)* | Remove the references from one address to another (all kinds), e.g. a wrong pointer the analyzer guessed. |
| `set_equate(address, value, name, operand_index=-1)` *(writes)* | Show a constant operand by name (an equate), e.g. 0x1f801810 as GPU_DATA or 4 as ITEM_POTION, in the listing and the decompiler. Finds the operand holding the value at the instruction; pass operand_index if several do. An empty name removes the equate there. |

## Emulation (1)

| Tool | Description |
|------|-------------|
| `emulate_function(address, registers={}, memory=[], read_memory=[], read_registers=[], max_steps=2000000, stack_pointer?)` | Run a function in Ghidra's P-code emulator with the register and memory inputs you give, until it returns (or a step limit, or an error), and report the result registers, the memory it wrote and any memory you ask to read. Use it to learn what a routine computes -- a decompressor, a checksum, a table lookup -- without running the game. The program itself is not modified. Calls into code that is not in the program (BIOS, hardware) stop the run with an error at that address. |

## Saving

Changes take effect in Ghidra immediately (and are in its undo history, named after the Claude Code session) but are written to the project only by `save_program()` or by the analyst.
