# ModelMesh API Documentation

## Overview

ModelMesh provides two distinct gRPC APIs defined in Protocol Buffer format:

1. **External Client API** (`model-mesh.proto`) - For managing models and virtual models in the cluster
2. **Runtime Integration API** (`model-runtime.proto`) - For model runtime containers to interface with ModelMesh

## External Client API (`model-mesh.proto`)

### Service: `ModelMesh`

The primary interface for external clients to manage models in a ModelMesh cluster.

#### Model Management Operations

##### `registerModel`
**Purpose**: Registers a trained model with the ModelMesh cluster

**Request**: `RegisterModelRequest`
- `modelId` (string) - Unique identifier for the model
- `modelInfo` (ModelInfo) - Model metadata (type, path, key)
- `loadNow` (bool) - Whether to load the model immediately
- `sync` (bool) - Whether to block until load completes (if loadNow=true)
- `lastUsedTime` (uint64) - Optional timestamp for cache priority

**Response**: `ModelStatusInfo` - Current status of the registered model

---

##### `unregisterModel`
**Purpose**: Removes a model from the cluster

**Request**: `UnregisterModelRequest`
- `modelId` (string) - ID of model to remove

**Response**: `UnregisterModelResponse` (empty)

---

##### `getModelStatus`
**Purpose**: Retrieves current status of a specific model

**Request**: `GetStatusRequest`
- `modelId` (string) - ID of model to query

**Response**: `ModelStatusInfo` - Detailed status information

---

##### `ensureLoaded`
**Purpose**: Ensures a model is loaded somewhere in the cluster

**Request**: `EnsureLoadedRequest`
- `modelId` (string) - ID of model to ensure is loaded
- `lastUsedTime` (uint64) - Timestamp for cache priority (0 = now)
- `sync` (bool) - Whether to block until loading completes

**Response**: `ModelStatusInfo` - Status after load operation

#### Virtual Model (VModel) Operations

##### `setVModel`
**Purpose**: Creates or updates a virtual model mapping

**Request**: `SetVModelRequest`
- `vModelId` (string) - Virtual model identifier
- `owner` (string) - Owner for access control
- `targetModelId` (string) - Concrete model this VModel points to
- `updateOnly` (bool) - Fail if VModel doesn't exist
- `modelInfo` (ModelInfo) - Optional info to create target model
- `autoDeleteTargetModel` (bool) - Auto-delete when no longer referenced
- `loadNow` (bool) - Load target model immediately
- `force` (bool) - Update active model immediately
- `sync` (bool) - Block until transition completes
- `expectedTargetModelId` (string) - Expected current target for atomic updates

**Response**: `VModelStatusInfo` - Status of the virtual model

---

##### `deleteVModel`
**Purpose**: Deletes a virtual model

**Request**: `DeleteVModelRequest`
- `vModelId` (string) - Virtual model to delete
- `owner` (string) - Must match VModel owner if provided

**Response**: `DeleteVModelResponse` (empty)

---

##### `getVModelStatus`
**Purpose**: Retrieves status of a virtual model

**Request**: `GetVModelStatusRequest`
- `vModelId` (string) - Virtual model to query
- `owner` (string) - Must match VModel owner if provided

**Response**: `VModelStatusInfo` - Virtual model status and transitions

### Data Types

#### `ModelInfo`
Model metadata passed to runtime for loading:
- `type` (string) - Model type identifier (required)
- `path` (string) - Model location/path
- `key` (string) - Additional model metadata

#### `ModelStatusInfo`
Comprehensive model status information:
- `status` (ModelStatus) - Overall model state
- `errors` (repeated string) - Error messages if any
- `modelCopyInfos` (repeated ModelCopyInfo) - Per-instance status details

**ModelStatus Enum**:
- `NOT_FOUND` - Model not registered
- `NOT_LOADED` - Registered but not loaded anywhere
- `LOADING` - Currently loading somewhere
- `LOADED` - Available in at least one instance
- `LOADING_FAILED` - Load failed, will retry
- `UNKNOWN` - Status uncertain

#### `VModelStatusInfo`
Virtual model status and transition information:
- `status` (VModelStatus) - Virtual model state
- `activeModelId` (string) - Currently active concrete model
- `targetModelId` (string) - Target model for transitions
- `activeModelStatus` (ModelStatusInfo) - Status of active model
- `targetModelStatus` (ModelStatusInfo) - Status of target model (if transitioning)
- `owner` (string) - VModel owner

**VModelStatus Enum**:
- `NOT_FOUND` - VModel not registered
- `DEFINED` - Steady state (active == target)
- `TRANSITIONING` - Switching to new target model
- `TRANSITION_FAILED` - Target model failed to load
- `UNKNOWN` - Status uncertain

## Runtime Integration API (`model-runtime.proto`)

### Service: `ModelRuntime`

Interface that model runtime containers must implement to integrate with ModelMesh.

#### Core Runtime Operations

##### `loadModel`
**Purpose**: Load a model into the runtime

**Request**: `LoadModelRequest`
- `modelId` (string) - Model identifier
- `modelType` (string) - Model type from ModelInfo
- `modelPath` (string) - Model path from ModelInfo
- `modelKey` (string) - Model key from ModelInfo

**Response**: `LoadModelResponse`
- `sizeInBytes` (uint64) - Memory consumed by loaded model
- `maxConcurrency` (uint32) - Max concurrent requests (experimental)

**Notes**: 
- Must handle cancellation gracefully
- Should return size if no additional cost involved
- Cancellation followed by unloadModel call

---

##### `unloadModel`
**Purpose**: Unload a previously loaded model

**Request**: `UnloadModelRequest`
- `modelId` (string) - Model to unload

**Response**: `UnloadModelResponse` (empty)

**Notes**: Return immediately if model not found/loaded

---

##### `predictModelSize`
**Purpose**: Estimate memory requirements before loading (optional)

**Request**: `PredictModelSizeRequest`
- `modelId` (string) - Model identifier
- `modelType` (string) - Model type
- `modelPath` (string) - Model path
- `modelKey` (string) - Model key

**Response**: `PredictModelSizeResponse`
- `sizeInBytes` (uint64) - Conservative size estimate

**Notes**: Must return quickly, no expensive computation

---

##### `modelSize`
**Purpose**: Calculate actual size of loaded model

**Request**: `ModelSizeRequest`
- `modelId` (string) - Loaded model to measure

**Response**: `ModelSizeResponse`
- `sizeInBytes` (uint64) - Actual memory consumption

**Notes**: Required only if size not returned from loadModel

---

##### `runtimeStatus`
**Purpose**: Provide runtime configuration and status

**Request**: `RuntimeStatusRequest` (empty)

**Response**: `RuntimeStatusResponse`
- `status` (Status) - Runtime state (STARTING/READY/FAILING)
- `capacityInBytes` (uint64) - Total memory capacity
- `maxLoadingConcurrency` (uint32) - Max parallel loads
- `modelLoadingTimeoutMs` (uint32) - Load timeout
- `defaultModelSizeInBytes` (uint64) - Conservative default size
- `runtimeVersion` (string) - Runtime version info
- `methodInfos` (map) - Method-specific configuration
- `limitModelConcurrency` (bool) - Enable per-model concurrency limits
- `allowAnyMethod` (bool) - Forward all RPC methods

**MethodInfo Structure**:
- `idInjectionPath` (repeated uint32) - Protobuf field path for model ID injection

**Notes**: 
- Called only during startup
- Must purge any existing loaded models before returning READY

## API Usage Patterns

### Basic Model Management
```
1. registerModel() -> ModelStatusInfo
2. getModelStatus() -> Check status
3. ensureLoaded() -> Guarantee availability
4. unregisterModel() -> Cleanup
```

### Virtual Model Workflow
```
1. setVModel() -> Create VModel pointing to concrete model
2. getVModelStatus() -> Monitor transition
3. setVModel() -> Update to new target model (blue/green deployment)
4. deleteVModel() -> Cleanup
```

### Runtime Integration
```
1. runtimeStatus() -> Announce capabilities
2. loadModel() -> Load on demand
3. modelSize() -> Report actual size
4. unloadModel() -> Free resources
```

## Access Control

### VModel Ownership
- **Owner-based Access**: VModels support optional ownership for multi-tenancy
- **Owner Validation**: Operations require matching owner or fail silently
- **Backwards Compatibility**: Operations without owner succeed regardless

### Security Features
- **Information Hiding**: Non-owners see NOT_FOUND instead of actual status
- **Silent Failures**: Invalid ownership operations fail without error indication
- **Immutable Ownership**: Owner set at creation cannot be changed

## Error Handling

### Model Operations
- **NOT_FOUND**: Model/VModel doesn't exist
- **LOADING_FAILED**: Load operation failed, automatic retry
- **ALREADY_EXISTS**: VModel creation with conflicting owner

### Runtime Operations
- **PRECONDITION_FAILED**: Runtime couldn't attempt load
- **INVALID_ARGUMENT**: Invalid request parameters
- **Cancellation**: Graceful handling of cancelled load operations

## Configuration

### Environment Variables
- `MM_DEFAULT_VMODEL_OWNER` - Default owner for new VModels
- Model-specific scaling parameters via dynamic configuration

### Runtime Capabilities
- Memory capacity and loading concurrency limits
- Method routing and ID injection configuration
- Concurrency control and latency-based autoscaling

## Best Practices

1. **Use VModels** for production deployments requiring zero-downtime updates
2. **Set loadNow=false** for registration unless immediate loading required
3. **Implement predictModelSize** for better resource planning
4. **Handle cancellation** properly in loadModel implementations
5. **Use owner fields** for multi-tenant VModel management
6. **Monitor ModelStatusInfo** for load failures and retry logic