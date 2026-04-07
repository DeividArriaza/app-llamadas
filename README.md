# Inspector — Detección de Llamadas Sospechosas

## Contexto Rápido (léelo primero)

**Inspector** es una app Android nativa escrita en **Kotlin** que detecta llamadas entrantes sospechosas (extorsión, spam, fraude). El sistema completo tendrá: app móvil + backend + base de datos compartida de números reportados + sistema de reputación. **Ahora solo estamos construyendo el Módulo 1: detección de llamadas entrantes.**

### Qué hace el Módulo 1

1. Detecta llamadas entrantes en tiempo real usando `CallScreeningService`.
2. Extrae el número de teléfono en formato E.164.
3. Emite un evento interno (`CallEvent`) con el número, timestamp y dirección.
4. Persiste los eventos localmente en Room.
5. Muestra una pantalla Compose con las llamadas detectadas recientes.
6. **Nunca bloquea ni rechaza llamadas** — solo detecta.

### Qué NO hace el Módulo 1

- No conecta con backend.
- No clasifica ni puntúa números.
- No bloquea llamadas.
- No envía notificaciones push.
- No detecta llamadas salientes.

---

## Stack Técnico

| Tecnología | Versión | Motivo |
|---|---|---|
| Kotlin | 2.0+ | Lenguaje principal Android; corrutinas; null safety |
| Jetpack Compose | BOM 2024+ | UI declarativa; sin XML |
| MVVM + Clean Architecture | — | Separación UI → Dominio ← Data |
| Coroutines + Flow | 1.8+ | Concurrencia estructurada; `SharedFlow` para eventos |
| Hilt (Dagger) | 2.51+ | Inyección de dependencias en tiempo de compilación |
| Room | 2.6+ | Persistencia local con soporte de corrutinas |
| CallScreeningService | API 29+ | Servicio del sistema para screening de llamadas |
| Timber | 5.0+ | Logging solo en debug; no-op en release |

**SDK mínimo:** API 29 (Android 10).  
**SDK de compilación:** API 35.

---

## Arquitectura de Detección de Llamadas

### ¿Por qué `CallScreeningService` y no `BroadcastReceiver`?

| Criterio | BroadcastReceiver | CallScreeningService ✅ |
|---|---|---|
| Permisos | Necesita `READ_PHONE_STATE` + `READ_CALL_LOG` (restringidos) | Solo `ROLE_CALL_SCREENING` (sin permisos restringidos) |
| Google Play | Requiere formulario de declaración de permisos | Cumple sin formularios extras |
| Acceso al número | Redactado en API 29+ sin `READ_CALL_LOG` | Proporcionado en `Call.Details` |
| Batería | Debe mantener proceso vivo manualmente | El SO gestiona el ciclo de vida |
| Apps simultáneas | Cualquier app recibe el broadcast | Solo una app tiene el rol — exclusividad |
| Bloqueo futuro | Requiere `MODIFY_PHONE_STATE` (solo sistema) | API nativa `respondToCall()` |

**Decisión:** `CallScreeningService` es el mecanismo elegido porque evita permisos restringidos, tiene cero costo de batería, y Google lo promueve activamente.

**Requisito crítico:** El usuario debe otorgar `ROLE_CALL_SCREENING` a la app mediante un diálogo del sistema. Si rechaza o ya hay otra app con el rol, la detección no funciona.

---

## Flujo del Sistema

```
┌─────────────────────────────────────────────────────────────┐
│                    SISTEMA ANDROID                          │
│                                                             │
│  Llamada entrante → Telecom → Bind CallScreeningService     │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────┐
│  CAPA DE SERVICIO                                            │
│                                                              │
│  IncomingCallScreeningService                                │
│    1. Extrae número de Call.Details                           │
│    2. Construye CallEvent                                    │
│    3. Emite vía CallEventEmitter (SharedFlow)                │
│    4. Responde PERMITIR (siempre en Módulo 1)                │
└────────────────────────────┬─────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────┐
│  CAPA DE DOMINIO (Kotlin puro — sin imports de Android)      │
│                                                              │
│  CallEventEmitter ──► ProcessIncomingCallUseCase             │
│                            │                                 │
│                            └──► CallRepository.save(event)   │
│                                                              │
│  GetRecentCallsUseCase ──► CallRepository.getRecent()        │
└────────────────────────────┬─────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────┐
│  CAPA DE DATOS                                               │
│                                                              │
│  CallRepositoryImpl → Room DAO → SQLite local                │
└────────────────────────────┬─────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────┐
│  CAPA DE UI                                                  │
│                                                              │
│  CallLogViewModel → CallLogScreen (Jetpack Compose)          │
│  Muestra llamadas detectadas en tiempo real                  │
└──────────────────────────────────────────────────────────────┘
```

**Regla de dependencias:**
```
UI → Dominio ← Data
       ↑
     Servicio
```
- UI depende de Dominio. Nunca de Data.
- Data implementa interfaces de Dominio.
- Servicio depende de Dominio. Nunca de UI ni Data.
- Dominio tiene **cero** imports de Android.

---

## Modelo de Datos

```kotlin
data class CallEvent(
    val id: String = UUID.randomUUID().toString(),
    val phoneNumber: String,          // E.164, ej: "+5215512345678"
    val timestamp: Instant,
    val direction: CallDirection,     // INCOMING
    val presentationType: Int,        // de Call.Details
)

enum class CallDirection { INCOMING, OUTGOING }
```

---

## Estructura del Proyecto

```
com.inspector.app/
│
├── InspectorApplication.kt               # Entry point Hilt
│
├── di/                                    # Módulos de inyección
│   ├── AppModule.kt
│   ├── DatabaseModule.kt
│   └── ServiceModule.kt
│
├── service/                               # Componentes Android
│   └── IncomingCallScreeningService.kt    # CallScreeningService
│
├── domain/                                # Kotlin puro — sin Android
│   ├── model/
│   │   ├── CallEvent.kt
│   │   └── CallDirection.kt
│   ├── repository/
│   │   └── CallRepository.kt             # Interfaz
│   ├── usecase/
│   │   ├── ProcessIncomingCallUseCase.kt
│   │   └── GetRecentCallsUseCase.kt
│   └── event/
│       └── CallEventEmitter.kt            # SharedFlow
│
├── data/                                  # Implementaciones
│   ├── local/
│   │   ├── CallDatabase.kt
│   │   ├── CallDao.kt
│   │   └── CallEntity.kt
│   ├── mapper/
│   │   └── CallMapper.kt
│   └── repository/
│       └── CallRepositoryImpl.kt
│
└── ui/
    ├── navigation/
    │   └── AppNavGraph.kt
    ├── screen/
    │   └── calllog/
    │       ├── CallLogScreen.kt
    │       └── CallLogViewModel.kt
    ├── component/
    │   └── CallEventCard.kt
    └── theme/
        ├── Theme.kt
        ├── Color.kt
        └── Type.kt
```

---

## Componentes Clave

### 1. IncomingCallScreeningService

Punto de entrada. El SO lo vincula cuando llega una llamada.

```kotlin
@AndroidEntryPoint
class IncomingCallScreeningService : CallScreeningService() {

    @Inject lateinit var callEventEmitter: CallEventEmitter

    override fun onScreenCall(callDetails: Call.Details) {
        val number = callDetails.handle?.schemeSpecificPart
        if (number != null) {
            val event = CallEvent(
                phoneNumber = number,
                timestamp = Instant.now(),
                direction = CallDirection.INCOMING,
                presentationType = callDetails.callerNumberVerificationStatus
            )
            CoroutineScope(Dispatchers.Default + SupervisorJob()).launch {
                callEventEmitter.emit(event)
            }
        }
        // SIEMPRE permitir la llamada en Módulo 1
        respondToCall(callDetails, CallResponse.Builder().build())
    }
}
```

**Manifest:**
```xml
<service
    android:name=".service.IncomingCallScreeningService"
    android:permission="android.permission.BIND_SCREENING_SERVICE"
    android:exported="true">
    <intent-filter>
        <action android:name="android.telecom.CallScreeningService" />
    </intent-filter>
</service>
```

### 2. CallEventEmitter

Bus de eventos reactivo singleton:

```kotlin
@Singleton
class CallEventEmitter @Inject constructor() {
    private val _events = MutableSharedFlow<CallEvent>(
        replay = 0,
        extraBufferCapacity = 64,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events: SharedFlow<CallEvent> = _events.asSharedFlow()

    suspend fun emit(event: CallEvent) = _events.emit(event)
}
```

### 3. Repository

```kotlin
// domain/repository/CallRepository.kt — Interfaz
interface CallRepository {
    suspend fun save(event: CallEvent)
    fun getRecentCalls(limit: Int = 50): Flow<List<CallEvent>>
}

// data/repository/CallRepositoryImpl.kt — Implementación
class CallRepositoryImpl @Inject constructor(
    private val dao: CallDao,
    private val mapper: CallMapper
) : CallRepository {
    override suspend fun save(event: CallEvent) = dao.insert(mapper.toEntity(event))
    override fun getRecentCalls(limit: Int) = dao.getRecent(limit).map { it.map(mapper::toDomain) }
}
```

### 4. Casos de Uso

```kotlin
class ProcessIncomingCallUseCase @Inject constructor(
    private val repository: CallRepository
) {
    suspend operator fun invoke(event: CallEvent) {
        repository.save(event)
        // Futuro: clasificar número aquí
    }
}

class GetRecentCallsUseCase @Inject constructor(
    private val repository: CallRepository
) {
    operator fun invoke(limit: Int = 50): Flow<List<CallEvent>> = repository.getRecentCalls(limit)
}
```

### 5. ViewModel

```kotlin
@HiltViewModel
class CallLogViewModel @Inject constructor(
    getRecentCalls: GetRecentCallsUseCase,
    callEventEmitter: CallEventEmitter,
    processIncomingCall: ProcessIncomingCallUseCase
) : ViewModel() {

    val recentCalls: StateFlow<List<CallEvent>> = getRecentCalls()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    init {
        viewModelScope.launch {
            callEventEmitter.events.collect { event ->
                processIncomingCall(event)
            }
        }
    }
}
```

---

## Permisos y Seguridad

### Permisos Requeridos

| Permiso | Tipo | ¿Necesario? | Notas |
|---|---|---|---|
| `BIND_SCREENING_SERVICE` | Firma (sistema) | Sí | Declarado en manifest; el sistema lo enforce |
| `ROLE_CALL_SCREENING` | Rol (API 29+) | Sí | Solicitado vía `RoleManager`; el usuario acepta en diálogo del sistema |
| `READ_PHONE_STATE` | Runtime, restringido | **No** | No necesario con `CallScreeningService` |
| `READ_CALL_LOG` | Runtime, muy restringido | **No** | No necesario para este módulo |

### Solicitud del Rol de Screening

```kotlin
val roleManager = getSystemService(RoleManager::class.java)
if (roleManager.isRoleAvailable(RoleManager.ROLE_CALL_SCREENING) &&
    !roleManager.isRoleHeld(RoleManager.ROLE_CALL_SCREENING)) {
    val intent = roleManager.createRequestRoleIntent(RoleManager.ROLE_CALL_SCREENING)
    startActivityForResult(intent, REQUEST_CODE_SCREENING_ROLE)
}
```

### Política de Google Play — CRÍTICO

1. **`READ_CALL_LOG` y `READ_PHONE_STATE` son permisos restringidos.** Apps que los usan deben enviar un formulario de declaración y demostrar necesidad. **Inspector NO los usa.**
2. **`CallScreeningService` es el mecanismo recomendado por Google** para apps de identificación de llamadas y spam.
3. **El propósito principal de la app debe ser screening de llamadas.** No abusar del rol para funcionalidad no relacionada.
4. **Números de teléfono son datos personales** (GDPR, CCPA, política de datos de usuario de Google Play):
   - Almacenamiento local cifrado (SQLCipher o `EncryptedSharedPreferences`).
   - Declarado en política de privacidad.
   - El usuario puede eliminar sus datos.

### Medidas de Seguridad

| Medida | Implementación |
|---|---|
| Cifrado local | Room + SQLCipher o `EncryptedSharedPreferences` |
| Sin PII en logs | Timber `Tree` que elimina números en builds de release |
| Ofuscación | ProGuard/R8 habilitado; servicio de screening excluido |
| Control de exportación | Servicio exportado con guard de permiso `BIND_SCREENING_SERVICE` |

---

## Limitaciones y Restricciones

### Versiones de Android

| API | Impacto |
|---|---|
| < 24 | `CallScreeningService` no existe — **no soportado** |
| 24–28 | Existe pero `Call.Details.handle` puede ser `null` según OEM. No confiable |
| **29+ (Android 10+)** | **Soporte completo.** `RoleManager` disponible. Número confiable |

### Dispositivos

- **Dual SIM:** `Call.Details` incluye `getAccountHandle()` para identificar qué SIM recibió la llamada. Se captura pero no se actúa.
- **Screening de operadora:** Algunas operadoras (T-Mobile Scam Shield, AT&T Call Protect) interceptan antes que `CallScreeningService`. Fuera del control de la app.
- **Modificaciones OEM:** Xiaomi, Huawei, Samsung modifican el framework Telecom. Pruebas en estos OEM son críticas.

### Acceso al Número

- **Números privados/restringidos:** `Call.Details.handle` será `null` o tipo `PRESENTATION_RESTRICTED`. La app lo maneja sin número.
- **Llamadas VOIP:** Algunas no pasan por el framework Telecom estándar y no activan `CallScreeningService`.
- **Wi-Fi Calling:** Generalmente funciona porque pasa por IMS de la operadora, pero varía por OEM.

### Restricción de App Única

Solo **una** app puede tener `ROLE_CALL_SCREENING` a la vez. Si el usuario tiene TrueCaller, Hiya, etc., debe elegir. Es limitación de plataforma sin workaround.

---

## Puntos de Integración Futura

### Backend API

La interfaz `CallRepository` es la costura de integración:

```
Actual:    CallRepositoryImpl → Room (solo local)
Futuro:    CallRepositoryImpl → Room + RemoteDataSource (API)
```

Pasos cuando el backend esté listo:
1. Crear `RemoteCallDataSource` que llame al API.
2. Modificar `CallRepositoryImpl` para persistir local **y** enviar al remoto.
3. Sin cambios en dominio ni servicio.

### Clasificación de Números

`ProcessIncomingCallUseCase` es el punto de integración:

```
Actual:    ProcessIncomingCallUseCase → save(event)
Futuro:    ProcessIncomingCallUseCase → save(event) + classifyNumber(phoneNumber)
```

Un nuevo `ClassifyNumberUseCase` consultará caché local → backend → retornará clasificación (seguro, sospechoso, peligroso, desconocido).

### Bloqueo de Llamadas (Módulo Futuro)

```kotlin
// Futuro — NO implementado en Módulo 1
val response = CallResponse.Builder()
    .setDisallowCall(true)
    .setRejectCall(true)
    .setSkipNotification(true)
    .build()
respondToCall(callDetails, response)
```

---

## Definición del MVP (Módulo 1)

### Criterios de Aceptación

El módulo está **terminado** cuando se cumplen todos:

| # | Criterio | Verificación |
|---|---|---|
| 1 | App compila y ejecuta en emulador API 29+ y al menos un dispositivo físico | Build CI + prueba manual |
| 2 | App solicita `ROLE_CALL_SCREENING` en primer lanzamiento con UX clara | Prueba manual |
| 3 | Cuando el rol está otorgado y llega una llamada, `onScreenCall` se invoca | Test instrumentado + Logcat |
| 4 | Número de teléfono extraído correctamente y emitido como `CallEvent` | Test unitario del mapper + test instrumentado |
| 5 | Todas las llamadas se **permiten** (nunca se bloquean o rechazan) | Code review + test instrumentado |
| 6 | `CallEvent` persiste en base de datos Room local | Test unitario del repositorio + test DAO |
| 7 | Pantalla Compose muestra lista de llamadas detectadas en tiempo real | Prueba manual |
| 8 | App maneja números `null` (privados/restringidos) sin crash | Test unitario |
| 9 | No aparecen números de teléfono en Logcat en builds de release | Verificación manual |
| 10 | App maneja rechazo del rol con gracia (muestra explicación, no crashea) | Prueba manual |

### Fuera del MVP

- Bloqueo o rechazo de llamadas
- Llamadas al backend API
- Búsqueda de reputación de números
- Reporte de números por el usuario
- Notificaciones push
- Widget u overlay durante llamadas
- Detección de llamadas salientes
- Soporte multi-idioma

### Estrategia de Testing

| Capa | Herramienta | Alcance |
|---|---|---|
| Dominio (use cases, mappers) | JUnit 5 + MockK | Tests unitarios — sin dependencias Android |
| Datos (Room DAO, repositorio) | AndroidX Test + Room in-memory | Tests instrumentados |
| Servicio (onScreenCall) | Robolectric o test instrumentado con mock `Call.Details` | Test de integración |
| UI (pantallas Compose) | Compose UI Test | Screenshot + tests de interacción |
| End-to-end | QA manual con simulación ADB | `adb shell am start -a android.intent.action.CALL -d tel:+5215551234567` |
