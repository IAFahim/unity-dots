BovineLabs Core & Ecosystem Master Documentation
1. Architectural Philosophy & Coding Standards

All modules in the BovineLabs ecosystem adhere strictly to Data-Oriented Design (DOD) and Test-Driven Development (TDD) principles.

1.1 The 6-Assembly Architecture

Every feature module MUST strictly follow this 6-assembly separation to ensure clean data flow and minimal compilation times:

Scripts.Data (Pure Types): Pure ECS Components, unmanaged structs. No logic.

Scripts (Logic / Systems): Stateless extensions, systems, pure functions. No Unity setup code.

Scripts.Authoring (Setup): MonoBehaviours, Bakers. Strictly for Editor-to-ECS data conversion.

Scripts.Debug (Tooling): Gizmos, visual debuggers (#if UNITY_EDITOR || BL_DEBUG).

Scripts.Editor (Tooling): Editor-only setup. Do not extend the editor here.

Scripts.Tests (Validation): TDD is mandatory. "Test is your bread and butter."

1.2 Strict Code Constraints

Cyclomatic Complexity (CC) = 1: OnUpdate must have CC=1. Extract all logic into stateless, pure functions or scheduled jobs.

Zero-Allocation: Strict zero-allocation hot paths. Native collections only. Maximum Burst compliance.

TryXOut Pattern: Prefer out parameters over direct returns. Prioritize pure functions.

Self-Documenting Code: NEVER WRITE COMMENTS. Self-document via variable naming, structural redesign, and method extraction.

2. Core ECS Framework & Paradigms
2.1 Facets (IFacet)

A source-generated replacement for Unity's IAspect. Allows you to declare the data you want to pull from an entity, automatically generating Lookup, TypeHandle, and ResolvedChunk accessors.

Features: Supports RefRO/RW, EnabledRefRO/RW, DynamicBuffer, [Singleton], and nested [Facet].

Usage: Define a partial struct implementing IFacet. The source generator handles the rest.

2.2 Entity Commands (IEntityCommands)

A unified interface for entity manipulation across different contexts (EntityManager, EntityCommandBuffer, EntityCommandBuffer.ParallelWriter, and IBaker), preventing code duplication.

Implementations: EntityManagerCommands, CommandBufferCommands, CommandBufferParallelCommands, BakerCommands.

2.3 Life Cycle Management

Provides a unified framework for entity initialization and destruction, automatically propagating destruction through LinkedEntityGroup.

Initialization: InitializeEntity (prefabs) and InitializeSubSceneEntity (subscenes) run in InitializeSystemGroup.

Destruction: DestroyEntity (Enableable) triggers destruction in DestroySystemGroup. DestroyOnDestroySystem handles complex hierarchies.

Timers: DestroyTimer<T> handles automatic delayed destruction.

2.4 Object Management & IDs (IUID)

Provides deterministic ID assignment across machines/builds, replacing standard entity instantiation for network/save safety.

ObjectDefinition: Auto-generated map of ObjectId -> Entity.

ObjectGroup: Collections of Object Definitions for spell targeting, unlocking systems, etc.

ObjectCategories: High-level 32-bit flag categorization.

2.5 States & K-Settings

States: Maps a byte/bitfield to component additions/removals automatically. Supports StateModel, StateFlagModel (multiple active), and History variants (forward/back caching).

K (KSettings): Type-safe, Burst-compatible alternative to Enums and LayerMasks. Defines key-value pairs in ScriptableObjects that convert human-readable strings to raw integers in Burst jobs.

3. High-Performance Collections & Memory
3.1 Dynamic Hash Maps

Reinterprets DynamicBuffer<byte> into fully Burst-compatible, entity-attached HashMaps.

Variants: IDynamicHashMap, IDynamicMultiHashMap, IDynamicHashSet, IDynamicUntypedHashMap, IDynamicPerfectHashMap.

Variable Maps: IDynamicVariableMap supports HashMaps with extra columns (e.g., storing ItemData + Weight array side-by-side).

3.2 Advanced Job Iterators

Custom job types that extend Unity's job system:

IJobForThread: Divides workload across a fixed number of worker threads (giving each thread a contiguous slice).

IJobParallelForDeferBatch: Deferred scheduling with batch processing.

IJob[Parallel]HashMapDefer: High-performance internal bucket iteration for native hash maps.

3.3 Managed-to-Burst Bridging (BurstTrampoline)

Provides a lightweight bridge from Burst-compiled code to managed code using a single payload pointer.

Usage: new BurstTrampoline(&ManagedCallback). Helper extensions like InvokeOut<TOut> pack and unpack the payload automatically without MonoPInvokeCallback boilerplate.

3.4 Pooling & Allocators

PooledNativeList<T>: Thread-safe pooling system for NativeList<T> that reuses memory, zeroing out allocations.

UnmanagedPool<T>, NativeSlabAllocator<T>.

4. Advanced Systems & Subsystems
4.1 SubScenes & Asset Loading

Enhanced control over DOTS SubScenes with world targeting (Game, Service, Client, Server).

Modes: Required Loading (blocking), Wait For Load, Auto Load.

Asset Loading: Synchronously loads and instantiates managed GameObject assets tied to specific DOTS worlds at runtime.

4.2 Pause & Time Management

Granular control over system updates.

Modes: Normal Pause (PauseAll=false - pauses simulation, keeps UI/Input) vs Full Pause (PauseAll=true).

Interfaces: IUpdateWhilePaused (forces system to run), IDisableWhilePaused (forces system to stop).

4.3 Physics Extensions

PhysicsStates: Stateful collision (StatefulCollisionEvent) and trigger (StatefulTriggerEvent) tracking (Enter/Stay/Exit) optimized for parallel processing.

PhysicsUpdate: Automatically rebuilds the physics spatial map when FPS exceeds the fixed timestep, preventing stale raycasts.

4.4 Spatial Mapping

SpatialMap<T> & SpatialKeyedMap<T>: Ultra-fast 2D/3D spatial hashing designed to be rebuilt entirely every frame for broad-phase neighbor queries.

5. System Capabilities (High-Level Frameworks)
5.1 UI Toolkit & MVVM (BovineLabs.Anchor)

High-performance, zero-allocation MVVM framework binding DOTS data to UI Toolkit.

ViewModels: [IsService] auto-registers for DI. Must be partial and inherit SystemObservableObject<T>.

ECS to UI: Use UIHelper<TViewModel, TData> in UISystemGroup to push ECS data to the managed UI layout safely.

UXML: Bind via <Bindings><ui:DataBinding data-source-path="MyProp"/></Bindings>.

5.2 Managed / DOTS Sync (BovineLabs.Bridge)

Strictly isolates managed object interaction to keep ECS simulation purely Burst-compiled.

Pipeline: ECS Data (LightData) -> BridgeCompanionSystem (GameObject Pooling) -> BridgeSyncSystemGroup (BurstTrampoline to Managed).

Audio Pooling: Fixed-size AudioSourcePool. Ambience uses spatial sorting by Priority + Distance; One-Shots auto-return to pool when finished.

Cinemachine: Spawns pooled dummy GameObjects (CMCameraTargetBridge) synced to ECS transforms for Cinemachine to track.

5.3 Event & Action Framework (BovineLabs.Reaction & Essence)

Data-driven condition-action gameplay framework.

Strict Pipeline: Initialization -> Conditions -> Active State -> Actions (Apply/Revert).

Conditions: Evaluates ConditionActive into ConditionAllActive. Supports single-frame Events or persistent States (Essence Stats).

Actions: Must be Reference-Counted. Actions execute on Active=true and revert on Active=false.

5.4 Timeline & Animation (BovineLabs.Timeline)

Bakes Unity PlayableDirector assets into pure ECS entities. Evaluates time, weights, and blended values exclusively in Burst.

Requirement: Target components must implement IAnimatedComponent<T>.

Execution: Calculates cross-fading and overlapping blend weights via TrackBlendImpl<T, TComponent> and applies them in TimelineComponentAnimationGroup.

5.5 Debug Drawing & Gizmos (BovineLabs.Quill)

Zero-allocation, thread-safe debug drawing replacing Debug.DrawLine.

Usage: Obtain via SystemAPI.GetSingleton<DrawSystem.Singleton>().CreateDrawer(). Safe to pass into IJobEntity.

Primitives: Drawer.Line(), Point(), Circle(), Text32(), SolidMesh().

Constraint: Must exist exclusively in Scripts.Debug assemblies wrapped in #if UNITY_EDITOR || BL_DEBUG.

6. Editor & Tooling
6.1 Settings Framework

Creates ScriptableObject-based settings with automatic ECS integration (SettingsSingleton / SettingsBase).

Global singletons initialize automatically before the splash screen and are forcibly included in builds.

World targeting ([SettingsWorld("Client")]) controls which ECS worlds receive which settings via SettingsAuthoring.

6.2 Analyzers

Automatic Roslyn analyzer integration. AnalyzersProjectFileGeneration scans the RoslynAnalyzers directory and injects .dll, .json, and .ruleset configurations into generated .csproj files, enforcing code quality without manual setup.

6.3 Change Filter Tracking

Editor-only tool ([ChangeFilterTracking]) that monitors IComponentData change frequency. Warns via the Change Filter window if components exceed optimal threshold (e.g., >85% update rate), usually indicating broken ref fields or unnecessary writes.

6.4 Inspectors & UI Toolkit Editor

ElementEditor / ElementProperty: Base classes replacing Unity's standard property drawers. Automatically builds VisualElements and provides CreatePropertyField() mapping.

[PrefabElement]: Routes edits directly to the prefab source object when modifying an instance in the scene.

6.5 Terminal Control (unity-cli)

Headless Unity Editor control for CI/CD or rapid tooling.

Integrity: Always run unity-cli reserialize <path> after modifying YAML files.

Execution: Run raw C# via unity-cli exec "<code>" for real-time querying.

Compilation: Always run unity-cli editor refresh --compile after touching .cs files before entering play mode.