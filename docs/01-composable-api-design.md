# Composable API Design & Design Systems

This document outlines best practices for authoring maintainable, scalable Composable functions and building robust Design Systems, adhering to official API guidelines and Material Design principles.

## 1. Core API Guidelines

### 1.1 Naming Conventions
Compose follows semantic naming rules distinct from standard Kotlin functions.

*   **Noun-Based for UI Elements**: Composables that emit UI (return `Unit`) must be Nouns and PascalCase. They should not contain verbs.
    *   *Do*: `ProfileCard`, `SubmissionButton`
    *   *Don't*: `DrawProfile`, `CreateButton`
*   **Verb-Based for Events/Effects**: Helper composables that perform actions or side effects often use verbs.
    *   *Example*: `BackHandler`, `LaunchedEffect`
*   **Return Values**: Composables that return a value should not use PascalCase unless they are factory functions that `remember` state.
    *   *State factories*: `rememberScrollState()`
    *   *Properties*: `currentComposer`

### 1.2 Parameter Ordering
Standardizing parameter order maximizes usability and consistency across the codebase.

1.  **Required Parameters**: Data necessary to render the UI (e.g., `user: User`).
2.  **Modifier**: A single `modifier: Modifier = Modifier`. This is essential for external layout control.
3.  **Optional Parameters**: Flags, colors, and styling options (with default values).
4.  **Events/Callbacks**: Lambda parameters for interaction (e.g., `onValueChange`, `onClick`).
5.  **Trailing Composable Lambda**: The primary "Slot API" content.

```kotlin
@Composable
fun MyAppButton(
    text: String,                          // 1. Required Data
    modifier: Modifier = Modifier,         // 2. Modifier
    enabled: Boolean = true,               // 3. Optional Config
    onClick: () -> Unit,                   // 4. Events
    leadingIcon: @Composable () -> Unit    // 5. Content slot
) { ... }
```

### 1.3 The Modifier Strategy
Correct handling of the `Modifier` parameter is critical for component reusability.

*   **Mandatory Parameter**: Every UI-emitting Composable must accept a `modifier` parameter.
*   **Default Value**: Provide a default value of `Modifier` (empty).
*   **Root Application**: The passed modifier **must** be applied to the *outermost* root element of the Composable.
*   **Single Application**: Do not reuse the passed modifier instance on children or multiple elements; this leads to unexpected layout behavior.

### 1.4 Scoped Layouts
Layouts like `Row`, `Column`, and `Box` provide specific receiver scopes (`RowScope`, `ColumnScope`, `BoxScope`) to their children.

**Purpose**: To provide context-aware modifiers that only make sense within that specific parent layout (e.g., `Modifier.weight` in `Row`, `Modifier.align` in `Box`).

**Design Decision**: This uses Kotlin's type-safe DSL to prevent invalid layout configurations at compile time. You cannot use `weight` inside a `Box` because `BoxScope` does not define it.

**Real-World Example: Custom Button Bar**
When building a reusable `ActionButtonBar`, exposing `RowScope` allows the caller to decide which buttons should be flexible (using `weight`) and which should be fixed-width, while the component enforces consistent padding and arrangement.

```kotlin
@Composable
fun ActionButtonBar(
    modifier: Modifier = Modifier,
    // Do: Expose RowScope to delegate layout control to the caller
    content: @Composable RowScope.() -> Unit
) {
    Row(
        modifier = modifier
            .fillMaxWidth()
            .padding(16.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        content()
    }
}
```

### 1.5 Modifier Factories
For complex modifier chains that are reused across multiple components, consider creating a **Modifier Factory** extension function. This improves readability, ensures consistency, and keeps your Composable code clean.

```kotlin
fun Modifier.messageBubbleAppearance(
    isUser: Boolean,
    colors: ChatColors,
    shape: Shape
): Modifier = this
    .clip(shape)
    .background(if (isUser) colors.userContainerColor else colors.assistantContainerColor)
    .padding(if (isUser) 12.dp else 0.dp)
```

## 2. Parameter Modeling Strategies

### 2.1 The "Exploded" Approach (Multiple Parameters)
Passing every piece of data as a separate primitive or stable argument.

```kotlin
@Composable
fun ProfileCard(
    name: String,
    age: Int,
    imageUrl: String,
    bio: String,
    isOnline: Boolean,
    modifier: Modifier = Modifier
) { ... }
```

*   **Pros**: 
    *   **Maximum Skippability**: Primitives are easily optimized by the Compose compiler. If one argument changes, only the relevant scopes recompose.
    *   **Decoupling**: The component is not tied to a specific domain model.
*   **Best For**: Low-level "Leaf" components (Buttons, Chips, Cards) where high performance and decoupling are priority.

### 2.2 The Data Class Approach
Passing a single UI-specific data class.

```kotlin
@Immutable 
data class UserState(
    val name: String,
    val bio: String,
    val tags: ImmutableList<String>
)

@Composable
fun ProfileCard(
    state: UserState,
    modifier: Modifier = Modifier
) { ... }
```

*   **Pros**: Clean API surface and easier maintenance when adding fields.
*   **Cons (The Stability Trap)**: If the data class contains unstable types (like standard `List<T>`), the compiler may mark it as **Unstable**, triggering unnecessary recompositions.
*   **The Fix**: Use `@Immutable` or `@Stable` annotations and immutable collections (`ImmutableList`).
*   **Best For**: High-level or "Feature" components that aggregate multiple data points.

### 2.3 The Interface / State Holder Approach
Passing an interface that represents the state and interactions of the component.

```kotlin
@Stable
interface ProfileState {
    val name: String
    val isOnline: Boolean
    fun toggleOnlineStatus()
}

@Composable
fun ProfileCard(
    state: ProfileState,
    modifier: Modifier = Modifier
) { ... }
```

*   **Pros**: Facilitates testing and Previews via fakes; encapsulates complex UI logic.
*   **Best For**: Complex, stateful widgets (e.g., Calendars, Map Controllers) with internal logic.

### 2.4 Decision Matrix

| Component Type / Scenario | Recommendation | Rationale |
| --- | --- | --- |
| **Low-level / Leaf components** (Button, Chip, Card) | Use Multiple Parameters | Maximizes skippability, performance, and decoupling from domain models. |
| **High-level / Feature components** (Screens, Complex Widgets) | Use Aggregated Objects | Reduces API bloat and simplifies state passing across complex hierarchies. |
| Data contains a `List` | Use Immutable Data Class | Requires a wrapper or `ImmutableList` to ensure stability. |
| Component has internal logic | Use Interface / State Holder | Encapsulates complex interactions and state transitions. |

### 2.5 Implementation Best Practice
If utilizing the **Data Class** approach, follow this pattern to ensure stability and performance:

```kotlin
// 1. Define a dedicated UI State object, NOT your database entity
@Immutable 
data class ProfileCardState(
    val name: String,
    val role: String,
    // 2. Use ImmutableList if you have collections
    val badges: ImmutableList<String> = persistentListOf() 
)

@Composable
fun ProfileCard(
    // 3. Pass the stable state object
    state: ProfileCardState, 
    modifier: Modifier = Modifier,
    // 4. Keep event callbacks separate! Don't put lambdas inside the data class.
    onBadgeClick: (String) -> Unit 
) { ... }
```
**Why keep callbacks separate?**
Lambdas do not implement value equality. Placing them inside a `data class` breaks the generated `equals()` method, causing the class to be seen as "different" during every recomposition and triggering unnecessary UI updates.

## 3. Stability and Annotations

### 3.1 The Stability Contract
To be considered "Stable", a type must guarantee:
1.  **Equality**: `equals()` results for two instances will always be the same for the same data.
2.  **Notification**: If a public property changes, Composition is notified (e.g., via `MutableState`).
3.  **Recursion**: All public properties must also be stable.

### 3.2 Annotation Usage
*   **`@Immutable`**: Use this for data classes where all properties are `val` and immutable. It is a stronger promise than `@Stable` and implies that the data *never* changes.
    *   *Usage*: UI State classes, models holding lists.
*   **`@Stable`**: Use this for classes that are mutable but adhere to the notification contract (typically classes holding `mutableStateOf` properties).
    *   *Usage*: State Holders, Controller classes.

### 3.3 Best Practices
1.  **Multi-Module Consistency**: The Compose compiler cannot infer stability across module boundaries. Always annotate UI state classes defined in a separate `domain` or `data` module.
2.  **Immutable Collections**: Prefer `kotlinx.collections.immutable.ImmutableList` over standard `List`. The standard `List` interface is considered unstable because the underlying implementation could be mutable (e.g., `ArrayList`).
3.  **Wrapper Classes**: If you must use an unstable class (e.g., from a third-party library), wrap it in an `@Immutable` data class to "stabilize" it for Compose.

### 3.4 Common Pitfalls
*   **`var` in Data Classes**: Using `var` without `MutableState` backing breaks the notification contract. Compose won't know when the value updates.
*   **Lambdas in Data Classes**: As noted in Section 2.5, lambdas break structural equality. Keep them as separate function parameters.
*   **Standard Collections**: Passing `List<T>` directly to a Composable often disables skipping.

## 4. Design System Components

### 4.1 The "Slot" First Approach
Avoid monolithic components. Use **Slots** to delegate content decisions to the caller while the component controls the layout.

```kotlin
import androidx.compose.material3.ListItem
import androidx.compose.material3.ListItemDefaults
import androidx.compose.material3.MaterialTheme

@Composable
fun StandardListItem(
    headlineContent: @Composable () -> Unit,
    modifier: Modifier = Modifier,
    supportingContent: (@Composable () -> Unit)? = null,
    leadingContent: (@Composable () -> Unit)? = null,
    trailingContent: (@Composable () -> Unit)? = null,
) {
    ListItem(
        headlineContent = headlineContent,
        modifier = modifier,
        supportingContent = supportingContent,
        leadingContent = leadingContent,
        trailingContent = trailingContent,
        colors = ListItemDefaults.colors(containerColor = MaterialTheme.colorScheme.surface)
    )
}
```

### 4.2 Styling Parameters
1.  **Modifier**: Always the first optional parameter.
2.  **Color/Typography/Shape**: Use parameters that default to themed values.

### 4.3 The "Defaults" Object Pattern
Material Compose uses `Defaults` objects (e.g., `ButtonDefaults`) to map theme tokens to component slots and handle state-aware styling. Use this pattern in custom components to maintain consistency and discoverability.

**Optimization Tip**: Annotate functions that only read from `CompositionLocal` (like the theme) with `@ReadOnlyComposable`. This allows the compiler to optimize the calls by skipping them if the read values haven't changed, as no UI nodes are emitted.

```kotlin
object ChatDefaults {
    @Composable
    @ReadOnlyComposable
    fun colors(
        userContainerColor: Color = MaterialTheme.colorScheme.primaryContainer,
        userContentColor: Color = MaterialTheme.colorScheme.onPrimaryContainer,
        assistantContainerColor: Color = Color.Transparent,
        assistantContentColor: Color = MaterialTheme.colorScheme.onSurface
    ): ChatColors = ChatColors(
        userContainerColor = userContainerColor,
        userContentColor = userContentColor,
        assistantContainerColor = assistantContainerColor,
        assistantContentColor = assistantContentColor
    )
}
```

### 4.4 Design Rationale: Wrappers vs. Style Objects
*   **Wrappers (`PrimaryButton`)**: Enforce semantic intent and consistent design "rules" (shape, height) while reducing API surface.
*   **Style Objects**: Best for primitives (e.g., `Text`) where maximum flexibility is required.

## 5. Theming Strategy

### 5.1 Extending MaterialTheme
Build on top of `MaterialTheme` by using its color and typography schemes as the source of truth, creating semantic extensions via `CompositionLocal` only when necessary.

### 5.2 Architectural Mechanics
`MaterialTheme` serves as a facade over **CompositionLocals**.

1.  **Providers**: When you wrap your app in `MaterialTheme(...)`, it calls `CompositionLocalProvider` internally.
    *   `LocalColorScheme` provides the colors.
    *   `LocalTypography` provides text styles.
    *   `LocalShapes` provides corner shapes.
2.  **Consumption**: When you call `MaterialTheme.colorScheme.primary`, you are actually calling `LocalColorScheme.current.primary`.
3.  **Ripple & Content Color**: `MaterialTheme` also sets up the `LocalIndication` (ripple) and `LocalContentColor`. The logic `contentColorFor(color)` works by checking the `LocalColorScheme` to see if the provided background color matches a known token (like `primary`), and if so, returns its pair (`onPrimary`).

### 5.3 Case Study: Adaptive Spacing System
Hardcoding Dp values (e.g., `padding(16.dp)`) makes it difficult to maintain consistency and adapt to different screen sizes. A best practice is to extend `MaterialTheme` with a custom Spacing system that reacts to the Window Size Class.

**1. Define the System**
Create an immutable data class to hold your semantic spacing values and a `CompositionLocal`.

```kotlin
@Immutable
data class Spacing(
    val default: Dp = 0.dp,
    val extraSmall: Dp = 4.dp,
    val small: Dp = 8.dp,
    val medium: Dp = 16.dp,
    val large: Dp = 24.dp,
    val extraLarge: Dp = 32.dp,
    // Semantic names are often better than t-shirt sizes
    val screenMargin: Dp = 16.dp,
    val borderThin: Dp = 0.5.dp
)

val LocalSpacing = staticCompositionLocalOf { Spacing() }

val MaterialTheme.spacing: Spacing
    @Composable
    @ReadOnlyComposable
    get() = LocalSpacing.current
```

**2. Provide Adaptive Values**
Calculate the window size class (using `calculateWindowSizeClass()` or `LocalConfiguration`) and provide the appropriate spacing configuration at the root of your screen.

```kotlin
@Composable
fun AdaptiveRoot() {
    val configuration = LocalConfiguration.current
    val screenWidth = configuration.screenWidthDp.dp

    // Determine Window Size Class
    val windowSizeClass = when {
        screenWidth < 600.dp -> WindowWidthSizeClass.Compact
        screenWidth < 840.dp -> WindowWidthSizeClass.Medium
        else -> WindowWidthSizeClass.Expanded
    }

    // Adaptive spacing configuration
    val spacing = when (windowSizeClass) {
        WindowWidthSizeClass.Compact -> Spacing(screenMargin = 16.dp)
        WindowWidthSizeClass.Medium -> Spacing(screenMargin = 24.dp)
        else -> Spacing(screenMargin = 32.dp, extraLarge = 48.dp)
    }

    CompositionLocalProvider(LocalSpacing provides spacing) {
        // Child composables use MaterialTheme.spacing.screenMargin
        // without knowing the specific Dp value or screen size.
        DashboardContent()
    }
}
```

## 6. Reference Implementation

```kotlin
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text

@Composable
fun InformationCard(
    title: String,
    modifier: Modifier = Modifier,
    icon: ImageVector? = null,
    containerColor: Color = MaterialTheme.colorScheme.surfaceVariant,
    content: @Composable ColumnScope.() -> Unit
) {
    Card(
        modifier = modifier,
        colors = CardDefaults.cardColors(containerColor = containerColor),
        shape = MaterialTheme.shapes.medium
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Row(verticalAlignment = Alignment.CenterVertically) {
                if (icon != null) {
                    Icon(imageVector = icon, contentDescription = null)
                    Spacer(modifier = Modifier.width(8.dp))
                }
                Text(
                    text = title,
                    style = MaterialTheme.typography.titleMedium,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
            Spacer(modifier = Modifier.height(8.dp))
            content()
        }
    }
}
```

### Checklist for Component APIs:
1.  [ ] **Modifier**: Is `modifier: Modifier = Modifier` present?
2.  [ ] **Slots**: Are strict parameters (`title: String`) used only when layout is rigid? Are Slots used for flexibility?
3.  [ ] **Defaults**: Do colors and styles default to `MaterialTheme`?
4.  [ ] **Naming**: Are slots named semantically (`headline`, `supportingText`, `leadingIcon`)?
5.  [ ] **Scopes**: Do slots have receiver scopes (e.g., `RowScope`) if they need layout data?
