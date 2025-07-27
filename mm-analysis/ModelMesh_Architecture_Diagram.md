# ModelMesh Top-Level Architecture

## Internal Component Architecture

```mermaid
graph TB
    subgraph "External Clients"
        CLI[Client Applications]
        API[API Gateway/REST Proxy]
    end

    subgraph "ModelMesh Core Layer"
        MM[ModelMesh Core<br/>- Orchestration<br/>- Request Routing<br/>- Load Balancing]
        VMM[VModelManager<br/>- Virtual Models<br/>- Version Management<br/>- Transitions]
        TCM[TypeConstraintManager<br/>- GPU Constraints<br/>- Instance Placement<br/>- Resource Affinity]
    end

    subgraph "Model Management"
        ML[ModelLoader<br/>- Abstract Interface<br/>- Runtime Loading<br/>- Size Prediction]
        SMM[SidecarModelMesh<br/>- External Runtime Integration<br/>- gRPC Communication<br/>- Failure Handling]
    end

    subgraph "Data Structures"
        MR[ModelRecord<br/>- Model State<br/>- Instance Placement<br/>- Reference Counting]
        IR[InstanceRecord<br/>- Capacity Tracking<br/>- Resource Utilization<br/>- Health Status]
        VR[VModelRecord<br/>- Virtual Model State<br/>- Target Mappings<br/>- Transition Status]
    end

    subgraph "Distributed Coordination"
        KVT[KVTable<br/>- Distributed Registry<br/>- Event Notifications<br/>- State Synchronization]
        LE[Leader Election<br/>- Cluster Coordination<br/>- Global Operations<br/>- Registry Maintenance]
        SN[SessionNode<br/>- Instance Presence<br/>- Heartbeat Management<br/>- Graceful Shutdown]
    end

    subgraph "Communication Protocols"
        GRPC[gRPC Services<br/>- ModelMesh API<br/>- ModelRuntime API<br/>- Protocol Buffers]
        THRIFT[Thrift Services<br/>- Legacy Support<br/>- Internal Forwarding<br/>- Backward Compatibility]
    end

    subgraph "Payload Processing"
        PP[PayloadProcessor Pipeline<br/>- Request/Response Handling<br/>- Logging & Analytics<br/>- Async Processing]
    end

    subgraph "External Dependencies"
        ETCD[(etcd/Zookeeper<br/>Distributed Storage)]
        TRITON[Triton Servers<br/>- Model Runtimes<br/>- GPU Processing<br/>- Inference Execution]
    end

    subgraph "Cache Management"
        CLHM[ConcurrentLinkedHashMap<br/>- LRU Cache<br/>- Thread-Safe Operations<br/>- Memory Management]
    end

    %% Client connections
    CLI --> GRPC
    API --> GRPC
    
    %% Core component relationships
    GRPC --> MM
    THRIFT --> MM
    MM --> VMM
    MM --> TCM
    MM --> ML
    MM --> PP
    
    %% Model management flow
    ML --> SMM
    SMM --> TRITON
    
    %% Data structure relationships
    MM --> MR
    MM --> IR
    VMM --> VR
    MM --> CLHM
    
    %% Distributed coordination
    MM --> KVT
    MM --> LE
    MM --> SN
    KVT --> ETCD
    LE --> ETCD
    SN --> ETCD
    
    %% Data persistence
    MR -.-> KVT
    IR -.-> KVT
    VR -.-> KVT
    
    %% Cache operations
    MM --> CLHM
    CLHM -.-> MR

    %% Styling
    classDef coreComponent fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef dataStructure fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef coordination fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px
    classDef external fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef communication fill:#fce4ec,stroke:#880e4f,stroke-width:2px

    class MM,VMM,TCM,ML,SMM coreComponent
    class MR,IR,VR,CLHM dataStructure
    class KVT,LE,SN coordination
    class ETCD,TRITON external
    class GRPC,THRIFT,PP communication
```

## Component Descriptions

### Core Components
- **ModelMesh Core**: Central orchestrator managing model lifecycle, request routing, and load balancing
- **VModelManager**: Manages virtual models for versioning, A/B testing, and seamless transitions
- **TypeConstraintManager**: Handles GPU affinity and resource constraints for optimal model placement

### Model Management
- **ModelLoader**: Abstract interface for loading different model types with size prediction
- **SidecarModelMesh**: Implements sidecar pattern for external model runtime integration

### Data Structures
- **ModelRecord**: Tracks model state, instance placement, and reference counting
- **InstanceRecord**: Monitors cluster capacity, utilization, and health status
- **VModelRecord**: Manages virtual model state and target mappings
- **ConcurrentLinkedHashMap**: Thread-safe LRU cache for efficient model memory management

### Distributed Coordination
- **KVTable**: Distributed registry with event notifications and state synchronization
- **Leader Election**: Ensures single coordinator for cluster-wide operations
- **SessionNode**: Manages instance presence and graceful lifecycle management

### Communication
- **gRPC Services**: Modern API for external model management and internal runtime communication
- **Thrift Services**: Legacy protocol support for backward compatibility
- **PayloadProcessor**: Pipeline for request/response processing and analytics

### External Dependencies
- **etcd/Zookeeper**: Distributed storage backend for cluster state and coordination
- **Triton Servers**: External model runtimes providing GPU-accelerated inference

## Key Architectural Patterns

1. **Sidecar Pattern**: ModelMesh runs alongside model runtimes for seamless integration
2. **Distributed Cache**: Intelligent LRU caching across cluster instances
3. **Leader Election**: Centralized coordination for global operations
4. **Observer Pattern**: Event-driven state synchronization across components
5. **Strategy Pattern**: Pluggable model loaders and payload processors