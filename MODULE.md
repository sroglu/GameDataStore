# GameDataStore

## Purpose
Two-layer data system: **Entries** defines a union-style value type (`Entry`) that can hold many primitive and custom types in a single struct; **DataStore** provides runtime data persistence and a database registry pattern.

## Assemblies

| Assembly | Path | Depends On |
|---|---|---|
| `mehmetsrl.GameDataStore.Entries` | `DataEntry/mehmetsrl.GameDataStore.Entries.asmdef` | Unity.Mathematics, ZString |
| `mehmetsrl.GameDataStore.Entries.Editor` | `DataEntry/Editor/mehmetsrl.GameDataStore.Entries.Editor.asmdef` | GameDataStore.Entries (Editor only) |
| `mehmetsrl.GameDataStore.Storage` | `DataStorage/mehmetsrl.GameDataStore.Storage.asmdef` | GameDataStore.Entries, Utilities.DataType, Utilities, ZString, Utilities.DataTypeTools, Unity.Mathematics |

## Key Classes

### Entries (`mehmetsrl.GameDataStore.Entries`)
- **`Entry`** — `[StructLayout.Explicit]` union struct holding int, float, bool, string, int2/int3, float2/float3, double, char, Entity, EntityLevel, EntityType, WeekDay. Implements `IEquatable`, `IComparable`, `IConvertible`
- **`EntryType`** — Enum of all supported Entry value types
- **`EntryTypeInfo`** / **`EntryComponentInfo`** — Metadata records describing an EntryType's fields
- **`ProcessedKey`** — String-based key for DataStore lookups
- **`GenericGameData`** — Pairs a `ProcessedKey` with an `Entry`; used for data injection into bindings
- **`EntryManager`** — Factory/converter for Entry values
- **`ITypeId`** / **`TypeIdTable`** — Type identity abstraction
- **`DataDefinition`**, **`DataDefinitionConfig`**, **`DataDefinitionList`**, **`DataDefinitionMap`** — ScriptableObject-based type definition configuration
- **Custom types**: `Entity`, `EntityType`, `EntityLevel`, `WeekDay`

### DataStore (`mehmetsrl.GameDataStore.Storage`)
- **`DataStoreManager`** — Singleton registry of `IDataStoreDatabase` instances; create/register/get/dispose databases
- **`IDataStoreDatabase`** — Database interface
- **`DataStoreDatabase`** — Abstract base database with persistence hooks
- **`LocalDataStore`** — Concrete implementation for local (in-memory/disk) storage
- **`DataStoreAttributes`** — Decorator attributes for database configuration

## Public API

```csharp
// Entry — create and compare typed values
Entry intEntry = new Entry(42);
Entry strEntry = new Entry("hello");
bool same = intEntry == new Entry(42); // true

// ProcessedKey — lightweight string key
var key = new ProcessedKey("player/health");

// DataStore
DataStoreManager.Instance.RegisterDatabase<PlayerDatabase>();
var db = DataStoreManager.Instance.GetDatabase<PlayerDatabase>();
db.Set(key, new Entry(100));
var health = db.Get(key); // Entry(100)
DataStoreManager.Instance.Dispose<PlayerDatabase>();
```

## File Structure
```
DataEntry/
  mehmetsrl.GameDataStore.Entries.asmdef
  EntryManager.cs
  Entry/EntryTypeInfo.cs
  Custom/TypeId/ITypeId.cs
  Custom/TypeId/TypeIdTable.cs
  Custom/TypeId/DataDefinitions/DataDefinition.cs
  Custom/TypeId/DataDefinitions/DataDefinitionConfig.cs
  Custom/TypeId/DataDefinitions/DataDefinitionList.cs
  Custom/TypeId/DataDefinitions/DataDefinitionMap.cs
  Custom/Types/DateTime/WeekDay.cs
  Custom/Types/Entity/Entity.cs
  Custom/Types/Entity/EntityLevel.cs
  Custom/Types/Entity/EntityType.cs
  Editor/
    mehmetsrl.GameDataStore.Entries.Editor.asmdef
    EntryPropertyDrawer.cs
DataStorage/
  mehmetsrl.GameDataStore.Storage.asmdef
  DataStoreAttributes.cs
  DataStoreManager.cs
  DataStoreDatabase.cs
Test/
  mehmetsrl.GameDataStore.Test.asmdef
  DataStoreTest.cs
  BookDataStore.cs
```

## Downstream Dependents

`mehmetsrl.Bindings`, `mehmetsrl.Presenter`, `mehmetsrl.ScreenManagement` (all in Infrastructural's Reusable layer) consume `Entries` and/or `Storage` via GUID asmdef refs — those refs survived the rename so consumer code only needs the namespace update (`using mehmetsrl.DataManagement.*` → `using mehmetsrl.GameDataStore.*`).

## Naming History

Originally named `DataManagement` (mehmetsrl.DataManagement.Entries / .DataStore). Renamed to `GameDataStore` in April 2026 because "DataManagement" was too generic and overlapped semantically with `AssetManagement`. The DataStore asmdef is now `Storage` (not `GameDataStore.DataStore`) to avoid the awkward name doubling.

## Scope Clarification

GameDataStore is a **runtime in-game data modeling layer** — NOT an app-level save/persistence system. Understand the distinction before picking where to write things:

- **Use GameDataStore when:** modeling typed game data at runtime (player stats as `Entry` values, item definitions as `DataDefinition` SOs, databases indexed by `ProcessedKey`), or when you need a typed Entry-union for bindings/UI.
- **Do NOT use GameDataStore when:** writing JSON save files with schema versioning, atomic file writes, migrations, or multi-namespace profile data. These concerns belong to an app-level save layer (e.g., a hub-app `Save` module built on `Utilities/FileSystemTools` atomic write).

The two can coexist: an app's save layer serializes a `GameDataStore` database snapshot into its own JSON schema on save, and rehydrates the database on load. GameDataStore does not own the file format, the atomic write, or the migration pipeline.

## Limitations / Known Gaps

- **No JSON persistence or atomic-write built in.** `LocalDataStore` can hold state in memory and persist via derived-class hooks, but the file I/O semantics (atomic write, fsync, .bak fallback, schema version migration) are NOT part of this module. Build those at the app-save layer.
- **`Entry` uses `[FieldOffset]` explicit overlay.** Reference fields (e.g., `string`) overlap value fields (e.g., `int`, `float`). Reading an overlapping slot after writing a different variant yields undefined results. Always read the variant that matches what was last written — `EntryType` tracks this.
- **`DataStoreManager` is a process singleton.** Initialize once (typically via a SubSystem) before any consumer touches `DataStoreManager.Instance`. There is no built-in per-profile scoping — if you need isolation (e.g., multi-user on one device), wrap the manager per profile yourself or namespace keys in `ProcessedKey`.
- **`ProcessedKey` is a flat string.** No hierarchical namespace built in — keys are opaque strings. Convention is slash-delimited (`player/health`, `inventory/potion_small`) but no API enforces it.
- **No built-in change notifications.** Mutating a database's Entry does not emit a signal or event. If downstream UI needs to react to data changes, wire Signaling (`SignalTracker.Emit<DataChangedSignal>()`) at the mutation call site, or wrap the database.
- **GUID asmdef refs survive the rename.** Consumers (`Bindings`, `Presenter`, `ScreenManagement`) reference GameDataStore via asmdef GUIDs, so the DataManagement → GameDataStore rename does not break references. Only `using` directives in source code need updating.

## Notes
- `Entry` uses `[FieldOffset]` overlapping — be aware when holding reference types (string) alongside value types
- `DataStoreManager` is a **singleton** — initialize in a SubSystem before consumers
- `ProcessedKey` is a simple string wrapper used as the flat key type
