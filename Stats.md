# SYSTEM CAPABILITY: BOVINELABS REACTION & ESSENCE (DOTS)
Data-driven, condition-action gameplay framework. High-performance, Burst-compiled state evaluation and event propagation pipeline.

## 🏗 CORE PIPELINE & DATA FLOW
The Reaction pipeline is strictly sequential. Data flows one way:
Initialization (`Targets`, `LocalTransform`) -> Conditions (`ConditionActive` -> `ConditionAllActive`) -> Active State (`Active`) -> Actions (Apply/Revert).

### ⚠️ CRITICAL ARCHITECTURAL CONSTRAINTS
1. Group Separation: `ActiveDisabledSystemGroup` MUST run BEFORE `ActiveEnabledSystemGroup`. NEVER merge them. Recursive state mutation during deactivation requires this isolation to prevent orphaned entities in nested hierarchies.
2. Ref-Counting Actions: ALL Actions (`ActionTag`, `ActionEnableable`) MUST be reference-counted. A target entity may have multiple reactions applying the same tag/component. Only remove when ref-count hits 0.
3. No Main-Thread Execution: Trigger events and state writes strictly via Burst-compiled `IJob` / `IJobChunk` / `IJobEntity`.
4. Target Context: Reactions operate on a `Targets` context (`Owner`, `Source`, `Target`, `Self`, `Custom0`, `Custom1`). Missing custom targets resolve to `Entity.Null`.

---

## ⚙️ 1. CONDITIONS & ESSENCE INTEGRATION
Conditions are 32-bit masks (`ConditionActive`) evaluated into `ConditionAllActive`. 

A. Event Conditions (Single-Frame)
Triggering: Use `ConditionEventWriter.Trigger(ConditionKey key, int value)`. Automatically flags `EventsDirty`.
Pipeline: Handled by `ConditionEventWriteSystem`. Resets automatically end-of-frame via `ConditionEventResetSystem`.
*Rule:* Events are intermittent. NEVER use accumulation (`ConditionFeature.Accumulate`) on State conditions.

B. State Conditions (Persistent - e.g., Essence Stats/Intrinsics)
Triggering: Call `ReactionUtil.WriteState(subscriber, value, comparisonValues, conditionActives, conditionValues)`. 
Essence Integration: `EssenceComparisonWriteSystem` updates `ConditionComparisonValue`. Stats/Intrinsics evaluate against these dynamically.
*Rule:* States persist until explicitly overwritten.

C. Logic Evaluation
Simple: Bitwise AND of `ConditionActive.Value`.
Composite: AST-based (`ConditionComposite`) evaluating `LogicOperation` (AND/OR/XOR/NOT) via `ConditionAllActiveSystem`. 

---

## ⏱️ 2. ACTIVE STATE MACHINE (16-Case Truth Table)
The `ActiveSystem` resolves `Active` state combining 4 components. 
Duration (`ActiveOnDuration`): Acts as `OR` (Overrides). Reset allows restart.
Cooldown (`ActiveOnCooldown`): Acts as `AND NOT` (Blocks).
Trigger (`ActiveTrigger`): Acts as `AND` (Single-frame gate). Auto-resets in `ActiveTriggerSystem`.
Condition (`ConditionAllActive`): Acts as `AND`.

*Advanced Cooldown:* If `ActiveCooldownAfterDuration` exists, cooldown tracking strictly waits for `ActiveOnDuration` to become disabled before ticking.

---

## 🎬 3. ACTIONS (EXECUTION & TEARDOWN)
Actions execute upon transition (`Active=true, ActivePrevious=false`) and revert on (`Active=false, ActivePrevious=true`).

ActionCreate: Instantiates entities. If `DestroyOnDisabled=true`, pushes to `ActionCreated` & parent's `LinkedEntityGroup`. Reverted by `ActionCreateDeactivatedSystem` which flags `DestroyEntity` and cascades deactivation recursively.
ActionTag: Ref-counted adding of zero-sized component tags. Validated by `ReactionValidationUtil`.
ActionEnableable: Ref-counted enabling of `IEnableableComponent` via `UnsafeEnableableLookup`.

---

## 💻 CODE IMPLEMENTATION PATTERNS

### Raising Events (Standard Pattern)
```csharp
[BurstCompile]
private struct RaiseEventJob : IJobEntity
{
    public ConditionEventWriter.Lookup EventWriters;
    public ConditionKey Key;
    
    private void Execute(Entity publisher, in HitData hit) // Publisher must have EventWriterAuthoring
    {
        if (this.EventWriters.TryGet(publisher, out var writer))
        {
            writer.Trigger(this.Key, hit.DamageAmount); // Value must be != 0
        }
    }
}