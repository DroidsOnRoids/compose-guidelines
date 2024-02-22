# How to represent the state

### 📖 Articles

- [UI State production](https://developer.android.com/topic/architecture/ui-layer/state-production)
- [UI layer](https://developer.android.com/topic/architecture/ui-layer)
- [Consuming flows safely in Jetpack Compose](https://medium.com/androiddevelopers/consuming-flows-safely-in-jetpack-compose-cde014d0d5a3)

### 🎬 Videos

- [State holders and state production in the UI Layer](https://www.youtube.com/watch?v=pCX9wvu-Bq0)
- [Peeling back the layers: Unmasking the UI-nknown](https://www.droidcon.com/2023/11/15/peeling-back-the-layers-unmasking-the-ui-nknown/)

## Single state vs multiple states

This is a common question when you start designing your UI state. Should you have a single state that represents the entire UI or should you have multiple states that represent different parts of the UI?
Both approaches can be used simultaneously, depending on the specific situation.

### 💚 Keep related states together

When you have multiple states that are related to each other, it's a good idea to keep them together in a single class. 
This makes it easier to understand the relationship between the states and how they change together.

This state contains logically related data. The `articles` list represent a list of articles which is presented to the user.
Then user can select one of them to see the details.

```kotlin
data class ArticleListUiState(
    val articles: List<Article> = emptyList(),
    val selectedArticle: Article? = null,
)

class ArticleListViewModel : ViewModel() {
    
    // ViewModel exposes a single UI State
    val uiState: StateFlow<ArticleListUiState> = ...
}
```

### 💚 Keep unrelated states separate

When you have states that are not related to each other, it's a good idea to keep them separate in different classes.

These two states have nothing in common. One represents the list of articles and the other represents the user toolbar with some information about currently logged-in user.

```kotlin
data class ArticleListUiState(
    val articles: List<Article> = emptyList(),
    val selectedArticle: Article? = null,
)

sealed interface UserToolbarUiState {
    
    data object SignedOut : UserToolbarUiState
    
    data class SignedIn(
        val userName: String,
        val subscriptionType: SubscriptionType,
    ) : UserToolbarUiState
}

class ArticleListViewModel : ViewModel() {
    
    // ViewModel exposes one state for article list ...
    val articleListUiState: StateFlow<ArticleListUiState> = ...
    
    // ... and another state for user toolbar
    val userToolbarUiState: StateFlow<UserToolbarUiState> = ...
}
```

## Data class vs sealed interface

When you design your UI state, you can use a data class, a sealed interface, or a combination of both to represent the state.
But be careful, because the choice of the representation can have a significant impact on the complexity of the code which uses this state.

### 💚 Use sealed interface to represent a sequence of states

When you have a state that can be in one of several states, the sealed interface is a proper way to represent it.
In this case user always goes from one state to another.
It can be a simple `Loading/Success/Failure` or more complex structure.

```kotlin
sealed inteface ArticleListUiState {
    
    data object Loading : ArticleListUiState
    
    data class Success(val articles: List<Article>) : ArticleListUiState
    
    data object Failure : ArticleListUiState
}
```

### 💚 Use data class when user can mutate data inside the state

When the state contains data that can be modified by the user, it's better to use a flat data class.
It simplifies the code which performs state updates.

```kotlin
data class ArticleListUiState {
    val articles: List<Article> = emptyList()
    val selectedArticle: Article? = null
}
```

### 💚 You can nest sealed interfaces inside data classes

When you need both a sequence of states and a mutable data, you can nest a sealed interface inside a data class.

```kotlin
sealed interface ArticlesLoadingUiState {

    data object Loading : ArticlesLoadingUiState

    data class Success(val articles: List<Article>) : ArticlesLoadingUiState

    data object Failure : ArticlesLoadingUiState
}

data class ArticleListUiState(
    val articlesLoadingUiState: ArticlesLoadingUiState = Loading,
    val selectedArticle: Article? = null,
)
```

### 💔 Don't put data class which can be modified by the user inside sealed interface

When some part of the state can me modified we should not put it inside sealed interface.
It adds extra complexity to the code which updates the state because we need to check if we are in the correct variant of the sealed interface.

```kotlin
sealed interface ArticleListUiState {

    data object Loading : ArticleListUiState

    data class Success(
        // This one is loaded from other layers
        val articles: List<Article>,
        // This one is modified by the user
        val selectedArticle: Article? = null,
    ) : ArticleListUiState

    data object Failure : ArticleListUiState
}

class ArticleListViewModel : ViewModel() {

    private val _uiState = MutableStateFlow<ArticleListUiState>(Loading)

    fun selectArticle(article: Article) {
        _uiState.value = when (val currentState = _uiState.value) {
            // Mutable data inside sealed interface adds unnecessary complexity
            is ArticleListUiState.Success -> currentState.copy(selectedArticle = article)
            else -> currentState
        }
    }
}
```

## StateFlow vs Compose State

UI state has to be represented in a observable way, so the Compose get notified about the changes and can update the UI.
There are two main ways to represent the UI state: `StateFlow` and Compose `State`.

### 💚 Both StateFlow and Compose State can be used inside the ViewModel

It is up to your preference to choose if you want to use `StateFlow` or Compose `State` to represent the UI State inside the ViewModel.
`State` is simpler solution which can be used as a property delegate. 
`StateFlow` on the other hand is more powerful with variety of extension functions.

```kotlin
class ArticleListViewModel : ViewModel() {
    
    // You can use StateFlow ...
    private val _uiState = MutableStateFlow(ArticleListUiState())
    val uiState: StateFlow<ArticleListUiState> = _uiState.asStateFlow()
    
    // ... as well as Compose State
    val uiState by mutableStateOf(ArticleListUiState())
        private set
}
```

### 💡 With Compose State you need to be aware of multi-threading

`StateFlow` is thread-safe out of the box. When you use Compose `State` you need to make sure that you use a dedicated function when you update the state from a background thread.

```kotlin
// State representation with StateFlow
private val _uiState = MutableStateFlow(ArticleListUiState.Loading)
val uiState = _uiState.asStateFlow()

// Loading data in the background
fun loadData() = viewModelScope.launch(defaultDispatcher) { 
    runCatching {
        articlesRepository.getArticles()
    }.onSuccess {
        // We can safely update the state from the background thread
        _uiState.value = ArticleListUiState.Success(it)
    }
} 
```

```kotlin
// State representation with Compose State
var uiState by mutableStateOf(ArticleListUiState.Loading)
    private set

// Loading data in the background
fun loadData() = viewModelScope.launch(defaultDispatcher) {
    runCatching {
        articlesRepository.getArticles()
    }.onSuccess {
        // We need to wrap the state update in withMutableSnapshot
        Snapshot.withMutableSnapshot {
            uiState = ArticleListUiState.Success(it)
        }
    }
} 
```

### 💡 With StateFlow you need to be aware of the lifecycle of the UI

`StateFlow` needs to be collected in the UI and transformed to Compose `State`. Collecting can be done in two ways: lifecycle-aware or not.
Standard `collectAsState` function keeps collecting the flow even when the UI is in the STOPPED state.
Dedicate `collectAsStateWithLifecyce` stops collecting the flow in that case.

```kotlin
@Composable
fun ArticleListScreen() {
    val viewModel: ArticleListViewModel = viewModel()
    // Lifecycle-aware collection
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    // Standard collection
    val uiState by viewModel.uiState.collectAsState()
}
```
