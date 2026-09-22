# Plan: Save Jotepad lists per character

## Goal

Change Jotepad so that each Valheim character keeps its own task list instead of sharing one list across the entire install.

## Current behavior

The mod currently saves a single config entry:

```csharp
JotepadStringConfig = Config.Bind("Client config", "JotepadString", "", "Jotepad list");
```

`SaveList()` and `ReadList()` operate on this single value, so all characters on the same install read and write the same tasks.

## Desired behavior

- Each character has its own task list.
- Switching characters loads that character's list.
- Adding, removing, or clearing tasks only affects the active character.
- Existing single-install data should be migrated safely when possible.

## Implementation plan

### 1. Add character-aware storage key logic

Create a helper to generate a storage key from the active character context.

Suggested structure:

```csharp
private string GetCharacterKey()
{
    // Use stable character/player id if available
    // Fallback to character name only if needed
}
```

Store data in a dictionary-like config structure keyed by the active character.

Example pattern:

```csharp
private ConfigEntry<string> JotepadStringConfig;
private string GetStorageKey()
{
    // e.g. "JotepadString_" + characterId
}
```

Prefer a stable, unique player/character identifier over the display name.

### 2. Replace single global config access with character-scoped config values

Instead of a single config value for all characters, define one config value per character or serialize multiple character entries into one config string.

Preferred approach:

- Keep one config key for all character data.
- Serialize the data as a small JSON object.

Example:

```json
{
  "char_12345": ["Build a house", "Collect wood"],
  "char_67890": ["Defeat Eikthyr"]
}
```

This keeps the BepInEx config simple and avoids creating a bunch of separate config entries.

### 3. Load the right list when the character loads

Current code reads the list during `Awake()`:

```csharp
private void Awake()
{
    numItems = 0;
    CreateConfigValues();
    ReadList();
    AddInputs();
}
```

This should be moved to a later point when the local player/character is available.

Add a safe loading path such as:

- after `Player.m_localPlayer` becomes available,
- after the player profile/character is fully initialized,
- or inside a delayed callback to retry until the character is ready.

### 4. Save to the active character only

Update `SaveList()` so it writes the list for the current active character, not the global per-install list.

Pseudo-flow:

```csharp
private void SaveList()
{
    var data = LoadAllCharacterData();
    data[GetCharacterKey()] = GetCurrentItems();
    SaveAllCharacterData(data);
}
```

### 5. Ensure the UI reflects the active character's list

The UI and item array logic can remain basically the same; only the backing data source changes.

- `ReadList()` should load data for the active character.
- `SaveList()` should persist only the current character's list.
- `ClearJotepad()` should clear only the active character's list.

### 6. Preserve backward compatibility for existing installs

If users already have a single global list, migrate it once when the mod loads.

Example behavior:

- If data for the current character is absent and a legacy single-entry value exists,
- copy the legacy list into a new key for the current character,
- then keep the single-entry value for compatibility or remove it after migration.

This avoids losing existing task data.

### 7. Validate with character switching behavior

Test the following cases:

- Character A creates tasks, logs out.
- Character B loads and sees an empty list.
- Character A logs back in and sees its own tasks.
- Clear List affects only the active character.
- Existing legacy data migrates correctly.

## Recommended data model

Use a single serialized data block rather than one BepInEx config key per character.

Example:

```csharp
private string GetAllCharacterDataString()
private Dictionary<string, string[]> LoadCharacterLists()
private void SaveCharacterLists(Dictionary<string, string[]> data)
```

This minimizes migration complexity and avoids needing to register many config entries.

## Risks / notes

- Character IDs are more reliable than character names.
- The mod currently reads at `Awake()`, which may be too early for character identity to be available.
- If the mod uses a JSON string inside config, ensure the serialized data cannot conflict with the current `SERIAL_TOKEN` logic.

## Acceptance criteria

- Each character gets an independent task list.
- Switching characters loads the correct saved list.
- Existing users do not lose their current task list after update.
- The current UI behavior remains intact.

## Suggested next step

Start by refactoring the storage layer first, before touching the visual UI logic. This keeps the change focused and makes character-specific behavior easier to verify.
