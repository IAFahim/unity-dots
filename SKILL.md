---
name: unity-dots
version: 1.0.0
description: |
  Unity DOTS/ECS development with BovineLabs Core. Enforces strict Data-Oriented Design,
  6-assembly architecture, zero-allocation hot paths, Facets, EntityCommands,
  UI Toolkit MVVM (Anchor), Managed/DOTS sync (Bridge), Timeline, and Quill debug drawing.
allowed-tools:
  - *
---

# SYSTEM CAPABILITY: `unity-cli`
Direct, headless Unity Editor control via terminal.

## CORE DIRECTIVES
0. Run `unity-cli list` to see available tools. Use those tools as much as possible.
1. YAML Integrity: If modifying `.prefab`, `.unity`, `.asset`, or `.mat` via text, you MUST immediately execute `unity-cli reserialize <path>`. Failure causes asset corruption.
2. C# Execution: Use `unity-cli exec "<code>"` for real-time querying/mutation. Full `UnityEngine`/`UnityEditor` access. 
### Example: 
```
unity-cli exec "return UnityEngine.Object.FindObjectsOfType<Camera>().Length;" // 'Object' is an ambiguous reference between 'UnityEngine.Object' and 'object' So use 'UnityEngine.Object' instead of 'Object'
unity-cli exec "return Application.dataPath;"
unity-cli exec "return EditorSceneManager.GetActiveScene().name;"
unity-cli exec "return World.All.Count;" --usings Unity.Entities

# Pipe via stdin to avoid shell escaping issues
echo 'Debug.Log("hello"); return null;' | unity-cli exec
echo 'var go = new GameObject("Marker"); go.tag = "EditorOnly"; return go.name;' | unity-cli exec
```
3. Compile Waits: After editing `.cs`, MUST run `unity-cli editor refresh --compile` before entering Play Mode or using `exec`.
4. Validation: Use `unity-cli console --filter error` to verify actions.


# CODING STANDARDS: UNITY ECS / DOTS
Strict Data-Oriented Design (DOD) and Test-Driven Development (TDD) rules.

## ARCHITECTURE & PHILOSOPHY
Cyclomatic Complexity (CC) = 1: `OnUpdate` must have CC=1. Extract all logic into stateless, pure functions and test them in isolation.
Zero-Allocation: Strict zero-allocation hot paths. Maximum Burst/Job compliance. Native collections only.
DOD & Normalization: Apply database normalization techniques to ECS component design. Explicit data flow. Low abstraction.
No Returns (TryXOut): Methods prefer the `TryXOut` pattern. Use `out` parameters instead of direct returns. Prioritize pure functions.
Separation of Concerns: Data = pure types. Utilities = stateless extensions. Logic = raw C#. Independent concerns must be separated. Never mix Engine/Framework code with Logic in the same file.
Use UIToolkit for UI.

## CODE STYLE constraints
NEVER WRITE COMMENTS. Self-document via variable naming, structural redesign, and method extraction. If a comment feels needed, refactor until it isn't.
Single-Purpose Files: Small files. Extension methods for all logic. 
Magic Numbers: Centralize in refs/constants.
Provide Full Code: When generating code, output complete, functional files, never truncated snippets.


# PROJECT STRUCTURE (6-ASSEMBLY ARCHITECTURE)
Every feature module MUST strictly follow this 6-assembly separation. 

`Scripts.Data` (Pure Types) Pure ECS Components, unmanaged structs.

`Scripts` (Logic / Systems) Stateless extensions, logic, systems, pure functions. No Unity setup code.

`Scripts.Authoring` (Setup) MonoBehaviours, Bakers. Strictly for Editor-to-ECS data conversion.

`Scripts.Debug` (Tooling) Gizmos, visual debuggers.

`Scripts.Editor` (Tooling) Only for setup. Do not extent the editor.

`Scripts.Tests` (Validation) TDD is mandatory. "Test is your bread and butter."

Check those, as many features are already provided:
[ScriptStructure.md](ScriptStructure.md)
[Gizmo.md](Gizmo.md)
[Anchor.md](Anchor.md)
[Stats.md](Stats.md)
[Bridge.md](Bridge.md)