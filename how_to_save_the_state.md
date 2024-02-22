# How to save the state

### 📖 Articles

- [Save UI state in Compose](https://developer.android.com/jetpack/compose/state-saving)
- [Save UI states](https://developer.android.com/topic/libraries/architecture/saving-states)
- [State and Jetpack Compose](https://developer.android.com/jetpack/compose/state)
- [Saved State module for ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel/viewmodel-savedstate)

### 🎬 Videos

- [Best practices for saving UI state on Android](https://www.youtube.com/watch?v=V-s4z7B_Gnc)

## Configuration changes

When the configuration changes, such as the device rotating or folding and unfolding, the system destroys and recreates the activity or fragment. 
This can cause the UI state to be lost, which can lead to a poor user experience.
The most common way to make sure the UI state is preserved across configuration changes is to use the `ViewModel` class.
However, Compose offers a new way to save and restore the UI state using the `rememberSaveable` function.

### 💚 Use `rememberSavebale` when you keep state in the UI

When you put some simple state directly in the composable function, you can use the `rememberSaveable` function instead of standard `remember` to save and restore the state across configuration changes.

```kotlin
@Composable
fun ExpandableCard() {
    // This way the card will remain expanded after configuration change
    var isExpanded by rememberSaveable { mutableStateOf(false) }
    
    AnimatedVisibility(isExpanded) {
        Card(onClick = { isExpanded = !isExpanded }) {
            // ...
        }
    }
}
```

### 💚 Implement custom `Saver` for plain class state holders

The `rememberSaveable` function can only save simple types like `Int`, `String`, `Boolean`, etc. and objects which are `Parcelable`.
If you have a plain class state holder, you can implement a custom `Saver` to save and restore the state.

```kotlin
// Plain class state holder for the expandable card
@Stable
class ExpandableCardState(initialIsExpanded: Boolean = false) {
    
    var isExpanded by mutableStateOf(initialIsExpanded)
}

// Helper function to remember an instance of the state holder
@Composable
fun rememberExpandableCardState(): ExpandableCardState {
    // Here we use rememberSaveable with custom Saver
    return rememberSaveable(saver = expandableCardStateSaver()) {
        ExpandableCardState()
    }
}

// Custom Saver for the state holder
private fun expandableCardStateSaver() = listSaver(
    save = { listOf(state.isExpanded) },
    restore = { ExpandableCardState(it[0]) }
)
```

### 💔 Don't use `rememberSaveable` to store complex business objects

The `rememberSaveable` function is the best option to save UI states. These are usually some data entered or selected by the user.
All the data is stored in the `Bundle` object which means it has to be serialized and deserialized. 

This process might be slow and can cause performance issues if you store complex data. If you have some complex data which you load from the API, database or other external source, don't use `rememberSaveable` to store it.
Instead, use a `ViewModel` class as a state holder.

```kotlin
// Storing business object, especially collections, in the rememberSaveable is not a good idea
@Composable
fun ArticleListScreen(articlesRepository: ArticlesRepository) {
    var articles by rememberSaveable(saver = articleListSaver()) {
        mutableStateOf(emptyList<Article>()) 
    }
    
    LaunchedEffect(Unit) {
        articles = articlesRepository.getArticles()
    }
}
```

### 💔 Don't put UI state in the ViewModel just to survive configuration changes

On the other hand, you should not overuse the `ViewModel` class to store the UI state just to survive configuration changes.
If some state is not related to the business logic, it's better to keep it in the UI directly or in a plain class state holder.

```kotlin
class ExpandableCardViewModel : ViewModel() {

    // We don't need ViewModel to store this state
    private val _isExpanded = MutableStateFlow(false)
    val isExpanded: StateFlow<Boolean> = _isExpanded
    
    fun toggleExpanded() {
        _isExpanded.value = !_isExpanded.value
    }
}
```

## Process death

Sometimes users navigate away from your app for a long time, and the system kills your app process to free up resources.
In that case all the states and objects, including `ViewModel` are destroyed.
When the user returns to your app, the system creates a new process and the UI state is lost.
Compose and Android offers us several ways to make our state survive the process death.

### 💚 Use `rememberSaveable` to survive process death

The `rememberSaveable` function survives the process death by default. No matter if you use it with a simple types, `Parcelable`, or a custom `Saver`.
The state is saved in the `Bundle` outside the application process and restored when the app is recreated.

```kotlin
@Composable
fun ExpandableCard() {
    // It's all you need to restore this state after process death
    var isExpanded by rememberSaveable { mutableStateOf(false) }
    
    AnimatedVisibility(isExpanded) {
        Card(onClick = { isExpanded = !isExpanded }) {
            // ...
        }
    }
}
```

### 💡 ViewModel doesn't survive process death

All the ViewModels are kept in the memory inside the application process. When the process is killed, all the ViewModels are destroyed.

```kotlin
class ArticleListViewModel : ViewModel() {

    // This state will be lost after process death
    private val _articles = MutableStateFlow(emptyList<Article>())
    val articles: StateFlow<List<Article>> = _articles
    
    fun loadArticles() = viewModelScope.launch {
        _articles.value = articlesRepository.getArticles()
    }
}
```

### 💚 Keep your data in the persistent storage if you want it to survive process death

The simplest option to make your business data survive the process death is to store it in the persistent storage.
You can use the `SharedPreferences`, `Room`, or any other database to store the data.

```kotlin
class ArticlesRepository(
    private val articlesLocalDataSource: ArticlesLocalDataSource,
    private val articlesNetworkDataSource: ArticlesNetworkDataSource,
) {
    
    fun getArticles(): List<Article> {
        // If data is stored locally, we don't need to load it from the network after process death
        val localArticles = articlesLocalDataSource.getArticles()
        if (localArticles) return localArticles
        val networkArticles = articlesNetworkDataSource.getArticles()
        articlesLocalDataSource.saveArticles(networkArticles)
        return networkArticles
    }
}
```

### 💚 Alternatively use `SavedStateHandle` to survive process death

If persistent storage is not an option, but you still want your data to survive the process death, you can use the `SavedStateHandle` class in your ViewModel.

```kotlin
private const val SELECTED_ARTICLE_ID = "selected_article_id"

class ArticlesListViewModel(
    private val savedStateHandle: SavedStateHandle,
) : ViewModel() {
    
    // You can keep the state in the SavedStateHandle to make it survive process death
    val selectedArticleId = savedStateHandle.getStateFlow(SELECTED_ARTICLE_ID, "")
    
    fun selectArticle(articleId: String) {
        savedStateHandle[SELECTED_ARTICLE_ID] = articleId
    }
}
```

