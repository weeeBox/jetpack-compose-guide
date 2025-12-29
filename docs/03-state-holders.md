# State Management & State Holders

Effective state management is the backbone of a robust Compose application. This document covers the Unidirectional Data Flow (UDF) pattern for screen-level architecture and the State Holder pattern for reusable component logic.

## 1. The State & Event Pattern (UDF)

The **State & Event** pattern (often called **Unidirectional Data Flow** or **UDF**) is the core architectural principle in Jetpack Compose. It decouples *what is shown* from *how it changes*.

### 1.1 The Concept

1.  **State (Flows Down ↓):**
    *   Represents the "source of truth" for your UI at a specific point in time.
    *   The UI observes this state and recomposes whenever it updates.
    *   *Example:* `isLoading`, `listOfItems`, `errorMessage`.

2.  **Events (Flows Up ↑):**
    *   Represents actions that happen (user interactions, system callbacks).
    *   The UI notifies the logic layer (usually a ViewModel) that an event occurred.
    *   This is often referred to as the **"Event Sink"** pattern when all events are funnelled through a single lambda or interface (e.g., `(ChatEvent) -> Unit`).
    *   *Example:* `OnButtonClicked`, `OnSearchQueryChanged`.

### 1.2 Code Example

Here is a standard implementation using a `ViewModel`.

**1. Define State and Events**

```kotlin
// Immutable state object
data class LoginUiState(
    val email: String = "",
    val isLoading: Boolean = false,
    val error: String? = null
)

// Sealed interface for all possible actions
sealed interface LoginUiEvent {
    data class EmailChanged(val newEmail: String) : LoginUiEvent
    data object LoginClicked : LoginUiEvent
}
```

**2. The Logic Layer (ViewModel)**

```kotlin
class LoginViewModel : ViewModel() {
    // Expose immutable StateFlow to the UI
    private val _uiState = MutableStateFlow(LoginUiState())
    val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()

    // Single entry point for all events
    fun onEvent(event: LoginUiEvent) {
        when(event) {
            is LoginUiEvent.EmailChanged -> {
                _uiState.update { it.copy(email = event.newEmail) }
            }
            is LoginUiEvent.LoginClicked -> {
                performLogin()
            }
        }
    }
    
    private fun performLogin() { /* ... */ }
}
```

**3. The UI Layer (Composable)**

```kotlin
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = viewModel()
) {
    // Collect state safely
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    LoginContent(
        state = state,
        // Pass events up to the ViewModel
        onEvent = viewModel::onEvent
    )
}

@Composable
fun LoginContent(
    state: LoginUiState,
    onEvent: (LoginUiEvent) -> Unit // Callback for events
) {
    if (state.isLoading) {
        CircularProgressIndicator()
    } else {
        Button(
            onClick = { onEvent(LoginUiEvent.LoginClicked) }
        ) {
            Text("Login")
        }
    }
}
```

### 1.3 Benefits

1.  **Single Source of Truth:** The UI never modifies the state directly; it only asks the ViewModel to do it.
2.  **Predictability:** It's easy to trace how a specific state was reached by looking at the stream of events.
3.  **Testability:** You can unit test the `ViewModel` by sending events and asserting the resulting state, without needing any Android/Compose dependencies.
4.  **"Stateless" Reusable Components:** When a Composable only takes State as input and exposes Events as callbacks, it can be easily previewed and reused.

## 2. Component State Holders

When Composable logic increases in complexity—involving state management, animations, or interaction handling—but is not business logic (like loading users), it should be extracted into a **State Holder**. This pattern involves creating a plain Kotlin class responsible for the "UI Logic" layer.

### 2.1 Core Responsibilities
*   **Source of Truth**: Maintains the `State<T>` (via `mutableStateOf`) for the component.
*   **Logic Processing**: Encapsulates functions to modify state (e.g., `scrollBy`, `expand`).
*   **State Persistence**: Instances are typically created via a `remember` factory function and utilize `rememberSaveable` to survive configuration changes.

### 2.2 Framework Example: `ScrollState`

The Jetpack Compose framework utilizes this pattern extensively. Consider `ScrollState` (simplified for clarity):

```kotlin
@Stable
class ScrollState(initial: Int) : ScrollableState {
    // 1. Public State backed by mutableStateOf
    var value: Int by mutableStateOf(initial, structuralEqualityPolicy())
        private set

    // 2. Logic to modify state
    suspend fun scrollTo(value: Int) {
        // ... animation and state update logic ...
        this.value = value
    }
    
    // ... implementation of ScrollableState ...
}
```

And its accompanying factory function:

```kotlin
@Composable
fun rememberScrollState(initial: Int = 0): ScrollState {
    return rememberSaveable(saver = ScrollState.Saver) {
        ScrollState(initial = initial)
    }
}
```

**Architectural Advantages:**
*   **Testability**: `ScrollState` is a plain class, allowing unit testing of logic (e.g., `scrollTo`) without UI instrumentation.
*   **Reusability**: Any Composable can accept a `ScrollState` instance to drive its behavior.
*   **Decoupling**: The UI component (e.g., `Column`) observes `state.value` without containing the logic itself.

## 3. Best Practices

### 3.1 Persistence with `Saver`
Implement a `Saver` if the state must survive process death or configuration changes.

```kotlin
companion object {
    val Saver: Saver<MyState, *> = listSaver(
        save = { listOf(it.currentValue, it.isExpanded) },
        restore = { MyState(it[0] as Int, it[1] as Boolean) }
    )
}
```

### 3.2 Platform Independence
State Holders should ideally be plain Kotlin classes. If Android resources (like `Context`) are required, inject them via the `remember` factory.

```kotlin
@Composable
fun rememberMyState(
    context: Context = LocalContext.current
): MyState {
    return remember { MyState(context) }
}
```

### 3.3 Stability Contract
Annotate the class with `@Stable` to inform the Compose compiler that public properties will notify composition upon change (via `MutableState`) and that the instance is immutable regarding its dependencies.

## 4. Common Pitfalls

### 4.1 Bundling Events in State Classes
**Avoid** moving event callbacks inside the State data class.

*   **Bad Pattern**:
    ```kotlin
    data class UiState(val data: String, val onEvent: () -> Unit)
    ```
*   **Why it fails**:
    1.  **Breaks Smart Recomposition**: Lambdas do not have structural equality. Even if data hasn't changed, a new lambda instance makes `oldState != newState`, triggering unnecessary recomposition.
    2.  **Serialization Issues**: You cannot `@Parcelize` a class containing function references, making it impossible to save state across process death.
    3.  **Debugging**: Logs become cluttered with `Function1<...>` instead of clean data.

**Recommendation**: Pass events as a separate parameter (e.g., `actions: LoginActions` or individual lambdas).

### 4.2 Leaking Composables
**Avoid** passing `@Composable` lambdas or Composable function references into a State Holder. State Holders must handle data and logic, not UI emission.

*   **Don't**: `class MyState(val content: @Composable () -> Unit)`
*   **Do**: Pass the content slot directly to the Composable function.

### 4.3 Managing Business Data
Do not perform data fetching (e.g., Repository calls) within a State Holder. This is the responsibility of the `ViewModel`. The State Holder should exclusively manage **UI state** (e.g., expansion status, scroll offset, input text).

*   **Don't**: calling `repo.getUsers()` inside `MyListState`.
*   **Do**: `ViewModel` loads users; `MyListState` manages the list's visual state.

### 4.4 Multiple Sources of Truth
If a parent Composable controls the state (Stateless component), do not duplicate that state inside a local State Holder. Adhere to "State Hoisting," where the State Holder observes the external state or the Composable acts as a pass-through.
