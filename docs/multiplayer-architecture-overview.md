# Multiplayer Architecture Overview

## **🎯 Core Architecture Principles**

### **Deterministic Lockstep Multiplayer**
- **All clients must tick at identical rates** - No client-specific adaptation
- **Server authority** - Server maintains definitive game state
- **Desync recovery** - Clients reconnect when desyncs occur
- **Synchronized calculations** - All deterministic systems must use shared data

---

## **👥 Player Tracking System**

### **Server-Side Player Management**

**PlayerManager** (`Source/Common/PlayerManager.cs`)
- Maintains complete list of all connected players
- Handles player connections, disconnections, and status updates
- Broadcasts player changes to all connected clients

**Key Data Structures:**
```csharp
public List<ServerPlayer> Players { get; } = new();
public IEnumerable<ServerPlayer> JoinedPlayers => Players.Where(p => p.HasJoined);
public IEnumerable<ServerPlayer> PlayingPlayers => Players.Where(p => p.IsPlaying);
```

**ServerPlayer Properties:**
- `id` - Unique player identifier
- `conn` - Connection object
- `status` - Current status (Simulating, Playing, Desynced)
- `FactionId` - Player's faction assignment
- `lastCursorTick` - Last cursor update tick
- `ticksBehind` - How many ticks behind server
- `simulating` - Whether player is in simulation mode

### **Client-Side Player Tracking**

**PlayerInfo** (`Source/Client/Session/PlayerInfo.cs`)
- Each client maintains `PlayerInfo` objects for all connected players
- Real-time tracking of player positions, status, and data

**Key Player Data:**
```csharp
public int id;
public string username;
public PlayerStatus status;
public Vector3 cursor;
public Vector3 lastCursor;
public byte map; // Current map index
public Dictionary<int, float> selectedThings;
```

### **Player Synchronization Protocol**

**PlayerListAction Packets:**
- `Add` - New player joins
- `Remove` - Player disconnects
- `List` - Complete player list (sent on join)
- `Status` - Status updates (playing/simulating/desynced)
- `Latencies` - Periodic latency updates

**Real-Time Updates:**
- **Cursor positions** - Continuously synced between all clients
- **Player status** - Broadcast to all clients on changes
- **Latency information** - Periodically shared
- **Faction assignments** - Known to all clients

**Answer: YES - All clients know about all other clients all the time**

---

## **🗺️ Map Processing System**

### **Server-Side Map Management**

**WorldData** (`Source/Common/WorldData.cs`)
- Central repository for all map data
- Compressed storage for efficiency
- Command tracking per map

**Key Data Structures:**
```csharp
public Dictionary<int, byte[]> mapData = new(); // Map id → compressed map data
public Dictionary<int, List<byte[]>> mapCmds = new(); // Map id → serialized commands
public byte[]? savedGame; // Compressed game save
public byte[]? sessionData; // Compressed semi-persistent data
```

### **Client-Side Map Handling**

**Map Data Loading:**
- **On Join**: Client receives ALL map data from server during loading phase
- **Compressed Transfer**: Server sends compressed map data via `Server_WorldData` packet
- **Local Storage**: Client stores all map data in `Session.dataSnapshot.MapData`

**Map Processing Strategy:**
- **Server Authority**: Server processes all maps continuously
- **Client Processing**: Clients only process maps they're actively viewing
- **Data Synchronization**: When client switches maps, data is already available locally
- **Command Synchronization**: Map-specific commands sent to all clients regardless of current map

### **Map Loading Process**

**Server Loading State** (`Source/Common/Networking/State/ServerLoadingState.cs`):
```csharp
writer.WriteInt32(Server.worldData.mapData.Count);
foreach (var kv in Server.worldData.mapData)
{
    int mapId = kv.Key;
    byte[] mapData = kv.Value;
    writer.WriteInt32(mapId);
    writer.WritePrefixedBytes(mapData);
}
```

**Client Loading State** (`Source/Client/Networking/State/ClientLoadingState.cs`):
```csharp
int mapDataCount = data.ReadInt32();
for (int i = 0; i < mapDataCount; i++)
{
    int mapId = data.ReadInt32();
    byte[] rawMapData = data.ReadPrefixedBytes();
    byte[] mapData = GZipStream.UncompressBuffer(rawMapData);
    mapDataDict[mapId] = mapData;
    mapsToLoad.Add(mapId);
}
```

**Answer: NO - Maps that clients aren't on are NOT processed on clients all the time**

---

## **🔄 Synchronization Architecture**

### **Data Flow Patterns**

**Player Data Flow:**
1. **Server** maintains authoritative player list
2. **Server** broadcasts player changes to all clients
3. **Clients** maintain local copies of all player data
4. **Real-time updates** for cursor positions, status, etc.

**Map Data Flow:**
1. **Server** processes all maps continuously
2. **Server** sends compressed map data to clients on join
3. **Clients** store all map data locally
4. **Clients** only process active maps for performance
5. **Commands** synchronized across all clients regardless of current map

### **Performance Considerations**

**Player Tracking:**
- **Low overhead** - Player data is small and infrequently updated
- **Real-time cursor** - Frequent updates but lightweight
- **Status changes** - Infrequent but important for UI

**Map Processing:**
- **Compressed storage** - Map data compressed for network efficiency
- **Selective processing** - Clients only process active maps
- **Command synchronization** - All map commands sent to all clients
- **Memory usage** - All map data stored locally but not all processed

---

## **🎯 VTR Integration Implications**

### **For VTR Deterministic Sync**

**Player Position Tracking:**
- **Available data**: All client positions are tracked and synchronized
- **Distance calculation**: Can use closest player distance for VTR
- **Real-time updates**: Player positions updated continuously
- **Performance**: Lightweight cursor tracking already in place

**Map Processing:**
- **Server authority**: Server processes all maps, maintains definitive state
- **Client caching**: All map data available locally
- **VTR calculation**: Can be server-authoritative or use synchronized player positions
- **Deterministic approach**: All clients can use same distance calculation

### **VTR Implementation Strategy**

**Option 1: Closest Player Distance**
- Use minimum distance from any player to VTR target
- Leverage existing player position tracking
- Deterministic across all clients

**Option 2: Server-Authoritative VTR**
- Server calculates VTR rate, broadcasts to all clients
- Eliminates client-side VTR calculation entirely
- Maximum determinism

**Option 3: Average Player Distance**
- Calculate mean distance from all players to VTR target
- More complex but potentially more stable
- Requires synchronized calculation

---

## **📊 Architecture Summary**

### **Player Tracking: Complete Visibility**
- ✅ All clients know about all players at all times
- ✅ Real-time position and status updates
- ✅ Lightweight, efficient synchronization
- ✅ Ready for VTR distance calculations

### **Map Processing: Server-Authoritative with Client Caching**
- ✅ Server processes all maps continuously
- ✅ Clients store all map data locally
- ✅ Clients only process active maps for performance
- ✅ Commands synchronized across all clients
- ✅ Efficient network usage with compression

### **VTR Integration Ready**
- ✅ Player position data available for distance calculations
- ✅ Server authority available for deterministic VTR
- ✅ Existing synchronization infrastructure can be leveraged
- ✅ Performance considerations already addressed

---

## **🔧 Technical Implementation Notes**

### **Key Files**
- `Source/Common/PlayerManager.cs` - Server player management
- `Source/Common/ServerPlayer.cs` - Individual player data
- `Source/Client/Session/PlayerInfo.cs` - Client player tracking
- `Source/Common/WorldData.cs` - Server map data management
- `Source/Common/Networking/State/ServerLoadingState.cs` - Map data transfer
- `Source/Client/Networking/State/ClientLoadingState.cs` - Map data reception

### **Network Protocol**
- **PlayerListAction** - Player synchronization packets
- **Server_WorldData** - Map data transfer
- **Client_Cursor/Server_Cursor** - Real-time cursor updates
- **Fragmented packets** - Large map data transfers

### **Performance Characteristics**
- **Player tracking**: Low overhead, real-time updates
- **Map data**: Compressed, transferred on join
- **Processing**: Server handles all maps, clients selective
- **Memory**: All map data cached locally

---

*Last Updated: 2025-06-29 - Comprehensive multiplayer architecture documentation for VTR implementation planning* 