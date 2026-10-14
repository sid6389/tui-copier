# Technical Specification Document
## TUI Multi-File Copier & Operations Utility (`tui-copier`)

**Status:** Draft v3 (pre-implementation)
**Target runtime:** Python 3.11+, `Textual` (pin a tested version in `pyproject.toml`)
**Primary platform:** Windows 10/11 in Windows Terminal (Linux/macOS secondary, see 9.1)

---

## 1. Overview & Goals

`tui-copier` is a keyboard-driven terminal application built with Python and Textual. It lets users browse a source directory tree, select files across multiple subfolders, optionally transform their filenames, and copy or rename them, with safe handling of conflicts.

### 1.1 Goals
* **Deep directory navigation:** browse source subfolders recursively in the terminal.
* **Live search filtering:** fuzzy matching by default, with a regex mode, applied to visible file names as the user types.
* **Cross-folder multi-selection:** one persistent selection set spanning multiple directories. Selecting a folder selects every file inside it, recursively.
* **Action palette:** a bottom-anchored list of file operations, each with a hotkey.
* **Extensible operations:** new operations are added by registering one object, with no UI changes.
* **Destination picker:** browse destinations inside a configured base directory and create new folders on the fly.
* **Safe by default:** preview every operation and prompt on conflicts before touching disk.

### 1.2 v1 Scope
| In scope (v1) | Out of scope (v1) |
| :--- | :--- |
| Copy (as-is) | Move / delete |
| Folder selection (recursive, all files inside) | Zip/archive creation or extraction |
| Fuzzy and regex search filter | Regex *rename* transforms (planned, see 8) |
| Copy & Rename (strip underscores) | Undo / history |
| Rename In-Place (strip underscores) | Mac/Linux-specific polish |
| Per-file conflict prompts | Remote / network paths |
| Config-file-based settings | Compound extensions (`.tar.gz`) |

### 1.3 Non-Goals
* Not a replacement for a full file manager (no permissions editing, no file content preview).
* No background daemon or watch mode.

---

## 2. Configuration & Startup

### 2.1 Config file
Location (first match wins): `--config <path>` CLI flag, then `%APPDATA%\tui-copier\config.toml` on Windows (`$XDG_CONFIG_HOME/tui-copier/config.toml` or `~/.config/tui-copier/config.toml` on Linux/macOS).

```toml
# config.toml
# Use single quotes (TOML literal strings) so backslashes are not escapes
source_root = 'C:\Users\me\projects'      # initial source tree root
destination_base = 'C:\Users\me\out'      # destination picker cannot navigate above this
show_hidden = false                         # hidden files/folders hidden by default
follow_symlinks = false                     # also applies to junctions
search_mode = "fuzzy"                       # "fuzzy" or "regex" (initial mode)
folder_select_warn_threshold = 1000         # confirm before selecting folders with more files
```

### 2.2 Rules
* If no config exists, the app starts with `source_root` = current working directory and shows a blocking error on any destination action: "destination_base is not configured." (It must never default to browsing the whole filesystem.)
* Paths are expanded (`~`, env vars) and resolved at startup. If `source_root` or `destination_base` does not exist or is not readable, show an error screen with the offending path and exit code 2.
* Optional CLI overrides: `--source <path>`, `--dest-base <path>`, `--config <path>`. CLI overrides config.

---

## 3. Architecture & UI

Command Palette & Tree View pattern, split into four zones plus two modal screens.

```
+-----------------------------------------------------------------------+
|  SEARCH & FILTER BAR                                                  |
|  [ Filter: type to filter source files...                           ] |
+-----------------------------------------------------------------------+
|  SOURCE WORKSPACE TREE                                                |
|  v src/                                                               |
|    v templates/                                                       |
|      [X] P_file1_dbs.xslt                                             |
|      [ ] config_v2.json                                               |
|    > output/                                                          |
+-----------------------------------------------------------------------+
|  SELECTION BANNER                                                     |
|  Selected (1): P_file1_dbs.xslt                                       |
+-----------------------------------------------------------------------+
|  ACTION PALETTE                                                       |
|  [1] Copy                                                             |
|  [2] Copy & Rename (strip underscores)                                |
|  [3] Rename In-Place (strip underscores)                              |
+-----------------------------------------------------------------------+
|  [Proceed]                                              [Quit]        |
+-----------------------------------------------------------------------+
```

### 3.1 Components
1. **Filter bar (`Input`):** filters the tree as the user types (debounced ~100 ms). Two modes, toggled with `Ctrl+R` and shown by a mode badge next to the input: **fuzzy** (default) and **regex**. See 3.3.
2. **Source tree (custom `DirectoryTree` subclass):** Textual's `DirectoryTree` has no built-in multi-select, so the subclass renders a `[X]` / `[-]` / `[ ]` marker (selected / partially selected / not selected) per node and toggles membership in the shared selection set. A folder shows `[X]` when all its files are selected and `[-]` when only some are. It overrides `filter_paths` to apply the search filter and hidden-file rules.
3. **Selection banner (`Static`):** bound to the reactive selection set; shows count and the first N names (`+ M more` beyond N=5).
4. **Action palette (`ListView`):** one `ListItem` per registered action showing label, hotkey, and one-line description. Not to be confused with Textual's built-in command palette (`Ctrl+P`), which is left enabled and unrelated.
5. **Destination screen (`ModalScreen`):** a tree rooted at `destination_base`, plus a Create Folder control (see 6).
6. **Preview/confirm screen (`ModalScreen`):** lists every `source -> target` pair, flags conflicts, and requires explicit confirmation (see 5).

### 3.2 Selection State Rules
* Selection is a single ordered set of absolute `Path`s, independent of filtering and folder expansion.
* Filtering hides rows but never changes the selection. Hidden-but-selected files remain selected and counted in the banner.
* The banner indicates when selected files are currently hidden by the filter (e.g. `Selected (4, 2 hidden by filter)`).
* **Folder selection:** pressing Space (or clicking the marker) on a folder selects **every file inside it, recursively**; pressing it again deselects them all. Enter still expands/collapses.
* Folder expansion uses the filesystem at toggle time, respects `show_hidden`, skips symlinks/junctions, and is limited to files that match the **active filter** if one is set (what the user sees is what gets selected).
* If a folder holds more files than `folder_select_warn_threshold`, a confirm prompt shows the count before selecting.
* Files in a folder can still be individually deselected afterwards; the folder then shows `[-]`.
* The selection set stores files only (folders are a UI convenience), so the banner counts are file counts.
* Files that disappear from disk between selection and execution are reported as errors in the result summary (see 9).

### 3.3 Search & Filter
* Matching is case-insensitive and runs against the **file name** (not the full path). Folders stay visible if any descendant file matches.
* **Fuzzy mode:** the query's characters must appear in the file name in order, not necessarily adjacent (subsequence match, e.g. `p1dbs` matches `P_file1_dbs.xslt`). Implemented in pure Python; no extra dependency required. The tree keeps its normal order (no re-ranking).
* **Regex mode:** the query is compiled with Python's `re` (`re.IGNORECASE`, `search` semantics). An invalid pattern shows an inline error under the input and keeps the previous valid filter active; it never crashes or empties the tree.
* An empty query shows everything. `Esc` clears the query.

---

## 4. Operation Registry

New operations are added without touching UI code. Each operation is one registered object.

### 4.1 Interface

```python
from dataclasses import dataclass
from pathlib import Path
from typing import Callable

@dataclass(frozen=True)
class Operation:
    id: str                                   # "copy", "copy_rename", "rename_inplace"
    label: str                                # shown in palette
    description: str
    hotkey: str                               # "1".."9"
    needs_destination: bool                   # False for in-place ops
    transform: Callable[[Path], str]          # source path -> target filename
    execute: Callable[[Path, Path], None]     # (source, resolved_target) -> None, one file
```

* `transform` is a pure function (no disk access), used for both preview and execution.
* `execute` handles exactly one file. The framework owns iteration, conflict resolution, progress, and error collection, so operations do not re-implement them.
* For `needs_destination = False`, the target directory is the source file's own directory.

### 4.2 Pipeline

```
Selected files
  -> transform(path)               -> target filename
  -> resolve target dir            -> destination dir + relative subfolder (or source dir if in-place)
  -> detect conflicts              -> prompt per conflict (section 5)
  -> preview/confirm screen
  -> execute(source, target) per file -> result summary
```

### 4.2.1 Target layout for folder selections
* Files selected **individually** are placed directly in the chosen destination folder.
* Files selected **via a folder** keep their path relative to that folder, **including the folder's own name**: selecting `src/templates` and copying to `D:\out` yields `D:\out\templates\...`. Intermediate subfolders are created as needed.
* The transform applies to **file names only**; folder names are never renamed.
* Rename In-Place renames each file where it lives; no folders are created or moved.

### 4.3 Registered Operations (v1)

#### `copy`: Copy
* `transform`: identity (`path.name`).
* `execute`: `shutil.copy2` (preserves metadata).

#### `copy_rename`: Copy & Rename (strip underscores)
* `transform`: `transform_strip_underscores` (below).
* `execute`: `shutil.copy2` to the transformed target name.

#### `rename_inplace`: Rename In-Place (strip underscores)
* `transform`: `transform_strip_underscores`.
* `execute`: `Path.rename` within the same directory.
* **Destructive:** the confirm screen shows an explicit warning ("Original files will be renamed. This cannot be undone."). There is no undo in v1.

### 4.4 Transformation: `transform_strip_underscores`

**Objective:** remove the text before the first underscore and after the last underscore in the filename stem, keeping the middle.

| Raw filename | Core (middle) | Target |
| :--- | :--- | :--- |
| `P_file1_dbs.xslt` | `file1` | `file1.xslt` |
| `TEMP_data_export_v1.json` | `data_export` | `data_export.json` |
| `prefix_main_suffix.py` | `main` | `main.py` |
| `no_underscore.txt` | n/a (only 2 parts) | `no_underscore.txt` (unchanged) |
| `plain.txt` | n/a (no underscore) | `plain.txt` (unchanged) |
| `a__b.txt` | empty middle | `a__b.txt` (unchanged, see edge cases) |
| `_x_.txt` | `x` | `x.txt` |
| `report_final_v3.zip` | `final` | `final.zip` |
| `backup_data_v1.tar.gz` | `data` | `data.gz` (limitation: only the last suffix is kept as the extension) |

**Edge-case rules**
* The stem is split on `_`. A name needs at least 3 parts to be transformed; otherwise it is returned unchanged.
* **Extensions:** the extension is the final suffix only (`Path.suffix`), e.g. `.zip`, `.xslt`. Compound extensions such as `.tar.gz` are not special-cased in v1.
* **Empty result:** if the middle is empty (e.g. `a__b.txt` gives `""`), the file is returned unchanged and flagged in the preview as "no change".
* **Leading dots:** a leading dot is part of the stem (`.env_local_x` has stem `.env_local_x` and no extension).
* **Windows-invalid results:** if a transformed name is not valid on Windows (see 9.1), the file is flagged as an error in the preview and skipped rather than failing mid-run.
* **Unchanged results:** files whose target equals their source name are shown in the preview as "no change". In-place rename skips them; Copy & Rename copies them under the same name.

**Reference implementation**

```python
def transform_strip_underscores(path: Path) -> str:
    parts = path.stem.split("_")
    if len(parts) >= 3:
        core = "_".join(parts[1:-1])
        if core:
            return f"{core}{path.suffix}"
    return path.name
```

### 4.5 Adding a New Operation
1. Write `transform: Path -> str`.
2. Write `execute: (source, target) -> None`.
3. Add an `Operation(...)` entry to the registry list in `operations/registry.py`. The palette, hotkeys, preview, conflict handling, and summary are picked up automatically.

Hotkeys must be unique; the registry validates this at startup and fails fast on duplicates.

---

## 5. Conflict Handling & Confirmation

### 5.1 Conflict definition
A conflict exists when:
* the resolved target path already exists on disk, or
* two or more selected files resolve to the same target path in the same run (intra-batch collision).

On Windows (and any case-insensitive filesystem), path comparison is **case-insensitive** (`Report.TXT` conflicts with `report.txt`). A case-only in-place rename of the same file (`File.txt` to `file.txt`) is not a conflict.

### 5.2 Per-file prompt
For each conflict, in order, a modal asks:

| Choice | Behavior |
| :--- | :--- |
| **Overwrite** | Replace the existing target. |
| **Skip** | Leave this file alone; record as skipped. |
| **Rename** | Prompt for a new filename (pre-filled with a suggested non-colliding name like `file_1.ext`); re-validated against conflicts. |
| **Apply to all remaining** (checkbox) | Reuses the chosen Overwrite/Skip for all later conflicts in this run. Not offered for Rename. |
| **Cancel run** | Abort before any file is processed if no execution has begun; otherwise stop after the current file. |

* For intra-batch collisions, the first file in selection order claims the name; later ones are treated as conflicts.
* In-place rename: a target that equals the source (no change) is not a conflict.
* Overwriting is never done silently: even with "Apply to all", the preview screen listed the conflicts beforehand.
* Overwriting a directory with a file (or vice versa) is refused and recorded as an error.

### 5.3 Preview/confirm screen
Shown after conflict resolution and before any disk write. It lists each `source -> target` row with a status (`new`, `overwrite`, `skip`, `no change`), the totals, and `Confirm` / `Back` buttons. Rename In-Place shows the destructive warning here.

---

## 6. Destination Selection

* The destination screen shows a `DirectoryTree` (directories only) rooted at `destination_base` (on Windows, a single drive-qualified path; drive switching is not offered). The user cannot navigate above it; symlinks/junctions that resolve outside it are not followed.
* Keys: `Up`/`Down` navigate, `Enter` expands/collapses, `s` (or the `Select` button) confirms the highlighted directory, `Esc` goes back.
* **Create Folder:** key `n` (or the `New Folder` button) opens an input for a folder name, created inside the highlighted directory. Names are validated: non-empty, no path separators, none of `< > : " | ? *`, not `.`/`..`, no trailing space or dot, not a reserved Windows name (`CON`, `NUL`, `COM1`, etc.), and not already existing (compared case-insensitively). Failure shows an inline error and keeps the dialog open. On success, the tree refreshes and the new folder is highlighted.
* Operations with `needs_destination = False` skip this screen entirely.

---

## 7. Keyboard Navigation & Interactions

| Key / Action | Context | Result |
| :--- | :--- | :--- |
| Type text | Filter bar | Filters visible files (case-insensitive substring). |
| `Esc` | Filter bar | Clears the filter and returns focus to the tree. |
| `Up` / `Down` | Source tree | Move highlight. |
| `Enter` | Source tree (folder) | Expand / collapse. |
| `Space` / Click | Source tree (file or folder) | Toggle selection (a folder toggles all files inside it). |
| `Ctrl+A` | Source tree | Select all currently visible (filtered) files. |
| `Ctrl+D` | Any | Clear selection. |
| `/` | Source tree | Focus filter bar. |
| `Ctrl+R` | Filter bar | Toggle fuzzy / regex mode. |
| `Tab` / `Shift+Tab` | App | Cycle focus: filter, tree, palette, buttons. |
| `Up` / `Down` | Action palette | Highlight an operation. |
| `Enter` | Action palette | Choose the operation and proceed. |
| `1`-`9` | Tree or palette focus only | Choose the operation by hotkey. **Disabled while the filter bar has focus** so it does not clash with typing. |
| `Enter` / Click | Proceed button | Continue to destination (or straight to preview for in-place ops). |
| `q` / `Ctrl+C` | Tree or palette focus | Quit (confirm if a run is in progress). |

Proceed is disabled with an inline hint when the selection is empty or no operation is chosen.

---

## 8. Future Considerations (not v1)
* Regex-based find/replace transform with live preview.
* Move operation and undo log.
* Directory selection (copy whole folders).
* Saved selection sets.

---

## 9. Execution, Errors & Results

* **Execution** runs in a Textual worker thread so the UI stays responsive. A progress bar shows `n / total` and the current file name.
* **Cancel:** `Esc` during a run stops after the current file; completed files are not rolled back (documented in the result summary).
* **Errors per file** (permission denied, source missing, disk full, etc.) are collected, not fatal: processing continues with remaining files.
* **Result summary screen** shows counts for copied/renamed, skipped, and failed, with a scrollable list of failures and their reasons. Selection is cleared for successfully processed files and retained for failed ones.
* **Symlinks/junctions:** not followed by default (`follow_symlinks = false`); a selected link is skipped with a warning.
* **Hidden files:** hidden by default; toggle with the `h` key and the config setting `show_hidden`. "Hidden" means the filesystem hidden attribute on Windows and a leading dot on Linux/macOS.
* **Large directories:** tree children load lazily on expand (Textual default); the filter applies only to loaded nodes and expands matching branches on demand.
* **Logging:** an optional log file (`~/.local/state/tui-copier/tui-copier.log` on Linux/macOS) records each operation, target, and result (`%LOCALAPPDATA%\tui-copier\tui-copier.log` on Windows).

### 9.1 Windows Considerations
* **Long paths:** paths over 260 characters can fail. Use `\\?\`-prefixed paths internally (via `os.fspath` helper) and show a clear error if the OS still refuses.
* **Locked files:** a file open in another program raises `PermissionError`/sharing violation; it is reported per file (with a "close the file and retry" hint) and does not abort the run.
* **Invalid/reserved names:** transformed and user-typed names are validated against Windows rules (invalid characters `< > : " | ? *`, reserved names like `CON`/`NUL`/`COM1`, trailing dot or space) before the preview.
* **Case-insensitivity:** all conflict detection and name comparison is case-insensitive on Windows.
* **Read-only files:** copying a read-only file succeeds; the read-only attribute is preserved by `copy2`. Overwriting a read-only target prompts a clear error instead of silently changing attributes.
* **Terminal:** target Windows Terminal. The legacy `cmd.exe` console is not a supported target (limited key and color support). Keybindings avoid combinations that Windows terminals commonly swallow (e.g. `Ctrl+Enter`).
* **Packaging:** distribute via `pipx`/`uv tool install` and/or a PyInstaller single-file `.exe`.

---

## 10. Project Structure

```
tui-copier/
├── pyproject.toml
├── README.md
├── src/tui_copier/
│   ├── __main__.py            # CLI entry point, arg parsing
│   ├── app.py                 # Textual App, screens wiring
│   ├── config.py              # TOML loading, validation
│   ├── state.py               # Selection set, reactive state
│   ├── operations/
│   │   ├── registry.py        # Operation dataclass + OPERATIONS list
│   │   ├── transforms.py      # transform_strip_underscores, etc.
│   │   └── executors.py       # copy/rename execute functions
│   ├── conflicts.py           # Conflict detection & resolution logic
│   └── screens/
│       ├── main.py
│       ├── destination.py
│       ├── preview.py
│       ├── conflict.py
│       └── results.py
└── tests/
```

---

## 11. Testing

* **Unit tests (pytest):** `transform_strip_underscores` (every row in the section 4.4 table plus edge cases), conflict detection (including case-insensitive matching), fuzzy and regex filter logic (including invalid regex), folder expansion, Windows name validation, config loading and validation, registry hotkey uniqueness.
* **Filesystem tests:** use `tmp_path` fixtures for copy, rename, overwrite, skip, and intra-batch collisions.
* **UI tests:** Textual's `App.run_test()` pilot for selection toggling, filtering with hidden-selected files, the hotkey/filter-focus clash, and the full copy flow.
* **CI:** run the suite on `windows-latest` (primary) and `ubuntu-latest`.
* Target: transforms, conflict logic, and filtering fully covered; UI covered for the main flows.

---

## 12. Acceptance Criteria (v1)

1. User can select files across at least two different subfolders, and a whole folder (all files inside, recursively), and see an accurate banner count.
2. Filtering (fuzzy and regex) never removes existing selections, hidden selected files are indicated, and an invalid regex never breaks the tree.
3. Copy preserves content and metadata and creates no changes before confirmation.
4. Copy & Rename and Rename In-Place produce exactly the targets in the section 4.4 table.
5. Every conflict triggers a per-file Overwrite/Skip/Rename prompt, including collisions within the same batch.
6. The destination screen cannot navigate above `destination_base`; Create Folder rejects invalid names.
7. A per-file failure never aborts the remaining files, and the result summary lists all failures.
8. A missing or invalid config produces a clear error rather than a crash or unrestricted browsing.
9. On Windows, conflicts are detected case-insensitively, invalid/reserved names are caught in the preview, and locked files are reported per file without aborting the run.
10. Copying a folder preserves its subfolder structure under the destination.
