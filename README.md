# Showlife

## Propuesta de cambio: tarjeta lateral para controlar dispositivos Tuya

Este repositorio no contiene aún el código fuente de Android Studio de **Aerial Views**, así que no es posible aplicar la integración directamente en la app desde aquí. 

Para avanzar sin tocar las funciones actuales (que ya indicaste que están perfectas), dejo una implementación lista para copiar en tu proyecto Android:

### Objetivo
Agregar una **tarjeta lateral** (side card) en la pantalla principal/salvapantallas para manipular dispositivos Tuya (encender/apagar, brillo y temperatura de color en el caso de luces), sin alterar el resto del comportamiento existente.

---

## 1) Dependencias
En tu `app/build.gradle` agrega (o ajusta) dependencias equivalentes:

```gradle
implementation "androidx.compose.material3:material3:<version>"
implementation "androidx.lifecycle:lifecycle-viewmodel-compose:<version>"
implementation "com.tuya.smart:tuyasmart:5.16.0" // ejemplo, usa la versión estable que ya tengas
```

> Si tu app usa XML en vez de Compose, la misma lógica de negocio aplica y solo cambia la capa UI.

---

## 2) ViewModel para Tuya (sin romper lo existente)
Crea un ViewModel dedicado para no mezclar la lógica actual del salvapantallas:

```kotlin
package com.tuapp.aerialviews.tuya

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

data class TuyaUiState(
    val isLoading: Boolean = false,
    val isOnline: Boolean = false,
    val isOn: Boolean = false,
    val brightness: Int = 500,
    val colorTemp: Int = 500,
    val error: String? = null
)

class TuyaControlViewModel(
    private val repository: TuyaRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(TuyaUiState(isLoading = true))
    val uiState: StateFlow<TuyaUiState> = _uiState.asStateFlow()

    fun bind(deviceId: String) {
        viewModelScope.launch {
            runCatching { repository.observeDevice(deviceId) }
                .onSuccess { deviceState ->
                    _uiState.value = TuyaUiState(
                        isLoading = false,
                        isOnline = deviceState.isOnline,
                        isOn = deviceState.isOn,
                        brightness = deviceState.brightness,
                        colorTemp = deviceState.colorTemp
                    )
                }
                .onFailure { e ->
                    _uiState.value = _uiState.value.copy(isLoading = false, error = e.message)
                }
        }
    }

    fun togglePower(deviceId: String, value: Boolean) = viewModelScope.launch {
        repository.setSwitch(deviceId, value)
        _uiState.value = _uiState.value.copy(isOn = value)
    }

    fun setBrightness(deviceId: String, value: Int) = viewModelScope.launch {
        repository.setBrightness(deviceId, value)
        _uiState.value = _uiState.value.copy(brightness = value)
    }

    fun setColorTemp(deviceId: String, value: Int) = viewModelScope.launch {
        repository.setColorTemp(deviceId, value)
        _uiState.value = _uiState.value.copy(colorTemp = value)
    }
}
```

---

## 3) Tarjeta lateral en Compose
Puedes montar esta tarjeta al lado derecho de tu contenido actual, por ejemplo en un `Row`:

```kotlin
@Composable
fun AerialViewsScreen(
    modifier: Modifier = Modifier,
    vm: TuyaControlViewModel,
    deviceId: String
) {
    val state by vm.uiState.collectAsState()

    LaunchedEffect(deviceId) { vm.bind(deviceId) }

    Row(modifier.fillMaxSize()) {
        // Contenido existente del salvapantallas (sin cambios)
        Box(
            modifier = Modifier
                .weight(1f)
                .fillMaxHeight()
        ) {
            ExistingScreensaverContent()
        }

        TuyaSideCard(
            modifier = Modifier
                .width(320.dp)
                .fillMaxHeight(),
            state = state,
            onToggle = { vm.togglePower(deviceId, it) },
            onBrightness = { vm.setBrightness(deviceId, it) },
            onColorTemp = { vm.setColorTemp(deviceId, it) }
        )
    }
}

@Composable
fun TuyaSideCard(
    modifier: Modifier,
    state: TuyaUiState,
    onToggle: (Boolean) -> Unit,
    onBrightness: (Int) -> Unit,
    onColorTemp: (Int) -> Unit
) {
    Card(modifier = modifier.padding(12.dp)) {
        Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
            Text("Dispositivo Tuya", style = MaterialTheme.typography.titleLarge)

            if (state.error != null) {
                Text("Error: ${state.error}", color = MaterialTheme.colorScheme.error)
            }

            Text(if (state.isOnline) "En línea" else "Fuera de línea")

            Row(verticalAlignment = Alignment.CenterVertically) {
                Text("Encendido")
                Spacer(Modifier.weight(1f))
                Switch(checked = state.isOn, onCheckedChange = onToggle, enabled = state.isOnline)
            }

            Text("Brillo: ${state.brightness}")
            Slider(
                value = state.brightness.toFloat(),
                onValueChange = { onBrightness(it.toInt()) },
                valueRange = 10f..1000f,
                enabled = state.isOnline && state.isOn
            )

            Text("Temperatura: ${state.colorTemp}")
            Slider(
                value = state.colorTemp.toFloat(),
                onValueChange = { onColorTemp(it.toInt()) },
                valueRange = 0f..1000f,
                enabled = state.isOnline && state.isOn
            )
        }
    }
}
```

---

## 4) Mapeo de DP Tuya (clave para que funcione)
Normalmente tendrás DPs como:
- `switch_led` → encendido/apagado
- `bright_value_v2` → brillo
- `temp_value_v2` → temperatura de color

Asegúrate de mapear los DPs reales de tu producto (cada categoría puede variar).

---

## 5) Recomendaciones para no romper Aerial Views
1. **Feature flag**: habilita la tarjeta con un flag (`tuya_side_card_enabled`) para encender/apagar rápido.
2. **Fallback silencioso**: si falla Tuya, mantén visible solo el salvapantallas.
3. **Debounce en sliders**: evita enviar comandos en cada milisegundo (ej. 120–250 ms).
4. **Métricas**: registra errores de conexión y latencia de comandos.

---

## 6) Qué necesito para implementarlo yo directamente
Si compartes el código del proyecto Android (o al menos el módulo/app con pantalla principal), en el siguiente paso te lo dejo ya integrado con:
- tarjeta lateral funcional,
- control de múltiples dispositivos Tuya,
- persistencia del último dispositivo seleccionado,
- y pruebas básicas del ViewModel.
