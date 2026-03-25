## File: Scripts.Authoring/AssemblyInfo.cs
```csharp
using System.Runtime.CompilerServices;
using Unity.Entities;

[assembly: DisableAutoTypeRegistration]

[assembly: InternalsVisibleTo("Scripts.Editor")]
[assembly: InternalsVisibleTo("Scripts.Tests")]
```

## File: Scripts.Authoring/Scripts.Authoring.asmdef
```authoring.asmdef
{
    "name": "Scripts.Authoring",
    "rootNamespace": "",
    "references": [
        "BovineLabs.Core",
        "BovineLabs.Core.Authoring",
        "BovineLabs.Core.Extensions",
        "BovineLabs.Core.Extensions.Authoring",
        "Scripts.Data",
        "Unity.Burst",
        "Unity.Collections",
        "Unity.Entities",
        "Unity.Entities.Graphics",
        "Unity.Entities.Hybrid",
        "Unity.InputSystem",
        "Unity.Mathematics",
        "Unity.Mathematics.Extensions",
        "Unity.Physics",
        "Unity.Transforms"
    ],
    "optionalUnityReferences": [],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": true,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": false,
    "defineConstraints": [
        "UNITY_EDITOR"
    ],
    "versionDefines": [],
    "noEngineReferences": false
}
```

## File: Scripts.Data/AssemblyInfo.cs
```csharp
using System.Runtime.CompilerServices;

[assembly: InternalsVisibleTo("Scripts")]
[assembly: InternalsVisibleTo("Scripts.Authoring")]
[assembly: InternalsVisibleTo("Scripts.Debug")]
[assembly: InternalsVisibleTo("Scripts.Editor")]
[assembly: InternalsVisibleTo("Scripts.Tests")]
```

## File: Scripts.Data/Scripts.Data.asmdef
```data.asmdef
{
    "name": "Scripts.Data",
    "rootNamespace": "",
    "references": [
        "BovineLabs.Core",
        "BovineLabs.Core.Extensions",
        "Unity.Burst",
        "Unity.Collections",
        "Unity.Entities",
        "Unity.Entities.Graphics",
        "Unity.Entities.Hybrid",
        "Unity.InputSystem",
        "Unity.Mathematics",
        "Unity.Mathematics.Extensions",
        "Unity.Physics",
        "Unity.Transforms"
    ],
    "optionalUnityReferences": [],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": true,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": false,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

## File: Scripts.Debug/Scripts.Debug.asmdef
```debug.asmdef
{
    "name": "Scripts.Debug",
    "rootNamespace": "",
    "references": [
        "BovineLabs.Core",
        "BovineLabs.Core.Editor",
        "Scripts",
        "Scripts.Data",
        "Unity.Burst",
        "Unity.Collections",
        "Unity.Entities",
        "Unity.Mathematics",
        "Unity.Transforms",
        "Unity.UIElements",
        "Unity.UIElements.Toolkit"
    ],
    "optionalUnityReferences": [],
    "includePlatforms": [
        "Editor"
    ],
    "excludePlatforms": [],
    "allowUnsafeCode": true,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": false,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}

```

## File: Scripts.Editor/Scripts.Editor.asmdef
```editor.asmdef
{
    "name": "Scripts.Editor",
    "rootNamespace": "",
    "references": [
        "BovineLabs.Core",
        "BovineLabs.Core.Editor",
        "BovineLabs.Core.Extensions",
        "BovineLabs.Core.Extensions.Editor",
        "Scripts",
        "Scripts.Authoring",
        "Scripts.Data",
        "Unity.Burst",
        "Unity.Collections",
        "Unity.Entities",
        "Unity.Entities.Graphics",
        "Unity.Entities.Hybrid",
        "Unity.InputSystem",
        "Unity.Mathematics",
        "Unity.Mathematics.Extensions",
        "Unity.Physics",
        "Unity.Transforms",
        "Unity.UIElements",
        "Unity.UIElements.Toolkit"
    ],
    "optionalUnityReferences": [],
    "includePlatforms": [
        "Editor"
    ],
    "excludePlatforms": [],
    "allowUnsafeCode": true,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": false,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```


## File: Scripts.Tests/AssemblyInfo.cs
```csharp
using Unity.Entities;

[assembly: DisableAutoCreation]
```

## File: Scripts.Tests/Scripts.Tests.asmdef
```tests.asmdef
{
    "name": "Scripts.Tests",
    "rootNamespace": "",
    "references": [
        "BovineLabs.Core",
        "BovineLabs.Core.Extensions",
        "BovineLabs.Testing",
        "Scripts",
        "Scripts.Data",
        "Unity.Burst",
        "Unity.Collections",
        "Unity.Entities",
        "Unity.Entities.Graphics",
        "Unity.Entities.Hybrid",
        "Unity.InputSystem",
        "Unity.Mathematics",
        "Unity.Mathematics.Extensions",
        "Unity.PerformanceTesting",
        "Unity.Physics",
        "Unity.Transforms"
    ],
    "optionalUnityReferences": [
        "TestAssemblies"
    ],
    "includePlatforms": [
        "Editor"
    ],
    "excludePlatforms": [],
    "allowUnsafeCode": true,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": false,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

## File: Scripts/AssemblyInfo.cs
```csharp
using System.Runtime.CompilerServices;

[assembly: InternalsVisibleTo("Scripts.Debug")]
[assembly: InternalsVisibleTo("Scripts.Editor")]
[assembly: InternalsVisibleTo("Scripts.Tests")]
```

## File: Scripts/Scripts.asmdef
```asmdef
{
    "name": "Scripts",
    "rootNamespace": "",
    "references": [
        "BovineLabs.Anchor",
        "BovineLabs.Core",
        "BovineLabs.Core.Extensions",
        "Scripts.Data",
        "Unity.AppUI",
        "Unity.AppUI.MVVM",
        "Unity.AppUI.Navigation",
        "Unity.Burst",
        "Unity.Collections",
        "Unity.Entities",
        "Unity.Entities.Graphics",
        "Unity.Entities.Hybrid",
        "Unity.InputSystem",
        "Unity.Mathematics",
        "Unity.Mathematics.Extensions",
        "Unity.Physics",
        "Unity.Transforms"
    ],
    "optionalUnityReferences": [],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": true,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": false,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

_Project Structure:_
```text
Scripts.Authoring/AssemblyInfo.cs
Scripts.Authoring/Scripts.Authoring.asmdef
Scripts.Data/AssemblyInfo.cs
Scripts.Data/Scripts.Data.asmdef
Scripts.Debug/Scripts.Debug.asmdef
Scripts.Editor/Scripts.Editor.asmdef
Scripts.Editor/SetupExamplesEditor.cs
Scripts.Tests/AssemblyInfo.cs
Scripts.Tests/Scripts.Tests.asmdef
Scripts/AssemblyInfo.cs
Scripts/Scripts.asmdef
```
