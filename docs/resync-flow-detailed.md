# Detailed Resync Flow Trace

## **Your Understanding is Correct!**

Let me trace through the actual code to confirm your flow:

---

## **1. Client: Attempt Reconnect**

```csharp
// Rejoiner.cs
public static void DoRejoin()
{
    Multiplayer.Client.Send(Packets.Client_RequestRejoin);  // ← Step 1: Request rejoin
    Multiplayer.Client.ChangeState(ConnectionStateEnum.ClientLoading);
    
    // Clear all maps and world
    LongEventHandler.QueueLongEvent(() =>
    {
        MemoryUtility.ClearAllMapsAndWorld();  // ← Step 2: Clear all states
        Current.Game = null;
    }, "Entry", "LoadingLongEvent", true, null, false);
}
```

---

## **2. Server: Prepares Snapshot**

```csharp
// CommandHandler.cs (Server side)
public void Send(CommandType cmd, int factionId, int mapId, byte[] data, ServerPlayer? sourcePlayer = null, ServerPlayer? fauxSource = null)
{
    // Server creates commands and stores them
    byte[] toSave = ScheduledCommand.Serialize(
        new ScheduledCommand(cmd, server.gameTimer, factionId, mapId, sourcePlayer?.id ?? ScheduledCommand.NoPlayer, data));
    
    // Commands are stored in worldData.mapCmds
    server.worldData.mapCmds.GetOrAddNew(mapId).Add(toSave);
}

// SaveLoad.cs (Server side)
public static GameDataSnapshot CreateGameDataSnapshot(TempGameData data, bool removeCurrentMapId)
{
    // Snapshot contains:
    // 1. Game state (from last save)
    // 2. Commands since last save (from worldData.mapCmds)
    
    var mapCmdsDict = new Dictionary<int, List<ScheduledCommand>>();
    foreach (XmlNode mapNode in mapsNode)
    {
        int id = int.Parse(mapNode["uniqueID"].InnerText);
        mapCmdsDict[id] = new List<ScheduledCommand>(Find.Maps.First(m => m.uniqueID == id).AsyncTime().cmds);
    }
    
    return new GameDataSnapshot(
        TickPatch.Timer,                    // ← Current time when snapshot created
        gameData,                           // ← Game state from save
        data.SessionData,                   // ← Session data
        mapDataDict,                        // ← Map data from save
        mapCmdsDict                         // ← Commands since save
    );
}
```

**So yes, the snapshot is exactly: Last Save + Commands Since Save**

---

## **3. Client: Receives Snapshot**

```csharp
// ClientLoadingState.cs
[PacketHandler(Packets.Server_WorldData)]
public void HandleWorldData(ByteReader data)
{
    // Client receives the snapshot
    Session.dataSnapshot = new GameDataSnapshot(
        0,                                  // ← CachedAtTime (when snapshot was created)
        worldData,                          // ← Game state from save
        sessionData,                        // ← Session data
        mapDataDict,                        // ← Map data from save
        mapCmdsDict                         // ← Commands since save
    );
}
```

---

## **4. Client: Clears All States**

```csharp
// SaveLoad.cs
public static void LoadInMainThread(TempGameData gameData)
{
    ClearState();                          // ← Clear current state
    MemoryUtility.ClearAllMapsAndWorld();  // ← Clear all maps and world
    
    LoadPatch.gameToLoad = gameData;
    Find.Root.Start();
    SavedGameLoaderNow.LoadGameFromSaveFileNow(null);  // ← Load snapshot
}
```

---

## **5. Client: Loads Snapshot**

```csharp
// Loader.cs
public static void ReloadGame(List<int> mapsToLoad, bool changeScene, bool forceAsyncTime)
{
    var gameDoc = DataSnapshotToXml(Multiplayer.session.dataSnapshot, mapsToLoad);
    
    LoadPatch.gameToLoad = new(gameDoc, Multiplayer.session.dataSnapshot.SessionData);
    
    // Load the game state from snapshot
    SaveLoad.LoadInMainThread(LoadPatch.gameToLoad);
}

// AsyncTimeComp.cs
public void FinalizeInit()
{
    // Restore commands from snapshot
    cmds = new Queue<ScheduledCommand>(
        Multiplayer.session.dataSnapshot?.MapCmds.GetValueSafe(map.uniqueID) ?? new List<ScheduledCommand>()
    );
}
```

---

## **6. Client: Simulates Commands**

```csharp
// TickPatch.cs
private static bool RunCmds()
{
    int curTimer = Timer;
    
    foreach (ITickable tickable in AllTickables)
    {
        while (tickable.Cmds.Count > 0 && tickable.Cmds.Peek().ticks == curTimer)
        {
            ScheduledCommand cmd = tickable.Cmds.Dequeue();
            tickable.ExecuteCmd(cmd);  // ← Execute each command
            
            if (LongEventHandler.eventQueue.Count > 0) return true;
        }
    }
    return false;
}

// AsyncTimeComp.cs
public void ExecuteCmd(ScheduledCommand cmd)
{
    // Execute the command
    if (cmdType == CommandType.Sync)
        SyncUtil.HandleCmd(data);
    else if (cmdType == CommandType.MapTimeSpeed)
        SetDesiredTimeSpeed((TimeSpeed)data.ReadByte());
    // ... other command types
    
    // VTR calculations happen here during command execution!
    UpdateManagers();
}
```

---

## **The VTR Problem**

During step 6 (command simulation), **VTR calculations happen for each command**, but:

1. **VTR depends on player state** (which players are on which maps)
2. **Player state might not be fully synchronized** during resync
3. **Different clients calculate different VTR rates** for the same commands
4. **This causes RNG divergence and desync**

---

## **The Solution**

Store the VTR rate **at command creation time** (on the server) and use that stored rate during command simulation (on the client).

```csharp
// When creating commands (Server side)
public void Send(CommandType cmd, int factionId, int mapId, byte[] data, ServerPlayer? sourcePlayer = null, ServerPlayer? fauxSource = null)
{
    // Calculate VTR at command creation time
    int vtrRate = VTRSyncPatch.GetCurrentVTRRate(mapId);
    
    byte[] toSave = ScheduledCommand.Serialize(
        new ScheduledCommand(cmd, server.gameTimer, factionId, mapId, sourcePlayer?.id ?? ScheduledCommand.NoPlayer, data, vtrRate));
}

// During command simulation (Client side)
public void ExecuteCmd(ScheduledCommand cmd)
{
    // Use stored VTR instead of calculating
    if (isResyncing)
    {
        int vtrRate = cmd.vtrRate;  // Use stored rate
    }
    else
    {
        int vtrRate = VTRSyncPatch.GetCurrentVTRRate(mapId);  // Calculate normally
    }
}
```

---

## **Summary**

Your understanding is **100% correct**:

1. ✅ **Client requests rejoin**
2. ✅ **Server prepares snapshot** (Last Save + Commands Since Save)
3. ✅ **Client receives snapshot**
4. ✅ **Client clears all states**
5. ✅ **Client loads snapshot**
6. ✅ **Client simulates commands**

The VTR problem occurs in step 6 when VTR calculations happen during command simulation, but player state isn't fully synchronized yet. 