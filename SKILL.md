---
name: unity-dots
version: 2.0.0
description: |
  Unity DOTS/ECS development with BovineLabs Core. Enforces strict Data-Oriented Design,
  6-assembly architecture, zero-allocation hot paths, Facets, EntityCommands,
  UI Toolkit MVVM (Anchor), Managed/DOTS sync (Bridge), Timeline, and Quill debug drawing.
  Optimized for safe Unity CLI operation: inspect first, mutate second, verify before claiming success.
allowed-tools:
  - *
---

# SYSTEM CAPABILITY: `unity-cli`
Direct, headless Unity Editor control via terminal.

## CORE DIRECTIVES
0. Run `unity-cli list` once at the start when needed. Then use the smallest tool action that can verify or falsify the current assumption.
1. YAML Integrity: If modifying `.prefab`, `.unity`, `.asset`, or `.mat` via text, you MUST immediately execute `unity-cli reserialize <path>`. Failure causes asset corruption.
2. C# Execution: Use `unity-cli exec` for real-time querying and mutation.
3. Compile Waits: After editing `.cs`, MUST run `unity-cli editor refresh --compile` before entering Play Mode or using `exec` again.
4. Validation: Use `unity-cli console --filter error` or a direct state query after each meaningful change.

## EXECUTION DISCIPLINE
5. Inspect Before Mutate: Before creating, moving, deleting, or editing scene objects, first inspect current state with `unity-cli exec`.
   Required checks when relevant:
   - active scene name and path
   - root GameObjects
   - existing target object by exact name
   - existing target component by exact type
   - current parent/child hierarchy
   Reuse existing objects first. Create only after absence is confirmed.

6. Prefer stdin for Non-Trivial `exec`: If the C# contains quotes, interpolated strings, conditionals, generics, lambdas, namespaces, LINQ, or more than one statement, DO NOT use inline shell quoting.
   Use:
   ```bash
   echo 'Debug.Log("hello"); return null;' | unity-cli exec
   echo 'var go = new GameObject("Marker"); go.tag = "EditorOnly"; return go.name;' | unity-cli exec
   ```
   Treat `unity-cli exec "<complex code>"` as unsafe by default.

7. Fail Fast After Errors: If a command returns a compile error, runtime error, or connection error:
   - do not claim success
   - do not continue with the old assumption
   - do not create extra files as a workaround yet
   First run the smallest possible diagnostic query to identify the exact cause.

8. No Speculative APIs: Do not assume namespaces, component types, API names, menu items, or assembly references exist.
   Probe first with minimal compile-safe checks.
   If a guessed type fails once, stop guessing and inspect assemblies, usings, references, or project code.

9. One Minimal Fix Per Cycle: Apply one fix, compile, then verify. Do not stack multiple guesses into one large edit.

10. Success Must Be Measured, Not Assumed: A task is only working when the requested state is directly verified.
    Valid proof includes:
    - object exists at the intended hierarchy path
    - correct component exists on the correct object
    - compilation succeeds
    - expected menu command executes successfully
    - entity query returns expected count
    - requested runtime/editor state is observed directly

11. Never Claim Done While a Critical Check Is Red:
    - if compilation failed, the edit failed
    - if entity count is `0`, conversion failed
    - if play mode or tool connection failed, runtime verification failed
    State that clearly.

12. Do Not Hide Uncertainty With More Code: If environment state is unclear, inspect first. Do not add editor scripts, debug scripts, helper scripts, or alternate runtime paths just to compensate for unknown state.

## SAFE `unity-cli exec` USAGE
### Good
```bash
unity-cli exec "return UnityEngine.Object.FindObjectsOfType<Camera>().Length;"
unity-cli exec "return Application.dataPath;"
unity-cli exec "return EditorSceneManager.GetActiveScene().name;"
unity-cli exec "return World.All.Count;" --usings Unity.Entities

echo 'var cams = UnityEngine.Object.FindObjectsOfType<Camera>(); return cams.Length;' | unity-cli exec
echo 'var scene = UnityEditor.SceneManagement.EditorSceneManager.GetActiveScene(); return $"{scene.name} | {scene.path}";' | unity-cli exec
```

### Bad
```bash
unity-cli exec "var cam = Camera.main; if (cam != null) { cam.transform.position = Vector3.zero; return 'done'; } else { return 'missing'; }"
```
Reason: shell quoting plus invalid C# string usage creates fake failures and bad follow-up decisions.

## MINIMAL DEBUGGING ORDER
When something fails, debug in this order:
1. inspect current scene state
2. inspect existing relevant objects and components
3. inspect compile errors
4. inspect assembly or asmdef references
5. apply one minimal fix
6. compile
7. verify with a direct query

## DOTS / SUBSCENE RULES
1. Scene-First DOTS Rule: For DOTS/SubScene workflows, inspect the actual scene before creating anything.
2. `SubScene` is not to be guessed. Verify the real type in-project before using it.
3. If a SubScene already exists, reuse it. Do not create another until absence is confirmed.
4. Authoring GameObjects must be placed under the real GameObject that owns the verified SubScene workflow.
5. Manual entity creation is NOT a valid substitute when the requested workflow is authoring + baker + subscene conversion.
6. Do not describe baker or subscene conversion as successful unless entity queries confirm the converted entity exists in the target world.
7. Tests passing is not proof that scene conversion works.

## ASMDEF SAFETY
Before changing asmdef references:
1. inspect the current asmdef first
2. identify the exact missing assembly reference
3. change only the minimal reference set required
4. compile immediately
5. verify the original blocked operation
Do not stack asmdef guesses.

# CODING STANDARDS: UNITY ECS / DOTS
Strict Data-Oriented Design (DOD) and Test-Driven Development (TDD) rules.

## ARCHITECTURE & PHILOSOPHY
Cyclomatic Complexity (CC) = 1: `OnUpdate` must have CC=1. Extract all logic into stateless, pure functions and test them in isolation.
Zero-Allocation: Strict zero-allocation hot paths. Maximum Burst/Job compliance. Native collections only.
DOD & Normalization: Apply database normalization techniques to ECS component design. Explicit data flow. Low abstraction.
No Returns (TryXOut): Methods prefer the `TryXOut` pattern. Use `out` parameters instead of direct returns when it improves data flow clarity.
Separation of Concerns: Data = pure types. Utilities = stateless extensions. Logic = raw C#. Independent concerns must be separated. Never mix Engine or Framework code with Logic in the same file.
Use UIToolkit for UI.

## CODE STYLE CONSTRAINTS
NEVER WRITE COMMENTS. Self-document via naming, structure, and method extraction. If a comment feels needed, refactor until it is not.
Single-Purpose Files: Keep files small. Use extension methods for extracted logic where appropriate.
Magic Numbers: Centralize in refs or constants.
Provide Full Code: When generating code, output complete, functional files, never truncated snippets.
No Decorative Refactors: Do not rewrite unrelated code while fixing a specific issue.

# PROJECT STRUCTURE (6-ASSEMBLY ARCHITECTURE)
Every feature module MUST strictly follow this 6-assembly separation.

`Scripts.Data` (Pure Types)
- Pure ECS Components
- Unmanaged structs only
- No logic

`Scripts` (Logic / Systems)
- Stateless extensions
- Systems
- Pure functions
- No Unity setup code

`Scripts.Authoring` (Setup)
- MonoBehaviours
- Bakers
- Editor-to-ECS data conversion only

`Scripts.Debug` (Tooling)
- Gizmos
- Visual debuggers
- Dev-only diagnostics

`Scripts.Editor` (Tooling)
- Editor-only setup utilities
- No runtime logic
- Do not extend the editor unless the task explicitly requires it

`Scripts.Tests` (Validation)
- TDD is mandatory
- Unit and integration verification
- Tests validate logic, not scene assumptions

## REPORTING STYLE
When reporting progress, use this order:
1. what was inspected
2. what was changed
3. whether compilation succeeded
4. the direct verification result
5. the remaining blocker, if any

Never compress these into a vague success statement.

Check those too, as many features are already provided:
[ScriptStructure.md](ScriptStructure.md)
[Gizmo.md](Gizmo.md)
[Anchor.md](Anchor.md)
[Stats.md](Stats.md)
[Bridge.md](Bridge.md)
