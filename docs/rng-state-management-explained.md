# RNG State Management: How PushState/PopState Makes Multiplayer Deterministic

## Overview

The multiplayer mod uses RimWorld's built-in `Rand` class state management to ensure deterministic random number generation across all clients. This document explains exactly how `Rand.PushState()` and `Rand.PopState()` work and why they're crucial for multiplayer synchronization.

## How Rand Works (The Basics)

### Rand State Components
```csharp
public static class Rand
{
    private static uint seed;           // Current seed value
    private static uint iterations;      // How many numbers generated since last seed
    private static readonly Stack<ulong> stateStack;  // Stack for PushState/PopState
}
```

### State Compression
```csharp
private static ulong StateCompressed
{
    get
    {
        return seed | ((ulong)iterations << 32);  // Combine seed + iterations
    }
    set
    {
        seed = (uint)(value & 0xFFFFFFFFu);       // Extract seed
        iterations = (uint)((value >> 32) & 0xFFFFFFFFu);  // Extract iterations
    }
}
```

### Random Number Generation
```csharp
public static float Value => (float)(((double)MurmurHash.GetInt(seed, iterations++) - -2147483648.0) / 4294967295.0);
public static int Int => MurmurHash.GetInt(seed, iterations++);
```

**Key Point**: Every time you call `Rand.Value` or `Rand.Int`, the `iterations` counter increments. This means the RNG state changes with each call.

## PushState/PopState Implementation

### PushState()
```csharp
public static void PushState()
{
    stateStack.Push(StateCompressed);  // Save current state to stack
}

public static void PushState(int replacementSeed)
{
    PushState();                       // Save current state
    Seed = replacementSeed;            // Set new seed
}
```

### PopState()
```csharp
public static void PopState()
{
    StateCompressed = stateStack.Pop();  // Restore state from stack
}
```

## How This Makes Multiplayer Deterministic

### The Problem Without State Management
Without proper state management, different clients might:
1. Generate different random numbers
2. Have different RNG states after the same operations
3. Desync because their game states diverge

### The Solution: Synchronized State Management

Here's exactly what happens in the multiplayer mod:

```csharp
// 1. BEFORE any random decisions
Rand.PushState();                    // Save current state to stack
Rand.StateCompressed = randState;    // Set synchronized state from saved value

// 2. DURING execution - game makes random decisions
// All these calls use the SAME synchronized state:
Rand.Value;    // Generates number, increments iterations
Rand.Int;      // Generates number, increments iterations  
Rand.Range();  // Uses Rand.Value internally
Rand.Chance(); // Uses Rand.Value internally

// 3. AFTER random decisions
randState = Rand.StateCompressed;    // Capture updated state (seed + iterations)
Rand.PopState();                     // Restore original state from stack
```

## Step-by-Step Example

Let's trace through a complete example:

### Initial State
```csharp
// Map's saved RNG state
randState = 0x12345678_00000000;  // seed=0x12345678, iterations=0

// Current Rand state (could be anything)
Rand.seed = 0x99999999;
Rand.iterations = 0x55555555;
```

### Step 1: PushState()
```csharp
Rand.PushState();
// Stack now contains: [0x99999999_55555555]
```

### Step 2: Set Synchronized State
```csharp
Rand.StateCompressed = 0x12345678_00000000;
// Now: Rand.seed = 0x12345678, Rand.iterations = 0
```

### Step 3: Generate Random Numbers
```csharp
Rand.Value;  // Uses seed=0x12345678, iterations=0, then increments to 1
Rand.Int;    // Uses seed=0x12345678, iterations=1, then increments to 2
Rand.Value;  // Uses seed=0x12345678, iterations=2, then increments to 3
```

### Step 4: Capture Updated State
```csharp
randState = Rand.StateCompressed;
// Now: randState = 0x12345678_00000003 (seed=0x12345678, iterations=3)
```

### Step 5: PopState()
```csharp
Rand.PopState();
// Restores: Rand.seed = 0x99999999, Rand.iterations = 0x55555555
```

## Why This Works for Multiplayer

### 1. **Deterministic State Restoration**
- All clients start with the same `randState` value
- All clients set the same `Rand.StateCompressed` 
- All clients generate the same sequence of random numbers
- All clients end with the same updated `randState`

### 2. **State Isolation**
- Each map has its own `randState`
- World has its own `randState`
- Commands have their own `randState`
- No interference between different contexts

### 3. **State Tracking**
- After each tick/command, the updated state is saved
- States are compared between clients for desync detection
- If states differ, a desync is detected

## Real-World Example

### Combat System
```csharp
// Before combat tick
Rand.PushState();
Rand.StateCompressed = map.randState;  // Set to saved state

// Combat makes random decisions
job.maxNumMeleeAttacks = Rand.RangeInclusive(2, 5);  // Uses synchronized state
job.expiryInterval = Rand.Range(2000, 4000);         // Uses synchronized state

// After combat tick  
map.randState = Rand.StateCompressed;  // Save updated state
Rand.PopState();                      // Restore original state
```

### Result
- All clients generate the same number of attacks
- All clients generate the same job duration
- All clients end with the same `map.randState`
- No desync occurs

## State Compression Details

### What StateCompressed Contains
```csharp
ulong StateCompressed = seed | ((ulong)iterations << 32);
// Example: 0x12345678_00000005
//         seed=0x12345678, iterations=5
```

### Why Use Both Seed and Iterations
- **Seed**: Determines the random number sequence
- **Iterations**: Tracks how far into the sequence we are
- **Combined**: Uniquely identifies the exact state of the RNG

### State Comparison
```csharp
// Client A: randState = 0x12345678_00000005
// Client B: randState = 0x12345678_00000005
// Result: States match, no desync

// Client A: randState = 0x12345678_00000005  
// Client B: randState = 0x12345678_00000006
// Result: States differ, desync detected
```

## Common Misconceptions

### ❌ "PushState creates a new RNG"
- PushState saves the current state, it doesn't create a new RNG
- The same Rand instance is used throughout

### ❌ "PopState returns to a random state"
- PopState returns to the exact state that was saved by PushState
- It's deterministic and predictable

### ❌ "StateCompressed is just the seed"
- StateCompressed contains both seed AND iterations
- Both are needed for complete state restoration

### ❌ "This is just setting a seed"
- Setting a seed only changes the sequence
- StateCompressed also sets the position within that sequence

## Debug Integration

The debug panel shows the current state:
```csharp
// Map RNG state display
string mapRngLow = $"{(uint)async.randState:X8}";      // Lower 32 bits (seed)
string mapRngHigh = $"{(uint)(async.randState >> 32):X8}"; // Upper 32 bits (iterations)
```

## Best Practices

1. **Always use PushState/PopState pairs**
2. **Set StateCompressed before any Rand calls**
3. **Capture StateCompressed after all Rand calls**
4. **Don't mix different RNG contexts**
5. **Test state synchronization thoroughly**

## Summary

The `Rand.PushState()` and `Rand.PopState()` system works by:

1. **Saving** the current RNG state to a stack
2. **Setting** the RNG to a synchronized state
3. **Generating** random numbers (which advance the state)
4. **Capturing** the updated state for synchronization
5. **Restoring** the original state from the stack

This ensures that all clients generate identical random numbers while maintaining the ability to use different RNG states for different contexts (map vs world vs commands). The system is deterministic, predictable, and prevents desyncs in multiplayer games. 