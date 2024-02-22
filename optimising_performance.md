# Optimising performance

### Articles

- [Jetpack Compose performance](https://developer.android.com/jetpack/compose/performance)
- [Stability in Compose](https://developer.android.com/jetpack/compose/performance/stability)
- [Jetpack Compose Stability Explained](https://medium.com/androiddevelopers/jetpack-compose-stability-explained-79c10db270c8)
- [Layout Inspector](https://developer.android.com/jetpack/compose/tooling/layout-inspector)

### Videos

- [Performance best practices for Jetpack Compose](https://www.youtube.com/watch?v=EOQB8PTLkpY)
- [More performance tips for Jetpack Compose](https://www.youtube.com/watch?v=ahXLwg2JYpc)
- [Jetpack Compose: Debugging recomposition](https://www.youtube.com/watch?v=SWBN0y0lFNY)

###

## Some theory

In order to understand different optimisation techniques in Composer, 
it's important to understand how the framework works under the hood.

### 💡 Composable functions are restarted when state which they read changes

Compose tracks places when we read the state, not where we declare it.
When the state changes, Compose restarts the closest composable function from the point where state was read.
This restarting process is called **recomposition**.

```kotlin
@Composable
fun Info(title: String, body: String) {
    // Declaration of the state is not tracked by Compose
    var isExpanded by remember { mutableStateOf(false) }
    
    // When the isExpanded changes the Card function is restarted as this is 
    // the closest composable function from the point where isExpanded was read
    Card {
        Text(title)
        // Read of the state is tracked by Compose
        if (isExpanded) {
            Text(body)
        }
        Button(onClick = { isExpanded = !isExpanded }) {
            Text("Toggle")
        }
    }
}
```

### 💡 Inline composable functions doesn't count

Inline functions are removed from the code during compilation and their content is pasted in the place where they are called.
It means that they are not treated as the closest composable function which can be restarted.

```kotlin
@Composable
fun Info(title: String, body: String) {
    var isExpanded by remember { mutableStateOf(false) }

    // Column is inline so now the Info function is the closest composable function
    Column {
        Text(title)
        if (isExpanded) {
            Text(body)
        }
        Button(onClick = { isExpanded = !isExpanded }) {
            Text("Toggle")
        }
    }
}
```

### 💡 Composable functions are skipped when their parameters don't change

Normally, restarting a function should result in restarting all of its children.
However, it would be inefficient to always restart all of them.
That's why Compose skips the composable function when its parameters don't change.

```kotlin
@Composable
fun TopAppBar(title: String) {
    // The TopAppBar function is skipped during recomposition 
    // when the title doesn't change
    TopAppBar(title = title) {
        Text(title)
    }
}
```

### 💡 You can monitor recompositions with Layout Inspector

It is a [tool](https://developer.android.com/jetpack/compose/tooling/layout-inspector) which allows you to see how many times each composable function was recomposed in running application.

## Stability

Unfortunately not every composable function can be skipped during recomposition.
Compose requires that all parameters of a composable function be stable.

### 💚 Basic types are stable

It means basic types like `String`, `Int`, `Double`, `Float`, `Boolean`.

```kotlin
@Composable
fun Example(
    string: String,
    int: Int,
    double: Double,
    float: Float,
    boolean: Boolean,
) {
    // ...
}
```

### 💚 Objects with read-only basic types are stable

Objects where all the properties are basic types and are defined as `val`.

```kotlin
data class ExampleData(
    val string: String,
    val int: Int,
    val double: Double,
    val float: Float,
    val boolean: Boolean,
)

@Composable
fun Example(data: ExampleData) {
    // ...
}
```

### 💔 All the collection types are unstable

Collections are interfaces so the compiler cannot guarantee that they have read-only implementation.

```kotlin
data class ExampleData(
    val string: String,
    val int: Int,
    val double: Double,
    val float: Float,
    val boolean: Boolean,
    // Whole ExampleData object is unstable because of the list
    val list: List<String>,
)

@Composable
fun Example(data: ExampleData) {
    // ...
}
```

### 💚 Classes with collections can be annotated with `@Immutable`

If our class contains a collection, we can annotate it with `@Immutable` to tell the Compose that we don't expect it to change.

```kotlin
@Immutable
data class ExampleData(
    val string: String,
    val int: Int,
    val double: Double,
    val float: Float,
    val boolean: Boolean,
    val list: List<String>,
)

@Composable
fun Example(data: ExampleData) {
    // ...
}
```

### 💔 The `@Immutable` annotation cannot be used with function parameters

The `@Immutable` annotation can be used only with classes. 
When we pass a list as a parameter of a composable function, we cannot use it.

```kotlin
@Composable
fun Example(
    // This code doesn't compile
    @Immutable list: List<String>
) {
    // ...
}
```

### 💚 We can use Immutable Collections library

As and alternative to the `@Immutable` annotation, we can use a dedicated library which adds immutable collections.

```kotlin
@Composable
fun Example(
    // Now compiler knows that the list will not change
    list: ImmutableList<String>
) {
    // ...
}
```

### 💔 Classes from non-Compose modules are unstable

Even if our class contains only basic types, it's still unstable when it comes from a module which does not have a Compose compiler applied.
These are typical some domain or data modules.

```kotlin
// :data:articles
data class Article(
    val id: String,
    val title: String,
    val content: String,
)

// :features:article-list
@Composable
fun ArticleItem(article: Article) {
    // ...
}
```

### 💚 Avoid passing full business object deep into the UI tree

It's better to pass only the necessary data, as separate parameters with basic types.

```kotlin
// :data:articles
data class Article(
    val id: String,
    val title: String,
    val content: String,
)

// :features:article-list
@Composable
fun ArticleItem(
    // Only the necessary data is passed
    title: String,
    content: String,
) {
    // ...
}
```

### 💚 Eventually introduce separate UI models

When we have a complex business object, it's better to introduce a separate UI model which will contain only the necessary data and guarantee stability.

```kotlin
// :data:articles
data class Article(
    val id: String,
    val title: String,
    val content: String,
    val image: URL,
    val author: String,
    val date: LocalDate
)

// :features:article-list
data class ArticleUiState(
    val title: String,
    val content: String,
    val imageUrl: String,
)

@Composable
fun ArticleItem(article: ArticleUiState) {
    // ...
}
```

### 💚 Use `@Stable` annotation for plain class state holders

Plain class state holders usually contain some `var` properties, so we can't annotate them as `@Immutable`.
However, we can use `@Stable` annotation to tell Compose that all the `var` properties are implemented as `MutableState` so the Compose will be notified about changes.

```kotlin
@Stable
class EmailFieldState {
    
    var email: String by mutableMapOf("")
}

@Composable
fun EmailField(state: EmailFieldState) {
    // ...
}
```

### Premature optimisation

You don't have to always apply all the optimisation techniques.
If performance is not an issue, it's better to keep the code simple and readable.

### 💚 Monitor recompositions with Layout Inspector
