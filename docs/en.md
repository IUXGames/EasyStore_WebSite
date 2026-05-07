<!-- doc-shell:page slug="introduccion" -->

# Introduction

[![Godot 4](https://img.shields.io/badge/Godot-4.4+-478cbf?logo=godotengine&logoColor=white)](https://godotengine.org/)
[![Version](https://img.shields.io/badge/version-1.0.0-3498db)](./plugin.cfg)
[![License](https://img.shields.io/badge/license-MIT-green)](./LICENSE)

**EasyStore** is a modular save system for **Godot 4** that unifies local and cloud save storage under a single clean API. Instead of writing your own file I/O, encryption, cloud sync, or migration logic, your game code only talks to EasyStore — and EasyStore handles the rest.

## Versions & compatibility

| Item | Value |
|------|--------|
| **Addon version** | **1.0.0** |
| **Target Godot version** | **Godot 4.4+** |
| **GDScript** | 4.x (static typing) |

## Philosophy

> One public API. Multiple swappable backends. Zero boilerplate.

EasyStore is built around three core ideas:

1. **Total storage abstraction** — Your game calls `EasyStore.save("player", data)`. Whether that writes a local file, uploads to Steam Cloud, or both simultaneously, is invisible to your game code.
2. **Resilience by default** — If the Steam backend fails (not installed, Steam offline, quota exceeded), EasyStore keeps going with whatever backends are available.
3. **Novice-friendly, expert-extensible** — Zero configuration gets you a working local save in seconds. Advanced users can plug in encryption, custom migrations, multi-slot management, and their own cloud backends.

## Layered Architecture

```
┌──────────────────────────────────────┐
│           Your game code             │  ← Only interact here
├──────────────────────────────────────┤
│        EasyStore — Public API        │  easystore.gd (Autoload)
├──────────────────────────────────────┤
│         Internal subsystems          │  SlotManager, SectionCache,
│                                      │  MigrationManager, SyncManager,
│                                      │  AutosaveTimer, AsyncWorker, etc.
├──────────────────────────────────────┤
│          Storage backends            │  StorageBackend (abstract)
├──────────────────────────────────────┤
│      Active backend(s)               │  Local (file) / Steam Cloud / custom
└──────────────────────────────────────┘
```

## Available Backends

| Backend | Status | Description |
|---------|--------|-------------|
| **Local** | ✅ Available | Saves to `user://saves/`. Supports encryption, backups, and JSON format. |
| **Steam Cloud** | ✅ Available | Saves to Steam Remote Storage via GodotSteam. Requires GodotSteam GDExtension 4.4+. |

## Key Features

- **Unified API** — `save()`, `load()`, `delete_slot()`: the same calls regardless of the active backend(s).
- **Multi-backend simultaneous** — Run Local + Steam Cloud at the same time. Every `save()` writes to both in parallel.
- **Offline resilience** — If Steam isn't available, EasyStore falls back to local storage transparently.
- **Section-based saves** — Divide your save data into named sections (`"player"`, `"world"`, `"settings"`). Load only what you need.
- **In-memory cache** — Reads are instant after the first load. Writes go to memory first (sync) then flushed to disk/cloud async.
- **Save slots** — Built-in multi-slot support (slot 0, 1, 2…). List, delete, and switch between them at runtime.
- **Save metadata** — Every slot has a lightweight sidecar with timestamp, playtime, custom fields, and a screenshot path.
- **Auto-save** — One call to start a background timer that saves automatically on a configurable interval.
- **Save migrations** — Register version-aware callbacks to transform old save data on load. Never break old saves again.
- **Conflict resolution** — When syncing multiple backends, pick a strategy: newest wins, cloud wins, local wins, or manual.
- **Optional encryption** — Encrypt local saves with a passphrase via Godot's `FileAccess.open_encrypted_with_pass()`.
- **Async I/O** — All disk operations run in a background thread (via `AsyncWorker`). The main thread is never blocked.
- **Extensible** — Add your own cloud backend by extending `StorageBackend` and implementing 8 virtual methods.

## Requirements

| Item | Required for | Notes |
|------|-------------|-------|
| **Godot 4.4+** | All | |
| **GodotSteam GDExtension 4.4+** | Steam backend only | Plugin by [Gramps](https://godotsteam.com/). Optional — EasyStore parses without it. |
| **Steam client running** | Steam backend only | Must be open and logged in. |

---

<!-- doc-shell:page slug="instalacion" -->

# Installation

## Step 1 — Copy the addon

Copy the `addons/easystore/` folder into your Godot project:

```
your-project/
└── addons/
    └── easystore/       ← paste the entire folder here
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

## Step 2 — Enable the plugin

Go to **Project → Project Settings → Plugins** and enable **EasyStore**:

```
[ EasyStore ]  [ enable ]
```

When enabled, the plugin registers the **`EasyStore`** autoload, which is globally accessible from any script.

## Step 3 — Verify the autoload

In **Project → Project Settings → Autoloads** you should see:

```
EasyStore   res://addons/easystore/easystore.tscn   ✓
```

> The autoload is added automatically. You do **not** need to add it manually.

## Step 4 — Initialize in your game

EasyStore **does not configure itself automatically**. Call `EasyStore.initialize()` once at startup (typically in your own Autoload), then add the backends you want:

```gdscript
# GLOBAL.gd — Your game autoload
extends Node

func _ready() -> void:
    _init_easystore()

func _init_easystore() -> void:
    # Optional: use a config resource to customize behavior
    var config := EasyStoreConfig.new()
    config.log_level = StoreEnums.LogLevel.DEBUG

    # await is required: initialize() may suspend internally if EasyStore's
    # _ready() hasn't run yet due to autoload ordering.
    await EasyStore.initialize(config)

    # Local backend: always available, no extra dependencies
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)

    # Steam backend: added only if GodotSteam is installed AND Steam is running
    # If Steam is unavailable, add_backend() warns and continues gracefully
    EasyStore.add_backend(StoreEnums.BackendType.STEAM_CLOUD)
```

> **Minimal setup (no config):** calling `EasyStore.initialize()` + `EasyStore.add_backend(StoreEnums.BackendType.LOCAL)` with no arguments is enough to get a working save system with default settings.

---

<!-- doc-shell:page slug="quickstart" -->

# Quick Start

This guide shows the minimum code needed to save and load game data in under 5 minutes.

## 1 — Save data

```gdscript
# Save a dictionary under the section name "player"
EasyStore.save("player", {
    "health": 100,
    "level":  3,
    "coins":  450,
})
```

`save()` is **synchronous from your game's perspective**: it writes to the in-memory cache instantly, then flushes to all active backends asynchronously (no frame drop).

## 2 — Wait for save to finish (optional)

If you need to know when the physical write completes (e.g. to show a "Saved!" indicator):

```gdscript
EasyStore.save_started.connect(func(slot): $SaveIcon.show())
EasyStore.save_completed.connect(func(slot, success):
    $SaveIcon.hide()
    if not success:
        $UI.show_error("Save failed!")
)
```

## 3 — Load data

```gdscript
# Trigger an async load for the current slot
EasyStore.load_all()

# Await the result
await EasyStore.load_completed

# Now read from cache — instant
var player := EasyStore.load("player")
print("Health: ", player.get("health", 100))
```

Or use the signal pattern:

```gdscript
func _ready() -> void:
    EasyStore.load_completed.connect(_on_load_completed)
    EasyStore.load_all()

func _on_load_completed(slot: int, data: Dictionary, success: bool) -> void:
    if not success:
        print("No save found — starting fresh")
        return
    var player := data.get("player", {})
    health = player.get("health", 100)
    level  = player.get("level", 1)
```

## 4 — Complete minimal example

```gdscript
# GLOBAL.gd
extends Node

func _ready() -> void:
    await EasyStore.initialize()
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)

    # Load any existing data
    EasyStore.load_completed.connect(_on_loaded)
    EasyStore.load_all()

func _on_loaded(slot: int, data: Dictionary, success: bool) -> void:
    if success:
        print("Loaded! Sections: ", data.keys())
    else:
        print("No save found for slot ", slot)

# Called by your game when it needs to save
func save_game() -> void:
    EasyStore.save("player", { "health": 100 })
    EasyStore.save("world",  { "level_name": "forest_01" })
```

---

<!-- doc-shell:page slug="configuracion" -->

# Configuration

EasyStore is configured through resource objects. All configuration is optional — sensible defaults are applied automatically.

## EasyStoreConfig

The main configuration resource, passed to `EasyStore.initialize(config)`.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `default_slot` | `int` | `0` | The save slot that becomes active after `initialize()`. |
| `conflict_strategy` | `StoreEnums.ConflictStrategy` | `NEWEST_WINS` | How to resolve conflicts when syncing multiple backends. |
| `current_save_version` | `int` | `1` | Current save format version. Used by the migration system. |
| `log_level` | `int` | `StoreEnums.LogLevel.INFO` | Log verbosity level. `NONE` = silent, `INFO` = recommended for development, `DEBUG` = verbose. See `StoreEnums.LogLevel`. |
| `local` | `LocalBackendConfig` | `null` | Local backend configuration (uses defaults if `null`). |
| `steam` | `SteamCloudBackendConfig` | `null` | Steam backend configuration (uses defaults if `null`). |

**Conflict strategies:**

| Value | Constant | Description |
|-------|----------|-------------|
| `0` | `NEWEST_WINS` | Whichever backend has the most recent timestamp wins. |
| `1` | `CLOUD_WINS` | Steam Cloud always overwrites local data. |
| `2` | `LOCAL_WINS` | Local always overwrites Steam Cloud data. |
| `3` | `MANUAL` | Emits `sync_conflict` for each differing key — your game resolves it. |

---

## LocalBackendConfig

Controls the behavior of the **Local** backend.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `save_directory` | `String` | `"saves"` | Relative path under `user://` where saves are stored. |
| `use_project_subfolder` | `bool` | `false` | If `true`, saves go to `user://saves/<ProjectName>/`. Useful for multi-project `user://` paths. |
| `format` | `StoreEnums.SerializationFormat` | `JSON` | `JSON` (readable) or `BINARY` (compact, uses `var_to_bytes`). |
| `encrypt` | `bool` | `false` | Encrypt save files with a passphrase. |
| `encryption_key` | `String` | `""` | The passphrase used for encryption. Keep this secret! |
| `max_backups` | `int` | `1` | Number of automatic `.bak` backup files to keep per slot. `0` disables backups. |

> **Where is `user://` on my machine?**
> - **Windows**: `%APPDATA%\Godot\app_userdata\<ProjectName>\`
> - **macOS**: `~/Library/Application Support/Godot/app_userdata/<ProjectName>/`
> - **Linux**: `~/.local/share/godot/app_userdata/<ProjectName>/`
>
> The exact path can change if you enable `application/config/use_custom_user_dir` in Project Settings. Use `EasyStore.get_save_path()` to always get the current absolute path at runtime.

---

## SteamCloudBackendConfig

Controls the behavior of the **Steam Cloud** backend.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `file_prefix` | `String` | `"easystore_"` | Prefix for all filenames written to Steam Remote Storage. Prevents collisions with other tools. |
| `auto_sync_on_connect` | `bool` | `true` | If `true`, calls `EasyStore.sync()` automatically when the Steam backend initializes. |
| `quota_warning_bytes` | `int` | `0` | If > 0, logs a warning when the used cloud space exceeds this threshold. `0` disables the check. |

---

## Full configuration example

```gdscript
func _init_easystore() -> void:
    var config := EasyStoreConfig.new()

    # General
    config.default_slot          = 0
    config.current_save_version  = 3
    config.conflict_strategy     = StoreEnums.ConflictStrategy.NEWEST_WINS
    config.log_level             = StoreEnums.LogLevel.NONE

    # Local backend
    config.local                      = LocalBackendConfig.new()
    config.local.save_directory       = "saves"
    config.local.use_project_subfolder = true
    config.local.encrypt              = true
    config.local.encryption_key       = "my-secret-key-123"
    config.local.max_backups          = 3

    # Steam backend
    config.steam_cloud             = SteamCloudBackendConfig.new()
    config.steam_cloud.file_prefix = "mygame_"

    await EasyStore.initialize(config)
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)
    EasyStore.add_backend(StoreEnums.BackendType.STEAM_CLOUD)
```

---

<!-- doc-shell:page slug="backend-local" -->

# Local Backend

The Local backend saves game data to the device's file system under `user://`. It is the most commonly used backend and works on all platforms without any extra dependencies.

## How it works

```
EasyStore.save("player", data)
        │
        ▼
  SectionCache (in memory) ← instant write, no I/O
        │
        ▼ (async — background thread)
  AsyncWorker
        │
        ├── slot_0.sav          ← full save file (JSON or binary)
        └── slot_0.meta.json    ← lightweight metadata sidecar
```

All disk operations are dispatched to a background `Thread` via `AsyncWorker`. The main thread is **never blocked**. Your game continues running while the file is being written.

## Default file locations

With default settings, saves go to:

```
user://saves/
    slot_0.sav
    slot_0.meta.json
    slot_0.sav.bak        ← automatic backup (if max_backups > 0)
    slot_1.sav
    slot_1.meta.json
```

Use `EasyStore.get_save_path()` to get the absolute OS path at runtime:

```gdscript
var path := EasyStore.get_save_path()
print("Saves stored at: ", path)
# Windows: C:\Users\YourName\AppData\Roaming\Godot\app_userdata\MyGame\saves\slot_0.sav
```

## Enabling the Local backend

```gdscript
await EasyStore.initialize()
EasyStore.add_backend(StoreEnums.BackendType.LOCAL)
```

That's it! With no config, all defaults apply.

## Enabling encryption

Encryption uses Godot's built-in `FileAccess.open_encrypted_with_pass()`. The passphrase is never stored in the file — only your code knows it.

```gdscript
var config := EasyStoreConfig.new()
config.local = LocalBackendConfig.new()
config.local.encrypt        = true
config.local.encryption_key = "super-secret-passphrase"
await EasyStore.initialize(config)
EasyStore.add_backend(StoreEnums.BackendType.LOCAL)
```

> ⚠️ **If you change or lose the encryption key, existing saves become unreadable.** Treat the key like a password. For most games, encryption is optional — consider it for competitive games where save tampering matters.

## Automatic backups

By default, before overwriting an existing save, the Local backend copies it to `slot_N.sav.bak`. This protects against corruption from power loss or crashes during a write.

```gdscript
config.local.max_backups = 3   # keep up to 3 backup files
```

## Saving only specific sections

If only part of your data changed, you can save just one section:

```gdscript
# Only "player" data will be written (other sections remain as-is on disk)
EasyStore.save("player", player_data)
```

To flush everything in the cache at once:

```gdscript
EasyStore.save_all()
```

## Custom save directory

```gdscript
config.local.save_directory        = "custom_saves"
config.local.use_project_subfolder = false
# Saves go to: user://custom_saves/slot_0.sav
```

With `use_project_subfolder = true`:

```gdscript
config.local.use_project_subfolder = true
# Saves go to: user://saves/MyGame/slot_0.sav
```

This is useful if multiple projects share the same `user://` root (e.g. during development).

---

<!-- doc-shell:page slug="backend-steam" -->

# Steam Backend

The Steam backend saves game data to **Steam Remote Storage** (Steam Cloud). Players can switch devices and their save is automatically available everywhere they play.

## Requirements

| Item | Details |
|------|---------|
| **GodotSteam GDExtension 4.4+** | Plugin by [Gramps](https://godotsteam.com/). Must be installed and enabled in your project. |
| **Steam client** | Must be running and logged in before the game starts. |
| **Valid Steam App ID** | Any App ID will work for local testing. Use your real App ID in production. |

> **EasyStore parses cleanly without GodotSteam.** The Steam backend resolves GodotSteam at runtime via `Engine.get_singleton("Steam")`. If the plugin is not installed, `add_backend(STEAM)` emits a clear warning and returns without crashing.

## Install GodotSteam

1. Download **GodotSteam GDExtension 4.4+** from [godotsteam.com](https://godotsteam.com/).
2. Copy the `addons/godotsteam/` folder into your project.
3. Enable it in **Project → Project Settings → Plugins**.

GodotSteam registers the `Steam` global singleton and provides the Steam API methods EasyStore uses (`fileWrite`, `fileRead`, `fileExists`, etc.).

## Enabling the Steam backend

```gdscript
func _init_easystore() -> void:
    await EasyStore.initialize()
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)   # always add local first
    EasyStore.add_backend(StoreEnums.BackendType.STEAM_CLOUD)   # add Steam — safe even if unavailable
```

EasyStore runs three checks during Steam backend initialization:

1. **Is GodotSteam installed?** (`Engine.has_singleton("Steam")`)
   - If not: logs a clear warning explaining how to install it. EasyStore continues with other backends.
2. **Is Steam running?** (`Steam.isSteamRunning()`)
   - If not: logs a warning explaining this is expected in offline mode. EasyStore continues with other backends.
3. **Apply config** — reads `file_prefix` from `SteamCloudBackendConfig`.

You can check if the Steam backend is ready at any time:

```gdscript
if EasyStore.is_backend_ready(StoreEnums.BackendType.STEAM_CLOUD):
    print("Steam Cloud is active!")
else:
    print("Steam Cloud not available — using local only")
```

## How Steam Cloud files are named

EasyStore stores two files per slot in Steam Remote Storage:

```
easystore_slot_0.sav          ← full save data
easystore_meta_0.meta.json    ← lightweight metadata
```

The `easystore_` prefix comes from `SteamCloudBackendConfig.file_prefix`. Steam namespaces these files automatically per App ID, so they won't collide with other games.

## Checking Steam quota

Steam Remote Storage has a quota per App ID (typically 100 MB for most games). EasyStore doesn't enforce quota limits — use `SteamCloudBackendConfig.quota_warning_bytes` to get a warning log when approaching your limit, then implement quota UI in your game if needed.

## Checking the backend status

```gdscript
# Is the backend in the active list (added but may have failed to init)?
if EasyStore.has_backend(StoreEnums.BackendType.STEAM_CLOUD):
    print("Steam backend was added")

# Is the backend fully initialized and ready to use?
if EasyStore.is_backend_ready(StoreEnums.BackendType.STEAM_CLOUD):
    print("Steam Cloud will receive saves")
```

## Full example with Local + Steam

```gdscript
# GLOBAL.gd
extends Node

func _ready() -> void:
    var config := EasyStoreConfig.new()
    config.conflict_strategy = StoreEnums.ConflictStrategy.NEWEST_WINS
    config.log_level         = StoreEnums.LogLevel.DEBUG

    await EasyStore.initialize(config)

    # Local: always available
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)

    # Steam: added safely — if unavailable, EasyStore ignores it
    EasyStore.add_backend(StoreEnums.BackendType.STEAM_CLOUD)

    # Sync local ↔ cloud on startup (resolves conflicts by timestamp)
    if EasyStore.is_backend_ready(StoreEnums.BackendType.STEAM_CLOUD):
        EasyStore.sync()
        await EasyStore.sync_completed
        print("Sync finished: ", EasyStore.get_active_backends())

    # Load the player's save
    EasyStore.load_completed.connect(_on_loaded)
    EasyStore.load_all()

func _on_loaded(slot: int, data: Dictionary, success: bool) -> void:
    if success:
        print("Save loaded! Player level: ", data.get("player", {}).get("level", 1))
    else:
        print("No save found for slot ", slot, " — starting fresh")
```

---

<!-- doc-shell:page slug="multi-backend" -->

# Multi-Backend

EasyStore is designed to run **multiple backends simultaneously**. When two or more backends are active, every `save()` call writes to all of them in parallel — so your player's progress is always protected both locally and in the cloud.

## Typical use case: Local + Steam Cloud

```
EasyStore.save("player", data)
    │
    ├── LocalBackend  → writes slot_0.sav      (background thread)
    └── SteamBackend  → writes to Steam Cloud  (Steam's own I/O)
```

If one backend fails (Steam offline, disk full, etc.), the other continues independently. EasyStore emits `error_occurred` for the failing backend, but the successful one still calls `save_completed`.

## Adding multiple backends

```gdscript
await EasyStore.initialize()
EasyStore.add_backend(StoreEnums.BackendType.LOCAL)
EasyStore.add_backend(StoreEnums.BackendType.STEAM_CLOUD)

print(EasyStore.get_active_backends())
# [0, 1]  ← BackendType.LOCAL=0, BackendType.STEAM_CLOUD=1
```

## Checking which backends are active

```gdscript
for bt in EasyStore.get_active_backends():
    match bt:
        StoreEnums.BackendType.LOCAL:  print("Local is active")
        StoreEnums.BackendType.STEAM_CLOUD:  print("Steam is active")
```

## Removing a backend at runtime

```gdscript
# If Steam goes offline mid-session, remove it gracefully
EasyStore.remove_backend(StoreEnums.BackendType.STEAM_CLOUD)
# Future saves go to Local only
```

## Sync — Resolving conflicts between backends

When a player plays on two different devices (one online, one offline), their local and cloud saves may diverge. `EasyStore.sync()` compares timestamps and resolves the conflict:

```gdscript
# Sync is automatic if SteamCloudBackendConfig.auto_sync_on_connect = true
# You can also call it manually at any time
await EasyStore.sync()
print("Sync done")
```

### Conflict strategies

Set `EasyStoreConfig.conflict_strategy` before calling `initialize()`:

```gdscript
config.conflict_strategy = StoreEnums.ConflictStrategy.NEWEST_WINS
```

| Strategy | Behavior |
|----------|----------|
| `NEWEST_WINS` | Compares `SaveMetadata.timestamp`. The backend with the most recent save wins. The older backend is overwritten. |
| `CLOUD_WINS` | Steam Cloud always wins, overwriting local data. |
| `LOCAL_WINS` | Local always wins, overwriting Steam Cloud. |
| `MANUAL` | EasyStore emits `sync_conflict` for each differing key. Your game shows a UI and calls `EasyStore.save()` to choose which version to keep. |

### Manual conflict resolution example

```gdscript
func _ready() -> void:
    EasyStore.sync_conflict.connect(_on_sync_conflict)
    EasyStore.sync()

func _on_sync_conflict(slot: int, key: String, local_data: Variant, cloud_data: Variant) -> void:
    # Show a dialog to the player so they choose which version to keep
    var dialog := $ConflictDialog
    dialog.setup(slot, key, local_data, cloud_data)
    dialog.popup()

# In the dialog:
func _on_keep_cloud_pressed() -> void:
    EasyStore.save(key, cloud_data)

func _on_keep_local_pressed() -> void:
    EasyStore.save(key, local_data)
```

## Load priority

When loading, EasyStore always prefers the **Local backend** first. If the slot doesn't exist locally, it tries the next available backend. After a successful `sync()`, both backends are identical and priority doesn't matter.

```gdscript
# Reads from Local if available; otherwise falls back to Steam
EasyStore.load_all()
```

---

<!-- doc-shell:page slug="slots" -->

# Save Slots

EasyStore supports multiple independent save slots out of the box. Each slot is a completely separate save with its own data, metadata, and files.

## The active slot

EasyStore always has an **active slot**. Operations that don't specify a slot (like `save()` and `load()`) act on the active slot.

```gdscript
EasyStore.set_slot(0)   # switch to slot 0
EasyStore.get_slot()    # returns 0
```

The default active slot after `initialize()` is `EasyStoreConfig.default_slot` (default: `0`).

## Listing all slots

```gdscript
var slots: Array[SaveMetadata] = EasyStore.list_slots()
for meta in slots:
    print("Slot %d — %s — %s" % [
        meta.slot,
        Time.get_datetime_string_from_unix_time(meta.timestamp),
        meta.custom.get("chapter", "Unknown")
    ])
```

`list_slots()` reads from the **metadata sidecar files** (`.meta.json`), which are small and fast to parse. The full save data is not loaded.

## Checking if a slot has data

```gdscript
if EasyStore.has_slot(1):
    print("Slot 1 has a save")
else:
    print("Slot 1 is empty")
```

## Switching slots

```gdscript
# Show the save select screen
for i in range(3):
    if EasyStore.has_slot(i):
        var meta := EasyStore.get_save_metadata(i)
        $SlotButton[i].text = "Slot %d — Level %d" % [i, meta.custom.get("level", 1)]
    else:
        $SlotButton[i].text = "Slot %d — Empty" % i

# Player selects slot 2
func _on_slot_2_pressed() -> void:
    EasyStore.set_slot(2)
    EasyStore.load_all()
    await EasyStore.load_completed
    get_tree().change_scene_to_file("res://scenes/game.tscn")
```

## Deleting a slot

```gdscript
EasyStore.delete_slot(1)
# Emits: slot_deleted(1)
# Removes files from all active backends and clears the in-memory cache
```

## Save metadata

Each slot carries a `SaveMetadata` object with descriptive information:

| Property | Type | Description |
|----------|------|-------------|
| `slot` | `int` | The slot index. |
| `timestamp` | `int` | Unix timestamp of when this slot was last saved. |
| `playtime_seconds` | `float` | Accumulated playtime in seconds (you update this). |
| `game_version` | `String` | Your game's version string (informational). |
| `save_version` | `int` | The save format version used when this slot was last saved. |
| `thumbnail_path` | `String` | Optional path to a screenshot taken at save time. |
| `custom` | `Dictionary` | Anything you want — chapter name, difficulty, player level, etc. |
| `is_empty` | `bool` | `true` if this slot has never been written to. |

```gdscript
var meta := EasyStore.get_save_metadata()   # gets metadata for the current slot
print("Last saved: ", Time.get_datetime_string_from_unix_time(meta.timestamp))
print("Playtime:   ", meta.playtime_seconds / 60.0, " minutes")
print("Chapter:    ", meta.custom.get("chapter", "Unknown"))
```

> EasyStore automatically fills `slot`, `timestamp`, and `is_empty` on every save. You are responsible for updating `playtime_seconds`, `game_version`, and `custom` through your game logic.

---

<!-- doc-shell:page slug="secciones" -->

# Sections & Cache

EasyStore organizes save data into **named sections**. Each section is an independent `Dictionary` that represents a logical part of your game state.

## Why sections?

Splitting data into sections gives you:
- **Partial loads** — read only `"settings"` at startup without loading all player data.
- **Partial saves** — if only the player moved, write `"player"` without re-writing `"world"`.
- **Clear organization** — `"player"`, `"world"`, `"settings"`, `"quests"` are instantly understandable.

## Writing to a section

```gdscript
EasyStore.save("player", {
    "health":   current_health,
    "mana":     current_mana,
    "level":    player_level,
    "position": Vector3(x, y, z),
})

EasyStore.save("settings", {
    "volume":   music_volume,
    "fullscreen": is_fullscreen,
    "language": "en",
})
```

Each call to `save()` **merges** the new data into the section (it does not erase previously written keys in the same save flush cycle). The section is marked **dirty** and will be included in the next backend write.

## Reading from a section

```gdscript
# If the section is in the cache, returns instantly (no I/O)
var player := EasyStore.load("player")
var health: int = player.get("health", 100)

# If the cache is empty (first load after startup), trigger a full load:
EasyStore.load_all()
await EasyStore.load_completed
var player := EasyStore.load("player")
```

## How the cache works

```
EasyStore.save("player", data)
    │
    ▼
SectionCache["slot_0"]["player"] = data   ← marked dirty
    │
    ▼  (async, on next flush)
SaveFile {
    sections: {
        "player":   {...},
        "world":    {...},
        "settings": {...},
    }
}  → written to backends
```

The cache holds all sections for all loaded slots in memory. Once loaded, subsequent `load()` calls are instant dictionary reads — no disk access.

**Dirty tracking** ensures only changed sections trigger a write. Calling `save_all()` builds a `SaveFile` from all dirty sections and dispatches a single write to each backend.

## Saving all sections at once

```gdscript
# Flush every dirty section to all backends in one write
EasyStore.save_all()
```

## Section naming convention (recommended)

| Section | Content |
|---------|---------|
| `"player"` | Health, stats, position, inventory |
| `"world"` | World state, opened doors, collected items |
| `"quests"` | Quest progress flags |
| `"settings"` | Audio, video, controls |
| `"achievements"` | Unlocked achievements |

---

<!-- doc-shell:page slug="autosave" -->

# Auto-save

EasyStore has a built-in auto-save system. Once enabled, it fires a timer in the background and calls `save_all()` automatically at the configured interval — without any extra code in your game.

## Enable / disable

```gdscript
# Start auto-saving every 2 minutes
EasyStore.enable_autosave(120.0)

# Stop auto-saving
EasyStore.disable_autosave()
```

The interval is in seconds. Common values:

| Interval | Seconds | When to use |
|----------|---------|-------------|
| 30 s | `30.0` | Fast-paced games where losing progress is very frustrating |
| 1 min | `60.0` | Action games |
| 2 min | `120.0` | RPGs and adventure games (**recommended default**) |
| 5 min | `300.0` | Slow-paced games, strategy |

## Reacting to the auto-save event

Use the `autosave_triggered` signal to show a save indicator in your UI:

```gdscript
func _ready() -> void:
    EasyStore.autosave_triggered.connect(_on_autosave)
    EasyStore.save_completed.connect(_on_save_completed)
    EasyStore.enable_autosave(120.0)

func _on_autosave(slot: int) -> void:
    $HUD/SaveIcon.show()  # show a spinning disk icon
    $HUD/SaveIcon.text = "Saving..."

func _on_save_completed(slot: int, success: bool) -> void:
    $HUD/SaveIcon.hide()
    if not success:
        $HUD/SaveLabel.text = "Save failed!"
```

## save_started and save_completed

These two signals bracket every save operation (both manual and auto-save):

| Signal | When |
|--------|------|
| `save_started(slot)` | Just before write is dispatched to backends |
| `save_completed(slot, success)` | When all active backends have finished |

```gdscript
EasyStore.save_started.connect(func(slot): print("Writing slot ", slot, "..."))
EasyStore.save_completed.connect(func(slot, ok): print("Done. Success: ", ok))
```

## load_started and load_completed

Similarly for loads:

| Signal | When |
|--------|------|
| `load_started(slot)` | Just before the backend read is dispatched |
| `load_completed(slot, data, success)` | When the data is available in memory |

```gdscript
EasyStore.load_started.connect(func(slot): $LoadScreen.show())
EasyStore.load_completed.connect(func(slot, data, ok): $LoadScreen.hide())
```

---

<!-- doc-shell:page slug="migraciones" -->

# Save Migrations

As your game evolves, your save format will change. EasyStore's migration system lets you write **version-aware upgrade functions** so old saves from earlier game versions are automatically updated when loaded — without losing player data.

## How versioning works

Every save file stores a `version` integer. You declare the current version in your config:

```gdscript
config.current_save_version = 3
EasyStore.initialize(config)
```

When EasyStore loads a save and finds `save.version < current_save_version`, it runs all registered migrations in order to bring it up to date.

## Registering migrations

```gdscript
# Run this BEFORE initialize(), so migrations are ready before any load happens

EasyStore.register_migration(1, 2, func(sections: Dictionary) -> Dictionary:
    # Version 1 → 2: "hp" was renamed to "health"
    if sections.has("player"):
        var p = sections["player"]
        if p.has("hp"):
            p["health"] = p["hp"]
            p.erase("hp")
    return sections
)

EasyStore.register_migration(2, 3, func(sections: Dictionary) -> Dictionary:
    # Version 2 → 3: new "stamina" stat added with a default of 100
    if sections.has("player"):
        sections["player"]["stamina"] = sections["player"].get("stamina", 100)
    # Version 2 → 3: "world" section now requires a "weather" key
    if sections.has("world"):
        sections["world"]["weather"] = sections["world"].get("weather", "clear")
    return sections
)
```

Each callable receives the full `sections: Dictionary` and must return the transformed dictionary.

## Migration signal

```gdscript
EasyStore.migration_applied.connect(func(old_ver: int, new_ver: int):
    print("Save migrated: v%d → v%d" % [old_ver, new_ver])
)
```

## Gaps in the migration chain

If you register 1→2 and 3→4 but NOT 2→3, EasyStore will apply 1→2, skip 2→3 with a warning, then apply 3→4. The save is not corrupted — only the missing step is skipped.

## Migrated data is not re-saved automatically

After migration, the updated data lives in the cache. It will be written to disk on the **next `save()` or `save_all()` call**. This is intentional — if something went wrong in your migration callback, the original file on disk is unchanged.

## Complete example with migrations

```gdscript
# GLOBAL.gd
extends Node

func _ready() -> void:
    # Register migrations FIRST
    EasyStore.register_migration(1, 2, _migrate_v1_to_v2)
    EasyStore.register_migration(2, 3, _migrate_v2_to_v3)

    # Then initialize with current version
    var config := EasyStoreConfig.new()
    config.current_save_version = 3
    await EasyStore.initialize(config)
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)

    EasyStore.migration_applied.connect(_on_migration)
    EasyStore.load_all()

func _migrate_v1_to_v2(sections: Dictionary) -> Dictionary:
    if sections.has("player"):
        sections["player"]["stamina"] = 100
    return sections

func _migrate_v2_to_v3(sections: Dictionary) -> Dictionary:
    if sections.has("world"):
        sections["world"].erase("legacy_seed")
        sections["world"]["biome"] = "forest"
    return sections

func _on_migration(old_ver: int, new_ver: int) -> void:
    print("Save file upgraded from v%d to v%d" % [old_ver, new_ver])
```

---

<!-- doc-shell:page slug="save-path" -->

# Save Files & Path

## Get the save path at runtime

```gdscript
# Returns the absolute OS path to the current slot's save file
var path := EasyStore.get_save_path()
print(path)
# Example: C:\Users\Alice\AppData\Roaming\Godot\app_userdata\MyGame\saves\slot_0.sav
```

This is useful for:
- Showing the save location in a settings menu
- Opening the folder with the OS file manager
- Debugging on a player's machine

```gdscript
# Show save path in a settings menu
func _on_open_save_folder_pressed() -> void:
    var path := EasyStore.get_save_path()
    var dir  := path.get_base_dir()
    OS.shell_open(dir)  # opens the folder in the OS file manager
```

## File format — JSON

The default format (`StoreEnums.SerializationFormat.JSON`) produces a human-readable `.sav` file:

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
        "custom": { "chapter": "act_2", "level": 14 }
    },
    "sections": {
        "player": { "health": 80, "level": 14, "coins": 1250 },
        "world":  { "level_name": "dungeon_01", "weather": "rain" },
        "settings": { "volume": 0.8, "language": "en" }
    }
}
```

## File format — BINARY

Setting `LocalBackendConfig.format = StoreEnums.SerializationFormat.BINARY` uses Godot's `var_to_bytes()` / `bytes_to_var()`. Binary files are smaller and slightly faster to parse, but not human-readable — useful for production builds when save file size matters.

## Metadata sidecar

Every slot also writes a small `.meta.json` sidecar alongside the main `.sav`:

```json
{
    "slot": 0,
    "timestamp": 1714500000,
    "playtime_seconds": 3720.5,
    "game_version": "1.2.0",
    "save_version": 3,
    "is_empty": false,
    "custom": { "chapter": "act_2", "level": 14 }
}
```

`EasyStore.list_slots()` reads **only the sidecars** — it never loads the full save files — making slot listing very fast regardless of save size.

---

<!-- doc-shell:page slug="custom-backends" -->

# Custom Backends

EasyStore is designed to be extended. If you want to sync saves with your own server, Google Play Games, Epic Games Store, or any other service, you can create a custom backend by implementing the `StorageBackend` interface.

## The StorageBackend contract

All backends extend `StorageBackend` (found at `addons/easystore/core/storage_backend.gd`). You must implement 8 virtual methods:

```gdscript
# backend_initialize: called by EasyStore after add_backend().
# Return OK on success, or any other Error code to signal failure.
func _backend_initialize(config: Resource) -> Error

# backend_shutdown: called by EasyStore when remove_backend() is called.
func _backend_shutdown() -> void

# Write a full SaveFile to your backend for the given slot.
# Must emit: backend_save_completed(slot, success, error_code)
func _backend_save(slot: int, save_file: SaveFile) -> void

# Read a SaveFile for the given slot from your backend.
# Must emit: backend_load_completed(slot, save_file, success, error_code)
func _backend_load(slot: int) -> void

# Delete all files for the given slot.
# Must emit: backend_delete_completed(slot, success)
func _backend_delete(slot: int) -> void

# List all available slots (reads metadata sidecars).
# Must emit: backend_list_completed(metadata_list: Array[SaveMetadata])
func _backend_list_slots() -> void

# Return true if the given slot has data in your backend.
func _backend_slot_exists(slot: int) -> bool

# Return a Dictionary describing your backend's capabilities.
func _backend_get_capabilities() -> Dictionary
```

**Signals your backend must emit:**

| Signal | Parameters | When |
|--------|------------|------|
| `backend_save_completed` | `slot, success, error_code` | After `_backend_save()` finishes |
| `backend_load_completed` | `slot, save_file, success, error_code` | After `_backend_load()` finishes |
| `backend_delete_completed` | `slot, success` | After `_backend_delete()` finishes |
| `backend_list_completed` | `metadata_list: Array` | After `_backend_list_slots()` finishes |
| `backend_error` | `code, message` | On any unexpected error |

## Step-by-step: adding a new backend

### 1 — Create the backend file

```
addons/easystore/backends/my_cloud/my_cloud_backend.gd
```

```gdscript
# my_cloud_backend.gd
class_name MyCloudBackend
extends StorageBackend

var _api_url: String = "https://api.mygame.com/saves"
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
    # ... similar HTTP DELETE request
    backend_delete_completed.emit(slot, true)


func _backend_list_slots() -> void:
    backend_list_completed.emit([])


func _backend_slot_exists(slot: int) -> bool:
    return false


func _backend_get_capabilities() -> Dictionary:
    return { "encryption": false, "compression": false, "cloud_sync": true }
```

### 2 — Add an entry to the BackendType enum

Open `addons/easystore/core/store_enums.gd`:

```gdscript
enum BackendType {
    LOCAL       = 0,
    STEAM_CLOUD = 1,
    GOOGLE_PLAY = 2,
    CUSTOM_HTTP = 3,   # ← your new backend
}
```

### 3 — Register the backend in EasyStore

Open `addons/easystore/easystore.gd`, find the `_create_backend()` function:

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
        StoreEnums.BackendType.CUSTOM_HTTP:      # ← add this case
            var b := MyCloudBackend.new()
            b.setup()
            return b
    return null
```

### 4 — (Optional) Add a config resource

Create `addons/easystore/config/my_cloud_backend_config.gd`:

```gdscript
class_name MyCloudBackendConfig
extends Resource

@export var api_url:    String = "https://api.mygame.com/saves"
@export var auth_token: String = ""
```

Then add it to `EasyStoreConfig`:

```gdscript
@export var my_cloud: MyCloudBackendConfig
```

And return it in `_resolve_default_config()` inside `easystore.gd`.

### 5 — Use your new backend

```gdscript
EasyStore.add_backend(StoreEnums.BackendType.CUSTOM_HTTP, my_cloud_config)
```

That's it! No other files need to change. EasyStore will automatically route saves, loads, and syncs to your new backend alongside any existing backends.

---

<!-- doc-shell:page slug="api" -->

# API Reference

Complete reference for all public methods of the **`EasyStore`** Autoload.

## Setup

| Method | Returns | Description |
|--------|---------|-------------|
| `initialize(config)` | `void` | Initializes EasyStore. Pass an `EasyStoreConfig` or `null` for defaults. Call once at startup. **Use `await`** — it may suspend internally if EasyStore's node isn't ready yet due to autoload ordering. |

## Backend management

| Method | Returns | Description |
|--------|---------|-------------|
| `add_backend(type, config)` | `void` | Adds and initializes a backend. `config` is optional — uses defaults from `EasyStoreConfig` if `null`. Multiple backends can be active simultaneously. |
| `remove_backend(type)` | `void` | Shuts down and removes a backend. Future saves will not go to it. |
| `get_active_backends()` | `Array[StoreEnums.BackendType]` | Returns a list of currently active (initialized) backend types. |
| `has_backend(type)` | `bool` | `true` if a backend of this type is in the active list. |
| `is_backend_ready(type)` | `bool` | `true` if a backend of this type is active **and** fully initialized. Use this to check if Steam Cloud is available. |

## Slot management

| Method | Returns | Description |
|--------|---------|-------------|
| `set_slot(slot)` | `void` | Sets the active save slot. |
| `get_slot()` | `int` | Returns the active save slot index. |
| `list_slots()` | `Array[SaveMetadata]` | Returns metadata for all known slots (reads from sidecar files only — fast). |
| `has_slot(slot)` | `bool` | `true` if the given slot has save data. |
| `delete_slot(slot)` | `void` | Deletes all files for this slot from all active backends and clears the cache. Emits `slot_deleted`. |
| `get_save_metadata(slot)` | `SaveMetadata` | Returns metadata for the slot, or `null` if unknown. `slot = -1` uses the active slot. |

## Save & Load

| Method | Returns | Description |
|--------|---------|-------------|
| `save(section, data, slot)` | `void` | Writes `data` to the named section. Dispatched to all active backends asynchronously. `slot = -1` uses the active slot. Emits `save_started` and `save_completed`. |
| `save_all(slot)` | `void` | Flushes all dirty cached sections for the slot to all backends. `slot = -1` uses the active slot. |
| `load(section, slot)` | `Dictionary` | Returns the section from cache if available. If not cached, triggers an async load and returns `{}`. Prefer `load_all()` + `await load_completed` for reliable data. |
| `load_all(slot)` | `void` | Triggers a full async load for the slot from the highest-priority backend. Emits `load_started` and `load_completed`. |
| `get_save_path(slot)` | `String` | Returns the absolute OS path for the slot's save file (Local backend only). Returns `""` if no local backend is active. |

## Multi-backend sync

| Method | Returns | Description |
|--------|---------|-------------|
| `sync(slot)` | `void` | Compares metadata timestamps between all active backends and resolves conflicts per the configured `ConflictStrategy`. Emits `sync_completed`. |

## Auto-save

| Method | Returns | Description |
|--------|---------|-------------|
| `enable_autosave(interval_seconds)` | `void` | Starts the auto-save timer. Default interval: `60.0` seconds. |
| `disable_autosave()` | `void` | Stops the auto-save timer. |

## Migrations

| Method | Returns | Description |
|--------|---------|-------------|
| `register_migration(from_version, to_version, fn)` | `void` | Registers a migration callable. `fn` receives and must return `sections: Dictionary`. |
| `set_current_version(version)` | `void` | Sets the current save version. Equivalent to `EasyStoreConfig.current_save_version`. |

## Debug

| Method | Returns | Description |
|--------|---------|-------------|
| `debug_mode(enabled)` | `void` | Shortcut to toggle logging. `true` → `LogLevel.DEBUG`, `false` → `LogLevel.NONE`. |
| `set_log_level(level)` | `void` | Sets the log verbosity level directly. More granular than `debug_mode()`. Example: `EasyStore.set_log_level(StoreEnums.LogLevel.DEBUG)`. |
| `get_logs(limit)` | `Array[Dictionary]` | Returns the last `limit` log entries. `limit = 0` returns all. |
| `get_debug_info()` | `Dictionary` | Returns a snapshot of the current EasyStore state (backends, slot, cache status, etc.). |

---

<!-- doc-shell:page slug="senales" -->

# Signals Reference

EasyStore exposes **13 signals** covering the entire save/load lifecycle. Connect to them to react to storage events without polling.

## Save / Load

| Signal | Parameters | When emitted |
|--------|------------|--------------|
| `save_started` | `slot: int` | Just before a save is dispatched to the backends. Use this to show a saving indicator. |
| `save_completed` | `slot: int, success: bool` | When all active backends have finished writing. `success` is `false` if any backend failed. |
| `load_started` | `slot: int` | Just before a backend read is dispatched. Use this to show a loading screen. |
| `load_completed` | `slot: int, data: Dictionary, success: bool` | When the save data is available in memory. `data` contains all sections. `success` is `false` if no save was found. |

## Backend management

| Signal | Parameters | When emitted |
|--------|------------|--------------|
| `backend_added` | `type: StoreEnums.BackendType` | After a backend is successfully initialized and added. |
| `backend_removed` | `type: StoreEnums.BackendType` | After a backend is removed. |

## Slots

| Signal | Parameters | When emitted |
|--------|------------|--------------|
| `slot_deleted` | `slot: int` | After a slot is deleted from all backends. |

## Auto-save

| Signal | Parameters | When emitted |
|--------|------------|--------------|
| `autosave_triggered` | `slot: int` | By the auto-save timer, before the save is dispatched. Use this to show a save icon. |

## Migrations

| Signal | Parameters | When emitted |
|--------|------------|--------------|
| `migration_applied` | `old_version: int, new_version: int` | Each time a migration step transforms a loaded save. |

## Sync

| Signal | Parameters | When emitted |
|--------|------------|--------------|
| `sync_completed` | `result: Dictionary` | When a multi-backend sync finishes. `result` maps `BackendType → bool`. |
| `sync_conflict` | `slot: int, key: String, local_data: Variant, cloud_data: Variant` | During a `MANUAL` strategy sync, for each section where backends disagree. |

## Errors

| Signal | Parameters | When emitted |
|--------|------------|--------------|
| `error_occurred` | `code: int, message: String` | On any internal error (backend failure, parse error, etc.). |

---

## Example — Connecting all critical signals

```gdscript
func _ready() -> void:
    EasyStore.save_started.connect(func(slot):
        $HUD/SaveIcon.show()
    )
    EasyStore.save_completed.connect(func(slot, success):
        $HUD/SaveIcon.hide()
        if not success:
            $HUD/ErrorLabel.text = "Save failed!"
    )
    EasyStore.load_started.connect(func(slot):
        $LoadScreen.show()
    )
    EasyStore.load_completed.connect(func(slot, data, success):
        $LoadScreen.hide()
        if success:
            _apply_save_data(data)
        else:
            _start_new_game()
    )
    EasyStore.error_occurred.connect(func(code, msg):
        push_warning("[EasyStore] Error %d: %s" % [code, msg])
    )
    EasyStore.migration_applied.connect(func(old, new):
        print("Save migrated: v%d → v%d" % [old, new])
    )
```

---

<!-- doc-shell:page slug="enums" -->

# Enums Reference

All enums are defined in `addons/easystore/core/store_enums.gd` and accessed via the `StoreEnums` class.

## StoreEnums.BackendType

```gdscript
enum BackendType {
    LOCAL       = 0,   # Local filesystem (user://)
    STEAM_CLOUD = 1,   # Steam Remote Storage via GodotSteam
    GOOGLE_PLAY = 2,   # Reserved for future Google Play Games backend
    CUSTOM_HTTP = 3,   # Reserved for custom HTTP/REST backends
}
```

## StoreEnums.ConflictStrategy

```gdscript
enum ConflictStrategy {
    NEWEST_WINS = 0,   # The backend with the most recent timestamp overwrites the other
    CLOUD_WINS  = 1,   # The cloud backend always wins
    LOCAL_WINS  = 2,   # The local backend always wins
    MANUAL      = 3,   # Emits sync_conflict for each differing section
}
```

## StoreEnums.SerializationFormat

```gdscript
enum SerializationFormat {
    JSON   = 0,   # Human-readable JSON (default)
    BINARY = 1,   # Godot var_to_bytes — smaller, not human-readable
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

Controls which log messages are printed to the console. All messages are always buffered internally regardless of level, so `get_logs()` always returns the full history.

```gdscript
enum LogLevel {
    NONE  = 0,   # Silent — no console output
    ERROR = 1,   # Errors only
    WARN  = 2,   # Errors and warnings
    INFO  = 3,   # Normal operation events (default)
    DEBUG = 4,   # Detailed internal events
    TRACE = 5,   # Extremely verbose — every internal step
}
```

---

<!-- doc-shell:page slug="debug" -->

# Debug & Diagnostics

EasyStore includes a logging and diagnostics system that you can use during development, in QA, and optionally in production.

Log output matches [LinkUx](https://github.com/iuxgames/LinkUx)'s format for consistency:
```
[EasyStore] LEVEL [Context]: message
```

## Setting the log level

```gdscript
# In configuration (before initialize) — INFO is the default
var config := EasyStoreConfig.new()
config.log_level = StoreEnums.LogLevel.DEBUG   # or INFO, WARN, ERROR, NONE
await EasyStore.initialize(config)

# Toggle at runtime
EasyStore.debug_mode(true)                              # shortcut → sets LogLevel.DEBUG
EasyStore.debug_mode(false)                             # shortcut → sets LogLevel.NONE
EasyStore.set_log_level(StoreEnums.LogLevel.DEBUG)      # granular control
```

| Level | Value | What gets printed |
|-------|-------|-------------------|
| `NONE` | `0` | Nothing |
| `ERROR` | `1` | Errors only |
| `WARN` | `2` | Errors + warnings |
| `INFO` | `3` | Normal operation (default) |
| `DEBUG` | `4` | Detailed internal events |
| `TRACE` | `5` | Every internal step |

EasyStore always buffers up to 500 log entries internally, regardless of the active level. Use `get_logs()` to retrieve them at any time.

## Reading log entries

```gdscript
# Get the last 50 log entries
var logs := EasyStore.get_logs(50)
for entry in logs:
    print("[%s] [%s]: %s" % [entry["level"], entry["context"], entry["message"]])

# Get all entries
var all_logs := EasyStore.get_logs(0)
```

Each log entry is a `Dictionary`:

```gdscript
{
    "level":     "INFO",          # level name string
    "context":   "Save",          # subsystem that emitted the log
    "message":   "Slot 0 saved successfully.",
    "timestamp": 1714500000,      # Unix timestamp
}
```

## Getting runtime debug info

```gdscript
var info := EasyStore.get_debug_info()
print(JSON.stringify(info, "  "))
# Output:
# {
#   "initialized": true,
#   "active_slot": 0,
#   "backends": [0, 1],
#   "cache_slots": [0],
#   "autosave_enabled": true,
#   "current_version": 3
# }
```

## In-game debug panel (example)

```gdscript
# debug_panel.gd
extends PanelContainer

func _process(_delta: float) -> void:
    if not visible:
        return
    var info := EasyStore.get_debug_info()
    $Grid/Backends.text = str(EasyStore.get_active_backends())
    $Grid/Slot.text     = str(EasyStore.get_slot())
    $Grid/Steam.text    = str(EasyStore.is_backend_ready(StoreEnums.BackendType.STEAM_CLOUD))
    $Grid/Version.text  = str(info.get("current_version", "?"))

    var logs := EasyStore.get_logs(5)
    var log_text := ""
    for entry in logs:
        log_text += "[%s] [%s]: %s\n" % [entry["level"], entry["context"], entry["message"]]
    $LogLabel.text = log_text
```

## Console output example

With `log_level = StoreEnums.LogLevel.INFO` (default), the Godot Output panel shows:

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

# Full Example

A realistic save system for a single-player RPG using EasyStore. Demonstrates multi-slot support, Steam Cloud, encryption, auto-save, migrations, and UI integration.

## Project structure

```
project/
├── autoloads/
│   └── GLOBAL.gd          ← Initializes EasyStore
├── scenes/
│   ├── main_menu.tscn      ← Slot selection screen
│   ├── game.tscn           ← Gameplay scene
│   └── hud.tscn            ← HUD with save indicator
└── scripts/
    ├── main_menu.gd
    ├── game.gd
    └── hud.gd
```

---

## GLOBAL.gd — Game autoload

```gdscript
# autoloads/GLOBAL.gd
extends Node

func _ready() -> void:
    _setup_easystore()

func _setup_easystore() -> void:
    # Register migrations BEFORE initialize
    EasyStore.register_migration(1, 2, func(sections: Dictionary) -> Dictionary:
        if sections.has("player"):
            sections["player"]["stamina"] = 100
        return sections
    )
    EasyStore.register_migration(2, 3, func(sections: Dictionary) -> Dictionary:
        if sections.has("world"):
            sections["world"]["weather"] = sections["world"].get("weather", "clear")
        return sections
    )

    # Configuration
    var config := EasyStoreConfig.new()
    config.current_save_version  = 3
    config.conflict_strategy     = StoreEnums.ConflictStrategy.NEWEST_WINS
    config.log_level             = StoreEnums.LogLevel.INFO if OS.is_debug_build() else StoreEnums.LogLevel.NONE

    config.local                 = LocalBackendConfig.new()
    config.local.encrypt         = true
    config.local.encryption_key  = "my-game-secret-key"
    config.local.max_backups     = 2

    config.steam_cloud                 = SteamCloudBackendConfig.new()
    config.steam_cloud.file_prefix     = "mygame_"

    await EasyStore.initialize(config)

    # Always add local backend first
    EasyStore.add_backend(StoreEnums.BackendType.LOCAL)

    # Add Steam — safe if unavailable
    EasyStore.add_backend(StoreEnums.BackendType.STEAM_CLOUD)

    # Sync if Steam is available
    if EasyStore.is_backend_ready(StoreEnums.BackendType.STEAM_CLOUD):
        EasyStore.sync()

    # Connect error handler
    EasyStore.error_occurred.connect(func(code, msg):
        push_warning("[GLOBAL] EasyStore error %d: %s" % [code, msg])
    )
    EasyStore.migration_applied.connect(func(old, new):
        print("Save upgraded: v%d → v%d" % [old, new])
    )
```

---

## main_menu.gd — Slot selection

```gdscript
# scripts/main_menu.gd
extends Control

@onready var slot_buttons := [$Slot0, $Slot1, $Slot2]

func _ready() -> void:
    _refresh_slots()

func _refresh_slots() -> void:
    var slots := EasyStore.list_slots()
    var slot_map := {}
    for meta in slots:
        slot_map[meta.slot] = meta

    for i in range(3):
        var btn: Button = slot_buttons[i]
        if slot_map.has(i):
            var meta: SaveMetadata = slot_map[i]
            var dt := Time.get_datetime_string_from_unix_time(meta.timestamp)
            var mins := int(meta.playtime_seconds / 60)
            btn.text = "Slot %d — Level %d\n%s (%d min)" % [
                i,
                meta.custom.get("level", 1),
                dt,
                mins,
            ]
        else:
            btn.text = "Slot %d — New Game" % i

func _on_slot_pressed(slot_index: int) -> void:
    EasyStore.set_slot(slot_index)
    if EasyStore.has_slot(slot_index):
        _load_and_enter(slot_index)
    else:
        _start_new_game(slot_index)

func _load_and_enter(slot: int) -> void:
    $LoadScreen.show()
    EasyStore.load_completed.connect(_on_loaded, CONNECT_ONE_SHOT)
    EasyStore.load_all(slot)

func _on_loaded(slot: int, data: Dictionary, success: bool) -> void:
    $LoadScreen.hide()
    if success:
        GameState.apply_save_data(data)
        get_tree().change_scene_to_file("res://scenes/game.tscn")
    else:
        _show_error("Failed to load save.")

func _start_new_game(slot: int) -> void:
    GameState.reset()
    GameState.current_slot = slot
    get_tree().change_scene_to_file("res://scenes/game.tscn")

func _on_delete_pressed(slot_index: int) -> void:
    EasyStore.delete_slot(slot_index)
    _refresh_slots()
```

---

## game.gd — Gameplay scene

```gdscript
# scripts/game.gd
extends Node

var playtime: float = 0.0

func _ready() -> void:
    # Start auto-save every 90 seconds
    EasyStore.enable_autosave(90.0)
    EasyStore.autosave_triggered.connect(_on_autosave)
    EasyStore.save_completed.connect(_on_save_completed)

func _process(delta: float) -> void:
    playtime += delta

# Called when the player reaches a checkpoint
func on_checkpoint_reached() -> void:
    _do_save()

# Called on game exit
func _notification(what: int) -> void:
    if what == NOTIFICATION_WM_CLOSE_REQUEST:
        EasyStore.disable_autosave()
        _do_save()

func _do_save() -> void:
    # Update metadata custom fields before saving
    var player_data := {
        "health":   $Player.health,
        "level":    $Player.level,
        "position": $Player.global_position,
        "inventory": $Player.inventory.to_dict(),
    }
    var world_data := {
        "level_name": get_tree().current_scene.name,
        "opened_doors": $World.opened_door_ids,
        "collected_items": $World.collected_item_ids,
        "weather": $World.weather,
    }

    EasyStore.save("player", player_data)
    EasyStore.save("world", world_data)
    # save_all() is called implicitly on the next autosave flush,
    # or you can force it now:
    EasyStore.save_all()

func _on_autosave(slot: int) -> void:
    $HUD.show_save_indicator()

func _on_save_completed(slot: int, success: bool) -> void:
    $HUD.hide_save_indicator()
    if not success:
        $HUD.show_error("Auto-save failed!")
```

---

## hud.gd — HUD save indicator

```gdscript
# scripts/hud.gd
extends CanvasLayer

@onready var save_icon    := $SaveIcon
@onready var error_label  := $ErrorLabel

func _ready() -> void:
    save_icon.hide()
    error_label.hide()

func show_save_indicator() -> void:
    save_icon.show()
    # Start a spin animation if you have one:
    $SaveIcon/AnimationPlayer.play("spin")

func hide_save_indicator() -> void:
    $SaveIcon/AnimationPlayer.stop()
    save_icon.hide()

func show_error(msg: String) -> void:
    error_label.text = msg
    error_label.show()
    await get_tree().create_timer(3.0).timeout
    error_label.hide()
```

---

## Complete flow

```
1. GLOBAL._ready()
   └── EasyStore.initialize() + add_backend(LOCAL) + add_backend(STEAM)
   └── Migrations registered
   └── Sync() if Steam available

2. MainMenu — player picks Slot 1
   └── EasyStore.set_slot(1) + load_all(1)
   └── load_completed → apply data → change_scene("game.tscn")

3. Game — player plays
   └── enable_autosave(90)
   └── Every 90s: autosave_triggered → HUD shows icon → save_all()
   └── save_completed → HUD hides icon

4. Player reaches checkpoint:
   └── save("player", data) + save("world", data) + save_all()
   └── Writes to LOCAL and STEAM simultaneously

5. Player quits:
   └── disable_autosave() + save_all()
   └── Game closes cleanly
```
