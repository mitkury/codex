## Agent Editing Toolkit: Reading, Patching, and Editing

This document explains how the agent performs file operations at a granular level for large codebases, focusing on three core capabilities: reading specific lines, applying context-safe patches, and performing precise fallback edits.

### Overview
- **Reading**: Use `functions.read_file` to fetch exact line ranges (or whole files) with numbered output.
- **Patching**: Use `functions.apply_patch` to apply structured, context-aware edits that are resilient to line-number shifts.
- **Editing (fallback)**: Use `functions.edit_file` for extremely targeted replacements when patching is ambiguous or repeatedly failing.

---

## Reading Lines with `functions.read_file`

**Purpose**: Efficiently read only the necessary parts of a file, with line numbers for accurate referencing and safe editing.

**Parameters**
- **`target_file`**: Absolute or relative path to the file.
- **`should_read_entire_file`**: Boolean. If `true`, reads the entire file (use sparingly on large files).
- **`start_line_one_indexed`**: Start line (1-indexed, inclusive). Required when not reading entire file.
- **`end_line_one_indexed_inclusive`**: End line (1-indexed, inclusive). Required when not reading entire file.
- **`explanation`**: One sentence explaining why this read is needed and how it contributes to the goal.

**Output Format**
- Each line is returned as: `LINE_NUMBER|LINE_CONTENT`
- Maximum lines per call: 1500. Prefer reading a generous but bounded window (e.g., 150–300 lines) around matches.

**Usage Template**
```json
{
  "target_file": "/abs/path/to/file.ts",
  "should_read_entire_file": false,
  "start_line_one_indexed": 1200,
  "end_line_one_indexed_inclusive": 1400,
  "explanation": "Read a focused window around the function we are about to edit to gather stable context."
}
```

**Best Practices**
- Read only what you need; avoid whole-file reads on very large files.
- Avoid re-reading the exact same range unless the file may have changed.
- When searching, pair with a code search step first, then read surrounding context.

---

## Patching with `functions.apply_patch`

**Purpose**: Apply robust, context-aware edits using a structured diff format that minimizes risk from shifting line numbers or concurrent changes.

**Parameters**
- **`file_path`**: Absolute or relative path to the file being edited.
- **`patch`**: A diff in the V4A format (see below) that specifies add/update operations with minimal, unique context.

**Patch (V4A) Format**
- Header per file action:
  - `*** Add File: path/to/file` to create a new file
  - `*** Update File: path/to/file` to modify an existing file
- Within an update, add scoped markers with `@@` to anchor changes to classes/functions/regions when helpful.
- Use `-` lines for content to remove/replace and `+` lines for new content.
- Include a few stable context lines before/after the change to ensure uniqueness.

**Example: Update an existing file**
```text
*** Update File: src/huge_file.ts
@@ class DataLoader
@@   public async fetchData():
-    const timeoutMs = 3000
+    const timeoutMs = 5000
@@
```

**Example: Add a new file**
```text
*** Add File: src/new_module.ts
export function greet(name: string): string {
  return `Hello, ${name}!`;
}
```

**Usage Template**
```json
{
  "file_path": "/abs/path/to/file.ts",
  "patch": "*** Update File: path/to/file.ts\n@@ function doWork()\n-  const retries = 1\n+  const retries = 3\n@@\n"
}
```

**Best Practices**
- Keep edits minimal and scoped; prefer multiple small, targeted edits over a single massive change.
- Provide 1–3 lines of stable context above and below each change.
- If a patch fails repeatedly due to context mismatches, re-read the file region to refresh context.

**Operational Guardrails**
- If you have not opened a file with `read_file` within your last few steps, re-read the relevant region before patching.
- Do not attempt more than three consecutive failing patches on the same file without re-reading with `read_file` to re-confirm contents.

---

## Precise Editing with `functions.edit_file` (Fallback)

**Purpose**: Perform pinpoint edits when patching is ambiguous or when you must replace a specific snippet without supplying full diff context.

**Parameters**
- **`target_file`**: Absolute or relative file path to modify.
- **`instructions`**: A single sentence describing what you are going to change. Used to assist the executor in applying the edit.
- **`code_edit`**: Only the exact lines you wish to change. All unchanged regions must be represented by the literal comment `// ... existing code ...` (or the language-appropriate comment marker).

**Code Edit Format**
- Do not include unrelated or unchanged code verbatim; replace large gaps with `// ... existing code ...`.
- Provide sufficient surrounding context to uniquely identify the edit location.
- Preserve the file’s existing indentation style (tabs vs spaces) and width.

**Example**
```json
{
  "target_file": "/abs/path/to/service.ts",
  "instructions": "I am increasing the default timeout from 3s to 5s in fetchData().",
  "code_edit": "@@ function fetchData()\n-  const timeoutMs = 3000\n+  const timeoutMs = 5000\n"
}
```

**Alternative Example with explicit context placeholders**
```text
// ... existing code ...
export async function fetchData() {
-  const timeoutMs = 3000
+  const timeoutMs = 5000
  return await doFetch(timeoutMs)
}
// ... existing code ...
```

**When to Use**
- After multiple failed `apply_patch` attempts due to ambiguous or unstable context.
- For very small, deterministic replacements where the exact snippet is unique.

---

## End-to-End Approach for Large Files (e.g., 10k lines)

1. Search and locate anchors (identifiers, imports, function names).
2. Use `functions.read_file` to load 150–300 lines around each match.
3. Prepare minimal, context-rich diffs and apply via `functions.apply_patch`.
4. If patching fails repeatedly, refresh context with `functions.read_file` and consider `functions.edit_file` as fallback.
5. Validate via tests/linters if available; iterate until green.

---

## Prompting/Description Guidance

When providing the `explanation`/`instructions` fields for these tools, write one concise sentence that states:
- What you are doing, and
- Why it is necessary to achieve the user’s goal.

**Examples**
- Read: "Read 200 lines around the `fetchData` function to capture stable context for a timeout change."
- Patch: "Increase the default timeout to 5s and ensure context markers uniquely identify `fetchData`."
- Edit: "Replace the literal timeout value in `fetchData` when the patch context is ambiguous."

---

## Finishing Behavior

The agent does not need to call a special finish tool. Completion is signaled by sending a normal assistant message with no tool calls once edits are applied and validated.

## Agent Editing Toolkit: Reading, Patching, and Editing

This document explains how the agent performs file operations at a granular level for large codebases, focusing on three core capabilities: reading specific lines, applying context-safe patches, and performing precise fallback edits.

### Overview
- **Reading**: Use `functions.read_file` to fetch exact line ranges (or whole files) with numbered output.
- **Patching**: Use `functions.apply_patch` to apply structured, context-aware edits that are resilient to line-number shifts.
- **Editing (fallback)**: Use `functions.edit_file` for extremely targeted replacements when patching is ambiguous or repeatedly failing.

---

## Reading Lines with `functions.read_file`

**Purpose**: Efficiently read only the necessary parts of a file, with line numbers for accurate referencing and safe editing.

**Parameters**
- **`target_file`**: Absolute or relative path to the file.
- **`should_read_entire_file`**: Boolean. If `true`, reads the entire file (use sparingly on large files).
- **`start_line_one_indexed`**: Start line (1-indexed, inclusive). Required when not reading entire file.
- **`end_line_one_indexed_inclusive`**: End line (1-indexed, inclusive). Required when not reading entire file.
- **`explanation`**: One sentence explaining why this read is needed and how it contributes to the goal.

**Output Format**
- Each line is returned as: `LINE_NUMBER|LINE_CONTENT`
- Maximum lines per call: 1500. Prefer reading a generous but bounded window (e.g., 150–300 lines) around matches.

**Usage Template**
```json
{
  "target_file": "/abs/path/to/file.ts",
  "should_read_entire_file": false,
  "start_line_one_indexed": 1200,
  "end_line_one_indexed_inclusive": 1400,
  "explanation": "Read a focused window around the function we are about to edit to gather stable context."
}
```

**Best Practices**
- Read only what you need; avoid whole-file reads on very large files.
- Avoid re-reading the exact same range unless the file may have changed.
- When searching, pair with a code search step first, then read surrounding context.

---

## Patching with `functions.apply_patch`

**Purpose**: Apply robust, context-aware edits using a structured diff format that minimizes risk from shifting line numbers or concurrent changes.

**Parameters**
- **`file_path`**: Absolute or relative path to the file being edited.
- **`patch`**: A diff in the V4A format (see below) that specifies add/update operations with minimal, unique context.

**Patch (V4A) Format**
- Header per file action:
  - `*** Add File: path/to/file` to create a new file
  - `*** Update File: path/to/file` to modify an existing file
- Within an update, add scoped markers with `@@` to anchor changes to classes/functions/regions when helpful.
- Use `-` lines for content to remove/replace and `+` lines for new content.
- Include a few stable context lines before/after the change to ensure uniqueness.

**Example: Update an existing file**
```text
*** Update File: src/huge_file.ts
@@ class DataLoader
@@   public async fetchData():
-    const timeoutMs = 3000
+    const timeoutMs = 5000
@@
```

**Example: Add a new file**
```text
*** Add File: src/new_module.ts
export function greet(name: string): string {
  return `Hello, ${name}!`;
}
```

**Usage Template**
```json
{
  "file_path": "/abs/path/to/file.ts",
  "patch": "*** Update File: path/to/file.ts\n@@ function doWork()\n-  const retries = 1\n+  const retries = 3\n@@\n"
}
```

**Best Practices**
- Keep edits minimal and scoped; prefer multiple small, targeted edits over a single massive change.
- Provide 1–3 lines of stable context above and below each change.
- If a patch fails repeatedly due to context mismatches, re-read the file region to refresh context.

**Operational Guardrails**
- If you have not opened a file with `read_file` within your last few steps, re-read the relevant region before patching.
- Do not attempt more than three consecutive failing patches on the same file without re-reading with `read_file` to re-confirm contents.

---

## Precise Editing with `functions.edit_file` (Fallback)

**Purpose**: Perform pinpoint edits when patching is ambiguous or when you must replace a specific snippet without supplying full diff context.

**Parameters**
- **`target_file`**: Absolute or relative file path to modify.
- **`instructions`**: A single sentence describing what you are going to change. Used to assist the executor in applying the edit.
- **`code_edit`**: Only the exact lines you wish to change. All unchanged regions must be represented by the literal comment `// ... existing code ...` (or the language-appropriate comment marker).

**Code Edit Format**
- Do not include unrelated or unchanged code verbatim; replace large gaps with `// ... existing code ...`.
- Provide sufficient surrounding context to uniquely identify the edit location.
- Preserve the file’s existing indentation style (tabs vs spaces) and width.

**Example**
```json
{
  "target_file": "/abs/path/to/service.ts",
  "instructions": "I am increasing the default timeout from 3s to 5s in fetchData().",
  "code_edit": "@@ function fetchData()\n-  const timeoutMs = 3000\n+  const timeoutMs = 5000\n"
}
```

**Alternative Example with explicit context placeholders**
```text
// ... existing code ...
export async function fetchData() {
-  const timeoutMs = 3000
+  const timeoutMs = 5000
  return await doFetch(timeoutMs)
}
// ... existing code ...
```

**When to Use**
- After multiple failed `apply_patch` attempts due to ambiguous or unstable context.
- For very small, deterministic replacements where the exact snippet is unique.

---

## End-to-End Approach for Large Files (e.g., 10k lines)

1. Search and locate anchors (identifiers, imports, function names).
2. Use `functions.read_file` to load 150–300 lines around each match.
3. Prepare minimal, context-rich diffs and apply via `functions.apply_patch`.
4. If patching fails repeatedly, refresh context with `functions.read_file` and consider `functions.edit_file` as fallback.
5. Validate via tests/linters if available; iterate until green.

---

## Prompting/Description Guidance

When providing the `explanation`/`instructions` fields for these tools, write one concise sentence that states:
- What you are doing, and
- Why it is necessary to achieve the user’s goal.

**Examples**
- Read: "Read 200 lines around the `fetchData` function to capture stable context for a timeout change."
- Patch: "Increase the default timeout to 5s and ensure context markers uniquely identify `fetchData`."
- Edit: "Replace the literal timeout value in `fetchData` when the patch context is ambiguous."

---

## Finishing Behavior

The agent does not need to call a special finish tool. Completion is signaled by sending a normal assistant message with no tool calls once edits are applied and validated.

