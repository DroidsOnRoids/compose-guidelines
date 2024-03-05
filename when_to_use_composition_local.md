# When to use Composition Local 

### 📖 Articles

- [Locally scoped data with CompositionLocal](https://developer.android.com/jetpack/compose/compositionlocal)
- [Custom design systems in Compose](https://developer.android.com/jetpack/compose/designsystems/custom)
- [Anatomy of a theme in Compose](https://developer.android.com/jetpack/compose/designsystems/anatomy)
- [Jetpack Compose CompositionLocal - What You Need to Know](https://medium.com/geekculture/jetpack-compose-compositionlocal-what-you-need-to-know-979a4aef6412)

Compose framework uses Composition Local mechanism to provide many different objects.
It is not only the `MaterialTheme` but also things like `LocalContext`, `LocalFocusManager`, `LocalConfiguration` or `LocalUriHandler`.
You can find a full list of Composition Local objects in the [official documentation](https://developer.android.com/reference/kotlin/androidx/compose/ui/platform/package-summary#top-level-properties).

## Use cases for Composition Local

We can use Composition Local to solve our own problems while building the compose UI.
Below are some examples of when it makes sense to use Composition Local.

### 💡 Data formatting

Data formatting is usually based on some market configuration. 
It comes either from the system settings or is handled by the application itself.
Composition local can be used to provide this market configuration to the UI tree.
Then it can be used to implement reusable formatting functions which we can use in the UI components.

```kotlin
// Class which holds market configuration
data class Market(
    val locale: Locale,
    val currencyFormat: NumberFormat,
    val dateFormatter: DateTimeFormatter,
)

// Default market configuration which we can use for development, preview or tests
val AmericanMarket = Market(
    locale = Locale.US,
    currencyFormat = NumberFormat.getCurrencyInstance(Locale.US),
    dateFormatter = DateTimeFormatter.ofLocalizedDate(FormatStyle.SHORT)
)

// Composition local to provide market to the UI tree
val LocalMarket = staticCompositionLocalOf<Market> { AmericanMarket }
```

```kotlin
// Function to format currency
@Composable
@ReadOnlyComposable
fun formatCurrency(amount: Double): String {
    val market = LocalMarket.current
    return market.currencyFormat.format(amount)
}

// Function to format date
@Composable
@ReadOnlyComposable
fun formatDate(date: LocalDate): String {
    val market = LocalMarket.current
    return market.dateFormatter.format(date)
}
```

```kotlin
@Composable
fun TransactionItem(amount: Double, date: LocalDate) {
    Column {
        // Currency and date will be formatted based on the market provided by the LocalMarket
        Text(text = formatCurrency(amount))
        Text(text = formatDate(date))
    }
}
```

### 💡 Feature flags

Feature flags are a common way to control the behavior of the application. 
We often use them to show or hide some parts of the UI.
Instead of loading features flags in the View Models of each screen, we can use Composition Local to provide them to the entire UI tree.

```kotlin
// Class which holds all the feature flags
data class FeatureFlags(
    val isChatEnabled: Boolean = true,
    val areDebitCardsEnabled: Boolean = true,
)

// Composition local to provide feature flags to the UI tree
val LocalFeatureFlags = staticCompositionLocalOf<FeatureFlags> { FeatureFlags() }
```

```kotlin
@Composable
fun BankProductsScreen(...) {
    // We can access feature flags directly in the UI tree
    val featureFlags = LocalFeatureFlags.current
    
    Column {
        ProductItem(name = "Loans", onClick = onNavigateToLoans)
        ProductItem(name = "Savings", onClick = onNavigateToSavings)
        // We can show or hide the Debit Cards button based on the feature flag
        if (featureFlags.areDebitCardsEnabled) {
            ProductItem(name = "Debit Cards", onClick = onNavigateToDebitCards)
        }
    }
}
```

### 💡 Analytics

Nowadays mobile applications track many analytics events when user uses the app.
Very often these events are triggered by some UI actions taken by the user.
Instead of adding analytics handling to the ViewModel of the screen we can handle it directly in the UI tree using Composition Local.

```kotlin
interface Analytics {
    
    fun logEvent(event: AnalyticsEvent)
}

// Stub implementation used for development and tests
class StubAnalytics : Analytics {
    // ...
}

// Production implementation which uses Firebase Analytics
class ProductionAnalytics(
    private val firebaseAnalytics: FirebaseAnalytics
) : Analytics {
    // ...   
}

// Composition local to provide analytics to the UI tree
val LocalAnalytics = staticCompositionLocalOf<Analytics> { StubAnalytics() }
```

```kotlin
// Side effect to log the event when the screen is viewed
@Composable
fun LogScreenViewedEffect(screenName: String) {
    val analytics = LocalAnalytics.current
    LaunchedEffect(Unit) {
        val event = ScreenViewedEvent(screenName)
        analytics.logEvent(event) 
    }
}
```

```kotlin
@Composable
fun NestListScreen(...) {
    // We send analytics event with a side effect
    LogScreenViewedEffect(screenName = "Nest List")
}
```

## Good practices

### 💚 Provide a reasonable default value

When creating a Composition Local it is a good practice to provide a reasonable default value.
This way we avoid unexpected runtime exceptions and we can easily use Composition Local in the preview or tests.

```kotlin
val LocalMarket = staticCompositionLocalOf<Market> { AmericanMarket }

val LocalFeatureFlags = staticCompositionLocalOf<FeatureFlags> { FeatureFlags() }

val LocalAnalytics = staticCompositionLocalOf<Analytics> { StubAnalytics() }
```

### Use `@ReadOnlyComposable` annotation

When composable function does not contain any UI and it only reads the value from the Composition Local it is a good practice to use `@ReadOnlyComposable` annotation.
It allows Compose compiler to skip this function during recompositions.

```kotlin
@Composable
@ReadOnlyComposable
fun formatCurrency(amount: Double): String {
    // No UI here, only reading the value from the Composition Local
    val market = LocalMarket.current
    return market.currencyFormat.format(amount)
}
```

### 💔 Don't use Composition Local for business logic 

Even though Composition Local is a powerful mechanism it should not be overused.
When it comes to business logic, so accessing use cases, repositories etc. it is better to use View Models and dependency injection.
Don't use Composition Local to access business logic from the UI.

```kotlin
// It should be injected to the ViewModel using a proper DI solution
val LocalGetUserUseCase = staticCompositionLocalOf<GetUserUseCase> {
    error("No GetUserUseCase provided")
}
```
