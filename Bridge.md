# SYSTEM CAPABILITY: BOVINELABS BRIDGE (DOTS <-> MANAGED)
High-performance, Burst-compiled hybrid layer for Unity ECS. Manages GameObject pooling, DOTS-to-Managed state synchronization

## CORE PIPELINE & DATA FLOW
Bridge strictly isolates managed object interaction into specific system groups using `BurstTrampoline` to keep ECS simulation fully Burst-compiled.
Data flows: ECS Data -> BridgeCompanionSystem (Pooling) -> BridgeSync (Trampoline to Managed).

### CRITICAL ARCHITECTURAL CONSTRAINTS
1. Burst-Safe Syncing: NEVER access Managed components (`Light`, `AudioSource`, `Volume`) directly inside `IJobEntity`. Write to ECS data components (`LightData`, `AudioSourceData`); Bridge syncs them to Managed objects via `BridgeSyncSystemGroup` using unsafe function pointers (`BurstTrampoline`).
2. Companion Lifecycle: `BridgeCompanionSystem` handles GameObject pooling automatically. It creates/destroys GameObjects based on the presence of `BridgeType` chunk components versus `BridgeObject`.
3. Transform Sync: Managed `Transform` components are synced from ECS `LocalToWorld` automatically in `BridgeTransformSyncSystemGroup` using `IJobParallelForTransform`. 
4. Cinemachine Targets: Cinemachine cannot track Entities directly. Bridge automatically spawns pooled dummy GameObjects (`CMCameraTargetBridge`) and syncs ECS `LocalToWorld` to them for Cinemachine to follow/look at.


## 1. PIPELINE GROUPS
`BridgeReadSystemGroup` (BeginSimulation): Managed -> ECS. Reads state into ECS early (e.g., `CameraMainSystem` updating `LocalTransform` from `Camera.main`).
`BridgeSimulationSystemGroup` (After Transform): ECS Logic. Calculates priorities, spatial distance sorting, and Audio pooling assignment.
`BridgeTransformSyncSystemGroup` (After Transform): ECS `LocalToWorld` -> Managed `Transform`.
`BridgeSyncSystemGroup` (Presentation): ECS Data -> Managed Components. Burst Trampolines push values to `AudioSource`, `Volume`, `Light`, `Cinemachine`, etc.


## 2. AUDIO POOLING ARCHITECTURE
Audio does not instantiate GameObjects per sound. It uses a strict, fixed-size `AudioSourcePool`. 
Looped (Ambiance): Entities with `AudioSourceEnabled`. `AudioSourceAmbiancePoolSystem` continuously spatial-sorts by `Priority` + distance to `CameraMain`. Only the N closest/highest-priority get a pool index.
One-Shot: Entities with `AudioSourceOneShot`. Triggers when `AudioSourceEnabled` flips true. Automatically returns to pool when playback finishes (`AudioSourceOneShotReturnSystem`).
Music: 2 dedicated pooled slots. Crossfades driven by `MusicState` and `MusicSelection.TrackId`.


## 3. CAMERA & CULLING
Bridge automatically finds `Camera.main` and creates a `CameraMain` entity.
Frustum Culling: `CameraFrustumSystem` writes `CameraFrustumPlanes`. Use `CameraFrustumPlanesPlanesExtensions.AnyIntersect(aabb)` for zero-allocation Culling in jobs.
Jitter/Offset: Write to `CameraViewSpaceOffset.ProjectionCenterOffset` to shift projection matrix (e.g., TAA jitter).


## CODE IMPLEMENTATION PATTERNS

### Playing a One-Shot Audio (Standard Pattern)
```csharp
// To trigger a one-shot, create/configure the entity, enable it, then immediately disable the request to prevent looping/re-triggering.
var audioEntity = ecb.CreateEntity(sortKey, myAudioArchetype); // Archetype needs AudioSourceOneShot, AudioSourceEnabled, etc.
ecb.SetComponent(sortKey, audioEntity, new AudioSourceDataExtended { Clip = myClip, Priority = 128, MaxDistance = 50f });
ecb.SetComponent(sortKey, audioEntity, new LocalToWorld { Value = float4x4.Translate(position) });
ecb.SetComponentEnabled<AudioSourceEnabled>(sortKey, audioEntity, true);
ecb.SetComponentEnabled<AudioSourceEnabled>(sortKey, audioEntity, false); // Required: prevents continuous re-triggering
```

### Frustum Culling in Jobs (Standard Pattern)

```csharp
[BurstCompile]
private struct FrustumCullJob : IJobEntity
{[ReadOnly] public CameraFrustumPlanes Frustum;

    private void Execute(EnabledRefRW<RenderableTag> renderable, in LocalTransform transform, in RenderBounds bounds)
    {
        var aabb = new AABB
        {
            Center = transform.TransformPoint(bounds.Value.Center),
            Extents = bounds.Value.Extents * transform.Scale
        };
        
        renderable.ValueRW = this.Frustum.AnyIntersect(aabb);
    }
}
```
