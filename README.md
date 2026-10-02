# Guía paso a paso: MVVM + Navigation Compose con ViewModel compartido

**Proyecto:** PrimerApp (`com.viajando.primerapp`)
**Curso:** Desarrollo Android — 1er ramo, 4to semestre
**Requisito previo:** Kotlin básico/intermedio (ya lo manejas)

Objetivo del ejercicio: dos pantallas (`InputScreen` y `DisplayScreen`) que comparten datos a través de **un mismo ViewModel**, navegando entre ellas con **Jetpack Navigation Compose**, sin pasar argumentos por la ruta.

---

## Parte 1 — Conceptos

### 1.1 ¿Qué es MVVM?

MVVM separa la app en tres capas para que cada una tenga una sola responsabilidad:

- **Model**: los datos puros y la lógica de negocio/datos (clases, repos, llamadas a red o BD). No sabe que existe una UI.
- **View** (en Compose, tus `@Composable`): solo dibuja lo que el ViewModel le dice y reporta eventos del usuario (clicks, texto escrito). No contiene lógica ni guarda estado "de verdad".
- **ViewModel**: el puente. Expone el estado de la UI (`uiState`) como algo observable, y expone funciones (`onNombreChanged()`, `saveData()`) que la View llama cuando el usuario interactúa. Sobrevive a cambios de configuración (rotación) porque su ciclo de vida está atado al `ViewModelStoreOwner`, no al Composable.

Flujo unidireccional (UDF — Unidirectional Data Flow):

```
Usuario interactúa → View llama función del ViewModel → ViewModel actualiza _uiState
→ Compose recompone automáticamente (porque observa uiState) → View se repinta
```

La View **nunca** modifica el estado directamente. Siempre pide al ViewModel que lo haga. Esto hace la UI predecible y testeable (puedes probar el ViewModel sin levantar ninguna pantalla).

### 1.2 `StateFlow` y el patrón UiState

El ViewModel no expone variables sueltas, expone **un solo objeto de estado** (`data class` con todos los valores que la pantalla necesita):

```kotlin
data class FormUiState(
    val nombre: String = "",
    val edad: String = "",
    val mensajeGuardado: String = ""
)
```

Y lo publica con `StateFlow`, que es observable y siempre tiene un valor actual:

```kotlin
private val _uiState = MutableStateFlow(FormUiState()) // privado y mutable, solo el VM escribe
val uiState: StateFlow<FormUiState> = _uiState.asStateFlow() // público e inmutable, la View solo lee
```

Esto se llama **encapsulamiento de estado**: afuera nadie puede hacer `uiState.nombre = "x"` directamente, solo a través de funciones del ViewModel. En el Composable se consume con `collectAsState()`, que suscribe la UI a los cambios y gatilla recomposición.

### 1.3 ¿Cómo se comparte el mismo ViewModel entre dos pantallas?

Si cada pantalla pidiera su propio `viewModel()`, cada una tendría una instancia **distinta** y los datos no viajarían entre ellas.

La solución: **crear la instancia una sola vez, en el nivel del `NavHost`**, y pasarla como parámetro a cada pantalla.

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    val sharedViewModel: SharedFormViewModel = viewModel() // 1 sola instancia

    NavHost(navController = navController, startDestination = "input") {
        composable("input") { InputScreen(viewModel = sharedViewModel, ...) }
        composable("display") { DisplayScreen(viewModel = sharedViewModel, ...) }
    }
}
```

`viewModel()` busca (o crea) la instancia en el `ViewModelStoreOwner` más cercano — en este caso, la Activity que contiene el `NavHost`. Como esa Activity vive mientras la app está abierta, el ViewModel **sobrevive** al navegar entre `input` y `display`. Por eso, cuando `InputScreen` guarda datos y navegas a `DisplayScreen`, esta última lee el `uiState` y ya tiene los valores actualizados, sin pasarlos por la ruta de navegación.

> Si en vez de esto cada pantalla llamara a `viewModel()` por su cuenta, cada `composable {}` del `NavHost` crea su propio `ViewModelStoreOwner` por defecto y obtendrías instancias distintas y datos perdidos al navegar.

### 1.4 `NavController`: pila de navegación

El `NavController` administra un **back stack** (pila) de pantallas:

- `navController.navigate("display")` → agrega "display" arriba de la pila y la muestra.
- `navController.popBackStack()` → sacar la pantalla actual de la pila y volver a la anterior (como el botón atrás del sistema, pero desde código).

### 1.5 Desacoplar la View del NavController (lambdas)

Una pantalla (`InputScreen`, `DisplayScreen`) **no debería conocer** al `NavController`. Si lo conociera, no podrías reusar esa pantalla en otro grafo de navegación ni probarla aislada. Por eso, la pantalla solo recibe una **lambda**:

```kotlin
@Composable
fun InputScreen(
    viewModel: SharedFormViewModel,
    onNavigateToDisplay: () -> Unit // la pantalla solo "avisa", no sabe cómo se navega
)
```

Y es el `NavHost` (quien sí conoce el `navController`) el que decide qué hacer cuando esa lambda se ejecuta:

```kotlin
composable("input") {
    InputScreen(
        viewModel = sharedViewModel,
        onNavigateToDisplay = { navController.navigate("display") }
    )
}
```

### 1.6 Resumen del flujo completo

1. El usuario escribe en `InputScreen` → se llama `viewModel.onNombreChanged(...)`.
2. El ViewModel actualiza `_uiState` con `.update { it.copy(...) }`.
3. Compose recompone `InputScreen` porque está leyendo `uiState` con `collectAsState()`.
4. El usuario pulsa "Guardar y Ver" → `viewModel.saveData()` y luego `onNavigateToDisplay()`.
5. El `NavHost` ejecuta `navController.navigate("display")`.
6. `DisplayScreen` se compone, lee el **mismo** `sharedViewModel.uiState` y ya tiene los datos guardados.
7. Al pulsar "Volver", se llama `onBack()` → el `NavHost` ejecuta `navController.popBackStack()`.

---

## Parte 2 — Configuración del proyecto

### 2.1 SDK

En `app/build.gradle.kts`, deja:

```kotlin
android {
    compileSdk = 37
    defaultConfig {
        minSdk = 37
        ...
    }
}
```

### 2.2 Librerías a instalar

El proyecto ya trae Compose, Material3 y `lifecycle-runtime-ktx`. Falta agregar **2 librerías**: Navigation Compose y el helper de ViewModel para Compose.

Edita `gradle/libs.versions.toml`:

```toml
[versions]
# ...versiones existentes...
navigationCompose = "2.9.7"

[libraries]
# ...librerías existentes...
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigationCompose" }
androidx-lifecycle-viewmodel-compose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycleRuntimeKtx" }
```

> `lifecycle-viewmodel-compose` reutiliza la misma versión (`lifecycleRuntimeKtx`) que ya usa `lifecycle-runtime-ktx` en este proyecto, para mantener todo el paquete `androidx.lifecycle` alineado.

Edita `app/build.gradle.kts` → bloque `dependencies`, agrega:

```kotlin
dependencies {
    // ...dependencias existentes...
    implementation(libs.androidx.navigation.compose)
    implementation(libs.androidx.lifecycle.viewmodel.compose)
}
```

Sincroniza Gradle (`Sync Now` en Android Studio) antes de seguir.

---

## Parte 3 — Implementación paso a paso

Estructura de carpetas sugerida dentro de `app/src/main/java/com/viajando/primerapp/`:

```
primerapp/
├── MainActivity.kt
├── ui/
│   ├── SharedFormViewModel.kt
│   ├── AppNavigation.kt
│   └── screens/
│       ├── InputScreen.kt
│       └── DisplayScreen.kt
```

### Paso 1 — Crear el `UiState` y el `ViewModel`

Archivo: `ui/SharedFormViewModel.kt`

```kotlin
package com.viajando.primerapp.ui

import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update

data class FormUiState(
    val nombre: String = "",
    val edad: String = "",
    val mensajeGuardado: String = ""
)

class SharedFormViewModel : ViewModel() {

    private val _uiState = MutableStateFlow(FormUiState())
    val uiState: StateFlow<FormUiState> = _uiState.asStateFlow()

    fun onNombreChanged(valor: String) {
        _uiState.update { it.copy(nombre = valor) }
    }

    fun onEdadChanged(valor: String) {
        _uiState.update { it.copy(edad = valor) }
    }

    fun saveData() {
        _uiState.update { it.copy(mensajeGuardado = "Datos guardados correctamente ✅") }
    }
}
```

### Paso 2 — Crear `InputScreen`

Archivo: `ui/screens/InputScreen.kt`

```kotlin
package com.viajando.primerapp.ui.screens

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import com.viajando.primerapp.ui.SharedFormViewModel

@Composable
fun InputScreen(
    viewModel: SharedFormViewModel,
    onNavigateToDisplay: () -> Unit
) {
    val uiState by viewModel.uiState.collectAsState()

    Column(
        modifier = Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.Center
    ) {
        Text("Ingresa tus datos", style = MaterialTheme.typography.headlineSmall)
        Spacer(Modifier.height(16.dp))

        OutlinedTextField(
            value = uiState.nombre,
            onValueChange = viewModel::onNombreChanged,
            label = { Text("Nombre") },
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(8.dp))

        OutlinedTextField(
            value = uiState.edad,
            onValueChange = viewModel::onEdadChanged,
            label = { Text("Edad") },
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(24.dp))

        Button(
            onClick = {
                viewModel.saveData()
                onNavigateToDisplay()
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("Guardar y Ver")
        }
    }
}
```

Nota: `InputScreen` no importa nada de `androidx.navigation`. No sabe qué es un `NavController`. Solo recibe una lambda.

### Paso 3 — Crear `DisplayScreen`

Archivo: `ui/screens/DisplayScreen.kt`

```kotlin
package com.viajando.primerapp.ui.screens

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import com.viajando.primerapp.ui.SharedFormViewModel

@Composable
fun DisplayScreen(
    viewModel: SharedFormViewModel,
    onBack: () -> Unit
) {
    val uiState by viewModel.uiState.collectAsState()

    Column(
        modifier = Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.Center
    ) {
        Text("Resumen", style = MaterialTheme.typography.headlineSmall)
        Spacer(Modifier.height(16.dp))

        Text("Nombre: ${uiState.nombre}")
        Text("Edad: ${uiState.edad}")
        Text(uiState.mensajeGuardado)

        Spacer(Modifier.height(24.dp))
        Button(onClick = onBack, modifier = Modifier.fillMaxWidth()) {
            Text("Volver")
        }
    }
}
```

`DisplayScreen` lee `viewModel.uiState` de la misma forma que `InputScreen`. Como es el **mismo** ViewModel (ver Parte 1.3), ya tiene los datos sin que se los hayas pasado por parámetro de ruta.

### Paso 4 — Crear el grafo de navegación

Archivo: `ui/AppNavigation.kt`

```kotlin
package com.viajando.primerapp.ui

import androidx.compose.runtime.Composable
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import com.viajando.primerapp.ui.screens.DisplayScreen
import com.viajando.primerapp.ui.screens.InputScreen

@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    val sharedViewModel: SharedFormViewModel = viewModel()

    NavHost(navController = navController, startDestination = "input") {
        composable("input") {
            InputScreen(
                viewModel = sharedViewModel,
                onNavigateToDisplay = { navController.navigate("display") }
            )
        }
        composable("display") {
            DisplayScreen(
                viewModel = sharedViewModel,
                onBack = { navController.popBackStack() }
            )
        }
    }
}
```

### Paso 5 — Conectar todo en `MainActivity`

Reemplaza el contenido de `MainActivity.kt`:

```kotlin
package com.viajando.primerapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import com.viajando.primerapp.ui.AppNavigation
import com.viajando.primerapp.ui.theme.PrimerAppTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            PrimerAppTheme {
                AppNavigation()
            }
        }
    }
}
```

Puedes borrar las funciones `Greeting` y `GreetingPreview` que venían de la plantilla por defecto (o dejarlas sin usar, no afectan la compilación).

### Paso 6 — Ejecutar y probar

1. `Build > Sync Project with Gradle Files`.
2. Ejecuta la app en un emulador o dispositivo.
3. Escribe un nombre y una edad en la primera pantalla.
4. Pulsa "Guardar y Ver" → debes llegar a la pantalla de resumen con esos mismos datos.
5. Pulsa "Volver" → regresas a la primera pantalla. Los campos deben seguir ahí (porque el ViewModel nunca se destruyó).

---

## Parte 4 — Checklist de verificación

- [ ] `compileSdk = 37` y `minSdk = 37` en `app/build.gradle.kts`.
- [ ] `navigation-compose` y `lifecycle-viewmodel-compose` agregados en `libs.versions.toml` y en `dependencies {}`.
- [ ] `SharedFormViewModel` expone `uiState` como `StateFlow`, nunca como `var` público.
- [ ] `InputScreen` y `DisplayScreen` no importan `androidx.navigation.*` (reciben lambdas, no el `NavController`).
- [ ] `viewModel()` se llama **una sola vez**, dentro de `AppNavigation()`, no dentro de cada pantalla.
- [ ] Navegar de ida y vuelta conserva los datos sin pasarlos por la ruta.

## Parte 5 — Retos para profundizar (opcional)

1. Agrega un tercer campo (ej: email) con validación básica (que no esté vacío) antes de habilitar el botón "Guardar".
2. Reemplaza las rutas de texto plano (`"input"`, `"display"`) por una `sealed class Screen(val route: String)` — más seguro ante errores de tipeo.
3. Agrega un botón "Limpiar" en `DisplayScreen` que resetee el `uiState` a `FormUiState()` usando una nueva función en el ViewModel.
4. Investiga `navController.navigate(route) { popUpTo("input") { inclusive = false } }` y qué problema de pila resuelve.
