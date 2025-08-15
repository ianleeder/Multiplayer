# Resync Process - File Relationships & Data Flow

## **📁 File Architecture Overview**

```mermaid
graph TB
    subgraph "Entry Point"
        A[Rejoiner.cs]
    end
    
    subgraph "Data Structures"
        B[GameDataSnapshot.cs]
        C[ScheduledCommand.cs]
    end
    
    subgraph "State Management"
        D[SaveLoad.cs]
        E[LoadPatch.cs]
        F[MultiplayerSession.cs]
    end
    
    subgraph "Command Processing"
        G[AsyncTimeComp.cs]
        H[SyncUtil.cs]
    end
    
    subgraph "VTR Fix (NEW)"
        I[VTRSyncPatch.cs]
    end
    
    subgraph "Networking"
        J[Multiplayer.cs]
        K[ConnectionStateEnum.cs]
    end
    
    A --> D
    A --> J
    D --> B
    D --> C
    D --> E
    D --> I
    E --> F
    G --> C
    G --> H
    G --> I
    I --> F
    
    style A fill:#ff6b6b,stroke:#d63031,color:white
    style B fill:#74b9ff,stroke:#0984e3,color:white
    style C fill:#fd79a8,stroke:#e84393,color:white
    style D fill:#55a3ff,stroke:#2d3436,color:white
    style E fill:#00b894,stroke:#00a085,color:white
    style F fill:#fdcb6e,stroke:#e17055,color:white
    style G fill:#6c5ce7,stroke:#5f3dc4,color:white
    style H fill:#a29bfe,stroke:#6c5ce7,color:white
    style I fill:#00cec9,stroke:#00b894,color:white
    style J fill:#ff7675,stroke:#d63031,color:white
    style K fill:#ffeaa7,stroke:#fdcb6e,color:black
```

---

## **🔄 Detailed Data Flow**

### **Phase 1: Rejoin Request Flow**

```mermaid
sequenceDiagram
    participant UI as User Interface
    participant Rejoiner as Rejoiner.cs
    participant Multiplayer as Multiplayer.cs
    participant Server as Game Server
    participant Session as MultiplayerSession.cs
    
    UI->>Rejoiner: User clicks "Rejoin"
    Rejoiner->>Rejoiner: Clear maps & world
    Rejoiner->>Multiplayer: Send Client_RequestRejoin
    Rejoiner->>Session: ChangeState(ClientLoading)
    Multiplayer->>Server: Send rejoin request
    Server->>Session: Prepare GameDataSnapshot
    Server->>Multiplayer: Send snapshot data
```

### **Phase 2: Snapshot Creation**

```mermaid
graph TD
    subgraph "Server Side"
        A[MultiplayerSession.cs] --> B[Create GameDataSnapshot]
        B --> C[Serialize Game Data]
        B --> D[Serialize Session Data]
        B --> E[Serialize Map Data]
        B --> F[Collect Commands]
        
        C --> G[GameDataSnapshot]
        D --> G
        E --> G
        F --> G
    end
    
    subgraph "Data Structure"
        G --> H[CachedAtTime: int]
        G --> I[GameData: byte[]]
        G --> J[SessionData: byte[]]
        G --> K[MapData: Dict<int, byte[]>]
        G --> L[MapCmds: Dict<int, List<ScheduledCommand>>]
    end
    
    style A fill:#74b9ff,stroke:#0984e3,color:white
    style B fill:#fd79a8,stroke:#e84393,color:white
    style C fill:#55a3ff,stroke:#2d3436,color:white
    style D fill:#00b894,stroke:#00a085,color:white
    style E fill:#fdcb6e,stroke:#e17055,color:white
    style F fill:#6c5ce7,stroke:#5f3dc4,color:white
    style G fill:#a29bfe,stroke:#6c5ce7,color:white
```

### **Phase 3: Client State Restoration**

```mermaid
graph TD
    subgraph "Client Side"
        A[SaveLoad.cs] --> B[LoadInMainThread]
        B --> C[ClearState]
        B --> D[MemoryUtility.ClearAllMapsAndWorld]
        B --> E[LoadPatch.gameToLoad = gameData]
        B --> F[Find.Root.Start]
        B --> G[SavedGameLoaderNow.LoadGameFromSaveFileNow]
        B --> H[InitializeVTRAfterLoad]
    end
    
    subgraph "VTR Initialization (NEW)"
        H --> I[VTRSyncPatch.InitializeVTRForResync]
        I --> J[Set isResyncing = true]
        I --> K[Log player state]
        I --> L[Enable deterministic VTR]
    end
    
    style A fill:#74b9ff,stroke:#0984e3,color:white
    style B fill:#fd79a8,stroke:#e84393,color:white
    style C fill:#55a3ff,stroke:#2d3436,color:white
    style D fill:#00b894,stroke:#00a085,color:white
    style E fill:#fdcb6e,stroke:#e17055,color:white
    style F fill:#6c5ce7,stroke:#5f3dc4,color:white
    style G fill:#a29bfe,stroke:#6c5ce7,color:white
    style H fill:#00cec9,stroke:#00b894,color:white
    style I fill:#ff7675,stroke:#d63031,color:white
```

### **Phase 4: Command Simulation**

```mermaid
graph TD
    subgraph "Command Processing"
        A[AsyncTimeComp.cs] --> B[Tick]
        B --> C[Process Commands]
        C --> D[ExecuteCmd for each command]
        D --> E[PreContext - Set RNG state]
        D --> F[Handle command type]
        D --> G[UpdateManagers]
        D --> H[PostContext - Restore RNG state]
    end
    
    subgraph "VTR Calculations During Simulation"
        F --> I[VTRSyncPatch.Prefix]
        I --> J[GetSynchronizedUpdateRate]
        J --> K[Check isResyncing]
        K --> L[GetDeterministicVTRForResync]
        L --> M[Return deterministic rate]
    end
    
    style A fill:#74b9ff,stroke:#0984e3,color:white
    style B fill:#fd79a8,stroke:#e84393,color:white
    style C fill:#55a3ff,stroke:#2d3436,color:white
    style D fill:#00b894,stroke:#00a085,color:white
    style E fill:#fdcb6e,stroke:#e17055,color:white
    style F fill:#6c5ce7,stroke:#5f3dc4,color:white
    style G fill:#a29bfe,stroke:#6c5ce7,color:white
    style H fill:#00cec9,stroke:#00b894,color:white
    style I fill:#ff7675,stroke:#d63031,color:white
```

---

## **🔧 Code Integration Points**

### **VTR Fix Integration**

```mermaid
graph LR
    subgraph "SaveLoad.cs"
        A[LoadInMainThread] --> B[InitializeVTRAfterLoad]
    end
    
    subgraph "VTRSyncPatch.cs"
        B --> C[InitializeVTRForResync]
        C --> D[Set isResyncing flag]
        C --> E[Log player state]
    end
    
    subgraph "AsyncTimeComp.cs"
        F[ExecuteCmd] --> G[VTR calculations]
        G --> H[VTRSyncPatch.Prefix]
        H --> I[Check isResyncing]
        I --> J[GetDeterministicVTRForResync]
    end
    
    subgraph "MultiplayerSession.cs"
        K[Player data] --> L[Session.players]
        L --> M[Player.map assignments]
        M --> J
    end
    
    style A fill:#74b9ff,stroke:#0984e3,color:white
    style B fill:#fd79a8,stroke:#e84393,color:white
    style C fill:#55a3ff,stroke:#2d3436,color:white
    style D fill:#00b894,stroke:#00a085,color:white
    style E fill:#fdcb6e,stroke:#e17055,color:white
    style F fill:#6c5ce7,stroke:#5f3dc4,color:white
    style G fill:#a29bfe,stroke:#6c5ce7,color:white
    style H fill:#00cec9,stroke:#00b894,color:white
    style I fill:#ff7675,stroke:#d63031,color:white
    style J fill:#ffeaa7,stroke:#fdcb6e,color:black
    style K fill:#dda0dd,stroke:#ba55d3,color:white
    style L fill:#98fb98,stroke:#32cd32,color:black
    style M fill:#f0e68c,stroke:#daa520,color:black
```

---

## **📊 Data Structures & Relationships**

### **GameDataSnapshot Structure**

```mermaid
classDiagram
    class GameDataSnapshot {
        +int CachedAtTime
        +byte[] GameData
        +byte[] SessionData
        +Dictionary<int, byte[]> MapData
        +Dictionary<int, List<ScheduledCommand>> MapCmds
    }
    
    class ScheduledCommand {
        +CommandType type
        +byte[] data
        +int mapId
        +int playerId
        +bool issuedBySelf
    }
    
    class CommandType {
        <<enumeration>>
        Sync
        MapTimeSpeed
        Designator
        WorldObject
        MapObject
        Global
    }
    
    GameDataSnapshot --> ScheduledCommand
    ScheduledCommand --> CommandType
```

### **VTR State Management**

```mermaid
stateDiagram-v2
    [*] --> NormalOperation
    NormalOperation --> ResyncInitialization : Client rejoins
    ResyncInitialization --> DeterministicVTR : InitializeVTRForResync
    DeterministicVTR --> CommandSimulation : Start simulation
    CommandSimulation --> NormalOperation : Resync complete
    
    state DeterministicVTR {
        [*] --> CheckPlayerState
        CheckPlayerState --> PlayerOnMap : hasPlayerOnMap = true
        CheckPlayerState --> NoPlayerOnMap : hasPlayerOnMap = false
        PlayerOnMap --> ReturnVTR1 : return 1
        NoPlayerOnMap --> ReturnVTR15 : return 15
    }
    
    state CommandSimulation {
        [*] --> ProcessCommands
        ProcessCommands --> VTRCalculation : Each tick
        VTRCalculation --> DeterministicVTR : Use deterministic logic
        DeterministicVTR --> ProcessCommands : Continue simulation
    }
```

---

## **🎯 Critical Integration Points**

### **1. Rejoiner → SaveLoad**
- **Trigger**: `Rejoiner.DoRejoin()` calls `SaveLoad.LoadInMainThread()`
- **Data**: `GameDataSnapshot` passed from server to client
- **Purpose**: Restore complete game state

### **2. SaveLoad → VTRSyncPatch**
- **Trigger**: `SaveLoad.InitializeVTRAfterLoad()` calls `VTRSyncPatch.InitializeVTRForResync()`
- **Data**: Session player state for debugging
- **Purpose**: Enable deterministic VTR during resync

### **3. AsyncTimeComp → VTRSyncPatch**
- **Trigger**: Every VTR calculation during command simulation
- **Data**: Current map and thing being processed
- **Purpose**: Ensure identical VTR rates across clients

### **4. VTRSyncPatch → MultiplayerSession**
- **Trigger**: VTR calculations need player state
- **Data**: `Multiplayer.session.players` for map assignments
- **Purpose**: Determine which maps have players for VTR calculation

---

## **🔍 Debugging Integration**

### **Logging Points**

```mermaid
graph TD
    A[VTRSyncPatch.InitializeVTRForResync] --> B[Log: "VTR: Initializing for resync"]
    B --> C[Log each player state]
    C --> D[Log: "VTR: Player {id} on map {map}, status {status}"]
    
    E[VTRSyncPatch.GetDeterministicVTRForResync] --> F[Log: "VTR: Deterministic calculation for map {mapId}"]
    F --> G[Log: "VTR: hasPlayerOnMap = {result}"]
    G --> H[Log: "VTR: Returning rate {rate}"]
    
    I[VTRSyncPatch.Prefix] --> J[Log: "VTR: Sync error: {message}"]
    J --> K[Log: "VTR: Using fallback rate 15"]
    
    style A fill:#74b9ff,stroke:#0984e3,color:white
    style B fill:#fd79a8,stroke:#e84393,color:white
    style C fill:#55a3ff,stroke:#2d3436,color:white
    style D fill:#00b894,stroke:#00a085,color:white
    style E fill:#fdcb6e,stroke:#e17055,color:white
    style F fill:#6c5ce7,stroke:#5f3dc4,color:white
    style G fill:#a29bfe,stroke:#6c5ce7,color:white
    style H fill:#00cec9,stroke:#00b894,color:white
    style I fill:#ff7675,stroke:#d63031,color:white
    style J fill:#ffeaa7,stroke:#fdcb6e,color:black
    style K fill:#dda0dd,stroke:#ba55d3,color:white
```

This comprehensive documentation shows how all the files work together during the resync process, with special emphasis on the VTR fix integration points. 