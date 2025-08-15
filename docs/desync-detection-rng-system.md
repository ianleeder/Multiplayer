# Desync Detection and RNG State Management System

## Overview

The multiplayer mod implements a sophisticated system to detect and prevent desynchronization between clients by tracking and comparing RNG (Random Number Generator) states and execution traces. This ensures that all clients maintain identical game states during multiplayer sessions.

## Core Components

### 1. SyncCoordinator
**Location:** `Source/Client/Desyncs/SyncCoordinator.cs`

The central orchestrator for desync detection:

- **AddClientOpinionAndCheckDesync()**: Main entry point for desync detection
- **HandleDesync()**: Processes detected desyncs
- **TryAddMapRandomState()**: Tracks map-specific RNG states
- **TryAddWorldRandomState()**: Tracks world-level RNG states
- **TryAddCommandRandomState()**: Tracks command execution RNG states

### 2. ClientSyncOpinion
**Location:** `Source/Client/Desyncs/ClientSyncOpinion.cs`

Represents a snapshot of game state for comparison:

```csharp
public class ClientSyncOpinion
{
    public List<uint> commandRandomStates = new();
    public List<uint> worldRandomStates = new();
    public List<MapRandomStateData> mapStates = new();
    public List<int> desyncStackTraceHashes = new();
    public bool simulating;
    public RoundModeEnum roundMode;
}
```

### 3. AsyncTimeComp (Map RNG Management)
**Location:** `Source/Client/AsyncTime/AsyncTimeComp.cs`

Manages per-map RNG state:

```csharp
public class AsyncTimeComp
{
    public ulong randState = 1;  // Map-specific RNG state
    
    public void PreContext()
    {
        Rand.PushState();
        Rand.StateCompressed = randState;  // Set RNG state before ticking
    }
    
    public void PostContext()
    {
        randState = Rand.StateCompressed;  // Update RNG state after ticking
        Rand.PopState();
    }
}
```

### 4. AsyncWorldTimeComp (World RNG Management)
**Location:** `Source/Client/AsyncTime/AsyncWorldTimeComp.cs`

Manages world-level RNG state:

```csharp
public class AsyncWorldTimeComp
{
    public ulong randState = 2;  // World-specific RNG state
    
    public void PreContext()
    {
        Rand.PushState();
        Rand.StateCompressed = randState;  // Set RNG state before ticking
    }
    
    public void PostContext()
    {
        randState = Rand.StateCompressed;  // Update RNG state after ticking
        Rand.PopState();
    }
}
```

## How Desync Detection Works

### 1. Opinion Collection
Each client periodically creates a `ClientSyncOpinion` containing:
- Map RNG states (high 32 bits of ulong)
- World RNG states (high 32 bits of ulong)
- Command RNG states (high 32 bits of ulong)
- Stack trace hashes for debugging
- Floating point round mode

### 2. Opinion Comparison
When a new opinion is received, it's compared to the most recent opinion from another client:

```csharp
public string CheckForDesync(ClientSyncOpinion other)
{
    if (roundMode != other.roundMode)
        return $"FP round mode doesn't match: {roundMode} != {other.roundMode}";
    
    if (!mapStates.Select(m => m.mapId).SequenceEqual(other.mapStates.Select(m => m.mapId)))
        return "Map instances don't match";
    
    foreach (var g in mapStates.Join(other.mapStates, m => m.mapId, m => m.mapId, (map1, map2) => (map1, map2)))
    {
        if (!g.map1.randomStates.SequenceEqual(g.map2.randomStates))
            return $"Wrong random state on map {g.map1.mapId}";
    }
    
    if (!worldRandomStates.SequenceEqual(other.worldRandomStates))
        return "Wrong random state for the world";
    
    if (!commandRandomStates.SequenceEqual(other.commandRandomStates))
        return "Random state from commands doesn't match";
    
    if (!simulating && !other.simulating && desyncStackTraceHashes.Any() && 
        other.desyncStackTraceHashes.Any() && 
        !desyncStackTraceHashes.SequenceEqual(other.desyncStackTraceHashes))
        return "Trace hashes don't match";
    
    return null; // No desync detected
}
```

### 3. Desync Handling
When a desync is detected:
1. Session is marked as desynced
2. Desync window is shown with details
3. Desync information is saved for debugging
4. Client sends desync notification to server

## RNG State Management

### State Flow
1. **Before Tick/Command**: `Rand.StateCompressed = randState`
2. **During Execution**: Game uses `Rand` for random decisions
3. **After Tick/Command**: `randState = Rand.StateCompressed`
4. **Tracking**: State is added to current `ClientSyncOpinion`

### State Types
- **Map States**: Each map has independent RNG state
- **World States**: Global world-level RNG state
- **Command States**: RNG state during command execution

## Stack Trace Logging

### DeferredStackTracing
**Location:** `Source/Client/Desyncs/DeferredStackTracing.cs`

Tracks execution paths for debugging:

- Hooks into `Rand.Value` and `Rand.Int` calls
- Collects stack traces at critical points
- Hashes stack traces for comparison
- Ignores certain non-deterministic systems (wild animals, plants, etc.)

### Key Hooks
```csharp
[HarmonyPatch(typeof(Rand), nameof(Rand.Value))]
[HarmonyPatch(typeof(Rand), nameof(Rand.Int))]
[HarmonyPatch(typeof(Thing), nameof(Thing.SpawnSetup))]
[HarmonyPatch(typeof(Thing), nameof(Thing.DeSpawn))]
[HarmonyPatch(typeof(Pawn_JobTracker), nameof(Pawn_JobTracker.EndCurrentJob))]
```

## Debug UI Integration

### SyncDebugPanel
**Location:** `Source/Client/UI/SyncDebugPanel.cs`

Provides real-time visibility into:
- Current RNG states (map and world)
- Sync status indicators
- Performance metrics
- Network information
- Detailed debug data

## Prevention Strategies

### 1. Deterministic Execution
- All random decisions use the synchronized `Rand` state
- RNG state is set before and captured after each tick/command
- Floating point round mode is tracked and compared

### 2. State Isolation
- Map and world RNG states are independent
- Each map maintains its own RNG state
- Commands have their own RNG state tracking

### 3. Trace Ignoring
- Certain systems are excluded from stack trace logging
- Wild animal/plant spawning
- Environmental effects
- Steam sprayers
- These systems are non-deterministic by design

## Common Desync Sources

1. **Mod Incompatibility**: Mods that modify game logic without proper synchronization
2. **Timing Issues**: Different execution speeds between clients
3. **State Corruption**: Incorrect RNG state management
4. **Floating Point Differences**: Different FPU configurations
5. **Non-Deterministic Code**: Code that doesn't use the synchronized RNG

## Debugging Desyncs

### 1. Check RNG States
- Compare map and world RNG states between clients
- Look for divergence points in RNG sequences

### 2. Analyze Stack Traces
- Stack trace hashes show where execution diverged
- Focus on the first point where hashes differ

### 3. Review Desync Logs
- Desync files contain detailed information
- Include game logs and mod lists
- Show exact divergence points

### 4. Use Debug Panel
- Real-time monitoring of RNG states
- Performance and network metrics
- Sync status indicators

## Integration Points

### Tick System
- RNG state is managed during map and world ticks
- Each tick updates the appropriate RNG state
- States are tracked in the current opinion

### Command System
- Commands execute with their own RNG state
- Command RNG states are tracked separately
- Ensures deterministic command execution

### Save/Load System
- RNG states are serialized with save data
- Ensures consistency across save/load cycles
- Prevents state corruption during saves

## Best Practices

1. **Always use Rand**: Never use System.Random or other RNG sources
2. **Test thoroughly**: Test multiplayer scenarios extensively
3. **Monitor states**: Use debug panel to monitor RNG states
4. **Handle errors**: Implement proper error handling for RNG operations
5. **Document changes**: Document any RNG-related code changes

## Future Improvements

1. **Enhanced Tracing**: More granular stack trace collection
2. **Predictive Detection**: Detect potential desyncs before they occur
3. **Automatic Recovery**: Attempt to recover from minor desyncs
4. **Better UI**: More intuitive debug interface
5. **Performance Optimization**: Reduce overhead of state tracking 