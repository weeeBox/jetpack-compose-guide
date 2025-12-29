# Common Pitfalls & Mitigations

*   **Missing `remember`**: Creating expensive objects or running logic on every recomposition. **Mitigation**: Wrap in `remember` or `rememberSaveable`.
*   **Unstable Lambdas**: Passing standard lambdas to sub-composables causes unnecessary recompositions. **Mitigation**: Use method references, `remember` the lambda, or use Kotlin 2.0 Strong Skipping mode.
*   **ConstraintLayout Overuse**: Using `ConstraintLayout` for simple linear layouts. **Mitigation**: Prefer nested `Column`/`Row` for better performance and readability, reserving `ConstraintLayout` for complex relative positioning.
*   **Writing to State during Composition**: Modifying state directly in the Composable body. **Mitigation**: Always move side effects to callbacks or Effect handlers (`LaunchedEffect`, `SideEffect`).
*   **Hardcoding Dimensions/Colors**: Scattering magic numbers and hex codes. **Mitigation**: Define them in the Theme or `Dimens` object to ensure consistency and ease of changes.
