# RNG Integration in Random Decisions

## Overview

The multiplayer mod ensures deterministic random number generation by intercepting all `Rand` calls and tracking the RNG state. This document shows how RNG is integrated into various game systems and provides specific examples.

## Core Integration Points

### 1. RNG State Management

**Location:** `Source/Client/AsyncTime/AsyncTimeComp.cs` and `Source/Client/AsyncTime/AsyncWorldTimeComp.cs`

```csharp
// Before any random decisions
Rand.PushState();
Rand.StateCompressed = randState;  // Set synchronized RNG state

// Game makes random decisions using Rand.Value, Rand.Int, etc.
// All these calls are tracked by DeferredStackTracing

// After random decisions
randState = Rand.StateCompressed;  // Capture updated state
Rand.PopState();
```

### 2. Stack Trace Tracking

**Location:** `Source/Client/Desyncs/DeferredStackTracing.cs`

The system hooks into all `Rand.Value` and `Rand.Int` calls:

```csharp
static IEnumerable<MethodBase> TargetMethods()
{
    yield return AccessTools.PropertyGetter(typeof(Rand), nameof(Rand.Value));
    yield return AccessTools.PropertyGetter(typeof(Rand), nameof(Rand.Int));
}

public static void Postfix()
{
    // Collect stack trace and hash it
    var logItem = StackTraceLogItemRaw.GetFromPool();
    var trace = logItem.raw;
    int hash = 0;
    int depth = DeferredStackTracingImpl.TraceImpl(trace, ref hash);
    
    // Add to current sync opinion
    Multiplayer.game.sync.TryAddStackTraceForDesyncLogRaw(logItem, depth, hash);
}
```

## Examples of RNG Integration

### 1. Combat System

**Example:** Melee attack randomization

```csharp
// From JobDriver_AttackMelee.cs
job.maxNumMeleeAttacks = Rand.RangeInclusive(2, 5);  // Random number of attacks
job.expiryInterval = Rand.Range(2000, 4000);          // Random job duration
```

**How it works:**
1. Before combat tick: `Rand.StateCompressed = map.randState`
2. Combat system calls `Rand.RangeInclusive()` and `Rand.Range()`
3. DeferredStackTracing captures stack trace and hashes it
4. After combat tick: `map.randState = Rand.StateCompressed`
5. State is added to current `ClientSyncOpinion`

### 2. Mental State System

**Example:** Mental break chances

```csharp
// From MentalBreaker.cs
return Rand.Chance(levelDef.anomalyMentalBreakChance);  // Random mental break
return Rand.MTBEventOccurs(0.5f, 60000f, 150f);        // Random event timing
```

**Example:** Social fighting outcomes

```csharp
// From MentalState_SocialFighting.cs
ThoughtDef thoughtDef = ((!(Rand.Value < 0.5f)) ? 
    ThoughtDefOf.HadCatharticFight : 
    ThoughtDefOf.HadAngeringFight);  // Random thought outcome
```

### 3. AI Decision Making

**Example:** Pawn behavior randomization

```csharp
// From Pawn_MindState.cs
if (Rand.Chance(PawnUtility.GetManhunterOnDamageChance(this.pawn, dinfo.Instigator)))
    // Random chance to go manhunter when damaged

if (Rand.Chance(0.4f) && CanStartFleeingBecauseOfPawnAction(pawn))
    // Random chance to flee from danger

if (Rand.Chance(pawn.RaceProps.leaveMapOnFleeChance))
    // Random chance to leave map when fleeing
```

**Example:** Job priority randomization

```csharp
// From ThinkNode_PrioritySorter.cs
workingNodes.Insert(Rand.Range(0, workingNodes.Count - 1), subNodes[i]);
// Random insertion position for job priorities
```

### 4. World Events

**Example:** Visitor gift giving

```csharp
// From TransitionAction_CheckGiveGift.cs
if (Rand.Chance(VisitorGiftForPlayerUtility.ChanceToLeaveGift(trans.target.lord.faction, trans.Map)))
    // Random chance for visitors to leave gifts
```

**Example:** Psychic ritual effects

```csharp
// From PsychicRitualToil_VoidProvocation.cs
if (Rand.Chance(psychicRitualDef_VoidProvocation.psychicShockChanceFromQualityCurve.Evaluate(psychicRitual.PowerPercent)))
    // Random psychic shock effect

// From PsychicRitualToil_SummonAnimals.cs
if (Rand.Chance(manhunterChance))
    // Random chance for summoned animals to go manhunter
```

### 5. Environmental Effects

**Example:** Baby crying animation

```csharp
// From MentalState_BabyCry.cs
MoteMaker.MakeAttachedOverlay(pawn, ThingDefOf.Mote_BabyCryingDots, 
    new Vector3(0.27f, 0f, 0.066f).RotatedBy(num)).exactRotation = Rand.Value * 180f;
// Random rotation for crying animation
```

### 6. Damage and Combat

**Example:** Battle log display chances

```csharp
// From BattleLogEntry_RangedImpact.cs
return Rand.ChanceSeeded(DisplayChanceOnMiss / (float)num, logID);
// Random chance to display miss in battle log
```

### 7. Name Generation

**Example:** Random gender assignment

```csharp
// From Rule_NamePerson.cs
gender = ((Rand.Value < 0.5f) ? Gender.Male : Gender.Female);
// Random gender for generated names
```

## Non-Deterministic Systems (Ignored)

Certain systems are intentionally excluded from RNG tracking because they are non-deterministic by design:

```csharp
// Wild animal spawning - excluded from tracking
[HarmonyPatch(typeof(WildAnimalSpawner), nameof(WildAnimalSpawner.WildAnimalSpawnerTick))]
static class WildAnimalSpawnerTickTraceIgnore
{
    static void Prefix() => DeferredStackTracing.ignoreTraces++;
    static void Finalizer() => DeferredStackTracing.ignoreTraces--;
}

// Wild plant spawning - excluded from tracking
[HarmonyPatch(typeof(WildPlantSpawner), nameof(WildPlantSpawner.WildPlantSpawnerTick))]
static class WildPlantSpawnerTickTraceIgnore
{
    static void Prefix() => DeferredStackTracing.ignoreTraces++;
    static void Finalizer() => DeferredStackTracing.ignoreTraces--;
}

// Environmental effects - excluded from tracking
[HarmonyPatch(typeof(SteadyEnvironmentEffects), nameof(SteadyEnvironmentEffects.SteadyEnvironmentEffectsTick))]
static class SteadyEnvironmentEffectsTickTraceIgnore
{
    static void Prefix() => DeferredStackTracing.ignoreTraces++;
    static void Finalizer() => DeferredStackTracing.ignoreTraces--;
}
```

## Critical Points for RNG Integration

### 1. Thing Spawning/Despawning

```csharp
[HarmonyPatch(typeof(Thing), nameof(Thing.SpawnSetup))]
public static class ThingSpawnPatch
{
    static void Postfix(Thing __instance)
    {
        if (__instance.def.HasThingIDNumber)
            DeferredStackTracing.Postfix();  // Track RNG state during spawning
    }
}

[HarmonyPatch(typeof(Thing), nameof(Thing.DeSpawn))]
public static class ThingDeSpawnPatch
{
    static void Postfix(Thing __instance)
    {
        if (__instance.def.HasThingIDNumber)
            DeferredStackTracing.Postfix();  // Track RNG state during despawning
    }
}
```

### 2. Job Completion

```csharp
[HarmonyPatch(typeof(Pawn_JobTracker), nameof(Pawn_JobTracker.EndCurrentJob))]
public static class EndCurrentJobPatch
{
    static void Prefix(Pawn_JobTracker __instance)
    {
        if (MpVersion.IsDebug && __instance.curJob != null && DeferredStackTracing.ShouldAddStackTraceForDesyncLog())
            Multiplayer.game.sync.TryAddInfoForDesyncLog($"EndCurrentJob for {__instance.pawn}: {__instance.curJob}", "");
    }
}
```

### 3. Unique ID Generation

```csharp
[HarmonyPatch(typeof(UniqueIDsManager), nameof(UniqueIDsManager.GetNextID))]
public static class UniqueIdsPatch
{
    static void Postfix()
    {
        DeferredStackTracing.Postfix();  // Track RNG state during ID generation
    }
}
```

## State Isolation Examples

### Map vs World RNG

```csharp
// Map-specific RNG (for map events, pawn behavior, etc.)
var async = Find.CurrentMap.AsyncTime();
string mapRngLow = $"{(uint)async.randState:X8}";
string mapRngHigh = $"{(uint)(async.randState >> 32):X8}";

// World-specific RNG (for world events, global effects, etc.)
var worldAsync = Multiplayer.AsyncWorldTime;
string worldRngLow = $"{(uint)worldAsync.randState:X8}";
string worldRngHigh = $"{(uint)(worldAsync.randState >> 32):X8}";
```

### Command RNG

```csharp
// Command execution has its own RNG state
Multiplayer.game.sync.TryAddCommandRandomState(randState);
```

## Debug Integration

The debug panel shows real-time RNG state information:

```csharp
// From SyncDebugPanel.cs
// Map RNG state
string mapRngLow = $"{(uint)async.randState:X8}";
string mapRngHigh = $"{(uint)(async.randState >> 32):X8}";
y = DrawStatusLine(x, y, width, "Map RNG:", $"{mapRngHigh} | {mapRngLow}", Color.white);

// World RNG state  
string worldRngLow = $"{(uint)worldAsync.randState:X8}";
string worldRngHigh = $"{(uint)(worldAsync.randState >> 32):X8}";
y = DrawStatusLine(x, y, width, "World RNG:", $"{worldRngHigh} | {worldRngLow}", Color.white);
```

## Best Practices for RNG Integration

1. **Always use Rand**: Never use `System.Random` or other RNG sources
2. **Track state changes**: Ensure RNG state is captured after each operation
3. **Isolate contexts**: Map and world RNG should be independent
4. **Ignore non-deterministic systems**: Exclude systems that should be random per client
5. **Test thoroughly**: Verify RNG behavior in multiplayer scenarios

## Common Pitfalls

1. **Using System.Random**: Will cause desyncs
2. **Not capturing state**: Missing state updates can cause divergence
3. **Mixing contexts**: Using world RNG for map events or vice versa
4. **Ignoring stack traces**: Missing critical execution points
5. **Non-deterministic code**: Code that doesn't use synchronized RNG

## Monitoring RNG States

Use the debug panel to monitor:
- Current map and world RNG states
- Stack trace collection status
- Desync detection indicators
- Performance impact of RNG tracking

This system ensures that all random decisions are deterministic across all clients in a multiplayer session, preventing desyncs while maintaining the game's random behavior. 