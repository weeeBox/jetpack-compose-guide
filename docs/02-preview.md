# Effective Compose Previews

## 1. Core Principles

### 1.1 Preview Stateless Composables
Preview the stateless content, not the stateful screen.
*   **Do**: Preview a `ProfileContent(state: ProfileState)` composable.
*   **Don't**: Attempt to preview `ProfileScreen(viewModel: ProfileViewModel)`. Mocks for ViewModels are heavy and often break preview rendering.

### 1.2 Isolation
Previews should run in isolation, but they often require a surrounding context to render correctly. If your component relies on `CompositionLocal` providers (Theme, Typography, Navigation, or custom Analytics), use a **Preview Harness**.

**Why use a Harness?**
*   **Theming**: Ensures `MaterialTheme` tokens (colors, shapes) are available.
*   **Surface**: Provides a default background and content color (otherwise, text might be invisible on a transparent background).
*   **Dependencies**: Provides "No-op" implementations for required `CompositionLocals` (like a `NavController`) to prevent runtime crashes during preview rendering.

```kotlin
@Composable
fun PreviewHarness(
    content: @Composable () -> Unit
) {
    MyAppTheme {
        // 1. Provide a Surface to set default background/content colors
        Surface(color = MaterialTheme.colorScheme.background) {
            // 2. Provide "Safe" defaults for required CompositionLocals
            CompositionLocalProvider(
                LocalAnalyticsProvider provides NoOpAnalytics,
                // Avoid using actual NavController; use a placeholder if necessary
            ) {
                content()
            }
        }
    }
}
```

## 2. Best Practices

### 2.1 Component "Sticker Sheets"
Dedicate a single file (e.g., `DesignSystemPreviews.kt`) to showcase all your core atoms (Buttons, Text Styles, Colors) in one place.
*   **Benefit**: Serves as "Visual Documentation" for the team.
*   **Organization**: Android Studio groups these previews, providing a quick catalog of available components.

### 2.2 PreviewParameterProvider
Avoid copying and pasting preview functions just to change one piece of data. Use `PreviewParameterProvider` to inject a sequence of data states (Loading, Success, Error, Edge Cases) into a single Preview function.

```kotlin
class UserProvider : PreviewParameterProvider<User> {
    override val values = sequenceOf(
        User(name = "Alice", isOnline = true),
        User(name = "Bob", isOnline = false),
        User(name = "Very Long Name That Might Break Layout", isOnline = true)
    )
}

@Preview(showBackground = true)
@Composable
fun UserCardPreview(
    @PreviewParameter(UserProvider::class) user: User
) {
    PreviewHarness {
        UserCard(user = user)
    }
}
```

### 2.3 Multipreview Annotations
Before creating custom annotations, leverage the standard **Compose Tooling Library** annotations (available in `androidx.compose.ui:ui-tooling-preview`).

*   **`@PreviewScreenSizes`**: Generates previews for reference devices (Phone, Foldable, Tablet).
*   **`@PreviewFontScale`**: Generates previews with various font scales (85% to 200%) to test accessibility.
*   **`@PreviewLightDark`**: Generates two previews: one in Light Mode and one in Dark Mode.
*   **`@PreviewDynamicColors`**: Generates previews using different dynamic color seeds (Android 12+).

#### Custom Multipreview
If the standard set is insufficient, create your own **Multipreview Annotation**. This reduces boilerplate and ensures consistency.

**Example: Device & Font Configuration**
```kotlin
@Preview(name = "Pixel 5", device = "spec:shape=Normal,width=1080,height=2340,unit=px,dpi=440")
@Preview(name = "Small Phone", device = "spec:shape=Normal,width=320,height=640,unit=dp,dpi=320")
@Preview(name = "Large Font", fontScale = 1.5f)
annotation class DeviceConfigPreviews
```

**Pro Tip: Stacking Annotations**
You can apply multiple Multipreview annotations to a single Composable to generate a combinatorial matrix of previews.

```kotlin
@DeviceConfigPreviews
@ThemePreviews // Assuming you defined this for Light/Dark mode
@Composable
fun LoginScreenPreview() {
    PreviewHarness {
        LoginContent(...)
    }
}
```

### 2.4 LocalInspectionMode
Use `LocalInspectionMode.current` to detect if the code is running inside a Preview. This is useful for skipping heavy initialization or logic (like video players or complex animations) that shouldn't run in the IDE preview.

```kotlin
@Composable
fun VideoPlayer(url: String) {
    if (LocalInspectionMode.current) {
        // Render a placeholder box in Preview
        Box(Modifier.background(Color.Black)) { 
            Text("Video Player Placeholder", Color.White) 
        }
    } else {
        // Initialize actual ExoPlayer
        RealVideoPlayer(url)
    }
}
```

## 3. Configuration & Organization

*   **`showBackground = true`**: Essential for components that don't have their own surface color (e.g., transparent text).
*   **`backgroundColor`**: Explicitly set this (e.g., `0xFFFFFFFF`) if `showBackground` defaults to a color that masks your component's issues.
*   **`group`**: Use the `group` parameter to organize previews in the Android Studio design view, making it easier to filter related components.

## 4. Common Pitfalls

### 4.1 "Works on my Machine" (Missing Themes)
**Pitfall**: A component looks fine in Preview but terrible in the app because the Preview didn't apply the AppTheme.
**Fix**: Always wrap previews in your `AppTheme` or a custom `PreviewHarness`.

### 4.2 Previewing `ViewModel` Dependencies
**Pitfall**: Passing a `ViewModel` into a previewable composable.
**Fix**: Hoist state! Refactor the Composable to accept a data class or interface (`State Holder`) instead of the ViewModel itself.

### 4.3 Private Previews
**Pitfall**: Marking `@Preview` functions as `private`.
**Nuance**: While this works for local development, some screenshot testing frameworks and tooling require Previews to be `internal` or `public` to discover and record them. Check your tooling requirements.

## 5. Decision Matrix: How to Preview

| Scenario | Strategy |
| --- | --- |
| **Simple Component** (Button) | Basic `@Preview` with hardcoded args. |
| **Data-Driven Component** (UserCard) | `@PreviewParameter` with a Provider for edge cases. |
| **Screen Level** | `@DevicePreviews` (Multipreview) on the Stateless Content composable. |
| **Dynamic content** (Video/Map) | Use `LocalInspectionMode` to swap with placeholders. |
