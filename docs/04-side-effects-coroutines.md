# Side Effects & Coroutines

## 1. Core Asynchronous Mechanisms

### 1.1 Composition-Scoped Work (`LaunchedEffect`)
Use `LaunchedEffect` to initiate a coroutine that matches the lifecycle of the Composable. The coroutine starts when the Composable enters the composition and is cancelled when it leaves.
*   **Use Case**: Animations, processing initial focus, or observing state changes over time.
*   **Restriction**: Do not use for user-initiated events (e.g., clicks).

### 1.2 Interaction-Scoped Work (`rememberCoroutineScope`)
Use `rememberCoroutineScope` to obtain a `CoroutineScope` that can be used to launch coroutines from outside the composition phase, such as inside interaction callbacks (`onClick`, `onDrag`).
*   **Restriction**: Do not launch coroutines directly in the Composable body using this scope.

## 2. `LaunchedEffect` Scenarios

`LaunchedEffect` is the appropriate tool when the UI must execute imperative logic in response to composition entry or parameter changes.

### 2.1 Requesting Focus
The `ViewModel` is aware of validation errors but lacks the context to manipulate focus.

```kotlin
@Composable
fun SearchScreen(
    modifier: Modifier = Modifier
) {
    val focusRequester = remember { FocusRequester() }

    LaunchedEffect(Unit) {
        focusRequester.requestFocus()
    }

    TextField(
        value = "",
        onValueChange = { /* ... */ },
        modifier = modifier.focusRequester(focusRequester)
    )
}
```

### 2.2 Complex / Sequencing Animations
While `animate*AsState` handles simple transitions, complex sequences require a coroutine scope.

```kotlin
@Composable
fun LoginErrorShake(
    triggerShake: Boolean,
    modifier: Modifier = Modifier
) {
    val offsetX = remember { Animatable(0f) }

    // Do: Implementation (animation sequence) is UI logic.
    LaunchedEffect(triggerShake) {
        if (triggerShake) {
            for (i in 0..5) {
                offsetX.animateTo(10f, tween(50))
                offsetX.animateTo(-10f, tween(50))
            }
            offsetX.animateTo(0f)
        }
    }
}
```

### 2.3 Propagating "One-Shot" Events [WIP]
Bridges `ViewModel` events to UI actions requiring `NavController` or `ScaffoldState`.

TBD: Figure out a better example.

```kotlin
@Composable
fun RegistrationScreen(
    viewModel: RegistrationViewModel, 
    navController: NavController,
    modifier: Modifier = Modifier
) {
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is RegEvent.Success -> navController.navigate("home")
                is RegEvent.Error -> { /* show snackbar */ }
            }
        }
    }
}
```

### 2.4 Interacting with Legacy System APIs
**Scenario:** Maintaining screen wake lock.

```kotlin
@Composable
fun QrCodeScreen(
    modifier: Modifier = Modifier
) {
    val context = LocalContext.current

    LaunchedEffect(Unit) {
        val window = (context as? Activity)?.window
        window?.addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)
        
        try {
            awaitCancellation()
        } finally {
            // Ensure cleanup occurs when leaving the composition
            window?.clearFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)
        }
    }
}
```

## 3. Scope Separation & The "Scope Trap"

A common architectural error is performing business logic within a UI-scoped coroutine.

### 3.1 The Scope Trap
Launching a network request via `rememberCoroutineScope` ties that request to the View's lifecycle. If the user rotates the device, the operation is cancelled immediately.

**Don't**: Launch business-critical operations in a UI scope.

```kotlin
// Don't: Network call dies on rotation
// NOTE: This is a BAD architectural decision for demonstration purposes only.
@Composable
fun LoginButton(
    viewModel: LoginViewModel,
    modifier: Modifier = Modifier
) {
    val scope = rememberCoroutineScope()
    
    Button(
        modifier = modifier,
        onClick = {
            scope.launch { 
                viewModel.login() 
            }
        }
    ) {
        Text("Login")
    }
}
```

**Do**: Hoist the event to the `ViewModel`, which owns a `viewModelScope` that survives configuration changes. Use `rememberCoroutineScope` strictly for UI feedback.

```kotlin
@Composable
fun LoginButton(
    onLoginClick: () -> Unit, 
    snackbarHostState: SnackbarHostState,
    modifier: Modifier = Modifier
) {
    val scope = rememberCoroutineScope()

    Button(
        modifier = modifier,
        onClick = {
            onLoginClick() // Fire-and-forget to the ViewModel
            
            scope.launch { 
                snackbarHostState.showSnackbar("Logging in...") 
            }
        }
    ) {
        Text("Login")
    }
}
```

## 4. Advanced Bridge Patterns

### 4.1 `SideEffect`: Syncing with Non-Compose Code
Executes **after** every successful composition. Use this to publish state to external systems (e.g., Analytics).

```kotlin
@Composable
fun AnalyticsTracker(
    user: User,
    modifier: Modifier = Modifier
) {
    SideEffect { 
        FirebaseAnalytics.setUserProperty("user_type", user.type) 
    }
}
```

### 4.2 `DisposableEffect`: Deterministic Cleanup
Mandatory for side effects requiring explicit unregistration (Sensors, BroadcastReceivers).

```kotlin
@Composable
fun SystemBroadcastReceiver(
    systemAction: String,
    onEvent: (Intent?) -> Unit,
    modifier: Modifier = Modifier
) {
    val context = LocalContext.current

    DisposableEffect(context, systemAction) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(context: Context?, intent: Intent?) {
                onEvent(intent)
            }
        }
        context.registerReceiver(receiver, IntentFilter(systemAction))
        
        onDispose { 
            context.unregisterReceiver(receiver) 
        }
    }
}
```

### 4.3 `produceState`: Callbacks to State
Provides a functional adaptation for converting non-Compose data sources (e.g., Sockets) into a Compose `State<T>`.

## 5. Flow Integration

### 5.1 Safe Collection (`collectAsStateWithLifecycle`)
*   **Do**: Use `collectAsStateWithLifecycle()` (requires `androidx.lifecycle:lifecycle-runtime-compose`) to safely observe Flows from the ViewModel.
*   **Don't**: Use `collectAsState()` for ViewLayer flows without explicit lifecycle handling.

### 5.2 State to Flow (`snapshotFlow`)
Converts Compose `State<T>` objects into a Flow to leverage operators like `debounce`.

```kotlin
@Composable
fun ScrollReporter(
    listState: LazyListState,
    viewModel: ScrollViewModel,
    modifier: Modifier = Modifier
) {
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .debounce(300)
            .distinctUntilChanged()
            .collect { index -> 
                viewModel.reportScrollPosition(index) 
            }
    }
}
```

## 6. Common Anti-Patterns

### 6.1 Launching in the Body
Never launch a coroutine directly in the Composable function body. This executes the side effect on **every recomposition**.

```kotlin
// Don't: This runs on every frame/recomposition
// NOTE: This is a BAD architectural decision for demonstration purposes only.
@Composable 
fun BadExample(
    modifier: Modifier = Modifier
) { 
    GlobalScope.launch { /* ... */ } 
}
```

### 6.2 Ignoring Cancellation
Compose relies on cooperative cancellation. If catching generic `Exception`, ensure `CancellationException` is re-thrown.

## 7. Architectural Validation Checklist

| API | Use Case | Valid Architectural Usage? |
| --- | --- | --- |
| **`LaunchedEffect`** | Composition-scoped coroutines | **Yes**, if logic relies on View-specific classes (Focus, Window) or is purely visual. |
| **`rememberCoroutineScope`** | Interaction-scoped coroutines | **Yes**, for UI feedback (Snackbar, Scroll). **No** for Business Logic (Network/DB). |
| **`SideEffect`** | Post-composition sync | **Yes**, for Analytics or logging state changes. |
| **`DisposableEffect`** | Cleanup | **Yes**, for Listeners/Receivers. |
| **`collectAsStateWithLifecycle`**| Observing Flows | **Yes**, mandatory for ViewModel flows. |