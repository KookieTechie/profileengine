# 🚀 ProfileEngine v3.0

### Next-Generation Data Persistence for Roblox

> *The spiritual successor to ProfileService, rebuilt from scratch with modern Luau, superior architecture, and features that go far beyond any existing DataStore wrapper.*

---

## ⚡ Why ProfileEngine?

You've probably used **ProfileService** by loleris. It was great, but it's no longer maintained, and Roblox has evolved. **ProfileEngine** takes everything ProfileService did right and pushes it further:

| Problem | ProfileService | **ProfileEngine** |
|---|---|---|
| Session lock stuck for 30+ min | Wait or restart | **MessagingService steal, instant release** |
| No real-time cross-server messages | GlobalUpdates only (polled) | **MessageAsync + GlobalUpdates** |
| No save lifecycle hooks | ❌ | **Full middleware pipeline** |
| No schema versioning | ❌ | **Numbered migration chain** |
| No operational visibility | ❌ | **Built-in analytics & telemetry** |
| Basic error handling | Retry loop | **Exponential backoff + jitter + critical state detection** |
| Studio testing awkward | Mock store | **Full mock with version history** |
| Data integrity checks | ❌ | **Corruption detection signal** |
| Know if data changed | ❌ | **HasUnsavedChanges() with deep equality** |

---

## 📦 Installation

1. Download `ProfileEngine.lua`
2. Place it as a **ModuleScript** inside `ServerStorage`
3. Require it from any server script:

```lua
local ProfileEngine = require(game.ServerStorage.ProfileEngine)
```

> **ProfileEngine is server-side only.** Never place it in ReplicatedStorage.

---

## 🏁 Quick Start (5 Minutes)

```lua
local Players = game:GetService("Players")
local ProfileEngine = require(game.ServerStorage.ProfileEngine)

-- 1. Create a store with your data template
local PlayerStore = ProfileEngine.GetProfileStore("PlayerData", {
    Coins = 0,
    Gems = 0,
    Level = 1,
    Inventory = {},
})

-- 2. Load profiles when players join
Players.PlayerAdded:Connect(function(player)
    local profile = PlayerStore:LoadProfileAsync(
        "Player_" .. player.UserId,
        "ForceLoad"
    )

    if not profile then
        player:Kick("Failed to load data")
        return
    end

    -- GDPR compliance
    profile:AddUserId(player.UserId)

    -- Fill in any new fields added to the template
    profile:Reconcile()

    -- Protect against session steal
    profile:ListenToRelease(function()
        player:Kick("Session ended")
    end)

    -- Use the data!
    profile.Data.Coins += 100
end)

-- 3. That's it! Auto-save and PlayerRemoving are handled automatically.
```

---

## 📖 Full API Reference

---

### 🔷 ProfileEngine (Top-Level Module)

The entry point. You get this when you `require` the module.

---

#### `ProfileEngine.GetProfileStore(storeName, template, scope?)`

Creates or retrieves a ProfileStore.

| Parameter | Type | Description |
|---|---|---|
| `storeName` | `string` | Name of the Roblox DataStore |
| `template` | `table` | Default data for new profiles |
| `scope` | `string?` | Optional DataStore scope |

**Returns:** `ProfileStore`

```lua
local Store = ProfileEngine.GetProfileStore("PlayerData", {
    Coins = 0,
    Gems = 0,
    Level = 1,
    Inventory = {},
    Settings = {
        MusicVolume = 0.5,
        SFXVolume = 0.8,
    },
})
```

> **Note:** Calling this twice with the same name returns the same cached store instance.

---

#### `ProfileEngine.GetAnalytics()`

Returns a detailed report of all DataStore operations.

**Returns:** `table`

```lua
local report = ProfileEngine.GetAnalytics()
--[[
{
    Uptime = 3600,
    TotalOperations = 150,
    TotalErrors = 2,
    Operations = { Load = 50, Save = 98, Wipe = 2 },
    Errors = { Save = 2 },
    Latencies = {
        Load = { Average = 0.12, Min = 0.05, Max = 0.45, Count = 50 },
        Save = { Average = 0.08, Min = 0.03, Max = 0.30, Count = 98 },
    },
}
--]]
```

---

#### `ProfileEngine.IsInCriticalState()`

Returns `true` if any store has hit 5+ consecutive DataStore errors.

```lua
if ProfileEngine.IsInCriticalState() then
    warn("DataStore is having issues!")
end
```

---

#### `ProfileEngine.ServiceLocked`

`boolean` — becomes `true` once `BindToClose` fires (server is shutting down). Use this to prevent new profile loads during shutdown.

```lua
if ProfileEngine.ServiceLocked then
    return -- Don't load new profiles, server is closing
end
```

---

#### `ProfileEngine.Version`

`string` — current version (`"3.0.0"`).

---

### 📡 ProfileEngine Signals

| Signal | Fires When | Callback Args |
|---|---|---|
| `ProfileEngine.IssueSignal` | Any DataStore operation fails | `(errorMessage: string)` |
| `ProfileEngine.CorruptionSignal` | Loaded data fails integrity check | `(key: string, reason: string)` |
| `ProfileEngine.CriticalStateSignal` | Entering or leaving critical state | `(isCritical: boolean)` |

```lua
ProfileEngine.IssueSignal:Connect(function(message)
    warn("DataStore issue: " .. message)
end)

ProfileEngine.CriticalStateSignal:Connect(function(isCritical)
    if isCritical then
        warn("⚠️ CRITICAL: DataStore is failing repeatedly!")
    else
        print("✅ DataStore recovered")
    end
end)
```

---

---

### 🔶 ProfileStore

Manages a collection of profiles backed by a single Roblox DataStore.

---

#### `ProfileStore:LoadProfileAsync(key, notReleasedHandler)`

Loads a profile with session locking. **This is the most important method.**

| Parameter | Type | Description |
|---|---|---|
| `key` | `string` | DataStore key (e.g. `"Player_12345"`) |
| `notReleasedHandler` | `string` or `function` | What to do if another server owns the session |

**`notReleasedHandler` options:**

| Value | Behavior |
|---|---|
| `"ForceLoad"` | Wait up to 30s, then steal the session *(default)* |
| `"Steal"` | Immediately notify the remote server via MessagingService and take over |
| `"Cancel"` | Return `nil` immediately if the profile is locked |
| `function(placeId, jobId)` | Custom handler — return `"ForceLoad"`, `"Steal"`, or `"Cancel"` |

**Returns:** `Profile` or `nil`

```lua
-- Basic usage
local profile = Store:LoadProfileAsync("Player_" .. player.UserId, "ForceLoad")

-- Custom handler
local profile = Store:LoadProfileAsync("Player_" .. player.UserId, function(placeId, jobId)
    warn("Profile locked by server:", jobId)
    return "Steal" -- Take it over
end)
```

---

#### `ProfileStore:ViewProfileAsync(key)`

Reads a profile **without** claiming the session lock. The returned profile is read-only (calling `Save()` does nothing).

Perfect for admin panels, leaderboards, or inspecting offline player data.

```lua
local profile = Store:ViewProfileAsync("Player_67890")
if profile then
    print("Their coins:", profile.Data.Coins)
end
```

---

#### `ProfileStore:GlobalUpdateProfileAsync(key, updateHandler)`

Sends a persistent update to a profile, even if the player is **offline** or on **another server.** The update is stored in the DataStore and processed when the profile is next loaded.

```lua
-- Send a gift to an offline player
Store:GlobalUpdateProfileAsync("Player_67890", function(globalUpdates)
    globalUpdates:AddActiveUpdate({
        Type = "Gift",
        Item = "DiamondSword",
        Amount = 1,
        From = "Admin",
    })
end)
```

---

#### `ProfileStore:MessageAsync(key, messageData)`

Sends a **real-time** message to a profile via `MessagingService`. Unlike GlobalUpdates, this is instant but **not persisted**,  if the player is offline, the message is lost.

```lua
Store:MessageAsync("Player_12345", {
    Type = "Notification",
    Text = "Your guild won the war!",
})
```

---

#### `ProfileStore:ProfileVersionQuery(key, sortDirection?, minDate?, maxDate?, pageSize?)`

Queries the DataStore version history for a profile key. Works with the mock store too.

```lua
local pages = Store:ProfileVersionQuery("Player_12345", Enum.SortDirection.Descending)
if pages then
    local versions = pages:GetCurrentPage()
    for _, v in versions do
        print("Version:", v.Version, "Created:", v.CreatedTime)
    end
end
```

---

#### `ProfileStore:WipeProfileAsync(key)`

Permanently deletes a profile from the DataStore. **Irreversible.**

```lua
local success = Store:WipeProfileAsync("Player_12345")
```

---

#### `ProfileStore:RegisterMigration(fromVersion, migrationFunc)`

Registers a data migration. When a profile with an older schema version is loaded, migrations run automatically in order.

```lua
-- Migrate from v1 → v2: rename "Gold" to "Coins"
Store:RegisterMigration(1, function(data)
    data.Coins = data.Gold or 0
    data.Gold = nil
    return data
end)

-- Migrate from v2 → v3: add Settings table
Store:RegisterMigration(2, function(data)
    data.Settings = { MusicVolume = 0.5, SFXVolume = 0.8 }
    return data
end)

-- Schema is now at version 3 automatically
```

---

#### `ProfileStore:GetProfile(key)`

Returns the loaded `Profile` for a key, or `nil` if not loaded.

#### `ProfileStore:GetLoadedProfiles()`

Returns a `{ [key] = Profile }` dictionary of all active profiles.

#### `ProfileStore:IsLoaded(key)`

Returns `true` if the key has an active profile.

#### `ProfileStore:SaveAll()`

Saves all dirty profiles immediately. Returns `(saved, failed)`.

#### `ProfileStore:ReleaseAll()`

Releases all loaded profiles (saves + removes session locks).

#### `ProfileStore:Destroy()`

Releases all profiles, stops auto-save, and cleans up signals.

---

#### `ProfileStore.Mock`

A version of the store that always uses the in-memory mock DataStore, even in a live game. Perfect for testing.

```lua
-- Use mock for tests
local MockStore = Store.Mock
local profile = MockStore:LoadProfileAsync("TestPlayer", "ForceLoad")
```

---

#### ProfileStore Signals

| Signal | Fires When | Callback Args |
|---|---|---|
| `ProfileStore.ProfileAdded` | A profile is loaded | `(profile: Profile)` |
| `ProfileStore.ProfileRemoved` | A profile is released | `(profile: Profile)` |
| `ProfileStore.CriticalState` | Critical state changes | `(isCritical: boolean)` |
| `ProfileStore.IssueSignal` | Any DS failure | `(message: string)` |
| `ProfileStore.CorruptionSignal` | Data corruption detected | `(key, reason)` |

---

---

### 🟢 Profile

The core data container. You get one from `LoadProfileAsync` or `ViewProfileAsync`.

---

#### Properties

| Property | Type | Description |
|---|---|---|
| `Profile.Data` | `table` | **Your data.** Read and write freely. |
| `Profile.MetaData` | `table` | Internal metadata (create time, session info, schema version, MetaTags) |
| `Profile.GlobalUpdates` | `GlobalUpdates` | Cross-server update handler |
| `Profile.UserIds` | `{number}` | GDPR-associated UserIds |
| `Profile.KeyInfo` | `DataStoreKeyInfo?` | Info from last save |
| `Profile.LastSavedData` | `table?` | Snapshot from last successful save |
| `Profile.Key` | `string` | The DataStore key |

---

#### State & Identity

```lua
profile:IsActive()    --> true if session-owned by this server
profile:Identify()    --> "[ProfileEngine] \"Player_123\" (Store: \"PlayerData\")"
```

---

#### MetaTags

Custom metadata that persists alongside the profile.

```lua
profile:SetMetaTag("VIP", true)
profile:SetMetaTag("JoinedEvent", "Summer2024")

local isVIP = profile:GetMetaTag("VIP") --> true
```

`Profile.MetaTagsUpdated` signal fires whenever a MetaTag changes.

---

#### Reconcile

Fills in any keys from the template that are missing in the loaded data. Call this after loading to handle schema additions without a full migration.

```lua
profile:Reconcile()
```

---

#### GDPR UserId Association

```lua
profile:AddUserId(player.UserId)    -- Associate
profile:RemoveUserId(player.UserId) -- Disassociate
```

These UserIds are passed to `DataStore:SetAsync()` for Roblox's GDPR right-to-erasure compliance.

---

#### Release & Teleportation

```lua
-- Listen for when the session ends (e.g., stolen by another server)
profile:ListenToRelease(function(hopTarget)
    if hopTarget then
        -- Profile is being teleported
        print("Heading to:", hopTarget.PlaceId, hopTarget.JobId)
    end
    player:Kick("Session ended")
end)

-- Release the profile (saves + removes lock)
profile:Release()

-- Release with teleportation hop
profile:Release(targetPlaceId, targetJobId)

-- Listen for hop-ready (for the receiving server)
profile:ListenToHopReady(function(placeId, jobId)
    print("Profile arrived from:", placeId, jobId)
end)
```

---

#### Manual Save

```lua
profile:Save() -- Queues an immediate save
```

> Auto-save runs every 120 seconds automatically. You only need `Save()` for critical moments.

---

#### Overwrite (Dangerous)

```lua
profile:OverwriteAsync() -- Bypasses session lock, use with extreme caution
```

---

#### Data Helpers

```lua
-- Deep-copy snapshot of current data
local snapshot = profile:GetSnapshot()

-- What was last saved to the DataStore?
local lastSaved = profile:GetLastSavedData()

-- Has anything changed since last save?
if profile:HasUnsavedChanges() then
    print("Unsaved changes detected!")
end
```

---

#### Middleware Pipeline

Hook into the save lifecycle:

```lua
-- Run code before every save
profile:AddMiddleware("BeforeSave", function(self)
    self.Data.LastSeen = os.time()
end)

-- Run code after every successful save
profile:AddMiddleware("AfterSave", function(self)
    print("Saved!", self.Key)
end)

-- Run code before release
profile:AddMiddleware("BeforeRelease", function(self)
    print("Goodbye!", self.Key)
end)
```

**Available phases:** `"BeforeSave"`, `"AfterSave"`, `"BeforeRelease"`, `"OnDataChanged"`

---

---

### 🌐 GlobalUpdates

The cross-server messaging system persisted in the DataStore. Updates survive server restarts and offline periods.

---

#### Sending Updates (from any server)

```lua
Store:GlobalUpdateProfileAsync("Player_12345", function(globalUpdates)
    -- Add a new update
    local id = globalUpdates:AddActiveUpdate({
        Type = "Gift",
        Item = "RocketLauncher",
        Quantity = 1,
    })

    -- Modify an existing active update
    globalUpdates:ChangeActiveUpdate(id, {
        Type = "Gift",
        Item = "RocketLauncher",
        Quantity = 3, -- Changed our mind, give 3
    })

    -- Or cancel it entirely
    globalUpdates:ClearActiveUpdate(id)
end)
```

---

#### Receiving Updates (on the owning server)

```lua
-- Listen for new updates arriving
profile.GlobalUpdates:ListenToNewActiveUpdate(function(updateId, data)
    if data.Type == "Gift" then
        -- Process the gift
        table.insert(profile.Data.Inventory, {
            Item = data.Item,
            Quantity = data.Quantity,
        })

        -- Mark as processed
        profile.GlobalUpdates:LockActiveUpdate(updateId)
    end
end)

-- Clean up processed updates
profile.GlobalUpdates:ListenToNewLockedUpdate(function(updateId, data)
    profile.GlobalUpdates:ClearLockedUpdate(updateId)
end)
```

---

#### Reading Updates

```lua
local active = profile.GlobalUpdates:GetActiveUpdates()   -- { {id, data}, ... }
local locked = profile.GlobalUpdates:GetLockedUpdates()   -- { {id, data}, ... }
```

---

---

## 🏗️ Architecture

```
ProfileEngine (singleton)
│
├── GetProfileStore("PlayerData", template)
│   │
│   ├── ProfileStore
│   │   ├── LoadProfileAsync(key) ──► Profile
│   │   │                              ├── .Data (your data)
│   │   │                              ├── .MetaData
│   │   │                              ├── .GlobalUpdates
│   │   │                              ├── .UserIds
│   │   │                              ├── Middleware[]
│   │   │                              └── Release listeners[]
│   │   │
│   │   ├── ViewProfileAsync(key) ──► Profile (read-only)
│   │   ├── GlobalUpdateProfileAsync(key, handler)
│   │   ├── MessageAsync(key, data)
│   │   ├── WipeProfileAsync(key)
│   │   ├── RegisterMigration(v, fn)
│   │   ├── Mock (in-memory testing)
│   │   └── Auto-save loop (120s)
│   │
│   └── Analytics Engine
│       ├── Per-operation latency tracking
│       ├── Error counters
│       └── Critical state detection
│
├── BindToClose (graceful shutdown)
├── PlayerRemoving (auto-release)
├── IssueSignal / CorruptionSignal / CriticalStateSignal
└── Session steal via MessagingService
```

---

## 🛡️ Session Locking — How It Works

```
Server A loads "Player_123"
    └── UpdateAsync: sets ActiveSession = {placeId_A, jobId_A}

Player joins Server B while Server A still has the lock:
    └── Server B calls LoadProfileAsync("Player_123", "ForceLoad")
        ├── Detects ActiveSession belongs to Server A
        ├── Publishes MessagingService steal request
        ├── Server A receives request
        │   ├── Releases the profile (saves + clears lock)
        │   ├── Fires ListenToRelease callbacks
        │   └── Publishes ACK back
        └── Server B receives ACK
            └── Claims the session via UpdateAsync
```

If Server A doesn't respond within 30 seconds, Server B force-claims the lock anyway. If the lock is older than 30 minutes, it's considered dead and claimed immediately.

---

## 🔄 Data Migrations

Handle schema changes gracefully:

```lua
local Store = ProfileEngine.GetProfileStore("PlayerData", {
    -- Current template (v3)
    Coins = 0,
    Gems = 0,
    Level = 1,
    Settings = { Music = 0.5, SFX = 0.8 },
    Achievements = {},
})

-- v1 → v2: Rename "Gold" to "Coins"
Store:RegisterMigration(1, function(data)
    data.Coins = data.Gold or 0
    data.Gold = nil
    return data
end)

-- v2 → v3: Add Achievements table
Store:RegisterMigration(2, function(data)
    data.Achievements = {}
    return data
end)
```

Migrations run **automatically** during `LoadProfileAsync` when a profile's schema version is behind. They execute in order: v1→v2, then v2→v3, etc.

---

## 🧪 Studio Testing

ProfileEngine automatically uses a **mock in-memory DataStore** when running in Studio (non-Run mode). This means:

- ✅ No API access errors in Studio
- ✅ Full functionality including version history
- ✅ Data resets each session (clean testing)

For explicit mock testing in live games:

```lua
local MockStore = Store.Mock
local profile = MockStore:LoadProfileAsync("TestKey", "ForceLoad")
-- Uses in-memory store, never touches real DataStore
```

---

## ⚙️ Configuration

ProfileEngine uses sensible defaults. All constants are at the top of the module:

| Constant | Default | Description |
|---|---|---|
| `AUTO_SAVE_INTERVAL` | `120` | Seconds between auto-saves |
| `SESSION_STEAL_TIMEOUT` | `30` | Seconds to wait for remote release |
| `LOCK_EXPIRY_SECONDS` | `1800` | Stale lock threshold (30 min) |
| `MAX_RETRY_ATTEMPTS` | `5` | Retries per DataStore operation |
| `RETRY_BASE_DELAY` | `1` | Base delay for exponential backoff |
| `RETRY_MAX_DELAY` | `15` | Maximum retry delay cap |
| `BUDGET_BUFFER` | `2` | Reserved DataStore budget units |
| `CRITICAL_STATE_THRESHOLD` | `5` | Consecutive errors for critical state |
| `BIND_TO_CLOSE_TIMEOUT` | `30` | Max shutdown wait time |
| `MAX_GLOBAL_UPDATES` | `100` | Max pending updates per profile |

---

## 📋 Complete Example, Full Game Server

```lua
local Players = game:GetService("Players")
local ProfileEngine = require(game.ServerStorage.ProfileEngine)

-- ── Store Setup ──
local PlayerStore = ProfileEngine.GetProfileStore("GameData_v1", {
    Coins = 0,
    Gems = 0,
    Level = 1,
    XP = 0,
    Kills = 0,
    Deaths = 0,
    Wins = 0,
    Inventory = {},
    Settings = {
        MusicVolume = 0.5,
        SFXVolume = 0.8,
    },
})

local Profiles = {} -- [Player] = Profile

-- ── Monitor Issues ──
ProfileEngine.IssueSignal:Connect(function(msg)
    warn("[DataStore Issue]", msg)
end)

ProfileEngine.CriticalStateSignal:Connect(function(isCritical)
    if isCritical then
        warn("🔴 CRITICAL STATE — DataStore is failing!")
    else
        print("🟢 DataStore recovered")
    end
end)

-- ── Player Join ──
local function onPlayerAdded(player)
    if ProfileEngine.ServiceLocked then return end

    local profile = PlayerStore:LoadProfileAsync(
        "Player_" .. player.UserId,
        "ForceLoad"
    )

    if not player.Parent then
        if profile then profile:Release() end
        return
    end

    if not profile then
        player:Kick("Unable to load data. Please rejoin.")
        return
    end

    profile:AddUserId(player.UserId)
    profile:Reconcile()

    profile:ListenToRelease(function()
        Profiles[player] = nil
        player:Kick("Your session has ended.")
    end)

    -- Handle gifts from other servers
    profile.GlobalUpdates:ListenToNewActiveUpdate(function(id, data)
        if data.Type == "GiftCoins" then
            profile.Data.Coins += (data.Amount or 0)
        elseif data.Type == "GiftItem" then
            table.insert(profile.Data.Inventory, data.Item)
        end
        profile.GlobalUpdates:LockActiveUpdate(id)
    end)

    profile.GlobalUpdates:ListenToNewLockedUpdate(function(id)
        profile.GlobalUpdates:ClearLockedUpdate(id)
    end)

    -- Middleware: stamp last seen time on every save
    profile:AddMiddleware("BeforeSave", function(p)
        p:SetMetaTag("LastSeen", os.time())
    end)

    -- Create leaderstats
    local leaderstats = Instance.new("Folder")
    leaderstats.Name = "leaderstats"

    local coins = Instance.new("IntValue")
    coins.Name = "Coins"
    coins.Value = profile.Data.Coins
    coins.Parent = leaderstats

    local level = Instance.new("IntValue")
    level.Name = "Level"
    level.Value = profile.Data.Level
    level.Parent = leaderstats

    leaderstats.Parent = player

    Profiles[player] = profile
    print(player.Name .. " loaded, Level " .. profile.Data.Level)
end

-- ── Player Leave ──
local function onPlayerRemoving(player)
    local profile = Profiles[player]
    if profile then
        Profiles[player] = nil
        profile:Release()
    end
end

-- ── Connect ──
Players.PlayerAdded:Connect(onPlayerAdded)
Players.PlayerRemoving:Connect(onPlayerRemoving)

for _, player in Players:GetPlayers() do
    task.spawn(onPlayerAdded, player)
end

print("✅ Game server initialized, ProfileEngine v" .. ProfileEngine.Version)
```

---

## 🔀 Migrating from ProfileService

ProfileEngine uses the same DataStore format as ProfileService. Your existing data **just works.**

| ProfileService | ProfileEngine |
|---|---|
| `ProfileService.GetProfileStore(name, template)` | `ProfileEngine.GetProfileStore(name, template)` |
| `ProfileStore:LoadProfileAsync(key, handler)` | Same ✅ |
| `ProfileStore:ViewProfileAsync(key)` | Same ✅ |
| `ProfileStore:GlobalUpdateProfileAsync(key, fn)` | Same ✅ |
| `ProfileStore:WipeProfileAsync(key)` | Same ✅ |
| `ProfileStore.Mock` | Same ✅ |
| `Profile.Data` | Same ✅ |
| `Profile.MetaData` | Same ✅ |
| `Profile:IsActive()` | Same ✅ |
| `Profile:Release()` | Same ✅ |
| `Profile:ListenToRelease(fn)` | Same ✅ |
| `Profile:Reconcile()` | Same ✅ |
| `Profile:AddUserId(id)` | Same ✅ |
| `Profile:SetMetaTag(name, value)` | Same ✅ |
| `Profile:GetMetaTag(name)` | Same ✅ |
| `Profile:Save()` | Same ✅ |
| N/A | `ProfileStore:MessageAsync()` 🆕 |
| N/A | `ProfileStore:RegisterMigration()` 🆕 |
| N/A | `Profile:AddMiddleware()` 🆕 |
| N/A | `Profile:HasUnsavedChanges()` 🆕 |
| N/A | `Profile:GetSnapshot()` 🆕 |
| N/A | `ProfileEngine.GetAnalytics()` 🆕 |
| N/A | `ProfileEngine.CriticalStateSignal` 🆕 |

**Migration steps:**
1. Replace `require(ProfileService)` with `require(ProfileEngine)`
2. That's it. The API is compatible.

---

## ❓ FAQ

**Q: Is this compatible with ProfileService data?**
A: Yes. ProfileEngine reads and writes the same `{ Data, MetaData, GlobalUpdates }` format. Existing profiles load without any changes.

**Q: What happens if the DataStore API is disabled in Studio?**
A: ProfileEngine auto-detects Studio and uses an in-memory mock DataStore. Everything works exactly the same, your data just won't persist between sessions.

**Q: How often does auto-save run?**
A: Every 120 seconds by default. Adjust `AUTO_SAVE_INTERVAL` at the top of the module. Only dirty (changed) profiles are saved.

**Q: What's the difference between GlobalUpdates and MessageAsync?**
A: GlobalUpdates are persisted in the DataStore, they survive reboots and work offline. MessageAsync uses MessagingService for instant delivery but is lost if the player isn't online.

**Q: Can I use this for non-player data?**
A: Absolutely. ProfileEngine is entity-agnostic. Use any string key, `"Guild_ABC"`, `"House_123"`, `"World_Seed_42"`.

**Q: What's "critical state"?**
A: If 5+ consecutive DataStore operations fail, ProfileEngine enters critical state and fires `CriticalStateSignal`. It auto-recovers when a call succeeds.

---

## 📄 License

MIT License, free to use in any Roblox project, commercial or otherwise.

---

<div align="center">

**ProfileEngine v3.0**, *Built for games that take data seriously.*

</div>
