# VS Code Build Tasks (AMI / Dell / Git Sync)

User-level VS Code tasks for building AMI AptioV BIOS projects and Dell (AGS) projects, and for syncing a Git
workspace with its submodules. They are available in every workspace.

| File | Location | Purpose |
|---|---|---|
| `tasks.json` | `%APPDATA%\Code\User\` | The `Build` task (menu: Build / Git Sync) and the list of required build tool defines |
| `aptio.json` | `%APPDATA%\Code\User\` | Saved settings: build tool paths, VEB → AptioV Tools version, last Dell selections. Created automatically |

## Quick start

1. Open the project folder in VS Code.
2. Press **Ctrl+Shift+B** (or **Terminal → Run Task… → Build**).
3. Pick what to do in the terminal:

   ```
   ---- task ----
   [1] Build *
   [2] Git Sync (current branch + submodules)
   [0] Exit
   Select task (0-2, Enter=1)
   ```

In every menu, **Enter** picks the default (marked `*`, usually your last choice). The task menu and the Dell menus also
accept **0** to exit.

> If the task does not show up or behaves like an old version after `tasks.json` was edited, run
> **Developer: Reload Window**.

## Build

The build process is chosen automatically from the workspace root:

| Workspace root | Build process |
|---|---|
| Has a `*.veb` file | [AMI build](#ami-build) (`make`) |
| No `*.veb` file | [Dell (AGS) build](#dell-ags-build) (`builda.bat`) |

Only real `.veb` files count; backups such as `Project.veb_` are ignored.

### AMI build

1. **Check build tool defines.** Every define listed in `APTIO_REQUIRED` (top of `tasks.json`) must have a value in
   `aptio.json`. Missing or empty ones are asked for in the terminal, one by one:

   ```
   Missing build tool defines in C:\Users\<you>\AppData\Roaming\Code\User\aptio.json: APTIO_TOOLS_ROOT, COX64DIR, ...
   Enter APTIO_TOOLS_ROOT (Enter = C:/AptioV)
   ```

   - Press Enter to accept the suggested default, or type a path (quotes are removed automatically).
   - If a plain path does not exist you are asked `Use it anyway? (y/N)`.
   - The values are saved to `aptio.json` immediately, so you are asked only once.
2. **Select the VEB** (`*.veb` in the workspace root). Skipped when there is only one.
3. **Select the AptioV Tools version** from the `Aptio_5.x_TOOLS_JRE_*` folders under `APTIO_TOOLS_ROOT`.
   The choice is remembered per VEB, so next time it is used automatically:
   `Using remembered AptioV Tools 59_x64 for WCL_NB`.
4. Paths that use placeholders (for example `TOOLS_DIR = {AptioTools}/BuildTools`) are checked now; a missing path only
   shows a yellow warning.
5. **Set the environment.** Every entry in `aptio.json` `env` becomes an environment variable, and `PATH` is built as:

   ```
   <pathPrepend...>;<system PATH>;<pathAppend...>;
   ```

6. **Choose the build action** (`[R] Re-Build  [C] Clean`, default Re-Build): **Re-Build** runs `make rebuild`, waits
   for a key press, then runs `make clean`; **Clean** runs `make clean`.

The Re-Build output and the total build time are logged to `Build.log` in the workspace root.

### Dell (AGS) build

Menus, in order:

| Menu | Options | Notes |
|---|---|---|
| build action | `Release_DRB`, `Debug_DRB`, `Release_DPF` (default), `Debug_DPF`, `All`, `Clean` | `All` = Clean → Release_DRB → Debug_DRB → Release_DPF → Debug_DPF, stops at the first failure |
| platform | `AMD` (default), `Intel` | AMD = `PLATFORM_SILICON_NAME = AGESA_*` in `DellPlatformPkg.dsc` |
| project | Folders `DellPkgs/DellPlatformPkgs/<type>/<Pkg>` that contain `builda.bat` and `DellPlatformPkg.dsc` | Skipped when there is only one. `<type>` becomes `Motherboard_Type` |
| Mount U drive | `Yes`, `No` (default) | `No` adds `-nou` |
| SimNow build | `Yes`, `No` (default) | AMD only. `Yes` adds `-simnow` |
| Dump performance | `Yes`, `No` (default) | `Yes` adds `-dp` |

All choices are remembered per workspace.

For AMD projects the `Build` folder of the matching `AmdCbsPkg` is deleted before building. The `AmdCbsPkg` path is
detected from `PLATFORM_SILICON_NAME`; if it cannot be detected you pick it from a list. The path is remembered per
project.

`builda.bat` arguments per action:

| Action | Arguments | Log |
|---|---|---|
| Release_DRB | `-drb -r -report -SafeAuthBypass` | `Build/<type>/<Proj>DrbPkg/RELEASE_<toolchain>/Build.log` |
| Debug_DRB | `-drb -dda -udc -report -SafeAuthBypass` | `Build/<type>/<Proj>DrbPkg/DEBUG_<toolchain>/Build.log` |
| Release_DPF | `-r -report -SafeAuthBypass` | `Build/<type>/<Proj>/RELEASE_<toolchain>/Build.log` |
| Debug_DPF | `-dda -udc -report -SafeAuthBypass` | `Build/<type>/<Proj>/DEBUG_<toolchain>/Build.log` |
| Clean | `-cleanall` | `Build/Clean.log` |

- `<toolchain>` is read from `TOOL_CHAIN_TAG=` in the build output (`VS2022` if not found).
- For a single action (not `All`), the log is also copied to `<project folder>/Build.log`.
- With SimNow = Yes, `SimNow.bat` is launched after a successful single build.

## Git Sync

Updates the **current branch** (no checkout) and all submodules:

1. `git fetch -p`
2. `git pull origin <current branch>`
3. `git submodule sync --recursive`
4. `git submodule update --init -f --recursive`

It does not check build tools or touch `aptio.json`. On an error (or if the folder is not a Git repository) it shows the
error and waits for a key press.

> `-f` in step 4 discards uncommitted changes inside submodules.

## Configuration

### Required build tool defines (`tasks.json`)

At the top of `tasks.json`:

```jsonc
"options": {
  "env": {
    "APTIO_REQUIRED": "APTIO_TOOLS_ROOT,COX64DIR,COX86DIR,EWDK_DIR"
  }
},
```

- Comma-separated list of defines that must have a value before an AMI build.
- To add one, append it, e.g. `...,EWDK_DIR,CLANG_BIN`. It is asked for on the next build.
- To give a new define a suggested default, add it to the `$suggest = @{ ... }` table in the `Build` command.

| Define | Meaning | Suggested default |
|---|---|---|
| `APTIO_TOOLS_ROOT` | Folder that holds all `Aptio_5.x_TOOLS_JRE_*` tool folders | `C:/AptioV` |
| `COX64DIR` | WinDDK amd64 bin folder | `C:/WinDDK/7600.16385.1/bin/x86/amd64` |
| `COX86DIR` | WinDDK x86 bin folder | `C:/WinDDK/7600.16385.1/bin/x86/x86` |
| `EWDK_DIR` | EWDK folder | `C:/AptioV/Compilers/EWDK_1703` |

### `aptio.json`

Created on the first AMI or Dell build. An existing `.vscode/aptio.json` or `.vscode/.aptio.vebmap` in a workspace is
migrated into it automatically. You can edit it by hand.

```jsonc
{
  "current": "WCL_NB",                        // last selected VEB
  "vebMap": { "WCL_NB": "59_x64" },           // VEB -> AptioV Tools version
  "search": { "toolsPrefix": "Aptio_5.x_TOOLS_JRE_" },
  "env": {                                    // set as environment variables for the AMI build
    "TOOLS_DIR": "{AptioTools}/BuildTools",
    "JAVA_HOME": "{AptioTools}/VisualeBios/zulu/bin",
    "VEB": "{Workspace}/{VEB}",
    "APTIO_TOOLS_ROOT": "C:/AptioV",
    "COX64DIR": "C:/WinDDK/7600.16385.1/bin/x86/amd64",
    "COX86DIR": "C:/WinDDK/7600.16385.1/bin/x86/x86",
    "EWDK_DIR": "C:/AptioV/Compilers/EWDK_1703"
  },
  "pathPrepend": ["JAVA_HOME", "VEB"],        // added BEFORE the system PATH (wins over system tools)
  "pathAppend": ["TOOLS_DIR", "COX64DIR", "COX86DIR", "EWDK_DIR"],  // added AFTER the system PATH
  "dell": {
    "last": { "AGS|Platform|C:\\proj": "AMD" },              // last Dell menu choices per workspace
    "cbsMap": { "FooPkg": "Amd/AGESA_Phx/Phx/AmdCbsPkg" }    // Dell project -> AmdCbsPkg path
  }
}
```

The `{...}` placeholders in `env` values are expanded at build time:

| Placeholder | Value |
|---|---|
| `{Workspace}` | Workspace folder |
| `{VEB}` | Selected VEB name |
| `{Version}` | Selected AptioV Tools version, e.g. `59_x64` |
| `{AptioTools}` | `<APTIO_TOOLS_ROOT>/<toolsPrefix><Version>`, e.g. `C:/AptioV/Aptio_5.x_TOOLS_JRE_59_x64` |

## Common tasks

| I want to… | Do this |
|---|---|
| Use another AptioV Tools version for a VEB | Delete the VEB entry in `aptio.json` `vebMap` (you are asked again), or edit the version |
| Change a build tool path | Edit `aptio.json` `env`, or delete the entry to be asked again |
| Add a required define | Append it to `APTIO_REQUIRED` in `tasks.json` |
| Add an optional environment variable | Add it to `aptio.json` `env` |
| Make a tool folder win over system tools | Move its name from `pathAppend` to `pathPrepend` |
| Re-pick the AmdCbsPkg path for a Dell project | Delete the project in `aptio.json` `dell.cbsMap` |
| Start over | Delete `aptio.json` |

## Notes

- `aptio.json` is shared by all workspaces. `current` is the last VEB selected in any workspace, and `vebMap` is keyed by
  VEB name only, so two projects with the same VEB name share one tools version.
- The Wildcat Lake release notes ask for LLVM 12.0.1 and a `CLANG_BIN` define for the UPL build (path with a trailing
  `/`, e.g. `C:/LLVM-12.0.1/bin/`). To be asked for it, add `CLANG_BIN` to `APTIO_REQUIRED`.

## Editing `tasks.json`

The task commands are PowerShell code stored as JSON string lines. VS Code passes them to `powershell.exe` on the
command line, which has some side effects:

- **Do not use double quotes (`"`)** inside the PowerShell code. They are stripped. Use single quotes, `[char]34`, or
  `\x22` in regular expressions.
- **Runs of spaces inside strings collapse to one space.** For example, `'  '` becomes `' '`. Use `(' ' * 2)` when the
  count matters.
- **Do not write `${...}` in PowerShell code**; VS Code treats it as a variable. `${workspaceFolder}` is intended.
- The `Build` task closes its terminal on exit, so end error paths with `pause` before `exit`.
- The whole `Build` command must stay under the Windows command line limit (~32,000 characters).
