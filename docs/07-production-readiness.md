# Production Readiness & General Best Practices

## 1. Semantics and Accessibility

Don't just build for sight; build for the system.

*   **Touch Targets**: Ensure interactive elements meet the minimum size (48.dp). Use standard Material components or `Modifier.minimumInteractiveComponentSize()`.
*   **Content Description**: Avoid suppressing nullability warnings. Use `contentDescription = null` only for decorative images. If it conveys info, provide a localized description.

### 1.1 Merging Descendants
For complex components like a Chat Bubble, individual text elements (timestamp, author, message) might clutter the talkback experience. Use `mergeDescendants = true` to treat the entire component as one focusable unit, and provide a comprehensive `contentDescription`.

```kotlin
Box(
    modifier = modifier
        .semantics(mergeDescendants = true) {
            contentDescription = if (isUser) {
                "You said: ${message.content}"
            } else {
                "Assistant said: ${message.content}"
            }
        }
) {
    // Child composables (Text, Icons)
}
```
