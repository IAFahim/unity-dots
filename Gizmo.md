# SYSTEM CAPABILITY: ECS Gizmo & Debug Drawing (`BovineLabs.Quill`)
High-performance, Burst-compiled, thread-safe debug drawing for Unity DOTS. Replaces `UnityEngine.Debug.DrawLine` and `OnDrawGizmos`.

## 🛠 CORE DIRECTIVES & ARCHITECTURE
1. **Zero-Allocation & Burst-Safe:** All drawing is batched via unmanaged native thread streams. The `Drawer` struct is safe to pass into parallel jobs (`IJobEntity`, `IJobChunk`).
2. **Assembly Constraint:** Debug drawing systems MUST exist exclusively in the `Scripts.Debug` assembly.
3. **Compilation Flags:** Wrap all debug drawing systems in `#if UNITY_EDITOR || BL_DEBUG`.
4. **Acquisition:** Obtain the unmanaged `Drawer` via `SystemAPI.GetSingleton<DrawSystem.Singleton>().CreateDrawer()`.
5. **Main Thread Fallback:** For non-job logic (e.g., Editor Scripts), use the static `GlobalDraw.Line(...)` API instead of `Drawer`.

## 📐 SHAPE API REFERENCE (Key Methods)
All methods accept `Color color` and optional `float duration = 0f`.
*   **Line / Lines:** `Drawer.Line(float3 p0, float3 p1, Color c)` | `Drawer.Lines(NativeArray<float3>, Color c)`
*   **Point:** `Drawer.Point(float3 point, float size, Color c)`
*   **Circle:** `Drawer.Circle(float3 center, float3 direction, Color c)`. *(Note: `direction` dictates BOTH the normal and the radius. e.g., `new float3(0, 5, 0)` = flat circle, radius 5).*
*   **Arrow:** `Drawer.Arrow(float3 origin, float3 vector, Color c)` *(Note: `vector` is the offset from origin, not a normalized direction).*
*   **3D Primitives:** `Cuboid`, `Sphere`, `Cylinder`, `Capsule`, `Cone`.
*   **Text (Burst Safe):** `Drawer.Text32(float3 pos, FixedString32Bytes text, Color c, float size)`
*   **Solid Meshes:** `SolidTriangle`, `SolidQuad`, `SolidTriangles` (Requires `BLDrawSolid` material pass).

## 💻 IMPLEMENTATION PATTERN (Standard Job Setup)
Follow this exact structural pattern for all debug visuals. Never mix simulation logic with draw logic.

```csharp
#if UNITY_EDITOR || BL_DEBUG
using BovineLabs.Quill;
using Unity.Burst;
using Unity.Collections;
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;
using UnityEngine;

namespace Scripts.Debug
{
    [UpdateInGroup(typeof(PresentationSystemGroup))][WorldSystemFilter(WorldSystemFilterFlags.LocalSimulation | WorldSystemFilterFlags.Editor)]
    public partial struct DebugEntityTargetingSystem : ISystem
    {
        public void OnCreate(ref SystemState state)
        {
            // Always require the DrawSystem.Singleton
            state.RequireForUpdate<DrawSystem.Singleton>();
        }

        [BurstCompile]
        public void OnUpdate(ref SystemState state)
        {
            // 1. Acquire the thread-safe drawer
            var drawer = SystemAPI.GetSingleton<DrawSystem.Singleton>().CreateDrawer();
            var transformLookup = SystemAPI.GetComponentLookup<LocalTransform>(true);

            // 2. Schedule parallel job, passing the drawer by value
            state.Dependency = new DrawTargetsJob
            {
                Drawer = drawer,
                TransformLookup = transformLookup
            }.ScheduleParallel(state.Dependency);
        }

        [BurstCompile]
        private partial struct DrawTargetsJob : IJobEntity
        {
            public Drawer Drawer;
            [ReadOnly] public ComponentLookup<LocalTransform> TransformLookup;

            private void Execute(in LocalTransform transform, in TargetEntity target) // TargetEntity = hypothetical data
            {
                // Draw radius indicator (direction vector = up, magnitude = radius)
                Drawer.Circle(transform.Position, new float3(0, 3f, 0), new Color(1f, 1f, 0f, 0.2f));

                if (TransformLookup.TryGetComponent(target.Value, out var targetTransform))
                {
                    // Draw tether
                    Drawer.Line(transform.Position, targetTransform.Position, Color.red);
                    // Draw anchor point
                    Drawer.Point(targetTransform.Position, 0.25f, Color.magenta);
                }
            }
        }
    }
}
#endif