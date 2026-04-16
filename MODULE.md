# DataManagement

## Purpose
Two-layer data system: **Entries** defines a union-style value type (`Entry`) that can hold many primitive and custom types in a single struct; **DataStore** provides runtime data persistence and a database registry pattern.

## Assemblies

| Assembly | Path | Depends On |
|---|---|---|
| `mehmetsrl.DataManagement.Entries` | `DataEntry/mehmetsrl.DataManagement.Entries.asmdef` | Unity.Mathematics, ZString |
| `mehmetsrl.DataManagement.Entries.Editor` | `DataEntry/Editor/mehmetsrl.DataManagement.Entries.Editor.asmdef` | DataManagement.Entries (Editor only) |
| `mehmetsrl.DataManagement.DataStore` | `DataStorage/mehmetsrl.DataManagement.DataStore.asmdef` | DataManagement.Entries, Utilities.DataType, Utilities, ZString, Utilities.DataTypeTools, Unity.Mathematics |

## Key Classes

### Entries (`mehmetsrl.DataManagement.Entries`)
- **`Entry`** — `[StructLayout.Explicit]` union struct holding int, float, bool, string, int2/int3, float2/float3, double, char, Entity, EntityLevel, EntityType, WeekDay. Implements `IEquatable`, `IComparable`, `IConvertible`
- **`EntryType`** — Enum of all supported Entry value types
- **`EntryTypeInfo`** / **`EntryComponentInfo`** — Metadata records describing an EntryType's fields
- **`ProcessedKey`** — String-based key for DataStore lookups
- **`GenericGameData`** — Pairs a `ProcessedKey` with an `Entry`; used for data injection into bindings
- **`EntryManager`** — Factory/converter for Entry values
- **`ITypeId`** / **`TypeIdTable`** — Type identity abstraction
- **`DataDefinition`**, **`DataDefinitionConfig`**, **`DataDefinitionList`**, **`DataDefinitionMap`** — ScriptableObject-based type definition configuration
- **Custom types**: `Entity`, `EntityType`, `EntityLevel`, `WeekDay`

### DataStore (`mehmetsrl.DataManagement.DataStore`)
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
DataManagement/DataEntry/
  mehmetsrl.DataManagement.Entries.asmdef
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
    mehmetsrl.DataManagement.Entries.Editor.asmdef
    EntryPropertyDrawer.cs
DataManagement/DataStorage/
  mehmetsrl.DataManagement.DataStore.asmdef
  DataStoreAttributes.cs
  DataStoreManager.cs
  DataStoreDatabase.cs
DataManagement/Test/
  mehmetsrl.DataManagement.Test.asmdef
  DataStoreTest.cs
  BookDataStore.cs
```

## Downstream Dependents
`mehmetsrl.Bindings`, `mehmetsrl.Presenter`, `mehmetsrl.ScreenManagement`, `mehmetsrl.DataManagement.DataStore`

## Notes
- `Entry` uses `[FieldOffset]` overlapping — be aware when holding reference types (string) alongside value types
- `DataStoreManager` is a **singleton** — initialize in a SubSystem before consumers
- `ProcessedKey` is a simple string wrapper used as the flat key type
