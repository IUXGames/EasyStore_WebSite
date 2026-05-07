<!-- doc-shell:page slug="introduccion" -->

# Introducción

[![Godot 4](https://img.shields.io/badge/Godot-4.4+-478cbf?logo=godotengine&logoColor=white)](https://godotengine.org/)
[![Versión](https://img.shields.io/badge/versión-1.0.0-3498db)](./plugin.cfg)
[![Licencia](https://img.shields.io/badge/licencia-MIT-green)](./LICENSE)

**EasyStore** es un sistema de guardado modular para **Godot 4** que unifica el almacenamiento local y en la nube bajo una única API limpia. En lugar de escribir tu propia lógica de I/O de archivos, cifrado, sincronización en la nube o migración, tu código de juego solo habla con EasyStore — y EasyStore se encarga del resto.

## Versiones y compatibilidad

| Elemento | Valor |
|----------|-------|
| **Versión del addon** | **1.0.0** |
| **Versión de Godot objetivo** | **Godot 4.4+** |
| **GDScript** | 4.x (tipado estático) |

## Filosofía

> Una API pública. Múltiples backends intercambiables. Cero código repetitivo.

EasyStore se basa en tres ideas fundamentales:

1. **Abstracción total del almacenamiento** — Tu juego llama a `EasyStore.save("player", data)`. Si eso escribe un archivo local, sube a Steam Cloud, o ambos a la vez, es invisible para tu código de juego.
2. **Resistencia por defecto** — Si el backend de Steam falla (no instalado, Steam sin conexión, cuota excedida), EasyStore continúa con los backends que estén disponibles.
3. **Amigable para principiantes, extensible para expertos** — Cero configuración te da un guardado local funcional en segundos. Los usuarios avanzados pueden añadir cifrado, migraciones personalizadas, gestión de múltiples slots y sus propios backends cloud.

## Arquitectura por capas

```
┌──────────────────────────────────────┐
│           Tu código de juego         │  ← Solo interactúas aquí
├──────────────────────────────────────┤
│        EasyStore — API Pública       │  easystore.gd (Autoload)
├──────────────────────────────────────┤
│         Subsistemas internos         │  SlotManager, SectionCache,
│                                      │  MigrationManager, SyncManager,
│                                      │  AutosaveTimer, AsyncWorker, etc.
├──────────────────────────────────────┤
│          Backends de almacenamiento  │  StorageBackend (abstracto)
├──────────────────────────────────────┤
│      Backend(s) activo(s)            │  Local (archivo) / Steam Cloud / personalizado
└──────────────────────────────────────┘
```

## Backends disponibles

| Backend | Estado | Descripción |
|---------|--------|-------------|
| **Local** | ✅ Disponible | Guarda en `user://saves/`. Soporta cifrado, copias de seguridad y formato JSON. |
| **Steam Cloud** | ✅ Disponible | Guarda en Steam Remote Storage mediante GodotSteam. Requiere GodotSteam GDExtension 4.4+. |

## Características principales

- **API unificada** — `save()`, `load()`, `delete_slot()`: las mismas llamadas independientemente de los backends activos.
- **Múltiples backends simultáneos** — Usa Local + Steam Cloud al mismo tiempo. Cada `save()` escribe en ambos en paralelo.
- **Resiliencia sin conexión** — Si Steam no está disponible, EasyStore recurre al almacenamiento local de forma transparente.
- **Guardado por secciones** — Divide los datos de guardado en secciones con nombre (`"player"`, `"world"`, `"settings"`). Carga solo lo que necesites.
- **Caché en memoria** — Las lecturas son instantáneas tras la primera carga. Las escrituras van primero a memoria (síncronas) y luego se vuelcan a disco/nube de forma asíncrona.
- **Slots de guardado** — Soporte multi-slot integrado (slot 0, 1, 2…). Lista, elimina y cambia entre ellos en tiempo de ejecución.
- **Metadatos de guardado** — Cada slot tiene un sidecar ligero con timestamp, tiempo de juego, campos personalizados y una ruta de captura de pantalla.
- **Auto-guardado** — Una sola llamada inicia un temporizador en segundo plano que guarda automáticamente en un intervalo configurable.
- **Migraciones de guardado** — Registra callbacks con conciencia de versión para transformar datos de guardado antiguos al cargarlos. Nunca rompas un guardado antiguo otra vez.
- **Resolución de conflictos** — Al sincronizar múltiples backends, elige una estrategia: gana el más nuevo, gana la nube, gana el local, o resolución manual.
- **Cifrado opcional** — Cifra los guardados locales con una contraseña usando `FileAccess.open_encrypted_with_pass()` de Godot.
- **I/O asíncrono** — Todas las operaciones de disco se ejecutan en un hilo secundario (via `AsyncWorker`). El hilo principal nunca se bloquea.
- **Extensible** — Añade tu propio backend cloud extendiendo `StorageBackend` e implementando 8 métodos virtuales.

## Requisitos

| Elemento | Necesario para | Notas |
|----------|---------------|-------|
| **Godot 4.4+** | Todo | |
| **GodotSteam GDExtension 4.4+** | Backend Steam únicamente | Plugin de [Gramps](https://godotsteam.com/). Opcional — EasyStore no da errores sin él. |
| **Cliente Steam en ejecución** | Backend Steam únicamente | Debe estar abierto y con sesión iniciada. |

---

<!-- doc-shell:page slug="instalacion" -->

# Instalación

## Paso 1 — Copiar el addon

Copia la carpeta `addons/easystore/` en tu proyecto de Godot:

```
tu-proyecto/
└── addons/
    └── easystore/       ← pega la carpeta completa aquí
        ├── easystore.gd
        ├── easystore.tscn
        ├── plugin.gd
        ├── plugin.cfg
        ├── core/
        ├── backends/
        ├── config/
        ├── subsystems/
        ├── debug/
        └── nodes/
```

## Paso 2 — Activar el plugin

Ve a **Proyecto → Configuración del Proyecto → Plugins** y activa **EasyStore**:

```
[ EasyStore ]  [ activar ]
```

Al activarlo, el plugin registra el autoload **`EasyStore`**, accesible globalmente desde cualquier script.

## Paso 3 — Verificar el autoload

En **Proyecto → Configuración del Proyecto → Autoloads** deberías ver:

```
EasyStore   res://addons/easystore/easystore.tscn   ✓
```

> El autoload se añade automáticamente. **No** necesitas añadirlo manualmente.

## Paso 4 — Inicializar en tu juego

EasyStore **no se configura solo**. Llama a `EasyStore.initialize()` una vez al inicio (típicamente en tu propio Autoload), luego añade los backends que quieras:

```gdscript
# GLOBAL.gd — Tu autoload del juego
extends Node

func _ready() -> void:
    _inicializar_easystore()

func _inicializar_easystore() -> void:
    # Opcional: usa un recurso de configuración para personalizar el comportamiento
    var config := EasyStoreConfig.new()
    config.log_level = StoreEnums.LogLevel.DEBUG

    # await es necesario: initialize() puede suspenderse internamente si el nodo
    # EasyStore aún no está listo por el orden de los autoloads.
    await EasyStore.initialize(config)

    # Backend Local: siempre disponible, sin dependencias adicionales
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)

    # Backend Steam: se añade de forma segura aunque Steam no esté disponible
    # Si Steam no está disponible, add_backend() advierte y continúa sin fallar
    EasyStore.add_backend(StoreEnums.BackendType.STEAM_CLOUD)
```

> **Configuración mínima (sin config):** llamar a `EasyStore.initialize()` + `EasyStore.add_backend(StoreEnums.BackendType.LOCAL)` sin argumentos es suficiente para tener un sistema de guardado funcional con la configuración predeterminada.

---

<!-- doc-shell:page slug="quickstart" -->

# Inicio Rápido

Esta guía muestra el código mínimo para guardar y cargar datos del juego en menos de 5 minutos.

## 1 — Guardar datos

```gdscript
# Guarda un diccionario bajo el nombre de sección "player"
EasyStore.save("player", {
    "salud":   100,
    "nivel":   3,
    "monedas": 450,
})
```

`save()` es **síncrono desde la perspectiva de tu juego**: escribe en la caché en memoria de forma instantánea, luego vuelca a todos los backends activos de forma asíncrona (sin caída de FPS).

## 2 — Esperar a que el guardado termine (opcional)

Si necesitas saber cuándo termina la escritura física (por ejemplo, para mostrar un indicador "¡Guardado!"):

```gdscript
EasyStore.save_started.connect(func(slot): $IconoGuardar.show())
EasyStore.save_completed.connect(func(slot, exito):
    $IconoGuardar.hide()
    if not exito:
        $UI.mostrar_error("¡El guardado falló!")
)
```

## 3 — Cargar datos

```gdscript
# Lanza una carga asíncrona para el slot actual
EasyStore.load_all()

# Espera el resultado
await EasyStore.load_completed

# Ahora lee de la caché — instantáneo
var player := EasyStore.load("player")
print("Salud: ", player.get("salud", 100))
```

O usa el patrón de señal:

```gdscript
func _ready() -> void:
    EasyStore.load_completed.connect(_al_cargar)
    EasyStore.load_all()

func _al_cargar(slot: int, data: Dictionary, exito: bool) -> void:
    if not exito:
        print("No se encontró guardado — empezando desde cero")
        return
    var player := data.get("player", {})
    salud = player.get("salud", 100)
    nivel = player.get("nivel", 1)
```

## 4 — Ejemplo mínimo completo

```gdscript
# GLOBAL.gd
extends Node

func _ready() -> void:
    await EasyStore.initialize()
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)

    # Cargar cualquier dato existente
    EasyStore.load_completed.connect(_al_cargar)
    EasyStore.load_all()

func _al_cargar(slot: int, data: Dictionary, exito: bool) -> void:
    if exito:
        print("¡Cargado! Secciones: ", data.keys())
    else:
        print("No hay guardado para el slot ", slot)

# Llamado por tu juego cuando necesita guardar
func guardar_juego() -> void:
    EasyStore.save("player", { "salud": 100 })
    EasyStore.save("world",  { "nombre_nivel": "bosque_01" })
```

---

<!-- doc-shell:page slug="configuracion" -->

# Configuración

EasyStore se configura a través de recursos (`Resource`). Toda la configuración es opcional — se aplican valores predeterminados razonables automáticamente.

## EasyStoreConfig

El recurso de configuración principal, que se pasa a `EasyStore.initialize(config)`.

| Propiedad | Tipo | Predeterminado | Descripción |
|-----------|------|----------------|-------------|
| `default_slot` | `int` | `0` | El slot de guardado que se activa después de `initialize()`. |
| `conflict_strategy` | `StoreEnums.ConflictStrategy` | `NEWEST_WINS` | Cómo resolver conflictos al sincronizar múltiples backends. |
| `current_save_version` | `int` | `1` | Versión actual del formato de guardado. Usada por el sistema de migración. |
| `log_level` | `int` | `StoreEnums.LogLevel.INFO` | Nivel de verbosidad del log. `NONE` = silencioso, `INFO` = recomendado para desarrollo, `DEBUG` = detallado. Ver `StoreEnums.LogLevel`. |
| `local` | `LocalBackendConfig` | `null` | Configuración del backend local (usa predeterminados si es `null`). |
| `steam` | `SteamCloudBackendConfig` | `null` | Configuración del backend Steam (usa predeterminados si es `null`). |

**Estrategias de conflicto:**

| Valor | Constante | Descripción |
|-------|-----------|-------------|
| `0` | `NEWEST_WINS` | El backend con el timestamp más reciente gana. |
| `1` | `CLOUD_WINS` | Steam Cloud siempre sobreescribe los datos locales. |
| `2` | `LOCAL_WINS` | El local siempre sobreescribe los datos de Steam Cloud. |
| `3` | `MANUAL` | Emite `sync_conflict` para cada clave diferente — tu juego lo resuelve. |

---

## LocalBackendConfig

Controla el comportamiento del backend **Local**.

| Propiedad | Tipo | Predeterminado | Descripción |
|-----------|------|----------------|-------------|
| `save_directory` | `String` | `"saves"` | Ruta relativa bajo `user://` donde se almacenan los guardados. |
| `use_project_subfolder` | `bool` | `false` | Si es `true`, los guardados van a `user://saves/<NombreProyecto>/`. |
| `format` | `StoreEnums.SerializationFormat` | `JSON` | `JSON` (legible) o `BINARY` (compacto, usa `var_to_bytes`). |
| `encrypt` | `bool` | `false` | Cifrar los archivos de guardado con una contraseña. |
| `encryption_key` | `String` | `""` | La contraseña usada para el cifrado. ¡Mantenla en secreto! |
| `max_backups` | `int` | `1` | Número de archivos de copia de seguridad `.bak` automáticos por slot. `0` desactiva las copias. |

> **¿Dónde está `user://` en mi máquina?**
> - **Windows**: `%APPDATA%\Godot\app_userdata\<NombreProyecto>\`
> - **macOS**: `~/Library/Application Support/Godot/app_userdata/<NombreProyecto>/`
> - **Linux**: `~/.local/share/godot/app_userdata/<NombreProyecto>/`
>
> La ruta exacta puede cambiar si activas `application/config/use_custom_user_dir` en la Configuración del Proyecto. Usa `EasyStore.get_save_path()` para obtener siempre la ruta absoluta actual en tiempo de ejecución.

---

## SteamCloudBackendConfig

Controla el comportamiento del backend **Steam Cloud**.

| Propiedad | Tipo | Predeterminado | Descripción |
|-----------|------|----------------|-------------|
| `file_prefix` | `String` | `"easystore_"` | Prefijo para todos los archivos escritos en Steam Remote Storage. Evita colisiones con otras herramientas. |
| `auto_sync_on_connect` | `bool` | `true` | Si es `true`, llama a `EasyStore.sync()` automáticamente cuando el backend Steam se inicializa. |
| `quota_warning_bytes` | `int` | `0` | Si es > 0, registra una advertencia cuando el espacio cloud usado supera este umbral. `0` desactiva la verificación. |

---

## Ejemplo de configuración completa

```gdscript
func _inicializar_easystore() -> void:
    var config := EasyStoreConfig.new()

    # General
    config.default_slot          = 0
    config.current_save_version  = 3
    config.conflict_strategy     = StoreEnums.ConflictStrategy.NEWEST_WINS
    config.log_level             = StoreEnums.LogLevel.NONE

    # Backend local
    config.local                        = LocalBackendConfig.new()
    config.local.save_directory         = "saves"
    config.local.use_project_subfolder  = true
    config.local.encrypt                = true
    config.local.encryption_key         = "mi-clave-secreta-123"
    config.local.max_backups            = 3

    # Backend Steam
    config.steam_cloud             = SteamCloudBackendConfig.new()
    config.steam_cloud.file_prefix = "mijuego_"

    await EasyStore.initialize(config)
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)
    EasyStore.add_backend(StoreEnums.BackendType.STEAM_CLOUD)
```

---

<!-- doc-shell:page slug="backend-local" -->

# Backend Local

El backend Local guarda los datos del juego en el sistema de archivos del dispositivo bajo `user://`. Es el backend más utilizado y funciona en todas las plataformas sin dependencias adicionales.

## Cómo funciona

```
EasyStore.save("player", data)
        │
        ▼
  SectionCache (en memoria) ← escritura instantánea, sin I/O
        │
        ▼ (asíncrono — hilo secundario)
  AsyncWorker
        │
        ├── slot_0.sav          ← archivo de guardado completo (JSON o binario)
        └── slot_0.meta.json    ← sidecar de metadatos ligero
```

Todas las operaciones de disco se despachan a un `Thread` secundario via `AsyncWorker`. El hilo principal **nunca se bloquea**. Tu juego sigue ejecutándose mientras se escribe el archivo.

## Ubicaciones de archivo predeterminadas

Con la configuración predeterminada, los guardados van a:

```
user://saves/
    slot_0.sav
    slot_0.meta.json
    slot_0.sav.bak        ← copia de seguridad automática (si max_backups > 0)
    slot_1.sav
    slot_1.meta.json
```

Usa `EasyStore.get_save_path()` para obtener la ruta absoluta del SO en tiempo de ejecución:

```gdscript
var ruta := EasyStore.get_save_path()
print("Guardados en: ", ruta)
# Windows: C:\Users\TuNombre\AppData\Roaming\Godot\app_userdata\MiJuego\saves\slot_0.sav
```

## Activar el backend Local

```gdscript
await EasyStore.initialize()
EasyStore.add_backend(StoreEnums.BackendType.LOCAL)
```

¡Eso es todo! Sin configuración, se aplican todos los valores predeterminados.

## Activar el cifrado

El cifrado usa el `FileAccess.open_encrypted_with_pass()` integrado de Godot. La contraseña nunca se almacena en el archivo — solo tu código la conoce.

```gdscript
var config := EasyStoreConfig.new()
config.local = LocalBackendConfig.new()
config.local.encrypt        = true
config.local.encryption_key = "contraseña-super-secreta"
await EasyStore.initialize(config)
EasyStore.add_backend(StoreEnums.BackendType.LOCAL)
```

> ⚠️ **Si cambias o pierdes la clave de cifrado, los guardados existentes se vuelven ilegibles.** Trata la clave como una contraseña. Para la mayoría de juegos, el cifrado es opcional — considéralo para juegos competitivos donde la manipulación de guardados importa.

## Copias de seguridad automáticas

Por defecto, antes de sobreescribir un guardado existente, el backend Local lo copia a `slot_N.sav.bak`. Esto protege contra la corrupción por corte de luz o fallos durante una escritura.

```gdscript
config.local.max_backups = 3   # mantener hasta 3 archivos de respaldo
```

## Guardar solo secciones específicas

Si solo cambió parte de tus datos, puedes guardar solo una sección:

```gdscript
# Solo se escribirán los datos de "player" (las demás secciones permanecen en disco)
EasyStore.save("player", datos_player)
```

Para volcar todo lo que está en caché de una vez:

```gdscript
EasyStore.save_all()
```

## Directorio de guardado personalizado

```gdscript
config.local.save_directory        = "guardados_personalizados"
config.local.use_project_subfolder = false
# Los guardados van a: user://guardados_personalizados/slot_0.sav
```

Con `use_project_subfolder = true`:

```gdscript
config.local.use_project_subfolder = true
# Los guardados van a: user://saves/MiJuego/slot_0.sav
```

Esto es útil si varios proyectos comparten la misma raíz `user://` (por ejemplo, durante el desarrollo).

---

<!-- doc-shell:page slug="backend-steam" -->

# Backend Steam

El backend Steam guarda los datos del juego en **Steam Remote Storage** (Steam Cloud). Los jugadores pueden cambiar de dispositivo y su guardado estará disponible automáticamente en todas partes donde jueguen.

## Requisitos

| Elemento | Detalles |
|----------|---------|
| **GodotSteam GDExtension 4.4+** | Plugin de [Gramps](https://godotsteam.com/). Debe estar instalado y activado en tu proyecto. |
| **Cliente Steam** | Debe estar en ejecución y con sesión iniciada antes de que el juego comience. |
| **App ID de Steam válido** | Cualquier App ID funciona para pruebas locales. Usa tu App ID real en producción. |

> **EasyStore no da errores de parseado sin GodotSteam.** El backend Steam resuelve GodotSteam en tiempo de ejecución via `Engine.get_singleton("Steam")`. Si el plugin no está instalado, `add_backend(STEAM)` emite una advertencia clara y continúa sin fallar.

## Instalar GodotSteam

1. Descarga **GodotSteam GDExtension 4.4+** desde [godotsteam.com](https://godotsteam.com/).
2. Copia la carpeta `addons/godotsteam/` en tu proyecto.
3. Actívala en **Proyecto → Configuración del Proyecto → Plugins**.

GodotSteam registra el singleton global `Steam` y proporciona los métodos de la API de Steam que EasyStore usa (`fileWrite`, `fileRead`, `fileExists`, etc.).

## Activar el backend Steam

```gdscript
func _inicializar_easystore() -> void:
    await EasyStore.initialize()
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)   # añade local primero
    EasyStore.add_backend(StoreEnums.BackendType.STEAM_CLOUD)   # añade Steam — seguro aunque no esté disponible
```

EasyStore ejecuta tres verificaciones durante la inicialización del backend Steam:

1. **¿Está GodotSteam instalado?** (`Engine.has_singleton("Steam")`)
   - Si no: registra una advertencia clara explicando cómo instalarlo. EasyStore continúa con otros backends.
2. **¿Está Steam en ejecución?** (`Steam.isSteamRunning()`)
   - Si no: registra una advertencia explicando que esto es esperado en modo sin conexión. EasyStore continúa con otros backends.
3. **Aplicar configuración** — lee `file_prefix` de `SteamCloudBackendConfig`.

Puedes verificar si el backend Steam está listo en cualquier momento:

```gdscript
if EasyStore.is_backend_ready(StoreEnums.BackendType.STEAM_CLOUD):
    print("¡Steam Cloud está activo!")
else:
    print("Steam Cloud no disponible — usando solo local")
```

## Cómo se nombran los archivos en Steam Cloud

EasyStore almacena dos archivos por slot en Steam Remote Storage:

```
easystore_slot_0.sav          ← datos de guardado completos
easystore_meta_0.meta.json    ← metadatos ligeros
```

El prefijo `easystore_` viene de `SteamCloudBackendConfig.file_prefix`. Steam gestiona estos archivos automáticamente por App ID, así que no colisionarán con otros juegos.

## Verificar el estado del backend

```gdscript
# ¿El backend está en la lista activa (añadido, pero puede haber fallado al inicializar)?
if EasyStore.has_backend(StoreEnums.BackendType.STEAM_CLOUD):
    print("El backend Steam fue añadido")

# ¿El backend está completamente inicializado y listo para usar?
if EasyStore.is_backend_ready(StoreEnums.BackendType.STEAM_CLOUD):
    print("Steam Cloud recibirá los guardados")
```

## Ejemplo completo con Local + Steam

```gdscript
# GLOBAL.gd
extends Node

func _ready() -> void:
    var config := EasyStoreConfig.new()
    config.conflict_strategy = StoreEnums.ConflictStrategy.NEWEST_WINS
    config.log_level         = StoreEnums.LogLevel.DEBUG

    await EasyStore.initialize(config)

    # Local: siempre disponible
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)

    # Steam: añadido con seguridad — si no está disponible, EasyStore lo ignora
    EasyStore.add_backend(StoreEnums.BackendType.STEAM_CLOUD)

    # Sincronizar local ↔ nube al arrancar (resuelve conflictos por timestamp)
    if EasyStore.is_backend_ready(StoreEnums.BackendType.STEAM_CLOUD):
        EasyStore.sync()
        await EasyStore.sync_completed
        print("Sincronización completada: ", EasyStore.get_active_backends())

    # Cargar el guardado del jugador
    EasyStore.load_completed.connect(_al_cargar)
    EasyStore.load_all()

func _al_cargar(slot: int, data: Dictionary, exito: bool) -> void:
    if exito:
        print("¡Guardado cargado! Nivel del jugador: ", data.get("player", {}).get("nivel", 1))
    else:
        print("No hay guardado para el slot ", slot, " — empezando desde cero")
```

---

<!-- doc-shell:page slug="multi-backend" -->

# Multi-Backend

EasyStore está diseñado para ejecutar **múltiples backends simultáneamente**. Cuando dos o más backends están activos, cada llamada a `save()` escribe en todos ellos en paralelo — así el progreso del jugador siempre está protegido tanto localmente como en la nube.

## Caso de uso típico: Local + Steam Cloud

```
EasyStore.save("player", data)
    │
    ├── LocalBackend  → escribe slot_0.sav      (hilo secundario)
    └── SteamBackend  → escribe en Steam Cloud  (I/O propio de Steam)
```

Si un backend falla (Steam sin conexión, disco lleno, etc.), el otro continúa de forma independiente. EasyStore emite `error_occurred` para el backend que falla, pero el que tiene éxito sigue llamando a `save_completed`.

## Añadir múltiples backends

```gdscript
await EasyStore.initialize()
EasyStore.add_backend(StoreEnums.BackendType.LOCAL)
EasyStore.add_backend(StoreEnums.BackendType.STEAM_CLOUD)

print(EasyStore.get_active_backends())
# [0, 1]  ← BackendType.LOCAL=0, BackendType.STEAM_CLOUD=1
```

## Verificar qué backends están activos

```gdscript
for bt in EasyStore.get_active_backends():
    match bt:
        StoreEnums.BackendType.LOCAL:  print("Local está activo")
        StoreEnums.BackendType.STEAM_CLOUD:  print("Steam está activo")
```

## Eliminar un backend en tiempo de ejecución

```gdscript
# Si Steam se desconecta a mitad de sesión, elimínalo de forma elegante
EasyStore.remove_backend(StoreEnums.BackendType.STEAM_CLOUD)
# Los guardados futuros irán solo a Local
```

## Sync — Resolver conflictos entre backends

Cuando un jugador juega en dos dispositivos distintos (uno con conexión, otro sin ella), sus guardados local y en la nube pueden divergir. `EasyStore.sync()` compara timestamps y resuelve el conflicto:

```gdscript
# La sincronización es automática si SteamCloudBackendConfig.auto_sync_on_connect = true
# También puedes llamarla manualmente en cualquier momento
await EasyStore.sync()
print("Sync completado")
```

### Estrategias de conflicto

Establece `EasyStoreConfig.conflict_strategy` antes de llamar a `initialize()`:

```gdscript
config.conflict_strategy = StoreEnums.ConflictStrategy.NEWEST_WINS
```

| Estrategia | Comportamiento |
|------------|---------------|
| `NEWEST_WINS` | Compara `SaveMetadata.timestamp`. El backend con el guardado más reciente gana. El más antiguo se sobreescribe. |
| `CLOUD_WINS` | Steam Cloud siempre gana, sobreescribiendo los datos locales. |
| `LOCAL_WINS` | El local siempre gana, sobreescribiendo Steam Cloud. |
| `MANUAL` | EasyStore emite `sync_conflict` para cada sección diferente. Tu juego muestra una UI y llama a `EasyStore.save()` para elegir qué versión conservar. |

### Ejemplo de resolución manual de conflictos

```gdscript
func _ready() -> void:
    EasyStore.sync_conflict.connect(_en_conflicto)
    EasyStore.sync()

func _en_conflicto(slot: int, seccion: String, datos_local: Variant, datos_cloud: Variant) -> void:
    # Muestra un diálogo al jugador para que elija qué versión conservar
    var dialogo := $DialogoConflicto
    dialogo.setup(slot, seccion, datos_local, datos_cloud)
    dialogo.popup()

# En el diálogo:
func _al_elegir_nube() -> void:
    EasyStore.save(seccion, datos_cloud)

func _al_elegir_local() -> void:
    EasyStore.save(seccion, datos_local)
```

## Prioridad de lectura

Al cargar, EasyStore siempre prefiere el **backend Local** primero. Si el slot no existe localmente, prueba con el siguiente backend disponible. Después de un `sync()` exitoso, ambos backends son idénticos y la prioridad no importa.

```gdscript
# Lee desde Local si está disponible; si no, recurre a Steam
EasyStore.load_all()
```

---

<!-- doc-shell:page slug="slots" -->

# Slots de Guardado

EasyStore soporta múltiples slots de guardado independientes de serie. Cada slot es un guardado completamente separado con sus propios datos, metadatos y archivos.

## El slot activo

EasyStore siempre tiene un **slot activo**. Las operaciones que no especifican un slot (como `save()` y `load()`) actúan sobre el slot activo.

```gdscript
EasyStore.set_slot(0)   # cambiar al slot 0
EasyStore.get_slot()    # devuelve 0
```

El slot activo predeterminado tras `initialize()` es `EasyStoreConfig.default_slot` (predeterminado: `0`).

## Listar todos los slots

```gdscript
var slots: Array[SaveMetadata] = EasyStore.list_slots()
for meta in slots:
    print("Slot %d — %s — %s" % [
        meta.slot,
        Time.get_datetime_string_from_unix_time(meta.timestamp),
        meta.custom.get("capitulo", "Desconocido")
    ])
```

`list_slots()` lee de los **archivos sidecar de metadatos** (`.meta.json`), que son pequeños y rápidos de parsear. Los datos de guardado completos no se cargan.

## Verificar si un slot tiene datos

```gdscript
if EasyStore.has_slot(1):
    print("El slot 1 tiene un guardado")
else:
    print("El slot 1 está vacío")
```

## Cambiar de slot

```gdscript
# Mostrar la pantalla de selección de guardado
for i in range(3):
    if EasyStore.has_slot(i):
        var meta := EasyStore.get_save_metadata(i)
        $BotonesSlot[i].text = "Slot %d — Nivel %d" % [i, meta.custom.get("nivel", 1)]
    else:
        $BotonesSlot[i].text = "Slot %d — Vacío" % i

# El jugador selecciona el slot 2
func _al_pulsar_slot_2() -> void:
    EasyStore.set_slot(2)
    EasyStore.load_all()
    await EasyStore.load_completed
    get_tree().change_scene_to_file("res://escenas/juego.tscn")
```

## Eliminar un slot

```gdscript
EasyStore.delete_slot(1)
# Emite: slot_deleted(1)
# Elimina los archivos de todos los backends activos y limpia la caché en memoria
```

## Metadatos del guardado

Cada slot lleva un objeto `SaveMetadata` con información descriptiva:

| Propiedad | Tipo | Descripción |
|-----------|------|-------------|
| `slot` | `int` | El índice del slot. |
| `timestamp` | `int` | Timestamp Unix de cuándo se guardó este slot por última vez. |
| `playtime_seconds` | `float` | Tiempo de juego acumulado en segundos (tú lo actualizas). |
| `game_version` | `String` | Tu cadena de versión del juego (informativo). |
| `save_version` | `int` | La versión del formato de guardado usada cuando se guardó este slot. |
| `thumbnail_path` | `String` | Ruta opcional a una captura de pantalla tomada al guardar. |
| `custom` | `Dictionary` | Lo que quieras — nombre de capítulo, dificultad, nivel del jugador, etc. |
| `is_empty` | `bool` | `true` si nunca se ha escrito en este slot. |

```gdscript
var meta := EasyStore.get_save_metadata()   # obtiene metadatos del slot actual
print("Último guardado: ", Time.get_datetime_string_from_unix_time(meta.timestamp))
print("Tiempo de juego: ", meta.playtime_seconds / 60.0, " minutos")
print("Capítulo:        ", meta.custom.get("capitulo", "Desconocido"))
```

> EasyStore rellena automáticamente `slot`, `timestamp` e `is_empty` en cada guardado. Tú eres responsable de actualizar `playtime_seconds`, `game_version` y `custom` desde la lógica de tu juego.

---

<!-- doc-shell:page slug="secciones" -->

# Secciones y Caché

EasyStore organiza los datos de guardado en **secciones con nombre**. Cada sección es un `Dictionary` independiente que representa una parte lógica del estado de tu juego.

## ¿Por qué secciones?

Dividir los datos en secciones te da:
- **Cargas parciales** — lee solo `"settings"` al arrancar sin cargar todos los datos del jugador.
- **Guardados parciales** — si solo se movió el jugador, escribe `"player"` sin re-escribir `"world"`.
- **Organización clara** — `"player"`, `"world"`, `"settings"`, `"quests"` son inmediatamente comprensibles.

## Escribir en una sección

```gdscript
EasyStore.save("player", {
    "salud":    salud_actual,
    "mana":     mana_actual,
    "nivel":    nivel_jugador,
    "posicion": Vector3(x, y, z),
})

EasyStore.save("settings", {
    "volumen":     volumen_musica,
    "pantalla_completa": es_pantalla_completa,
    "idioma": "es",
})
```

Cada llamada a `save()` **fusiona** los nuevos datos en la sección. La sección se marca como **sucia** y se incluirá en la próxima escritura al backend.

## Leer de una sección

```gdscript
# Si la sección está en caché, devuelve instantáneamente (sin I/O)
var player := EasyStore.load("player")
var salud: int = player.get("salud", 100)

# Si la caché está vacía (primera carga al arrancar), lanza una carga completa:
EasyStore.load_all()
await EasyStore.load_completed
var player := EasyStore.load("player")
```

## Cómo funciona la caché

```
EasyStore.save("player", data)
    │
    ▼
SectionCache["slot_0"]["player"] = data   ← marcado como sucio
    │
    ▼  (asíncrono, en el próximo volcado)
SaveFile {
    sections: {
        "player":   {...},
        "world":    {...},
        "settings": {...},
    }
}  → escrito en los backends
```

La caché mantiene todas las secciones de todos los slots cargados en memoria. Una vez cargadas, las llamadas posteriores a `load()` son lecturas instantáneas de diccionario — sin acceso a disco.

## Guardar todas las secciones de una vez

```gdscript
# Volcar todas las secciones sucias a todos los backends en una sola escritura
EasyStore.save_all()
```

## Convención de nombres de sección (recomendada)

| Sección | Contenido |
|---------|-----------|
| `"player"` | Salud, estadísticas, posición, inventario |
| `"world"` | Estado del mundo, puertas abiertas, objetos recogidos |
| `"quests"` | Indicadores de progreso de misiones |
| `"settings"` | Audio, vídeo, controles |
| `"achievements"` | Logros desbloqueados |

---

<!-- doc-shell:page slug="autosave" -->

# Auto-guardado

EasyStore tiene un sistema de auto-guardado integrado. Una vez activado, dispara un temporizador en segundo plano y llama a `save_all()` automáticamente en el intervalo configurado — sin ningún código adicional en tu juego.

## Activar / desactivar

```gdscript
# Empezar a guardar automáticamente cada 2 minutos
EasyStore.enable_autosave(120.0)

# Detener el auto-guardado
EasyStore.disable_autosave()
```

El intervalo está en segundos. Valores comunes:

| Intervalo | Segundos | Cuándo usarlo |
|-----------|---------|---------------|
| 30 s | `30.0` | Juegos rápidos donde perder progreso es muy frustrante |
| 1 min | `60.0` | Juegos de acción |
| 2 min | `120.0` | RPGs y juegos de aventura (**predeterminado recomendado**) |
| 5 min | `300.0` | Juegos de ritmo lento, estrategia |

## Reaccionar al evento de auto-guardado

Usa la señal `autosave_triggered` para mostrar un indicador de guardado en tu UI:

```gdscript
func _ready() -> void:
    EasyStore.autosave_triggered.connect(_al_autoguardar)
    EasyStore.save_completed.connect(_al_terminar_guardado)
    EasyStore.enable_autosave(120.0)

func _al_autoguardar(slot: int) -> void:
    $HUD/IconoGuardar.show()
    $HUD/IconoGuardar.text = "Guardando..."

func _al_terminar_guardado(slot: int, exito: bool) -> void:
    $HUD/IconoGuardar.hide()
    if not exito:
        $HUD/EtiquetaGuardar.text = "¡Error al guardar!"
```

## save_started y save_completed

Estas dos señales enmarcan cada operación de guardado (tanto manual como auto-guardado):

| Señal | Cuándo |
|-------|--------|
| `save_started(slot)` | Justo antes de que la escritura se despache a los backends |
| `save_completed(slot, exito)` | Cuando todos los backends activos han terminado |

## load_started y load_completed

De forma similar para las cargas:

| Señal | Cuándo |
|-------|--------|
| `load_started(slot)` | Justo antes de que se despache la lectura del backend |
| `load_completed(slot, data, exito)` | Cuando los datos están disponibles en memoria |

```gdscript
EasyStore.load_started.connect(func(slot): $PantallaCarga.show())
EasyStore.load_completed.connect(func(slot, data, ok): $PantallaCarga.hide())
```

---

<!-- doc-shell:page slug="migraciones" -->

# Migraciones de Guardado

A medida que tu juego evoluciona, el formato de guardado cambiará. El sistema de migración de EasyStore te permite escribir **funciones de actualización con conciencia de versión** para que los guardados antiguos se actualicen automáticamente al cargarlos — sin perder los datos del jugador.

## Cómo funciona el versionado

Cada archivo de guardado almacena un entero `version`. Declaras la versión actual en tu configuración:

```gdscript
config.current_save_version = 3
EasyStore.initialize(config)
```

Cuando EasyStore carga un guardado y encuentra `save.version < current_save_version`, ejecuta todas las migraciones registradas en orden para actualizarlo.

## Registrar migraciones

```gdscript
# Ejecuta esto ANTES de initialize(), para que las migraciones estén listas antes de cualquier carga

EasyStore.register_migration(1, 2, func(sections: Dictionary) -> Dictionary:
    # Versión 1 → 2: "hp" se renombró a "salud"
    if sections.has("player"):
        var p = sections["player"]
        if p.has("hp"):
            p["salud"] = p["hp"]
            p.erase("hp")
    return sections
)

EasyStore.register_migration(2, 3, func(sections: Dictionary) -> Dictionary:
    # Versión 2 → 3: nueva estadística "stamina" añadida con valor predeterminado 100
    if sections.has("player"):
        sections["player"]["stamina"] = sections["player"].get("stamina", 100)
    # Versión 2 → 3: la sección "world" ahora requiere una clave "clima"
    if sections.has("world"):
        sections["world"]["clima"] = sections["world"].get("clima", "despejado")
    return sections
)
```

## Señal de migración

```gdscript
EasyStore.migration_applied.connect(func(ver_ant: int, ver_nueva: int):
    print("Guardado migrado: v%d → v%d" % [ver_ant, ver_nueva])
)
```

## Huecos en la cadena de migración

Si registras 1→2 y 3→4 pero NO 2→3, EasyStore aplicará 1→2, saltará 2→3 con una advertencia y luego aplicará 3→4. El guardado no se corrompe — solo se omite el paso que falta.

## Los datos migrados no se re-guardan automáticamente

Tras la migración, los datos actualizados viven en la caché. Se escribirán en disco en la **siguiente llamada a `save()` o `save_all()`**. Esto es intencional — si algo salió mal en tu callback de migración, el archivo original en disco permanece sin cambios.

## Ejemplo completo con migraciones

```gdscript
# GLOBAL.gd
extends Node

func _ready() -> void:
    # Registrar migraciones PRIMERO
    EasyStore.register_migration(1, 2, _migrar_v1_a_v2)
    EasyStore.register_migration(2, 3, _migrar_v2_a_v3)

    # Luego inicializar con la versión actual
    var config := EasyStoreConfig.new()
    config.current_save_version = 3
    await EasyStore.initialize(config)
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)

    EasyStore.migration_applied.connect(_al_migrar)
    EasyStore.load_all()

func _migrar_v1_a_v2(sections: Dictionary) -> Dictionary:
    if sections.has("player"):
        sections["player"]["stamina"] = 100
    return sections

func _migrar_v2_a_v3(sections: Dictionary) -> Dictionary:
    if sections.has("world"):
        sections["world"].erase("semilla_legacy")
        sections["world"]["bioma"] = "bosque"
    return sections

func _al_migrar(ver_ant: int, ver_nueva: int) -> void:
    print("Archivo de guardado actualizado de v%d a v%d" % [ver_ant, ver_nueva])
```

---

<!-- doc-shell:page slug="save-path" -->

# Archivos de Guardado y Ruta

## Obtener la ruta de guardado en tiempo de ejecución

```gdscript
# Devuelve la ruta absoluta del SO al archivo de guardado del slot actual
var ruta := EasyStore.get_save_path()
print(ruta)
# Ejemplo: C:\Users\Alice\AppData\Roaming\Godot\app_userdata\MiJuego\saves\slot_0.sav
```

Esto es útil para:
- Mostrar la ubicación del guardado en un menú de configuración
- Abrir la carpeta con el explorador de archivos del SO
- Depuración en la máquina de un jugador

```gdscript
# Mostrar la ruta de guardado en un menú de configuración
func _al_abrir_carpeta_guardado() -> void:
    var ruta := EasyStore.get_save_path()
    var dir  := ruta.get_base_dir()
    OS.shell_open(dir)  # abre la carpeta en el explorador del SO
```

## Formato de archivo — JSON

El formato predeterminado (`StoreEnums.SerializationFormat.JSON`) produce un archivo `.sav` legible por humanos:

```json
{
    "version": 3,
    "slot": 0,
    "metadata": {
        "slot": 0,
        "timestamp": 1714500000,
        "playtime_seconds": 3720.5,
        "game_version": "1.2.0",
        "save_version": 3,
        "is_empty": false,
        "custom": { "capitulo": "acto_2", "nivel": 14 }
    },
    "sections": {
        "player": { "salud": 80, "nivel": 14, "monedas": 1250 },
        "world":  { "nombre_nivel": "mazmorra_01", "clima": "lluvia" },
        "settings": { "volumen": 0.8, "idioma": "es" }
    }
}
```

## Formato de archivo — BINARIO

Establecer `LocalBackendConfig.format = StoreEnums.SerializationFormat.BINARY` usa `var_to_bytes()` / `bytes_to_var()` de Godot. Los archivos binarios son más pequeños y ligeramente más rápidos de parsear, pero no son legibles por humanos — útil para builds de producción cuando el tamaño del archivo de guardado importa.

## Sidecar de metadatos

Cada slot también escribe un pequeño `.meta.json` junto al `.sav` principal:

```json
{
    "slot": 0,
    "timestamp": 1714500000,
    "playtime_seconds": 3720.5,
    "game_version": "1.2.0",
    "save_version": 3,
    "is_empty": false,
    "custom": { "capitulo": "acto_2", "nivel": 14 }
}
```

`EasyStore.list_slots()` lee **solo los sidecars** — nunca carga los archivos de guardado completos — haciendo que el listado de slots sea muy rápido.

---

<!-- doc-shell:page slug="custom-backends" -->

# Backends Personalizados

EasyStore está diseñado para ser extendido. Si quieres sincronizar guardados con tu propio servidor, Google Play Games, Epic Games Store u otro servicio, puedes crear un backend personalizado implementando la interfaz `StorageBackend`.

## El contrato StorageBackend

Todos los backends extienden `StorageBackend` (en `addons/easystore/core/storage_backend.gd`). Debes implementar 8 métodos virtuales:

```gdscript
func _backend_initialize(config: Resource) -> Error
func _backend_shutdown() -> void
func _backend_save(slot: int, save_file: SaveFile) -> void
func _backend_load(slot: int) -> void
func _backend_delete(slot: int) -> void
func _backend_list_slots() -> void
func _backend_slot_exists(slot: int) -> bool
func _backend_get_capabilities() -> Dictionary
```

**Señales que tu backend debe emitir:**

| Señal | Parámetros | Cuándo |
|-------|------------|--------|
| `backend_save_completed` | `slot, exito, codigo_error` | Después de que `_backend_save()` termine |
| `backend_load_completed` | `slot, save_file, exito, codigo_error` | Después de que `_backend_load()` termine |
| `backend_delete_completed` | `slot, exito` | Después de que `_backend_delete()` termine |
| `backend_list_completed` | `lista_metadata: Array` | Después de que `_backend_list_slots()` termine |
| `backend_error` | `codigo, mensaje` | En cualquier error inesperado |

## Paso a paso: añadir un nuevo backend

### 1 — Crear el archivo del backend

```
addons/easystore/backends/mi_nube/mi_nube_backend.gd
```

```gdscript
# mi_nube_backend.gd
class_name MiNubeBackend
extends StorageBackend

var _api_url: String = "https://api.mijuego.com/saves"
var _token:   String = ""


func setup() -> void:
    _backend_type = StoreEnums.BackendType.CUSTOM_HTTP


func _backend_initialize(config: Resource) -> Error:
    if config and config.has_method("get") and config.get("api_url"):
        _api_url = config.api_url
        _token   = config.auth_token
    return OK


func _backend_shutdown() -> void:
    _token = ""


func _backend_save(slot: int, save_file: SaveFile) -> void:
    var body := JSON.stringify(save_file.to_dict())
    var headers := [
        "Content-Type: application/json",
        "Authorization: Bearer " + _token,
    ]
    var http := HTTPRequest.new()
    add_child(http)
    http.request_completed.connect(func(result, code, _h, _b):
        http.queue_free()
        var ok := (result == HTTPRequest.RESULT_SUCCESS and code == 200)
        backend_save_completed.emit(slot, ok, 0 if ok else StoreEnums.ErrorCode.CLOUD_UNAVAILABLE)
    )
    http.request(_api_url + "/slot/%d" % slot, headers, HTTPClient.METHOD_PUT, body)


func _backend_load(slot: int) -> void:
    var headers := ["Authorization: Bearer " + _token]
    var http := HTTPRequest.new()
    add_child(http)
    http.request_completed.connect(func(result, code, _h, body):
        http.queue_free()
        if result != HTTPRequest.RESULT_SUCCESS or code != 200:
            backend_load_completed.emit(slot, null, false, StoreEnums.ErrorCode.CLOUD_UNAVAILABLE)
            return
        var json := JSON.new()
        if json.parse(body.get_string_from_utf8()) != OK:
            backend_load_completed.emit(slot, null, false, StoreEnums.ErrorCode.PARSE_ERROR)
            return
        var sf := SaveFile.new()
        sf.from_dict(json.data)
        backend_load_completed.emit(slot, sf, true, StoreEnums.ErrorCode.OK)
    )
    http.request(_api_url + "/slot/%d" % slot, headers)


func _backend_delete(slot: int) -> void:
    backend_delete_completed.emit(slot, true)


func _backend_list_slots() -> void:
    backend_list_completed.emit([])


func _backend_slot_exists(slot: int) -> bool:
    return false


func _backend_get_capabilities() -> Dictionary:
    return { "encryption": false, "compression": false, "cloud_sync": true }
```

### 2 — Añadir una entrada al enum BackendType

Abre `addons/easystore/core/store_enums.gd`:

```gdscript
enum BackendType {
    LOCAL       = 0,
    STEAM_CLOUD = 1,
    GOOGLE_PLAY = 2,
    CUSTOM_HTTP = 3,   # ← tu nuevo backend
}
```

### 3 — Registrar el backend en EasyStore

Abre `addons/easystore/easystore.gd`, encuentra `_create_backend()`:

```gdscript
func _create_backend(type: StoreEnums.BackendType) -> Node:
    match type:
        StoreEnums.BackendType.LOCAL:
            var b := LocalBackend.new()
            b.setup(_worker)
            return b
        StoreEnums.BackendType.STEAM_CLOUD:
            var b: StorageBackend = load("res://addons/easystore/backends/steam/steam_backend.gd").new()
            b.setup()
            return b
        StoreEnums.BackendType.CUSTOM_HTTP:      # ← añade este caso
            var b := MiNubeBackend.new()
            b.setup()
            return b
    return null
```

### 4 — (Opcional) Añadir un recurso de configuración

```gdscript
# mi_nube_backend_config.gd
class_name MiNubeBackendConfig
extends Resource

@export var api_url:    String = "https://api.mijuego.com/saves"
@export var auth_token: String = ""
```

### 5 — Usar tu nuevo backend

```gdscript
EasyStore.add_backend(StoreEnums.BackendType.CUSTOM_HTTP, mi_config_nube)
```

¡Eso es todo! No es necesario cambiar ningún otro archivo. EasyStore enrutará automáticamente los guardados, cargas y sincronizaciones a tu nuevo backend junto a cualquier backend existente.

---

<!-- doc-shell:page slug="api" -->

# Referencia de API

Referencia completa de todos los métodos públicos del Autoload **`EasyStore`**.

## Configuración

| Método | Devuelve | Descripción |
|--------|---------|-------------|
| `initialize(config)` | `void` | Inicializa EasyStore. Pasa un `EasyStoreConfig` o `null` para los valores predeterminados. Llama una vez al arrancar. **Usa `await`** — puede suspenderse internamente si el nodo EasyStore aún no está listo por el orden de los autoloads. |

## Gestión de backends

| Método | Devuelve | Descripción |
|--------|---------|-------------|
| `add_backend(tipo, config)` | `void` | Añade e inicializa un backend. `config` es opcional. Pueden estar activos múltiples backends simultáneamente. |
| `remove_backend(tipo)` | `void` | Cierra y elimina un backend. |
| `get_active_backends()` | `Array[StoreEnums.BackendType]` | Lista los tipos de backend actualmente activos. |
| `has_backend(tipo)` | `bool` | `true` si un backend de este tipo está en la lista activa. |
| `is_backend_ready(tipo)` | `bool` | `true` si el backend está activo **y** completamente inicializado. |

## Gestión de slots

| Método | Devuelve | Descripción |
|--------|---------|-------------|
| `set_slot(slot)` | `void` | Establece el slot de guardado activo. |
| `get_slot()` | `int` | Devuelve el índice del slot activo. |
| `list_slots()` | `Array[SaveMetadata]` | Lista todos los slots conocidos (lee solo sidecars — rápido). |
| `has_slot(slot)` | `bool` | `true` si el slot tiene datos de guardado. |
| `delete_slot(slot)` | `void` | Elimina un slot de todos los backends y de la caché. Emite `slot_deleted`. |
| `get_save_metadata(slot)` | `SaveMetadata` | Metadatos del slot, o `null` si es desconocido. `slot = -1` usa el activo. |

## Guardar y Cargar

| Método | Devuelve | Descripción |
|--------|---------|-------------|
| `save(seccion, datos, slot)` | `void` | Escribe `datos` en la sección. Asíncrono hacia los backends. `slot = -1` usa el activo. Emite `save_started` y `save_completed`. |
| `save_all(slot)` | `void` | Vuelca todas las secciones sucias a todos los backends. |
| `load(seccion, slot)` | `Dictionary` | Lee de la caché si está disponible; si no, lanza una carga y devuelve `{}`. |
| `load_all(slot)` | `void` | Lanza una carga asíncrona completa. Emite `load_started` y `load_completed`. |
| `get_save_path(slot)` | `String` | Ruta absoluta del archivo de guardado (solo backend Local). `""` si no hay backend local. |

## Sincronización multi-backend

| Método | Devuelve | Descripción |
|--------|---------|-------------|
| `sync(slot)` | `void` | Compara timestamps entre backends y resuelve conflictos. Emite `sync_completed`. |

## Auto-guardado

| Método | Devuelve | Descripción |
|--------|---------|-------------|
| `enable_autosave(intervalo_segundos)` | `void` | Inicia el temporizador de auto-guardado. Predeterminado: `60.0` segundos. |
| `disable_autosave()` | `void` | Detiene el temporizador de auto-guardado. |

## Migraciones

| Método | Devuelve | Descripción |
|--------|---------|-------------|
| `register_migration(desde, hasta, fn)` | `void` | Registra un callable de migración. `fn` recibe y devuelve `sections: Dictionary`. |
| `set_current_version(version)` | `void` | Establece la versión actual de guardado. |

## Debug

| Método | Devuelve | Descripción |
|--------|---------|-------------|
| `debug_mode(activado)` | `void` | Atajo para alternar el log. `true` → `LogLevel.DEBUG`, `false` → `LogLevel.NONE`. |
| `set_log_level(nivel)` | `void` | Establece el nivel de verbosidad directamente. Más granular que `debug_mode()`. Ejemplo: `EasyStore.set_log_level(StoreEnums.LogLevel.DEBUG)`. |
| `get_logs(limite)` | `Array[Dictionary]` | Últimas `limite` entradas de log. `0` devuelve todas. |
| `get_debug_info()` | `Dictionary` | Snapshot del estado actual de EasyStore. |

---

<!-- doc-shell:page slug="senales" -->

# Referencia de Señales

EasyStore expone **13 señales** que cubren todo el ciclo de vida del guardado/carga.

## Guardar / Cargar

| Señal | Parámetros | Cuándo se emite |
|-------|------------|-----------------|
| `save_started` | `slot: int` | Justo antes de que un guardado se despache a los backends. Ideal para mostrar un indicador de guardado. |
| `save_completed` | `slot: int, exito: bool` | Cuando todos los backends han terminado. `exito` es `false` si alguno falló. |
| `load_started` | `slot: int` | Justo antes de la lectura del backend. Ideal para mostrar pantalla de carga. |
| `load_completed` | `slot: int, data: Dictionary, exito: bool` | Cuando los datos están disponibles en memoria. |

## Gestión de backends

| Señal | Parámetros | Cuándo se emite |
|-------|------------|-----------------|
| `backend_added` | `tipo: StoreEnums.BackendType` | Tras añadir un backend correctamente. |
| `backend_removed` | `tipo: StoreEnums.BackendType` | Tras eliminar un backend. |

## Slots

| Señal | Parámetros | Cuándo se emite |
|-------|------------|-----------------|
| `slot_deleted` | `slot: int` | Tras eliminar un slot de todos los backends. |

## Auto-guardado

| Señal | Parámetros | Cuándo se emite |
|-------|------------|-----------------|
| `autosave_triggered` | `slot: int` | Por el temporizador de auto-guardado, antes del guardado. |

## Migraciones

| Señal | Parámetros | Cuándo se emite |
|-------|------------|-----------------|
| `migration_applied` | `version_anterior: int, version_nueva: int` | Cada vez que un paso de migración transforma un guardado. |

## Sync

| Señal | Parámetros | Cuándo se emite |
|-------|------------|-----------------|
| `sync_completed` | `resultado: Dictionary` | Cuando termina una sincronización multi-backend. |
| `sync_conflict` | `slot: int, clave: String, datos_local: Variant, datos_cloud: Variant` | Durante sync con estrategia `MANUAL`, para cada sección diferente. |

## Errores

| Señal | Parámetros | Cuándo se emite |
|-------|------------|-----------------|
| `error_occurred` | `codigo: int, mensaje: String` | En cualquier error interno. |

---

## Ejemplo — Conectar todas las señales críticas

```gdscript
func _ready() -> void:
    EasyStore.save_started.connect(func(slot):
        $HUD/IconoGuardar.show()
    )
    EasyStore.save_completed.connect(func(slot, exito):
        $HUD/IconoGuardar.hide()
        if not exito:
            $HUD/EtiquetaError.text = "¡Error al guardar!"
    )
    EasyStore.load_started.connect(func(slot):
        $PantallaCarga.show()
    )
    EasyStore.load_completed.connect(func(slot, data, exito):
        $PantallaCarga.hide()
        if exito:
            _aplicar_datos_guardado(data)
        else:
            _iniciar_juego_nuevo()
    )
    EasyStore.error_occurred.connect(func(codigo, msg):
        push_warning("[EasyStore] Error %d: %s" % [codigo, msg])
    )
    EasyStore.migration_applied.connect(func(ant, nuevo):
        print("Guardado migrado: v%d → v%d" % [ant, nuevo])
    )
```

---

<!-- doc-shell:page slug="enums" -->

# Referencia de Enums

Todos los enums están en `addons/easystore/core/store_enums.gd` y se acceden mediante la clase `StoreEnums`.

## StoreEnums.BackendType

```gdscript
enum BackendType {
    LOCAL       = 0,   # Sistema de archivos local (user://)
    STEAM_CLOUD = 1,   # Steam Remote Storage via GodotSteam
    GOOGLE_PLAY = 2,   # Reservado para un futuro backend de Google Play Games
    CUSTOM_HTTP = 3,   # Reservado para backends HTTP/REST personalizados
}
```

## StoreEnums.ConflictStrategy

```gdscript
enum ConflictStrategy {
    NEWEST_WINS = 0,   # El backend con el timestamp más reciente sobreescribe al otro
    CLOUD_WINS  = 1,   # El backend cloud siempre gana
    LOCAL_WINS  = 2,   # El backend local siempre gana
    MANUAL      = 3,   # Emite sync_conflict para cada sección diferente
}
```

## StoreEnums.SerializationFormat

```gdscript
enum SerializationFormat {
    JSON   = 0,   # JSON legible por humanos (predeterminado)
    BINARY = 1,   # var_to_bytes de Godot — más pequeño, no legible
}
```

## StoreEnums.ErrorCode

```gdscript
enum ErrorCode {
    OK                = 0,
    FILE_NOT_FOUND    = 1,
    PARSE_ERROR       = 2,
    CLOUD_UNAVAILABLE = 3,
    SLOT_EMPTY        = 4,
    MIGRATION_FAILED  = 5,
    BACKEND_NOT_READY = 6,
}
```

## StoreEnums.LogLevel

Controla qué mensajes se imprimen en consola. Todos los mensajes se almacenan internamente siempre (hasta 500), independientemente del nivel activo, por lo que `get_logs()` siempre devuelve el historial completo.

```gdscript
enum LogLevel {
    NONE  = 0,   # Silencioso — sin salida por consola
    ERROR = 1,   # Solo errores
    WARN  = 2,   # Errores y advertencias
    INFO  = 3,   # Eventos de operación normal (predeterminado)
    DEBUG = 4,   # Eventos internos detallados
    TRACE = 5,   # Extremadamente verboso — cada paso interno
}
```

---

<!-- doc-shell:page slug="debug" -->

# Debug y Diagnósticos

EasyStore incluye un sistema de registro y diagnósticos que puedes usar durante el desarrollo, en QA y, opcionalmente, en producción.

La salida de log sigue el mismo formato que [LinkUx](https://github.com/iuxgames/LinkUx) para mantener consistencia:
```
[EasyStore] LEVEL [Context]: message
```

## Establecer el nivel de log

```gdscript
# En la configuración (antes de initialize) — INFO es el predeterminado
var config := EasyStoreConfig.new()
config.log_level = StoreEnums.LogLevel.DEBUG   # o INFO, WARN, ERROR, NONE
await EasyStore.initialize(config)

# Alternar en tiempo de ejecución
EasyStore.debug_mode(true)                              # atajo → establece LogLevel.DEBUG
EasyStore.debug_mode(false)                             # atajo → establece LogLevel.NONE
EasyStore.set_log_level(StoreEnums.LogLevel.DEBUG)      # control granular
```

| Nivel | Valor | Qué se imprime |
|-------|-------|----------------|
| `NONE` | `0` | Nada |
| `ERROR` | `1` | Solo errores |
| `WARN` | `2` | Errores + advertencias |
| `INFO` | `3` | Operación normal (predeterminado) |
| `DEBUG` | `4` | Eventos internos detallados |
| `TRACE` | `5` | Cada paso interno |

EasyStore siempre almacena internamente hasta 500 entradas de log, independientemente del nivel activo. Usa `get_logs()` para recuperarlas en cualquier momento.

## Leer entradas de log

```gdscript
# Obtener las últimas 50 entradas de log
var logs := EasyStore.get_logs(50)
for entrada in logs:
    print("[%s] [%s]: %s" % [entrada["level"], entrada["context"], entrada["message"]])
```

Cada entrada de log es un `Dictionary`:

```gdscript
{
    "level":     "INFO",                        # nombre del nivel
    "context":   "Save",                        # subsistema que emitió el log
    "message":   "Slot 0 saved successfully.",
    "timestamp": 1714500000,                    # Unix timestamp
}
```

## Obtener información de debug en tiempo de ejecución

```gdscript
var info := EasyStore.get_debug_info()
print(JSON.stringify(info, "  "))
# {
#   "initialized": true,
#   "active_slot": 0,
#   "backends": [0, 1],
#   "autosave_enabled": true,
#   "current_version": 3
# }
```

## Panel de debug en el juego (ejemplo)

```gdscript
# panel_debug.gd
extends PanelContainer

func _process(_delta: float) -> void:
    if not visible:
        return
    $Grid/Backends.text = str(EasyStore.get_active_backends())
    $Grid/Slot.text     = str(EasyStore.get_slot())
    $Grid/Steam.text    = str(EasyStore.is_backend_ready(StoreEnums.BackendType.STEAM_CLOUD))

    var logs := EasyStore.get_logs(5)
    var texto := ""
    for e in logs:
        texto += "[%s] [%s]: %s\n" % [e["level"], e["context"], e["message"]]
    $EtiquetaLog.text = texto
```

## Salida de consola de ejemplo

Con `log_level = StoreEnums.LogLevel.INFO` (predeterminado), el panel de salida de Godot muestra:

```
[EasyStore] INFO [Core]: EasyStore ready. (save_version=3, strategy=NEWEST_WINS)
[EasyStore] INFO [Backend]: Local backend ready. Saves will be stored in user://saves/
[EasyStore] WARN [Backend]: STEAM backend unavailable: Steam is not running. Make sure the Steam client is open before launching the game.
[EasyStore] INFO [Save]: Slot 0 saved successfully.
[EasyStore] INFO [Load]: Slot 0 loaded successfully. (save_version=3)
[EasyStore] INFO [Sync]: Starting sync for slot 0 — comparing timestamps across 2 backends.
[EasyStore] INFO [Sync]: Local save is newer — pushing to cloud. (slot=0)
```

---

<!-- doc-shell:page slug="ejemplo-completo" -->

# Ejemplo Completo

Un sistema de guardado realista para un RPG de un solo jugador usando EasyStore. Demuestra soporte multi-slot, Steam Cloud, cifrado, auto-guardado, migraciones e integración de UI.

## Estructura del proyecto

```
proyecto/
├── autoloads/
│   └── GLOBAL.gd          ← Inicializa EasyStore
├── escenas/
│   ├── menu_principal.tscn ← Pantalla de selección de slots
│   ├── juego.tscn          ← Escena de juego
│   └── hud.tscn            ← HUD con indicador de guardado
└── scripts/
    ├── menu_principal.gd
    ├── juego.gd
    └── hud.gd
```

---

## GLOBAL.gd — Autoload del juego

```gdscript
# autoloads/GLOBAL.gd
extends Node

func _ready() -> void:
    _configurar_easystore()

func _configurar_easystore() -> void:
    # Registrar migraciones ANTES de initialize
    EasyStore.register_migration(1, 2, func(sections: Dictionary) -> Dictionary:
        if sections.has("player"):
            sections["player"]["stamina"] = 100
        return sections
    )
    EasyStore.register_migration(2, 3, func(sections: Dictionary) -> Dictionary:
        if sections.has("world"):
            sections["world"]["clima"] = sections["world"].get("clima", "despejado")
        return sections
    )

    var config := EasyStoreConfig.new()
    config.current_save_version  = 3
    config.conflict_strategy     = StoreEnums.ConflictStrategy.NEWEST_WINS
    config.log_level             = StoreEnums.LogLevel.INFO if OS.is_debug_build() else StoreEnums.LogLevel.NONE

    config.local                 = LocalBackendConfig.new()
    config.local.encrypt         = true
    config.local.encryption_key  = "mi-clave-secreta-de-juego"
    config.local.max_backups     = 2

    config.steam_cloud                 = SteamCloudBackendConfig.new()
    config.steam_cloud.file_prefix     = "mijuego_"

    await EasyStore.initialize(config)
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)
    EasyStore.add_backend(StoreEnums.BackendType.STEAM_CLOUD)

    if EasyStore.is_backend_ready(StoreEnums.BackendType.STEAM_CLOUD):
        EasyStore.sync()

    EasyStore.error_occurred.connect(func(codigo, msg):
        push_warning("[GLOBAL] Error de EasyStore %d: %s" % [codigo, msg])
    )
    EasyStore.migration_applied.connect(func(ant, nuevo):
        print("Guardado actualizado: v%d → v%d" % [ant, nuevo])
    )
```

---

## menu_principal.gd — Selección de slot

```gdscript
# scripts/menu_principal.gd
extends Control

@onready var botones_slot := [$Slot0, $Slot1, $Slot2]

func _ready() -> void:
    _actualizar_slots()

func _actualizar_slots() -> void:
    var slots := EasyStore.list_slots()
    var mapa  := {}
    for meta in slots:
        mapa[meta.slot] = meta

    for i in range(3):
        var btn: Button = botones_slot[i]
        if mapa.has(i):
            var meta: SaveMetadata = mapa[i]
            var dt   := Time.get_datetime_string_from_unix_time(meta.timestamp)
            var mins := int(meta.playtime_seconds / 60)
            btn.text = "Slot %d — Nivel %d\n%s (%d min)" % [
                i, meta.custom.get("nivel", 1), dt, mins
            ]
        else:
            btn.text = "Slot %d — Nuevo Juego" % i

func _al_pulsar_slot(i: int) -> void:
    EasyStore.set_slot(i)
    if EasyStore.has_slot(i):
        $PantallaCarga.show()
        EasyStore.load_completed.connect(_al_cargar, CONNECT_ONE_SHOT)
        EasyStore.load_all(i)
    else:
        get_tree().change_scene_to_file("res://escenas/juego.tscn")

func _al_cargar(slot: int, data: Dictionary, exito: bool) -> void:
    $PantallaCarga.hide()
    if exito:
        get_tree().change_scene_to_file("res://escenas/juego.tscn")
    else:
        $EtiquetaError.text = "Error al cargar el guardado."

func _al_pulsar_eliminar(i: int) -> void:
    EasyStore.delete_slot(i)
    _actualizar_slots()
```

---

## juego.gd — Escena de juego

```gdscript
# scripts/juego.gd
extends Node

var tiempo_juego: float = 0.0

func _ready() -> void:
    EasyStore.enable_autosave(90.0)
    EasyStore.autosave_triggered.connect(func(slot): $HUD.mostrar_icono_guardado())
    EasyStore.save_completed.connect(func(slot, ok): $HUD.ocultar_icono_guardado())

func _process(delta: float) -> void:
    tiempo_juego += delta

func al_llegar_checkpoint() -> void:
    _guardar()

func _notification(what: int) -> void:
    if what == NOTIFICATION_WM_CLOSE_REQUEST:
        EasyStore.disable_autosave()
        _guardar()

func _guardar() -> void:
    EasyStore.save("player", {
        "salud":     $Jugador.salud,
        "nivel":     $Jugador.nivel,
        "posicion":  $Jugador.global_position,
    })
    EasyStore.save("world", {
        "nombre_nivel": get_tree().current_scene.name,
        "clima":        $Mundo.clima,
    })
    EasyStore.save_all()
```

---

## Flujo completo

```
1. GLOBAL._ready()
   └── initialize() + add_backend(LOCAL) + add_backend(STEAM)
   └── Migraciones registradas
   └── sync() si Steam está disponible

2. MenuPrincipal — jugador elige Slot 1
   └── set_slot(1) + load_all(1)
   └── load_completed → change_scene("juego.tscn")

3. Juego — el jugador juega
   └── enable_autosave(90)
   └── Cada 90s: autosave_triggered → HUD icono → save_all()
   └── save_completed → HUD oculta icono

4. Jugador llega a un checkpoint:
   └── save("player", ...) + save("world", ...) + save_all()
   └── Escribe en LOCAL y STEAM simultáneamente

5. Jugador cierra el juego:
   └── disable_autosave() + save_all()
   └── El juego cierra limpiamente
```
