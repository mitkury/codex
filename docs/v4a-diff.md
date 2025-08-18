## V4A Diff Format (Concise Guide)

V4A is a context-based diff format used by the agent for safe, line-number-free edits.

### Purpose
- Make edits resilient to line shifts by anchoring changes with nearby, unique context instead of line numbers.

### Patch Wrapper
- Every patch passed to the editor must be wrapped:
```text
*** Begin Patch
... file sections and hunks ...
*** End Patch
```

### File Section Header
- One file per patch invocation.
- Actions supported:
  - `*** Add File: path/to/file`
  - `*** Update File: path/to/file`

### Change Hunks and Anchors
- Inside an Update section, specify one or more hunks. Hunks may be anchored to scopes using `@@` lines:
  - `@@ class ClassName`
  - `@@ def methodName():` (or language equivalent)
  - You can nest anchors for extra precision:
    - `@@ class ClassName`
    - `@@   def methodName()`
- Within each hunk:
  - Provide 1–3 stable context lines before/after the change when useful.
  - Use `-` for old lines and `+` for new lines.
  - Do not include line numbers.

### Rules
- Keep context minimal but unique; prefer highly distinctive anchors.
- Preserve indentation style (tabs vs spaces) and width of surrounding code.
- If a patch fails due to context mismatch, re-read the file region and update context.
- Deletions of entire files are not part of V4A; use the delete-file operation instead.

### Examples

Add a new file:
```text
*** Begin Patch
*** Add File: src/new_module.ts
export function greet(name: string): string {
  return `Hello, ${name}!`;
}
*** End Patch
```

Update an existing file with anchors and minimal context:
```text
*** Begin Patch
*** Update File: src/huge_file.ts
@@ class DataLoader
@@   public async fetchData():
-    const timeoutMs = 3000
+    const timeoutMs = 5000
@@
*** End Patch
```

Multiple hunks in the same file (separate changes in one update):
```text
*** Begin Patch
*** Update File: src/service.ts
@@ function initClient()
-  const retries = 1
+  const retries = 3
@@
@@ function getBaseUrl()
-  return "https://api.example.com"
+  return getEnvBaseUrl()
@@
*** End Patch
```

That’s all you need to author V4A diffs: a file header (Add/Update), optional `@@` anchors, minimal stable context, and `-`/`+` change lines.

