# Layouts & Surfaces

## 1. Core Layout Primitives

### 1.1 Row & Column

#### Built-in Spacing
Avoid manually placing `Spacer` composables between every item. Use `Arrangement.spacedBy()` for consistent, clearer code.

```kotlin
// Don't: Manual spacers clutter the code
Column {
    Text("Title")
    Spacer(Modifier.height(8.dp))
    Text("Subtitle")
    Spacer(Modifier.height(8.dp))
    Text("Body")
}

// Do: Use Arrangement for uniform spacing
Column(
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    Text("Title")
    Text("Subtitle")
    Text("Body")
}
```
> For more details, see [Official Documentation: Standard Layouts](https://developer.android.com/develop/ui/compose/layouts/basics).

#### Weight Distribution
Use the `weight` modifier (available in `RowScope` and `ColumnScope`) to distribute remaining space dynamically.

```kotlin
Row(modifier = Modifier.fillMaxWidth()) {
    // Takes up as much space as it needs
    Icon(Icons.Default.Menu, contentDescription = null)
    
    // Takes up all remaining space
    Text(
        "Page Title",
        modifier = Modifier.weight(1f),
        textAlign = TextAlign.Center
    )
    
    // Takes up as much space as it needs
    Icon(Icons.Default.Settings, contentDescription = null)
}
```

#### Intrinsic Measurements
A common challenge is making a child match the height of the tallest element in a Row (e.g., a vertical Divider). By default, Composables measure themselves independently. Use `IntrinsicSize` to force a measurement pass.

**Scenario**: You want a vertical divider between two texts to stretch to the height of the taller text.

```kotlin
// Apply IntrinsicSize.Min to the parent Row
Row(modifier = Modifier.height(IntrinsicSize.Min)) {
    Text("Short", Modifier.weight(1f))
    
    // Divider now knows "Min" height means "height of tallest sibling"
    VerticalDivider(color = Color.Black)
    
    Text("Very\nTall\nText", Modifier.weight(1f))
}
```

#### Fill Max Width/Height
*   Inside a **Column**, use `fillMaxWidth()` to make children stretch horizontally.
*   Inside a **Row**, use `fillMaxHeight()` to make children stretch vertically (often requires fixed height on parent).

### 1.2 Box
`Box` stacks elements on top of each other (Z-axis). It is equivalent to a `FrameLayout`.

*   **Alignment**: The `contentAlignment` parameter sets the default position for all children.
*   **Scope Modifiers**: Use `Modifier.align()` to override the position for specific children.

#### Match Parent Size
As detailed in the [Modifiers Guide](08-modifiers.md), use `Modifier.matchParentSize()` if a child needs to be as large as the parent *without* affecting the parent's size calculation.

```kotlin
Box {
    Text("Overlay Text", modifier = Modifier.align(Alignment.Center))
    
    // Background image scales to fit the text, not the screen
    Image(
        painter = painterResource(id = R.drawable.bg),
        contentDescription = null,
        modifier = Modifier.matchParentSize()
    )
}
```

## 2. Material Surfaces

### 2.1 The `Surface` Composable
`Surface` is the fundamental canvas of Material Design. It is more powerful than a simple `Box` because it handles:

1.  **Color**: Sets the background color.
2.  **Content Color**: Automatically provides the correct `LocalContentColor` (e.g., setting `color = Primary` makes text `OnPrimary`).
3.  **Shape**: Clips content to a shape.
4.  **Border**: Draws a border.
5.  **Elevation**: Handles both shadow elevation and **tonal elevation** (color overlays in MD3).

#### Prefer Surface over Box
When building UI elements that represent a "material" object, use `Surface` instead of a `Box` with manual modifiers.

```kotlin
// Don't: Manually re-implementing Material rules
Box(
    modifier = Modifier
        .clip(RoundedCornerShape(8.dp))
        .background(MaterialTheme.colorScheme.primary)
        .border(1.dp, MaterialTheme.colorScheme.outline)
) {
    // You also have to manually set the text color!
    CompositionLocalProvider(LocalContentColor provides MaterialTheme.colorScheme.onPrimary) {
        Text("Button-like")
    }
}

// Do: Use Surface
Surface(
    shape = MaterialTheme.shapes.small,
    color = MaterialTheme.colorScheme.primary,
    border = BorderStroke(1.dp, MaterialTheme.colorScheme.outline)
) {
    // Text color is automatically correct (onPrimary)
    Text("Button-like")
}
```

### 2.2 Handling Interactions
`Surface` has overloads that accept `onClick`, `onLongClick`, etc. Using these overloads ensures that accessibility events and ripple effects are handled correctly (and clipped to the Surface's shape).

```kotlin
Surface(
    onClick = { /* ... */ },
    shape = CircleShape,
    color = MaterialTheme.colorScheme.surfaceContainerHigh
) {
    Icon(Icons.Default.Add, contentDescription = "Add")
}
```

## 3. Cards

### 3.1 Card Types

*   **`Card` (Filled)**: Low emphasis. No shadow, just a filled background color. Good for separating content sections.
*   **`ElevatedCard`**: Medium emphasis. Uses a shadow and tonal elevation. Good for draggable or floating items.
*   **`OutlinedCard`**: High emphasis boundary. Transparent background with a border. Good for secondary content that needs visual containment.

```kotlin
Column(verticalArrangement = Arrangement.spacedBy(16.dp)) {
    // Filled
    Card {
        Text("Filled Card Content", modifier = Modifier.padding(16.dp))
    }
    
    // Elevated
    ElevatedCard {
        Text("Elevated Card Content", modifier = Modifier.padding(16.dp))
    }
    
    // Outlined
    OutlinedCard {
        Text("Outlined Card Content", modifier = Modifier.padding(16.dp))
    }
}
```

### 3.2 Best Practices

#### Clickable Cards
Do not add `Modifier.clickable` to a Card if possible. Use the `onClick` parameter provided by the Card composable API. This ensures the ripple effect is drawn *over* the card content and respects the card's shape and state.

```kotlin
// Do
ElevatedCard(
    onClick = { navigateToDetails() },
    modifier = Modifier.fillMaxWidth()
) {
    // ...
}
```

#### Inner Padding
Cards do not apply padding to their content by default. You must apply padding to the inner layout (usually a Column) or the content itself.

```kotlin
Card {
    Column(
        modifier = Modifier.padding(16.dp) // Essential!
    ) {
        Text(title, style = MaterialTheme.typography.titleMedium)
        Text(body, style = MaterialTheme.typography.bodyMedium)
    }
}
```

### 4. Common Pitfalls

#### The "Scrollable Column" Trap
`Column` renders all its children at once and does not scroll. If content might exceed the screen height:
*   **Small, fixed content**: Use `Column(Modifier.verticalScroll(rememberScrollState()))`.
*   **Lists/Grids**: Use `LazyColumn` or `LazyVerticalGrid`.

#### Deep Nesting
Avoid unnecessary nesting of Boxes and Columns. Each layer adds complexity to the layout phase.

*   *Bad*: `Box { Column { Row { Text } } }` (just to add padding or background).
*   *Fix*: Use `Modifier.padding()` and `Modifier.background()` on a single element where possible.
