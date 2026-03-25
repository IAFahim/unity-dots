# Unity DOTS Skills

Claude Code skills for Unity DOTS/ECS development with BovineLabs Core.

## Installation

The skills are loaded from `~/.claude/skills/unity-dots/SKILL.md`.

To install:

```bash
mkdir -p ~/.claude/skills/unity-dots
# Copy SKILL.md to the directory above
```

## Usage

Invoke via `/unity-dots` or it **auto-triggers** for ANY Unity project:
- Files in `Assets/`, `Packages/`, `Scripts/`, `ProjectSettings/`
- Unity file extensions: `.unity`, `.asset`, `.prefab`, `.asmdef`, `.unitypackage`
- C# files importing Unity namespaces (`Unity.Entities`, `UnityEngine`, etc.)
- When you ask about Unity, ECS, DOTS, or BovineLabs

## Documentation Files

- **Core.md** - Unity CLI and basic coding standards
- **SKILL.md** - Complete BovineLabs Core & Ecosystem documentation
- **Anchor.md** - UI Toolkit & MVVM framework
- **Bridge.md** - Managed/DOTS synchronization
- **Gizmo.md** - Debug drawing and visualization
- **Stats.md** - Stats system
- **Timeline.md** - Timeline and animation
- **ScriptStructure.md** - Project structure guidelines

## Key Concepts

### 6-Assembly Architecture

Every feature MUST follow this separation:

1. **Scripts.Data** - Pure ECS Components, unmanaged structs
2. **Scripts** - Stateless extensions, logic, systems
3. **Scripts.Authoring** - MonoBehaviours, Bakers
4. **Scripts.Debug** - Gizmos, visual debuggers
5. **Scripts.Editor** - Editor-only setup
6. **Scripts.Tests** - TDD is mandatory

### Strict Constraints

- **Cyclomatic Complexity = 1** for OnUpdate
- **Zero-allocation** hot paths
- **TryXOut pattern** for returns
- **NEVER write comments** - self-documenting code only

## License

See [LICENSE](LICENSE)
