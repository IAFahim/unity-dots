# SYSTEM CAPABILITY: UI Toolkit & MVVM (`BovineLabs.Anchor`)
High-performance, MVVM-driven UI framework for Unity DOTS using UI Toolkit. Provides dependency injection, navigation, and zero-allocation Burst-compiled data binding.

## CORE DIRECTIVES & ARCHITECTURE
1. App Bootstrap: The UI lifecycle is driven by a subclass of `AnchorAppBuilder` attached to a `GameObject` alongside a `UIDocument`.
2. Dependency Injection: ViewModels and Services must be decorated with `[IsService]` to be automatically registered. Fetch services via `AnchorApp.Current.Services.GetRequiredService<T>()`.
3. Source Generators: ViewModels MUST be `partial` classes. Use `[ObservableProperty]` on private fields (generates public PascalCase properties) and `[ICommand]` on methods (generates `MethodNameCommand`).
4. ECS / Burst Bridge: Never modify UI elements directly from Burst. Instead, ViewModels inherit from `SystemObservableObject<T>` (where `T` is an unmanaged struct). Burst systems use `UIHelper<TViewModel, TData>` to safely write to this struct.
5. UXML Data Binding: The root `<ui:VisualElement>` in your UXML must declare `data-source-type="Namespace.ViewModel, Assembly"`. Bindings are established using the `<Bindings><ui:DataBinding/></Bindings>` syntax.

## UITK & ANCHOR API REFERENCE (Key Elements & Methods)
*   Safe Area: `<bl:AnchorSafeArea edges="All">` Automatically applies padding for mobile notches/bezels.
*   Navigation: `AnchorApp.Current.NavHost.Navigate("view-key", AnchorNavArgument.String("param", "val"))`
*   Screen Callbacks: ViewModels implement `IAnchorNavigationScreen` to receive `OnEnter(AnchorNavArgument[])` and `OnExit()` callbacks.
*   Option Pager: `<bl:OptionPager>` - A left/right arrow cycle control (alternative to Dropdown).
*   Custom Sliders: `<bl:AnchorTouchSliderFloat>` and `<bl:AnchorTouchSliderInt>` - Touch-friendly sliders.
*   Class Binding: Bind a boolean to a USS class toggle using `<bl:ClassBinding class="my-uss-class" data-source-path="MyBoolProp" />`.

## IMPLEMENTATION PATTERN 1: The ViewModel (ECS-Linked)
Follow this pattern to create a ViewModel that can be safely written to from Burst jobs.

```csharp
using BovineLabs.Anchor;
using BovineLabs.Anchor.MVVM;
using BovineLabs.Anchor.Nav;
using Unity.Collections;

namespace Scripts.UI
{
    [IsService] // Auto-registers for DI
    public partial class PlayerStatsViewModel : SystemObservableObject<PlayerStatsViewModel.Data>, IAnchorNavigationScreen
    {
        // 1. STANDARD MVVM (Managed)
        [ObservableProperty] private bool showDetails; // Generates: public bool ShowDetails { get; set; }

        [ICommand] // Generates: public ICommand ToggleDetailsCommand { get; }
        private void ToggleDetails() => this.ShowDetails = !this.ShowDetails;

        // 2. ECS-LINKED DATA (Unmanaged)
        // The UI binds to these managed accessors, which read from the unmanaged Value struct
        [CreateProperty] public string HealthText => this.Value.HealthText.ToString();[CreateProperty] public float HealthPercent => this.Value.HealthPercent;

        public struct Data
        {
            [SystemProperty] public FixedString64Bytes HealthText;
            [SystemProperty] public float HealthPercent;
        }

        // 3. NAVIGATION CALLBACKS
        public void OnEnter(AnchorNavArgument[] args) { /* Setup when screen opens */ }
        public void OnExit(AnchorNavArgument[] args) { /* Cleanup when screen closes */ }
    }
}
```

## 💻 IMPLEMENTATION PATTERN 2: The Burst System
Follow this pattern to safely push data from ECS/Burst to the UI Toolkit ViewModel.

```csharp
using BovineLabs.Anchor;
using BovineLabs.Anchor.Binding;
using Unity.Burst;
using Unity.Entities;

namespace Scripts.UI
{
    [UpdateInGroup(typeof(UISystemGroup))] // Anchor systems run in UISystemGroup
    public partial struct PlayerStatsUISystem : ISystem, ISystemStartStop
    {
        // The UIHelper bridges Burst to the ViewModel
        private UIHelper<PlayerStatsViewModel, PlayerStatsViewModel.Data> helper;

        public void OnCreate(ref SystemState state)
        {
            state.RequireForUpdate<PlayerTag>(); // Only run if player exists
            
            // Link helper to our ViewModel. The required component acts as a safety gate.
            this.helper = new UIHelper<PlayerStatsViewModel, PlayerStatsViewModel.Data>(
                ref state, ComponentType.ReadOnly<PlayerTag>());
        }

        public void OnStartRunning(ref SystemState state) => this.helper.Bind();
        public void OnStopRunning(ref SystemState state) => this.helper.Unbind();

        [BurstCompile]
        public void OnUpdate(ref SystemState state)
        {
            // 1. Force completion of previous jobs writing to our data
            state.Dependency.Complete();

            // 2. Fetch ECS data
            var playerEntity = SystemAPI.GetSingletonEntity<PlayerTag>();
            var health = SystemAPI.GetComponent<Health>(playerEntity);

            // 3. Format strings in burst
            FixedString64Bytes healthStr = default;
            healthStr.Append(health.Current);
            healthStr.Append('/');
            healthStr.Append(health.Max);

            // 4. Safely push to UI using SetProperty (triggers managed INotifyPropertyChanged)
            ref var binding = ref this.helper.Binding;
            binding.SetProperty(ref binding.HealthText, healthStr, "HealthText");
            binding.SetProperty(ref binding.HealthPercent, health.Current / (float)health.Max, "HealthPercent");
        }
    }
}
```

## IMPLEMENTATION PATTERN 3: The UXML View
Standard layout for binding the UI Toolkit elements to the generated ViewModel properties.

```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements" xmlns:bl="BovineLabs.Anchor.Elements" editor-extension-mode="False">
    <ui:VisualElement data-source-type="HomeViewModel, Assembly-CSharp" style="flex-grow: 1; align-items: center; justify-content: center;">
        
        <!-- 2. Apply Safe Area for mobile devices -->
        <bl:AnchorSafeArea edges="All" style="flex-grow: 1;">
            
            <!-- 3. Bind to Burst-driven ECS properties -->
            <ui:Label text="100/100">
                <Bindings>
                    <ui:DataBinding property="text" data-source-path="HealthText" binding-mode="ToTarget" />
                </Bindings>
            </ui:Label>

            <!-- 4. Bind to Managed Commands -->
            <ui:Button text="Toggle Details">
                <Bindings>
                    <ui:DataBinding property="command" data-source-path="ToggleDetailsCommand" binding-mode="ToTarget" />
                </Bindings>
            </ui:Button>

        </bl:AnchorSafeArea>
    </ui:VisualElement>
</ui:UXML>
```