# How to handle UI events

### 📖 Articles

- [UI events](https://developer.android.com/topic/architecture/ui-layer/events)
- [ViewModel: One-off event antipatterns](https://medium.com/androiddevelopers/viewmodel-one-off-event-antipatterns-16a1da869b95)
- [ViewModel: Events as State are an Antipattern](https://proandroiddev.com/viewmodel-events-as-state-are-an-antipattern-35ff4fbc6fb6)
- [Exercises in futility: One-time events in Android](https://itnext.io/exercises-in-futility-one-time-events-in-android-ddbdd7b5bd1c)
- [How to handle single-event in Jetpack Compose](https://marco-cattaneo.medium.com/how-to-handle-single-event-in-jetpack-compose-f90b6220e8c8)

### 🎬 Videos

- [Architecture: Handling UI events](https://www.youtube.com/watch?v=lwGtp0Yr0PE)
- [Are One-Time Events an Anti-Pattern?](https://www.youtube.com/watch?v=njchj9d_Lf8)

## User events

User events are actions that the user performs in the UI, such as clicking a button, swiping a list, or entering text in a text field. 

### 💚 When user event requires a business logic delegate it to the ViewModel

For example, if the user clicks a button to like an article, the ViewModel should handle that business logic.

```kotlin
fun NavGraphBuilder.articleListScreen() {
    composable(route = ARTICLE_LIST_ROUTE) {
        val viewModel: ArticleListViewModel = hiltViewModel()
        val state: ArticleListState by viewModel.state.collectAsState()
        
        ArticleListScreen(
            state = state,
            // Delegate business logic to the ViewModel
            onArticleLike = { articleId -> viewModel.onArticleLike(articleId) }
        )
    }
}
```

### 💚 When navigation doesn't require business logic, handle it in the UI

Sometimes, when user clicks a button, we want to navigate to another screen. 
In this case, we can handle the navigation directly in the UI.

```kotlin
fun NavGraphBuilder.articleListScreen(
    onNavigateToArticle: (String) -> Unit
) {
    composable(route = ARTICLE_LIST_ROUTE) {
        val viewModel: ArticleListViewModel = hiltViewModel()
        val state: ArticleListState by viewModel.state.collectAsState()
        
        ArticleListScreen(
            state = state,
            // Handle navigation in the UI
            onNavigateToArticle = onNavigateToArticle,
        )
    }
}
```

### 💚 User event can update the UI state

In other cases, some user event may result in updating the UI state.
No matter if this state is directly in the UI or in a separate plain class state holder, we can update it directly in the UI.

```kotlin
@Composable
fun ExpandableCard() {
    var isExpanded by rememberSaveable { mutableStateOf(false) }

    AnimatedVisibility(isExpanded) {
        // User event can update the UI state directly
        Card(onClick = { isExpanded = !isExpanded }) {
            // ...
        }
    }
}
```

### 💔 Don't put complex logic in the UI when you handle user events

When you handle user events in the UI, don't put complex logic there.
If you need to perform some decisions or calculations, create a separate plain class state holder.

```kotlin
@Composable
fun ArticleListScreen(articles: List<Article>) {
    var displayedArticles by remember(articles) { mutableStateOf(articles) }
    
    Button(
        onClick = {
            // Not a good thing to put in the UI
            // Create a separate plain class state holder instead
            displayedArticles = articles.sortedBy { it.date }
        }
    ) {
        Text("Sort by date")
    }
}
```

## ViewModel events - Google way

Second type of UI events are ViewModel events. They usually occur when some business logic is done and the UI needs to be updated.
Google has its own recommendations how to handle these situations.

### 💚 When something happens in the ViewModel, you should update the state

For example when ViewModel completes some business operation with either success or failure, it should update the state to inform the UI about the result.

```kotlin
class LoginViewModel(private val loginUseCase: LoginUseCase) : ViewModel() {
    
    var uiState by mutableStateOf<LoginUiState>(LoginUiState.FillingForm)
        private set
    
    fun submitLogin(credentials: Credentials) = viewModelScope.launch {
        // We inform that we are submitting the form
        // UI can display some loading indicator
        uiState = LoginUiState.Submitting
        val result = loginUseCase.login(credentials)
        loginUseCase(credentials).onSuccess {
            // We inform that user is logged in
            // UI can go to a different screen 
            uiState = LoginUiState.LoggedIn
        }.onFailure {
            // We inform that credentials are invalid
            // UI can display some error message
            uiState = LoginUiState.InvalidCredentials
        }
    }
} 
```

### 💚 Handle state changes with `LaunchedEffect`

The best way to handle state changes in the UI is to use the `LaunchedEffect`.
This way we can call navigation or display Snackbar or Dialog.

```kotlin
val snackbarHostState = remember { SnackbarHostState() }
val viewModel: LoginViewModel = hiltViewModel()
val state by viewModel.uiState.collectAsStateWithLifecycle()

LaunchedEffect(state) {
    when (state) {
        is LoginUiState.InvalidCredentials -> {
            snackbarHostState.showSnackbar(message = "Invalid credentials")
        }
        is LoginUiState.LoggedIn -> onNavigateToHome()
    }
}
```

### 💚 UI can inform the ViewModel that state change was handled

UI decides how to react to the state change. When ViewModel informs about the invalid credentials, UI may display a Snackbar. 
Snackbar is visible for a few seconds and then it disappears. Once it disappears, the UI can inform the ViewModel that the state change was handled.
Then ViewModel resets the state to the initial state.

```kotlin
val snackbarHostState = remember { SnackbarHostState() }
val viewModel: LoginViewModel = hiltViewModel()
val state by viewModel.uiState.collectAsStateWithLifecycle()

LaunchedEffect(state) {
    when (state) {
        is LoginUiState.InvalidCredentials -> {
            snackbarHostState.showSnackbar(message = "Invalid credentials")
            // Inform the ViewModel that Snackbar was shown
            viewModel.invalidCredentialsMessageShown()
        }
        is LoginUiState.LoggedIn -> onNavigateToHome()
    }
}
```

```kotlin
class LoginViewModel : ViewModel() {
    
    var uiState by mutableStateOf<LoginUiState>(LoginUiState.FillingForm)
        private set
    
    fun invalidCredentialsMessageShown() {
        // Reset the state to the initial state
        uiState = LoginUiState.FillingForm
    }
}
```

### 💔 Don't tell the UI what to do. Just say what happened

When ViewModel informs the UI about the state change, it should not tell the UI what to do.
It is the UI's responsibility to decide how to react to the state change.

```kotlin
class LoginViewModel : ViewModel() {
    
    var uiState by mutableStateOf<LoginUiState>(LoginUiState.FillingForm)
        private set
    
    fun submitLogin(credentials: Credentials) = viewModelScope.launch {
        // Don't say how to present the loading
        uiState = LoginUiState.ShowLoadingSpinner
        val result = loginUseCase.login(credentials)
        loginUseCase(credentials).onSuccess {
            // Don't say where to navigate
            uiState = LoginUiState.NavigateToHome
        }.onFailure {
            // Don't say how to present the error
            uiState = LoginUiState.ShowInvalidCredentialsSnackbar
        }
    }
}
```

## ViewModel events - Community way

The community doesn't fully agree with Google's approach to handle ViewModel events by updating the state.
The alternative approach is to use separate side effects and a `Channel` to inform the UI about what happened in the ViewModel.

### 💚 Define separate SideEffect model

In this approach, we define a separate model which represents all the side effect which may come from a single ViewModel.

```kotlin
data class LoginUiState(
    val isSubmitting: Boolean = false,
)

sealed interface LoginSideEffect {
    
    data object CompleteLogin : LoginSideEffect
    
    data object NotifyAboutInvalidCredentials : LoginSideEffect
}
```

### 💚 Use `Channel` to emit side effects from the ViewModel

Next to the UI state, ViewModel contains a `Channel` that we can use to send side effects to the UI.

```kotlin
class LoginViewModel(private val loginUseCase: LoginUseCase) : ViewModel() {

    var uiState by mutableStateOf<LoginUiState>(LoginUiState())
        private set
    
    private val _sideEffects = Channel<LoginSideEffect>()
    val sidesEffects = _sideEffects.receiveAsFlow()

    fun submitLogin(credentials: Credentials) = viewModelScope.launch {
        // Inform that we are submitting the form
        // UI can display some loading indicator
        uiState = uiState.copy(isSubmitting = true)
        val result = loginUseCase.login(credentials)
        loginUseCase(credentials).onSuccess {
            // Inform that user is logged in
            // UI can go to a different screen 
            _sideEffects.send(LoginSideEffect.CompleteLogin)
        }.onFailure {
            // Inform that credentials are invalid
            // UI can display some error message
            _sideEffects.send(LoginSideEffect.NotifyAboutInvalidCredentials)
        }
        // Inform that we finished submitting the form
        // UI can hide the loading indicator
        uiState = uiState.copy(isSubmitting = false)
    }
} 
```

### 💚 Create a helper function to collect side effects in the UI

In order to safely collect side effects in the UI we should scope it to the lifecycle and use a proper dispatcher.
To not repeat this code in every screen, we can create a helper function.

```kotlin
@Composable
fun <T> CollectSideEffects(
    sideEffects: Flow<T>,
    onSideEffect: (T) -> Unit
) {
    val lifecycleOwner = LocalLifecycleOwner.current
    LaunchedEffect(Unit) {
        // Collect side effects only when the UI is in the STARTED state
        lifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
            // Guarantee that we handle side effect immediately without delays
            withContext(Dispatchers.Main.immediate) {
                sideEffects.collect(onSideEffect)
            }
        }
    }
}
```

### 💚 Collect side effects in the UI using helper function

We can use our custom helper function to safely handle side effects in the UI.

```kotlin
val snackbarHostState = remember { SnackbarHostState() }
val viewModel: LoginViewModel = hiltViewModel()

// We handle side effect using our helper function
CollectSideEffects(viewModel.sidesEffects) { sideEffect ->
    when (sideEffect) {
        is LoginSideEffect.NotifyAboutInvalidCredentials -> {
            snackbarHostState.showSnackbar(message = "Invalid credentials")
        }
        is LoginSideEffect.CompleteLogin -> onNavigateToHome()
    }
}
```
