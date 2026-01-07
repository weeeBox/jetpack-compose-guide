# Styling & Design Systems

## 1. Material Design 3 Foundations

### 1.1 The Theme Wrapper
Your application should be wrapped in a custom theme function that configures `MaterialTheme` and provides additional overrides.

```kotlin
@Composable
fun MyAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    ...

    // Provide custom systems or overrides
    CompositionLocalProvider(
        LocalSpacing provides Spacing(),
    ) {
        MaterialTheme(
            colorScheme = colorScheme,
            typography = AppTypography,
            shapes = AppShapes,
            content = content
        )
    }
}
```

### 1.2 Semantic Colors
Material 3 separates colors into semantic roles, allowing the UI to adapt automatically to light, dark, and dynamic themes.

#### The Core Palette
*   **Primary**: The source of truth for your brand. Used for high-emphasis components (FAB, active states).
*   **Secondary**: Less dominant than primary. Used for filter chips, tonal buttons.
*   **Tertiary**: A contrasting accent color. Used to break up monotony or draw attention to distinct elements.
*   **Error**: Indicates errors or dangerous actions.

#### Role Variants
For each core color (Primary, Secondary, Tertiary, Error), there are four associated tokens:
1.  **Base** (e.g., `Primary`): The main color.
2.  **On-Color** (e.g., `OnPrimary`): Color for text/icons sitting *on top* of the Base color.
3.  **Container** (e.g., `PrimaryContainer`): A lighter (or tonal) version of the color, often used for filling large areas like cards or navigation rails.
4.  **On-Container** (e.g., `OnPrimaryContainer`): Color for text/icons sitting *on top* of the Container color.

#### Surface & Background
*   **Background / OnBackground**: The scrollable area behind all content.
*   **Surface / OnSurface**: The default color for "Sheets" of material (Cards, BottomSheet, Menus).
*   **SurfaceVariant / OnSurfaceVariant**: A dedicated variation for decorative elements or medium-emphasis text.
*   **Outline**: Used for borders (e.g., OutlinedButton) and dividers.
*   **OutlineVariant**: Softer version for decorative dividers.

#### Scheme Builders
To define these colors, use the builder functions provided by the library. These builders provide sensible default values for any token you don't explicitly override.

*   **`lightColorScheme(...)`**: Creates a scheme optimized for light mode.
*   **`darkColorScheme(...)`**: Creates a scheme optimized for dark mode.
    *   *Note*: Dark mode colors are usually desaturated (pastel) versions of the light mode colors to reduce eye strain and meet contrast standards.

```kotlin
private val LightColors = lightColorScheme(
    primary = Purple40,
    onPrimary = Color.White,
    primaryContainer = Purple90,
    onPrimaryContainer = Purple10,
    // ... other overrides ...
)

private val DarkColors = darkColorScheme(
    primary = Purple80,
    onPrimary = Purple20,
    primaryContainer = Purple30,
    onPrimaryContainer = Purple90,
)
```

#### How Components Use Semantic Colors
Standard Material 3 components are pre-configured to use specific semantic tokens based on their emphasis level.

*   **Primary Components** (e.g., `Button`, `FloatingActionButton`): Use `Primary` for the container and `OnPrimary` for content.
*   **Tonal Components** (e.g., `FilledTonalButton`): Use `SecondaryContainer` for the container and `OnSecondaryContainer` for content.
*   **Surface Components** (e.g., `Card`, `ListItem`): Use `Surface` and `OnSurface`.

**Automatic Content Color Propagation**:
When you use a Material component like `Surface` or `Button`, it calls `contentColorFor(backgroundColor)` internally to find the matching "On" color from your theme (e.g., if background is `primary`, it returns `onPrimary`).

It then uses `CompositionLocalProvider` to provide this color as `LocalContentColor`. Child components like `Text` and `Icon` read `LocalContentColor.current` by default, ensuring they automatically appear in the correct, legible color.

```kotlin
Surface(color = MaterialTheme.colorScheme.primary) {
    // 1. Surface calculates contentColor = onPrimary
    // 2. Surface provides LocalContentColor = onPrimary
    
    // 3. Text reads LocalContentColor.current (onPrimary)
    Text("Automatically readable") 
}
```

### 1.3 Surface Tones & Elevation
In Material3, elevation is often represented by **Surface Tones** (colors) rather than just shadows. As a surface gets "higher" (more elevated), it becomes lighter (in dark mode) or tinted with the primary color.

**Best Practice**: Use `Surface` or `Card` with the `tonalElevation` parameter instead of just `shadow`.

```kotlin
Surface(
    tonalElevation = 2.dp, // Adds a subtle primary tint
    shadowElevation = 2.dp // Adds the drop shadow
) {
    /* Content */
}
```

## 2. Typography & Shapes

### 2.1 Type Scale
Material3 defines a standardized scale:
*   **Display**: Large, short, high-impact text.
*   **Headline**: Markers for primary passages of text.
*   **Title**: Shorter, prominent text.
*   **Body**: Long-form writing.
*   **Label**: Utility text (buttons, captions).

**Best Practice**: Never hardcode `fontSize` or `fontFamily`. Always reference `MaterialTheme.typography`.

```kotlin
Text(
    text = "Dashboard",
    style = MaterialTheme.typography.headlineMedium
)
```

**Overriding Styles**:
If you need a slight variation, use `.copy()`.

```kotlin
Text(
    text = "Subtitle",
    style = MaterialTheme.typography.bodyMedium.copy(
        color = MaterialTheme.colorScheme.onSurfaceVariant,
        fontWeight = FontWeight.Bold
    )
)
```

### 2.2 Shape Scale
Shapes are also tokenized (ExtraSmall, Small, Medium, Large, ExtraLarge).

```kotlin
Button(
    shape = MaterialTheme.shapes.small, // e.g., 4.dp rounded corners
    onClick = { }
) { ... }
```

## 3. Dimensions & Spacing

Avoid hardcoding `dp` values directly in your UI code.

### 3.1 The Spacing Object
As described in [Composable API Design](01-composable-api-design.md), use a centralized `Spacing` object integrated into your theme.

```kotlin
// Do: Use semantic spacing
Modifier.padding(MaterialTheme.spacing.medium)

// Don't: Magic numbers
Modifier.padding(16.dp)
```

### 3.2 Component Sizing
Standard components often have minimum touch target requirements (48.dp).
*   **Buttons**: Default to min height 40.dp, but interactive area is 48.dp.
*   **Icons**: Standard size is 24.dp.

## 4. Component Defaults & Customization

Material3 components provide `Defaults` objects to easily access standard values.

### 4.1 Customizing Colors
Instead of manually setting `background` modifiers on components, use the dedicated `colors` parameter and the component's default mapping function.

```kotlin
Button(
    onClick = { },
    colors = ButtonDefaults.buttonColors(
        containerColor = MaterialTheme.colorScheme.tertiary,
        contentColor = MaterialTheme.colorScheme.onTertiary
    )
) {
    Text("Tertiary Action")
}
```

### 4.2 Content Padding
Components often come with built-in padding. Check the `contentPadding` parameter before applying your own `padding` modifier to avoid double-spacing.

```kotlin
// Adjusting internal padding of a button
Button(
    onClick = { },
    contentPadding = PaddingValues(horizontal = 24.dp, vertical = 8.dp)
) { ... }
```

## 5. Common Pitfalls

### 5.1 Hardcoded Colors
**Pitfall**: Using `Color.Black` or `Color.White`.
**Why it fails**: It breaks Dark Mode. `Color.Black` text disappears on a dark background.
**Fix**: Use `MaterialTheme.colorScheme.onBackground` or `onSurface`.

### 5.2 Ignoring "On" Colors
**Pitfall**: Using `primary` background with `onSurface` text.
**Why it fails**: Poor contrast ratios. If `primary` is dark blue and `onSurface` is black, the text is unreadable.
**Fix**: Always match `primary` with `onPrimary`, `error` with `onError`, etc.

### 5.3 Font Scaling (`sp` vs `dp`)
**Pitfall**: Defining text size in `dp`.
**Why it fails**: Text won't scale if the user changes their system font size setting (Accessibility).
**Fix**: Always use `sp` for text sizes (or better yet, `MaterialTheme.typography`).

### 5.4 Transparency
**Pitfall**: Using `alpha` modifiers on text to make it grey.
**Why it fails**: It can cause blending issues.
**Fix**: Use the specific "Variant" colors provided by Material3, such as `onSurfaceVariant` for secondary text.

```kotlin
// Do
Text(
    text = "Metadata",
    color = MaterialTheme.colorScheme.onSurfaceVariant
)

// Don't
Text(
    text = "Metadata",
    modifier = Modifier.alpha(0.6f)
)
```
