## File: Timeline.md
# SYSTEM CAPABILITY: ECS TIMELINE & ANIMATION (`BovineLabs.Timeline`)
High-performance, Burst-compiled Unity Timeline playback, track blending, and animation evaluation for DOTS. Bakes Unity Timeline assets into pure ECS entities and evaluates time, weights, and blended values exclusively in Burst jobs.

## CORE PIPELINE & DATA FLOW
The Timeline pipeline converts `PlayableDirector` and `TimelineAsset` structures into hierarchical DOTS entities.
Data flows: Unity Time -> `ClockData` -> `Timer` -> Clip `LocalTime` & `ClipWeight` -> Blended `MixData<T>` -> Target Component.

### CRITICAL ARCHITECTURAL CONSTRAINTS
1. Execution Order: Custom track evaluation systems MUST be scheduled in `TimelineComponentAnimationGroup`. This ensures timers and clip weights are fully calculated before track data is applied.
2. Interface Requirement: Any ECS component driven by a timeline MUST implement `IAnimatedComponent<T>` where `T` is the unmanaged data type being blended.
3. Playback Control: NEVER destroy timeline entities to stop playback. Toggle the `TimelineActive` (evaluates time) and `TimerPaused` (freezes time) enableable components on the Director entity.
4. Blending Operations: NEVER manually calculate clip overlapping blend weights. Always use `TrackBlendImpl<T, TComponent>` to gather data and `JobHelpers.Blend<T, TMixer>` to resolve cross-fading and overlap logic.
5. Deactivation State: To reset an entity when a timeline finishes, rely on `TimelineActivePrevious` changing from `true` to `false` (or assign `TrackResetOnDeactivate` during baking).

## TIMELINE UPDATE GROUPS
`ScheduleSystemGroup`: Calculates delta time (`ClockUpdateSystem`) and increments active timers (`TimerUpdateSystem`). Handles bounds and playback ranges (Loop, AutoStop, AutoPause).
`TimelineUpdateSystemGroup`: Maps global timer data to individual clip `LocalTime` (`ClipLocalTimeSystem`) and evaluates curve-based weights (`ClipWeightSystem`). Handles Extrapolation.
`TimelineComponentAnimationGroup`: User-defined track systems read weights and apply blended values to target entities here.

## IMPLEMENTATION PATTERN 1: The Animated Data Component (`Scripts.Data`)
Defines the unmanaged data container and implements the required animation interface for the blender.

```csharp
using BovineLabs.Timeline.Data;
using Unity.Entities;

namespace Scripts.Data
{
    public struct ScaleAnimated : IComponentData, IAnimatedComponent<float>
    {
        public float CurrentScale;
        public float Value => this.CurrentScale;
    }

    public struct ScaleClipData : IComponentData
    {
        public float TargetScale;
    }
}
```

## IMPLEMENTATION PATTERN 2: Authoring Clips and Tracks (`Scripts.Authoring`)
Converts Unity Editor Timeline blocks into the pure ECS data components.

```csharp
using BovineLabs.Timeline.Authoring;
using Scripts.Data;
using Unity.Entities;
using UnityEngine;
using UnityEngine.Timeline;

namespace Scripts.Authoring
{
    public class ScaleClip : DOTSClip, ITimelineClipAsset
    {
        public float TargetScale = 1f;

        public ClipCaps clipCaps => ClipCaps.Blending;

        public override void Bake(Entity clipEntity, BakingContext context)
        {
            context.Baker.AddComponent(clipEntity, new ScaleClipData { TargetScale = this.TargetScale });
            context.Baker.AddComponent(clipEntity, new ScaleAnimated { CurrentScale = this.TargetScale });
            context.Baker.AddTransformUsageFlags(context.Binding.Target, TransformUsageFlags.Dynamic);
        }
    }

    [TrackClipType(typeof(ScaleClip))]
    [TrackBindingType(typeof(Transform))]
    public class ScaleTrack : DOTSTrack
    {
        protected override void Bake(BakingContext context)
        {
        }
    }
}
```

## IMPLEMENTATION PATTERN 3: Blending & Applying System (`Scripts`)
Executes the high-performance collection of active clips, handles blending math via `FloatMixer`, and applies the result to the target entity.

```csharp
using BovineLabs.Timeline;
using BovineLabs.Timeline.Data;
using Scripts.Data;
using Unity.Burst;
using Unity.Collections;
using Unity.Entities;
using Unity.Transforms;

namespace Scripts
{[UpdateInGroup(typeof(TimelineComponentAnimationGroup))]
    public partial struct ScaleTrackSystem : ISystem
    {
        private TrackBlendImpl<float, ScaleAnimated> blendImpl;

        [BurstCompile]
        public void OnCreate(ref SystemState state)
        {
            this.blendImpl.OnCreate(ref state);
        }

        [BurstCompile]
        public void OnDestroy(ref SystemState state)
        {
            this.blendImpl.OnDestroy(ref state);
        }

        [BurstCompile]
        public void OnUpdate(ref SystemState state)
        {
            var localTransforms = SystemAPI.GetComponentLookup<LocalTransform>();
            var blendData = this.blendImpl.Update(ref state);

            state.Dependency = new WriteScaleJob
            {
                BlendData = blendData,
                LocalTransforms = localTransforms
            }.ScheduleParallel(blendData, 64, state.Dependency);
        }

        [BurstCompile]
        private struct WriteScaleJob : IJobParallelHashMapDefer
        {
            [ReadOnly]
            public NativeParallelHashMap<Entity, MixData<float>>.ReadOnly BlendData;

            [NativeDisableParallelForRestriction]
            public ComponentLookup<LocalTransform> LocalTransforms;

            public void ExecuteNext(int entryIndex, int jobIndex)
            {
                this.Read(this.BlendData, entryIndex, out var entity, out var mixData);

                var transformRef = this.LocalTransforms.GetRefRWOptional(entity);
                if (!transformRef.IsValid)
                {
                    return;
                }

                transformRef.ValueRW.Scale = JobHelpers.Blend<float, FloatMixer>(ref mixData, transformRef.ValueRO.Scale);
            }
        }
    }
}
```
```