# Performance Optimization

Compose uses a "Stability" system to determine if it can skip recomposition. Understanding this is key to smooth UI.

## 1. Smart Recomposition Strategies

### 1.1 Deferring State Reads
Use lambda modifiers (e.g., `Modifier.offset { IntOffset(x, y) }`) to move state reading to the layout or drawing phase, skipping the composition phase entirely.

### 1.2 Derived State (`derivedStateOf`)
Use `derivedStateOf` to throttle updates when a state changes more frequently than you need to react to. A classic example is observing scroll position to determine if a "Scroll to Bottom" button should be visible or if we are at the bottom of the list.

```kotlin
val listState = rememberLazyListState()

// BEST PRACTICE: derivedStateOf
// We only want 'isAtBottom' to update when the boolean result changes,
// NOT every time the scroll offset changes (which happens every frame).
val isAtBottom by remember {
    derivedStateOf {
        val layoutInfo = listState.layoutInfo
        val visibleItemsInfo = layoutInfo.visibleItemsInfo
        if (layoutInfo.totalItemsCount == 0) {
            true
        } else {
            val lastVisibleItem = visibleItemsInfo.lastOrNull()
            val lastItemIndex = layoutInfo.totalItemsCount - 1

            // Item is at the bottom if the last item is visible and its bottom matches the viewport bottom
            lastVisibleItem != null &&
                    lastVisibleItem.index == lastItemIndex &&
                    lastVisibleItem.offset + lastVisibleItem.size <= layoutInfo.viewportEndOffset
        }
    }
}
```

### 1.3 Strict List Performance
For `LazyColumn` and `LazyRow`, relying on the default behavior is often insufficient for complex lists.

*   **`key`**: Unique identifier for each item. Prevents unnecessary recompositions and maintains state (like scroll position or text input) when items are moved or removed.
*   **`contentType`**: Helps the `LazyLayout` reuse compositions more efficiently. If you have different types of items (e.g., "User Message" vs "System Message"), telling Compose which is which allows it to recycle the correct nodes.

```kotlin
LazyColumn(
    state = listState,
    contentPadding = PaddingValues(16.dp)
) {
    items(
        items = messages,
        key = { message -> message.id },
        contentType = { message -> message.role }
    ) { item ->
        MessageBubble(item)
    }
}
```

## 2. Stability Annotations

### 2.1 The `@Immutable` Promise
Marking your UI state classes with `@Immutable` acts as a contract with the compiler. It promises that if the object reference hasn't changed, its content hasn't changed either. This allows Compose to skip recomposing the entire `ChatScreen` if the `ChatUiState` instance is the same.

```kotlin
@Immutable
data class ChatUiState(
    val messages: List<Message> = emptyList(),
    val inputValue: String = "",
)
```

### 2.2 `@ReadOnlyComposable`
As mentioned in the API Design guide, use `@ReadOnlyComposable` for helper functions that only read CompositionLocals. This completely skips the composition grouping for that call.

```kotlin
@Composable
@ReadOnlyComposable
fun colors(): ChatColors = ...
```

## 3. Performance Monitoring

Optimizing performance is an iterative process that requires measurement.

### 3.1 Tracking Scroll Jank
In production, it's important to monitor whether your lists are stuttering (janking) during scrolls.

**Best Practice**: Use specialized utilities like `TrackScrollJank` to monitor the `ScrollableState` (like `LazyListState`) and report performance metrics.

```kotlin
@Composable
fun MyScreen() {
    val state = rememberLazyListState()
    
    // Monitors scroll performance and reports jank
    TrackScrollJank(scrollableState = state, stateName = "my_screen:list")
    
    LazyColumn(state = state) {
        /* ... */
    }
}
```

This ensures you have visibility into real-world performance across different devices.