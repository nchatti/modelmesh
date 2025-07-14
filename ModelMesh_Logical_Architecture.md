# ModelMesh Logical Architecture

## System Overview

This diagram shows the logical architecture of ModelMesh, illustrating the conceptual layers, data flow, and key interactions between different parts of the system.

```mermaid
graph TD
    subgraph "Client Layer"
        WEB[Web Applications]
        CLI[CLI Tools]
        SDK[SDKs & Libraries]
    end

    subgraph "API Gateway Layer"
        REST[REST API Gateway]
        GRPC_GW[gRPC Gateway]
    end

    subgraph "ModelMesh Control Plane"
        subgraph "API Layer"
            MM_API[ModelMesh API<br/>- Model Registration<br/>- Status Queries<br/>- VModel Management]
            RT_API[Runtime API<br/>- Model Loading<br/>- Health Checks<br/>- Capacity Reporting]
        end

        subgraph "Orchestration Layer"
            ORCHESTRATOR[Model Orchestrator<br/>- Request Routing<br/>- Load Balancing<br/>- Capacity Planning]
            VMODEL_MGR[Virtual Model Manager<br/>- Version Control<br/>- A/B Testing<br/>- Gradual Rollouts]
            PLACEMENT[Placement Engine<br/>- Resource Constraints<br/>- GPU Affinity<br/>- Cost Optimization]
        end

        subgraph "Caching Layer"
            CACHE_MGR[Cache Manager<br/>- LRU Eviction<br/>- Memory Management<br/>- Hit Rate Optimization]
            MODEL_CACHE[(Model Cache<br/>Distributed LRU)]
        end
    end

    subgraph "Data Plane"
        subgraph "Model Runtime Layer"
            TRITON_1[Triton Runtime 1<br/>GPU Instance]
            TRITON_2[Triton Runtime 2<br/>GPU Instance]
            TRITON_N[Triton Runtime N<br/>GPU Instance]
            OTHER_RT[Other Runtimes<br/>TensorFlow Serving<br/>MLServer, etc.]
        end

        subgraph "Inference Processing"
            INFERENCE_ROUTER[Inference Router<br/>- Model Resolution<br/>- Instance Selection<br/>- Request Forwarding]
            PAYLOAD_PROC[Payload Processor<br/>- Protocol Translation<br/>- Data Transformation<br/>- Logging & Analytics]
        end
    end

    subgraph "State Management Layer"
        subgraph "Distributed Storage"
            MODEL_REGISTRY[(Model Registry<br/>- Model Metadata<br/>- Instance Mapping<br/>- Health Status)]
            INSTANCE_REGISTRY[(Instance Registry<br/>- Capacity Info<br/>- Resource Usage<br/>- Constraints)]
            CONFIG_STORE[(Configuration Store<br/>- Runtime Config<br/>- Policies<br/>- Feature Flags)]
        end

        subgraph "Coordination Services"
            LEADER_ELECTION[Leader Election<br/>- Cluster Coordination<br/>- Global Operations]
            SESSION_MGR[Session Manager<br/>- Instance Lifecycle<br/>- Heartbeat Monitoring]
            EVENT_BUS[Event Bus<br/>- State Notifications<br/>- Change Propagation]
        end
    end

    subgraph "External Infrastructure"
        STORAGE[Model Storage<br/>S3, NFS, Git]
        ETCD[(etcd Cluster<br/>Distributed KV Store)]
        METRICS[Metrics & Monitoring<br/>Prometheus, Grafana]
    end

    %% Client connections
    WEB --> REST
    CLI --> GRPC_GW
    SDK --> MM_API

    %% API Gateway routing
    REST --> MM_API
    GRPC_GW --> MM_API

    %% Control plane flow
    MM_API --> ORCHESTRATOR
    MM_API --> VMODEL_MGR
    RT_API --> PLACEMENT
    
    ORCHESTRATOR --> CACHE_MGR
    ORCHESTRATOR --> INFERENCE_ROUTER
    VMODEL_MGR --> MODEL_REGISTRY
    PLACEMENT --> INSTANCE_REGISTRY
    
    %% Cache management
    CACHE_MGR --> MODEL_CACHE
    MODEL_CACHE --> TRITON_1
    MODEL_CACHE --> TRITON_2
    MODEL_CACHE --> TRITON_N

    %% Data plane processing
    INFERENCE_ROUTER --> PAYLOAD_PROC
    PAYLOAD_PROC --> TRITON_1
    PAYLOAD_PROC --> TRITON_2  
    PAYLOAD_PROC --> TRITON_N
    PAYLOAD_PROC --> OTHER_RT

    %% State management
    ORCHESTRATOR --> MODEL_REGISTRY
    ORCHESTRATOR --> INSTANCE_REGISTRY
    PLACEMENT --> CONFIG_STORE
    
    LEADER_ELECTION --> ETCD
    SESSION_MGR --> ETCD
    EVENT_BUS --> ETCD
    
    MODEL_REGISTRY --> ETCD
    INSTANCE_REGISTRY --> ETCD
    CONFIG_STORE --> ETCD

    %% External dependencies
    TRITON_1 --> STORAGE
    TRITON_2 --> STORAGE
    TRITON_N --> STORAGE
    OTHER_RT --> STORAGE
    
    ORCHESTRATOR --> METRICS
    CACHE_MGR --> METRICS
    INFERENCE_ROUTER --> METRICS

    %% Styling
    classDef clientLayer fill:#e3f2fd,stroke:#0277bd,stroke-width:2px
    classDef apiLayer fill:#f1f8e9,stroke:#33691e,stroke-width:2px
    classDef controlPlane fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    classDef dataPlane fill:#fce4ec,stroke:#c2185b,stroke-width:2px
    classDef stateLayer fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef external fill:#eceff1,stroke:#455a64,stroke-width:2px

    class WEB,CLI,SDK clientLayer
    class REST,GRPC_GW,MM_API,RT_API apiLayer
    class ORCHESTRATOR,VMODEL_MGR,PLACEMENT,CACHE_MGR,MODEL_CACHE controlPlane
    class TRITON_1,TRITON_2,TRITON_N,OTHER_RT,INFERENCE_ROUTER,PAYLOAD_PROC dataPlane
    class MODEL_REGISTRY,INSTANCE_REGISTRY,CONFIG_STORE,LEADER_ELECTION,SESSION_MGR,EVENT_BUS stateLayer
    class STORAGE,ETCD,METRICS external
```

## Request Flow Patterns

```mermaid
sequenceDiagram
    participant Client
    participant API as ModelMesh API
    participant Orchestrator
    participant Cache as Model Cache
    participant Runtime as Triton Runtime
    participant Registry as Model Registry

    Note over Client,Registry: Model Deployment Flow
    Client->>API: Deploy Model Request
    API->>Orchestrator: Register Model
    Orchestrator->>Registry: Store Model Metadata
    Orchestrator->>Runtime: Load Model
    Runtime-->>Orchestrator: Model Loaded
    Orchestrator-->>API: Deployment Success
    API-->>Client: 201 Created

    Note over Client,Registry: Inference Request Flow
    Client->>API: Inference Request
    API->>Orchestrator: Route Request
    Orchestrator->>Cache: Check Model Location
    
    alt Model in Cache
        Cache->>Runtime: Forward Request
        Runtime-->>Cache: Inference Result
        Cache-->>Orchestrator: Result
    else Model Not Cached
        Orchestrator->>Runtime: Load & Infer
        Runtime->>Cache: Cache Model
        Runtime-->>Orchestrator: Result
    end
    
    Orchestrator-->>API: Response
    API-->>Client: 200 OK + Result
```

## Data Flow Architecture

```mermaid
flowchart LR
    subgraph "Ingress"
        REQ[Client Requests]
    end

    subgraph "Processing Pipeline"
        ROUTE[Request Router]
        RESOLVE[Model Resolver]
        LOAD_BAL[Load Balancer]
        TRANSFORM[Data Transformer]
    end

    subgraph "Decision Engine"
        CACHE_CHECK{Model in Cache?}
        CAPACITY_CHECK{Capacity Available?}
        CONSTRAINT_CHECK{Constraints Met?}
    end

    subgraph "Execution"
        CACHE_HIT[Cache Hit - Direct Execution]
        LOAD_MODEL[Load Model]
        EXECUTE[Execute Inference]
        EVICT[Evict if Needed]
    end

    subgraph "Response"
        AGGREGATE[Response Aggregation]
        RESP[Client Response]
    end

    REQ --> ROUTE
    ROUTE --> RESOLVE
    RESOLVE --> LOAD_BAL
    LOAD_BAL --> TRANSFORM
    TRANSFORM --> CACHE_CHECK

    CACHE_CHECK -->|Yes| CACHE_HIT
    CACHE_CHECK -->|No| CAPACITY_CHECK
    
    CAPACITY_CHECK -->|Yes| CONSTRAINT_CHECK
    CAPACITY_CHECK -->|No| EVICT
    
    CONSTRAINT_CHECK -->|Met| LOAD_MODEL
    CONSTRAINT_CHECK -->|Not Met| ROUTE
    
    EVICT --> LOAD_MODEL
    LOAD_MODEL --> EXECUTE
    CACHE_HIT --> EXECUTE
    
    EXECUTE --> AGGREGATE
    AGGREGATE --> RESP

    class CACHE_CHECK,CAPACITY_CHECK,CONSTRAINT_CHECK decision
    class ROUTE,RESOLVE,LOAD_BAL,TRANSFORM,LOAD_MODEL,EXECUTE,EVICT,AGGREGATE process
    class REQ,RESP,CACHE_HIT endpoint
```

## Logical Layer Descriptions

### Client Layer
- **Multiple Interfaces**: Web apps, CLI tools, SDKs for different programming languages
- **Protocol Abstraction**: Clients use familiar interfaces (REST, native gRPC)

### API Gateway Layer  
- **Protocol Translation**: REST to gRPC conversion for legacy clients
- **Request Validation**: Input validation and authentication
- **Rate Limiting**: Traffic control and quotas

### Control Plane
- **Orchestration**: Central brain coordinating all model operations
- **Virtual Models**: Abstraction layer for model versioning and transitions
- **Placement Logic**: Intelligent model placement based on constraints
- **Caching Strategy**: LRU cache management across cluster

### Data Plane
- **Runtime Abstraction**: Support for multiple model serving frameworks
- **Request Processing**: Protocol translation and payload handling
- **Performance Optimization**: Direct routing to loaded models

### State Management
- **Distributed Registries**: Centralized state storage for models and instances
- **Coordination**: Leader election and distributed consensus
- **Event Propagation**: Real-time state change notifications

### External Infrastructure
- **Model Storage**: Persistent storage for model artifacts
- **Configuration**: Distributed configuration management via etcd
- **Observability**: Metrics collection and monitoring integration

## Key Logical Principles

1. **Separation of Concerns**: Clear boundaries between control plane and data plane
2. **Event-Driven Architecture**: State changes propagate through event notifications
3. **Distributed Caching**: Intelligent model placement and caching decisions
4. **Pluggable Runtimes**: Support for multiple model serving frameworks
5. **Horizontal Scalability**: Stateless components with shared distributed state
6. **Fault Tolerance**: Leader election and graceful degradation mechanisms