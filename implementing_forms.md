# Implementing forms

### 📖 Articles

- [Effective state management for TextField in Compose](https://medium.com/androiddevelopers/effective-state-management-for-textfield-in-compose-d6e5b070fbe5)
- [BasicTextField2: A TextField of Dreams [1/2]](https://proandroiddev.com/basictextfield2-a-textfield-of-dreams-1-2-0103fd7cc0ec)
- [BasicTextField2: A TextField of Dreams [2/2]](https://proandroiddev.com/basictextfield2-a-textfield-of-dreams-2-2-fdc7fbbf9ffb)

### 🎬 Videos

- [Reimagining text fields in Compose](https://www.droidcon.com/2023/07/20/reimagining-text-fields-in-compose/)
- [TextField in Jetpack Compose: past, present, and future](https://www.droidcon.com/2023/11/15/textfield-in-jetpack-composepast-present-and-future/)

## TextFieldState

Compose offers a `TextField` component that allows users to input text. We use this component to create forms in our applications.

### 💚 Define a `TextFieldState` to manage the state of a `TextField`

Current implementation of the `TextField` uses a simple state hoisting mechanism with `value` an `onValueChange` parameters. 
However, this solution is widely questioned by the community for being bug prone and difficult to use in more complex scenarios. 

For this reason, the upcoming version of Compose is going to introduce a dedicated `TextFieldState` class which acts as a plain class state holder for a `TextField`. 
Right now we can already implement our own `TextFieldState`. A good example of it can be found in the [compose-samples](https://github.com/android/compose-samples/blob/main/Jetsurvey/app/src/main/java/com/example/compose/jetsurvey/signinsignup/TextFieldState.k) repository.

```kotlin
// Plain class state holder for a TextField
@Stable
open class TextFieldState(initialText: String = "") {
    
    var text: String by mutableStateOf(initialText)
        private set
    
    fun setText(newText: String) {
        text = newText
    }
}

// Saver which allows to save and restore the state
fun textFieldStateSaver() = listSaver(
    save = { listOf(it.text) },
    restore = { TextFieldState(it[0]) }
)

// Helper function to remember the state with given saver
@Composable
fun rememberTextFieldState(initialText: String = ""): TextFieldState {
    return rememberSaveable(initialText, saver = textFieldStateSaver()) {
        TextFieldState(initialText) 
    }
}
```

### 💚 Define a wrapper for a `TextField` that uses `TextFieldState`

```kotlin
@Composable
fun TextField(
    // We can hoist the state holder to the parent composable
    state: TextFieldState = rememberTextFieldState(),
    // Other parameters
) {
    TextField(
        // Original Text field uses the state holder
        value = state.text,
        onValueChange = state::setText,
        // Other parameters
    )
}
```

### 💚 Create more specific states for different types of input

The `TextFieldState` class can be used to create more specific states for different types of input. 
For example, we can create an `EmailFieldState` class which adds validation for email addresses.

```kotlin
class EmailFieldState(initialText: String = "") : TextFieldState(initialText) {
    
    // Additional validation for email addresses
    val isValid: Boolean
        get() = text.contains("@")
}

fun emailFieldStateSaver() = listSaver(
    save = { listOf(it.text) },
    restore = { EmailFieldState(it[0]) }
)

@Composable
fun rememberEmailFieldState(initialText: String = ""): EmailFieldState {
    return rememberSaveable(initialText, saver = emailFieldStateSaver()) {
        EmailFieldState(initialText) 
    }
}
```

### 💚 Combine different states into a form state

When we have multiple `TextFieldState` instances, we can create a `FormState` class to manage the state of the entire form.

```kotlin
@Stable
class LoginForm(
    // Separate states for different types of input
    val email: EmailFieldState = EmailFieldState(),
    val password: PasswordFieldState = PasswordFieldState(),
) {
    
    // Additional validation for the entire form
    val isValid: Boolean
        get() = email.isValid && password.isValid
}
```

## ViewModel

The `TextFieldState` is a great solution for managing the state of a `TextField`. 
But each form has to be submitted at some point and we need some business logic to do that.
This is where the `ViewModel` comes into play.

### 💚 ViewModel don't need to keep the form state

For most cases, the `ViewModel` doesn't need to keep the form state. 
It can only accept entered data in some `submit` method and execute the business logic.
In that case, the `FormState` can be kept directly in the composable function, just like other plain class state holders.

```kotlin
class LoginViewModel : ViewModel() {
    
    fun submit(credentials: Credentials) {
        // Execute the business logic using received data
    }
}
```

```kotlin
@Composable
fun LoginScreen(onSubmit: (Credentials) -> Unit) {
    // Form state is kept directly in the composable function
    val formState = rememberLoginFormState()
    
    Button(
        onClick = {
            // Send entered data in the onSubmit event
            val credentials = Credentials(
                email = formState.email.text,
                password = formState.password.text
            )
            onSubmit(credentials)
        },
        // Button is enabled only when the form is valid
        enabled = formState.isValid
    ) {
        Text("Submit")
    }
}
```

### 💚 `FieldState` or `FormState` can be hoisted to the `ViewModel`

In some cases, it might be necessary to hoist the `FieldState` or `FormState` to the `ViewModel`.
For example to observe the input changes and perform some online validation using a proper business logic.

```kotlin
class LoginViewModel(
    // Business logic for online validation
    private val checkIfEmailIsTakenUseCase: CheckIfEmailIsTakenUseCase
) : ViewModel() {
    
    // Form state is hoisted to the ViewModel
    val formState = LoginFormState()
    
    // Observe input changes and perform online validation
    val isEmailTaken: StateFlow<Boolean> =
        snapshotFlow { formState.email.text }
            .mapLatest { checkIfEmailIsTakenUseCase(it) }
            .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), false)
}
```

### 💔 Don't use `StateFlow` to manage the state of a `TextField`

It was proven that using a `StateFlow` to manage the state of a `TextField` might lead to bugs and unpredictable UI behaviors.
This is why Google is introducing a dedicated `TextFieldState` class in the upcoming version of Compose.
And until this time it is recommended to use a custom plain class state holder for a `TextField`.

```kotlin
data class LoginUiState(
    val email: String = "",
    val password: String = "",
)

class LoginViewModel : ViewModel() {
   
    // State of the form should not be managed by StateFlow 
    private val _uiState = MutableStateFlow(LoginUiState())
    val uiState: StateFlow<LoginUiState> = _uiState
    
    fun onEmailChanged(email: String) {
        _uiState.update {
            it.copy(email = email)
        }
    }
    
    fun onPasswordChanged(password: String) {
        _uiState.update {
            it.copy(password = password)
        }
    }
}
```
