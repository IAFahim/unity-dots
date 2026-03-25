---
name: unity-dots
version: 3.0.0
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
- Run `unity-cli list` once at the start when needed. Then use the smallest tool action that can verify or falsify the current assumption.
- Scene Context First: Before changing any scene object, report the active scene path and whether the target lives in the parent scene or in a referenced subscene asset.
- YAML Integrity: If modifying `.prefab`, `.unity`, `.asset`, or `.mat` via text, you MUST immediately execute `unity-cli reserialize <path>`. Failure causes asset corruption.
- C# Execution: Use `unity-cli exec` for real-time querying and mutation.
- Compile Waits: After editing `.cs`, MUST run `unity-cli editor refresh --compile` before entering Play Mode or using `exec` again.
- Validation: Use `unity-cli console --filter error` or a direct state query after each meaningful change.

## EXECUTION DISCIPLINE
- Inspect Before Mutate: Before creating, moving, deleting, or editing scene objects, first inspect current state with `unity-cli exec`.
  Required checks when relevant:
  - active scene name and path
  - root GameObjects
  - existing target object by exact name
  - existing target component by exact type
  - current parent/child hierarchy
  Reuse existing objects first. Create only after absence is confirmed.

- Prefer stdin for Non-Trivial `exec`: If the C# contains quotes, interpolated strings, conditionals, generics, lambdas, namespaces, LINQ, or more than one statement, DO NOT use inline shell quoting.
  Use:
  ```bash
  echo 'Debug.Log("hello"); return null;' | unity-cli exec
  echo 'var go = new GameObject("Marker"); go.tag = "EditorOnly"; return go.name;' | unity-cli exec
  ```
  Treat `unity-cli exec "<complex code>"` as unsafe by default.

- Fail Fast After Errors: If a command returns a compile error, runtime error, or connection error:
  - do not claim success
  - do not continue with the old assumption
  - do not create extra files as a workaround yet
  First run the smallest possible diagnostic query to identify the exact cause.

- No Speculative APIs: Do not assume namespaces, component types, API names, menu items, or assembly references exist.
  Probe first with minimal compile-safe checks.
  If a guessed type fails once, stop guessing and inspect assemblies, usings, references, or project code.

- One Minimal Fix Per Cycle: Apply one fix, compile, then verify. Do not stack multiple guesses into one large edit.

- Success Must Be Measured, Not Assumed: A task is only working when the requested state is directly verified.
  Valid proof includes:
  - object exists at the intended hierarchy path
  - correct component exists on the correct object
  - compilation succeeds
  - expected menu command executes successfully
  - entity query returns expected count
  - requested runtime/editor state is observed directly

- Never Claim Done While a Critical Check Is Red:
  - if compilation failed, the edit failed
  - if entity count is `0`, conversion failed
  - if play mode or tool connection failed, runtime verification failed
  State that clearly.

- Do Not Hide Uncertainty With More Code: If environment state is unclear, inspect first. Do not add editor scripts, debug scripts, helper scripts, or alternate runtime paths just to compensate for unknown state.

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
- inspect current scene state
- inspect existing relevant objects and components
- inspect compile errors
- inspect assembly or asmdef references
- apply one minimal fix
- compile
- verify with a direct query

## DOTS / SUBSCENE RULES
- Scene-First DOTS Rule: For DOTS/SubScene workflows, inspect the actual scene before creating anything.
- `SubScene` is not to be guessed. Verify the real type in-project before using it.
- If a SubScene already exists, reuse it. Do not create another until absence is confirmed.
- Authoring GameObjects must be placed under the real GameObject that owns the verified SubScene workflow.
- Manual entity creation is NOT a valid substitute when the requested workflow is authoring + baker + subscene conversion.
- Do not describe baker or subscene conversion as successful unless entity queries confirm the converted entity exists in the target world.
- Tests passing is not proof that scene conversion works.

## SUBSCENE CONTEXT INVARIANTS
- Distinguish the two things:
  - Parent scene object: a GameObject in the main scene with `Unity.Scenes.SubScene`
  - Referenced subscene asset: the separate `.unity` scene file pointed to by that component
  Never treat them as the same thing.

- Never verify parent-scene setup from the referenced subscene asset:
  - inspect the active parent scene first
  - find existing `Unity.Scenes.SubScene` GameObjects first
  - verify hierarchy from the parent scene first
  Do not open the referenced subscene asset and then claim the parent scene is correct.

- Do not open a subscene asset with `OpenSceneMode.Single` unless the task explicitly requires editing that asset.
  Opening a referenced subscene asset with `OpenSceneMode.Single` replaces scene context and invalidates parent-scene verification.
  If editing the subscene asset is required:
  - record the previous active scene path first
  - state that context is being switched
  - restore the previous scene after the edit
  - re-verify from the parent scene afterward

- Subscene Editing Rule: Before any SubScene mutation, inspect and report:
  - active scene path
  - whether current scene is the parent scene or the referenced subscene asset
  - existing `Unity.Scenes.SubScene` GameObjects by name
  - the target SubScene GameObject exact name
  - the referenced scene asset path of that SubScene component
  Do not mutate until those facts are known.

- Reuse Existing SubScene Containers:
  - If a `Sub Scene` or existing SubScene GameObject already exists, reuse it.
  - Never create `SpiralSubScene`, `ProperSubscene`, or any other new SubScene container unless absence is explicitly confirmed in the current parent scene.

- No Speculative SubScene Type Lookups:
  - Do not guess `Unity.Entities.SubScene`.
  - Prefer direct compile-safe reference to `Unity.Scenes.SubScene` when available.
  - Otherwise inspect assemblies via `AppDomain.CurrentDomain.GetAssemblies()`.
  - Do not use `someGameObject.GetType().Assembly.GetType("Unity.Scenes.SubScene")` to determine whether the type exists. That checks the wrong assembly.

- No MenuItem Scaffolding to Work Around a Failed Query:
  - If one `unity-cli exec` query fails, do not immediately create Editor helper scripts, menu items, runtime converters, or test tools.
  - First simplify the query, use stdin, remove interpolation and escaping risk, and verify the exact scene context.
  - Only create permanent code if the user asked for permanent code.

- Manual Entity Creation Is Not Proof of SubScene Success:
  - Creating test entities, editor-created entities, or runtime conversion helpers does NOT prove that authoring + baker + subscene conversion works.
  - Those are separate workflows and must not be treated as equivalent.

- Success Criteria for SubScene Tasks:
  - correct parent scene is active or restored
  - intended GameObject exists under the intended existing SubScene container
  - referenced subscene asset linkage is correct
  - compile is clean
  - entering play mode or loading the relevant world succeeds
  - entity query for the authored or baked component returns expected count greater than zero
  If the entity query is `0`, the task is not done.

- Entity Count Zero Is a Hard Failure:
  - `0` matching entities means conversion or load failure until proven otherwise
  - do not say `working`, `ready`, `fully functional`, or `done`
  - do not hide this with tests passing, helper tools, or manual entity creation

- Restore Context After SubScene Edits:
  - save the referenced subscene asset
  - reopen or restore the parent scene
  - verify the `Unity.Scenes.SubScene` object still points to the intended asset
  - verify the authored object exists in the intended place from the parent-scene perspective

- One Failure, One Diagnosis:
  - After two consecutive compile or exec failures caused by quoting, escaping, or context confusion, stop expanding scope.
  - Stop writing new scripts.
  - Print the minimal diagnostic plan.
  - Run only context inspection commands until the exact mistake is located.

## ASMDEF SAFETY
Before changing asmdef references:
- inspect the current asmdef first
- identify the exact missing assembly reference
- change only the minimal reference set required
- compile immediately
- verify the original blocked operation
Do not stack asmdef guesses.

# CODING STANDARDS: UNITY ECS / DOTS
Strict Data-Oriented Design and Test-Driven Development rules.

## ARCHITECTURE & PHILOSOPHY
- Cyclomatic Complexity = 1: `OnUpdate` must have CC=1. Extract all logic into stateless, pure functions and test them in isolation.
- Zero-Allocation: Strict zero-allocation hot paths. Maximum Burst and Job compliance. Native collections only.
- DOD & Normalization: Apply database normalization techniques to ECS component design. Explicit data flow. Low abstraction.
- No Returns (TryXOut): Methods prefer the `TryXOut` pattern. Use `out` parameters instead of direct returns when it improves data flow clarity.
- Separation of Concerns: Data = pure types. Utilities = stateless extensions. Logic = raw C#. Independent concerns must be separated. Never mix Engine or Framework code with Logic in the same file.
- Use UIToolkit for UI.

## CODE STYLE CONSTRAINTS
- NEVER WRITE COMMENTS. Self-document via naming, structure, and method extraction. If a comment feels needed, refactor until it is not.
- Single-Purpose Files: Keep files small. Use extension methods for extracted logic where appropriate.
- Magic Numbers: Centralize in refs or constants.
- Provide Full Code: When generating code, output complete, functional files, never truncated snippets.
- No Decorative Refactors: Do not rewrite unrelated code while fixing a specific issue.

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
- what was inspected
- what was changed
- whether compilation succeeded
- the direct verification result
- the remaining blocker, if any

Never compress these into a vague success statement.

Check those too, as many features are already provided:
- [ScriptStructure.md](ScriptStructure.md)
- [Gizmo.md](Gizmo.md)
- [Anchor.md](Anchor.md)
- [Stats.md](Stats.md)
- [Bridge.md](Bridge.md)
