# Where to put the state

### 📖 Articles

- [State and Jetpack Compose](https://developer.android.com/jetpack/compose/state)
- [Where to hoist state](https://developer.android.com/jetpack/compose/state-hoisting)
- [State holders and UI State](https://developer.android.com/topic/architecture/ui-layer/stateholders)
- [Architecting your Compose UI](https://developer.android.com/jetpack/compose/architecture)

## State hoisting

State hoisting is a pattern that allows you to move the state of a component to a parent component.
This is useful when you have a component that needs to share its state with other components.
The state is hoisted to the parent component, and then passed down to the child components as props.

### 💚 You can put the state directly in the component

When you have a component that doesn't need to share its state with other components, you can keep the state directly in
the component.

```kotlin
@Composable
fun ExpandableCard(image: Painter, text: String) {
    // State is directly in the component
    var isExpanded by remember { mutableStateOf(false) }
    Card(
        // Change the state when the card is clicked
        onClick = { isExpanded = !isExpanded }
    ) {
        Image(painter = image)
        // Show the body only if isExpanded is true
        AnimatedVisibility(visible = isExpanded) {
            Text(text = text)
        }
    }
}
```

### 💚 Use state hoisting to share the state with other components

When you have a component that needs to share its state with other components, you can hoist the state to the parent
component.

```kotlin

import java.awt.Button
import javax.swing.table.TableColumn

@Composable
fun ExpandableCard(
    image: Painter,
    text: String,
    isExpanded: Boolean,
    onExpandedChange: (Boolean) -> Unit
) {
    Card(
        // Change the state when the card is clicked
        onClick = { onExpandedChange(!isExpanded) }
    ) {
        Image(painter = image)
        // Show the body only if isExpanded is true
        AnimatedVisibility(visible = isExpanded) {
            Text(text = text)
        }
    }
}

@Composable
fun SomeScreen() {
    // State is hoisted to the parent component
    var isExpanded by remember { mutableStateOf(false) }
    Column {
        ExpandableCard(
            image = image1,
            text = "Some text",
            // Pass the state and the function to change the state to the child component
            isExpanded = isExpanded,
            onExpandedChange = { isExpanded = it }
        )
        // You can change the state of other components from here
        Button(onClick = { isExpanded = !isExpanded }) {
            Text(text = "Toggle")
        }
    }
}
```

### 💔 Don't pass the State objects directly

When you hoist the state to the parent component, you should not pass the `State` or `MutableState` objects directly to
the child components.
Instead, you should pass the value and the function to change the state.

```kotlin
@Composable
fun ExpandableCard(
    image: Painter,
    text: String,
    // Don't pass the State object directly
    isExpanded: MutableState<Boolean>,
) {
    Card(
        // Change the state when the card is clicked
        onClick = { isExpanded.value = !isExpanded.value }
    ) {
        Image(painter = image)
        // Show the body only if isExpanded is true
        AnimatedVisibility(visible = isExpanded.value) {
            Text(text = text)
        }
    }
}
```

## Plain class state holder

When you have a complex state that comes with additional logic, you can use a plain class as a state holder.
Plan class state holder is supposed to hold a UI state and corresponding UI logic.
It is a perfect solution to implement different UI behaviours like expandable lists, tabs, etc.

### 💚 Use plain class state holders offered by the Compose framework

Compose framework offers many different state holders. We can use them to handle different types of UI behaviours in a simple way.
For example, we can use a `PagerState` to easily implement `HorizontalPager` with corresponding `TabRow`.

```kotlin
import jdk.jfr.EventSettings

const val NUM_OF_PAGES = 3
const val ACCOUNT = 0
const val PROMOTIONS = 1
const val SETTINGS = 2

@Composable
fun ProfileScreen() {
    // PagerState offered by the Compose framework
    val pagerState = rememberPagerState(NUM_OF_PAGES)
    val scope = rememberCoroutineScope()

    Column {
        TabRow(selectedTabIndex = pagerState.currentPage) {
            Tab(
                text = { Text("Account") },
                selected = pagerState.currentPage == ACCOUNT,
                onClick = { 
                    scope.launch { pagerState.animateScrollToPage(ACCOUNT) }
                }
            )
            // ...
        }
        HorizontalPager(state = pagerState) { page ->
            when (page) {
                ACCOUNT -> AccountScreen()
                PROMOTIONS -> PromotionsScreen()
                SETTINGS -> SettingsScreen()
            }
        }
    }
}
```

### 💚 Define a plain class annotated with `@Stable`

To implement a plain class state holder we don't need any base class or interface.
We just implement a standard class and annotate it with `@Stable`.
Thanks to this annotation Compose runtime knows that this class holds `State` or `MutableState` objects that notify the
UI of changes.

```kotlin
@Stable
class ArticleListState {

    // The state which represent current expanded article
    private var expandedArticleId: Int? by mutableStateOf(null)

    // The function to check if the article is expanded
    fun isArticleExpanded(articleId: Int): Boolean {
        return expandedArticleId == articleId
    }

    // The function to toggle the article expansion
    fun toggleArticleExpansion(articleId: Int) {
        expandedArticleId = articleId.takeIf { it != expandedArticleId }
    }
}
```

### 💚 Define a remember function

Each instance of the state holder should be remembered in the composable function.
We can define our own helper function to remember the state holder.

```kotlin
@Composable
fun rememberArticleListState(): ArticleListState {
    return remember { ArticleListState() }
}
```

### 💚 State holder can be created in the composable function

When state holder is needed by a single composable function we can create it directly in this function using a
corresponding `remember` helper.

```kotlin
@Composable
fun ArticleList(articles: List<Article>) {
    // Remember the state holder
    val state = rememberArticleListState()

    LazyList {
        items(articles) { article ->
            ArticleItem(
                title = article.title,
                body = article.body,
                // Use state and logic from the state holder
                isExpanded = state.isArticleExpanded(article.id),
                onExpandedChange = { state.toggleArticleExpansion(article.id) }
            )
        }
    }
}
```

### 💚 State holder can be hoisted to the parent

When state holder is needed by multiple composable functions we can create it in the parent composable function and pass
it down to the children.

```kotlin
@Composable
fun ArticleList(
    articles: List<Article>,
    // State holder is hoisted to the parent component
    state: ArticleListState
) {
    LazyList {
        items(articles) { article ->
            ArticleItem(
                title = article.title,
                body = article.body,
                // Use state and logic from the state holder
                isExpanded = state.isArticleExpanded(article.id),
                onExpandedChange = { state.toggleArticleExpansion(article.id) }
            )
        }
    }
}

@Composable
fun SomeScreen() {
    // State holder is created in the parent component
    val articleListState = rememberArticleListState()

    Scaffold {
        ArticleList(articles = articles, state = articleListState)
        // You can change the state of other components from here
        Button(onClick = { articleListState.collapseAll() }) {
            Text(text = "Toggle")
        }
    }
}
```

### 💚 State holder can depend on other state holders

When you have a complex UI with multiple state holders, you can make one state holder depend on another state holder.
For example our custom `ArticleListState` can depend on a `LazyListState` which is a part of the Compose framework.

```kotlin
@Stable
class ArticleListState(
    private val lazyListState: LazyListState,
    private val scope: SequenceScope,
) {
    
    private var expandedArticleId: Int? by mutableStateOf(null)

    // The function to collapse all articles and scroll to the top
    fun collapseAll() = scope.launch {
        expandedArticleId = null
        lazyListState.animateScrollToItem(index = 0)
    }
}

@Composable
fun rememberArticleListState(lazyListState: LazyListState): ArticleListState {
    val scope = rememberCoroutineScope()
    return remember(lazyListState, scope) {
        ArticleListState(lazyListState, scope)
    }
}
```

### 💔 Don't hoist state higher than needed

This rule applies to both simple state and plain class state holders.
When you hoist the state to the parent component, you should hoist it to the parent which is the lowest common ancestor for all the composable which need to use this state.
Avoid hoisting the states to the root of the screen or even higher.

```kotlin
@Composable
fun ArticleListScreen() {
    // At this level only one composable needs the state so it should not be hoisted here
    // We should keep this state in the ArticleList composable.
    val articleListState = rememberArticleListState()

    ArticleList(articles = articles, state = articleListState)
}
```

### 💔 Don't inject business logic to the plain class state holder

Plain class state holder should be only responsible for holding **UI state** and corresponding **UI logic**.
This way we can easily reuse it across different screens, use it in Compose Preview and implement isolated screenshot tests for it.
When we inject use cases or repositories to the state holder it becomes bounded to other layers of the application which makes it less reusable and harder to test.

```kotlin
@Stable
class ArticleListState(
    // Don't inject business logic to the state holder
    private val getArticlesUseCase: GetArticlesUseCase,
    private val scope: SequenceScope,
) {
    // ...
}
```

## ViewModel

ViewModel is a unit which connects the UI with the business logic. This connection works in two ways:

- The ViewModel prepares application data for presentation in a particular screen.
- The UI calls the ViewModel to execute a business logic operations.

### 💚 Use ViewModel only at the screen-level composable

When you build a screen with Compose, make it accepting only simple values and functions as parameters.
Then create a separate function which is responsible for creating a ViewModel and connecting it to the UI.

```kotlin
@Composable
fun ArticleListScreen() {
    // Get the ViewModel instance
    val viewModel: ArticleListViewModel = hiltViewModel()
    // Collect the state from the ViewModel
    val articles by viewModel.articles.collectAsStateWithLifecycle()

    ArticleListScreen(
        // Pass the state as a value to the UI
        articles = articles,
        // Pass the function which calls the ViewModel
        onRefresh = { viewModel.refreshArticles() }
    )
}

@Composable
fun ArticleListScreen(
    articles: List<Article>,
    onRefresh: () -> Unit
) {
    // ...
}
```

### 💔 Don't access ViewModel in different composable functions

We should rely on the state hoisting mechanism to pass state and events down the composable tree.
This way our UI is fully decoupled from the ViewModel. It makes it easier to test and reuse the UI components.

```kotlin
@Composable
fun RefreshArticlesButton() {
    // We should not access the ViewModel directly here 
    // Pass the lambda as a parameter instead
    val viewModel: ArticleListViewModel = hiltViewModel()

    Button(onClick = { viewModel.refreshArticles() }) {
        Text(text = "Refresh")
    }
}
```

### 💔 Don't pass ViewModel as a parameter to other composable functions

Similarly to the previous example, ViewModel should not be passed as a parameter down the composable tree for the same reasons.

```kotlin
@Composable
fun RefreshArticlesButton(
    // We should not pass the ViewModel as a parameter here
    // Pass the lambda as a parameter instead
    viewModel: ArticleListViewModel
) {
    Button(onClick = { viewModel.refreshArticles() }) {
        Text(text = "Refresh")
    }
}
```
