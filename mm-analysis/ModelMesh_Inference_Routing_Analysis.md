# ModelMesh Inference Request Routing Strategy Analysis

*Based exclusively on current codebase examination*

## Main Entry Point: invokeModel Method

**File**: `src/main/java/com/ibm/watson/modelmesh/ModelMesh.java`  
**Lines**: 3421-4000+

The primary inference routing logic is implemented in the `invokeModel` method, which serves as the central hub for all inference requests.

### Request Classification (Lines 3434-3455)

```java
// Determines request type based on TAS_INTERNAL context
final String tasInternal = contextMap.get(TAS_INTERNAL_CXT_KEY);

if (tasInternal == null) {
    externalReq = true; hopType = "ex"; // External request
} else if (INTERNAL_REQ.equals(tasInternal)) {
    externalReq = true; hopType = "in"; // Internal request  
} else if (HIT_ONLY.equals(tasInternal)) {
    externalReq = false; local = true; hopType = "ho"; // Hit-only (cache hit)
} else {
    externalReq = false; hopType = "ll"; // Load-local (cache miss)
}
```

**Request Types:**
- **External ("ex")**: Original client requests
- **Internal ("in")**: Inter-cluster forwarding
- **Hit-only ("ho")**: Cache hit forwarding
- **Load-local ("ll")**: Cache miss placement

## Load Balancing Strategy

### Source Balancing Logic (Lines 3521-3535)

```java
// Determine if request was "balanced" at source
final boolean sourceIsBalanced = !"true".equals(contextMap.get(UNBALANCED_KEY));
final boolean favourSelfForHits = (!limitModelConcurrency && sourceIsBalanced) || method == null;
final boolean preferSelfForHits = !favourSelfForHits && sourceIsBalanced;
```

**Balancing Behavior:**
- **Source balanced**: Prefer local instance to reduce hops
- **Unbalanced**: Use standard load balancing across cluster
- **Self-favor**: Always choose local instance for cache hits when possible

### Load Balancer Classes

#### ForwardingLB (Lines 4309-4408)
**Purpose**: Routes requests to instances with loaded models (cache hits)

**Selection Logic:**
1. **In-use count priority**: Prefer instances with fewer active requests
2. **LRU consideration**: Among equal load, prefer more recent activity  
3. **Loading state**: Avoid instances still loading models
4. **Self-exclusion**: Skip local instance if `excludeSelf` is true

```java
// Line 4356: Load balancing logic
final int inuse = us ? localInvokesInFlight.get() : si.getInUseCount();
if (inuse > min) {
    continue; // Skip higher-loaded instances
}
```

#### CacheMissForwardingLB (Lines 4757+)
**Purpose**: Routes requests for model placement (cache misses)
- Uses `PLACEMENT_ORDER` comparator for instance ranking
- Applies type constraints filtering
- Implements tolerance-based candidate selection

## Cache Hit vs Cache Miss Handling

### Cache Hit Logic (Lines 3594-3760)

**Main Loop** (MAX_ITERATIONS = 12):
```java
for (int n = 0; n < MAX_ITERATIONS; n++) {
    int filteredCount = filteredInstances.size();
    if (filteredCount > 0) {
        // Model loaded somewhere - cache hit logic
        
        Long localLoaded = filteredInstances.get(instanceId);
        boolean goLocal = false;
        
        if (localLoaded != null) {
            goLocal = filteredCount == 1; // Only copy
            if (!goLocal && favourSelfForHits) {
                // Check if local copy is ready or loading
                cacheEntry = getFromCache(modelId, lastUsedTime);
                if (cacheEntry != null) {
                    if (cacheEntry.isDone()) {
                        goLocal = true; // Loaded locally
                    } else {
                        // Loading locally - check timing
                        long oldest = oldest(filteredInstances);
                        if (oldest == localLoaded || age(oldest) < 1500L) {
                            goLocal = true; // We're oldest or recent enough
                        }
                    }
                }
            }
        }
    }
}
```

**Cache Hit Decision Factors:**
1. **Single copy**: Always use local if only one copy exists
2. **Local preference**: Use local copy if loaded or loading with good timing
3. **Load balancing**: Forward to least loaded instance otherwise

### Cache Miss Logic (Lines 3760-3850)

```java
// Global cache miss! 
if (externalReq) {
    // Create cache miss exclude set
    loadTargetFilter = new CacheMissExcludeSet(sourceIsBalanced, mr.getType());
    cacheMissExcludeTl.set(loadTargetFilter);
    
    // Add exclusions and context
    updateCacheMissExcludeSet(loadTargetFilter, cacheMissExcludeString, 
                             explicitExcludes, mr, lastUsedTime);
    contextMap = addFilterMapToContext(contextMap, filtered);
    contextMap.put(TAS_INTERNAL_CXT_KEY, LOAD_LOCAL_ONLY);
    
    // Try to place model on suitable instance
    Object result = invokeRemote(cacheMissClient, method, remoteMeth, modelId, args);
}
```

**Cache Miss Process:**
1. **Failure checks**: Verify load failure limits not exceeded
2. **Constraint verification**: Check type constraints before loading
3. **Exclude set creation**: Build list of unsuitable instances
4. **Remote invocation**: Use CacheMissForwardingLB for placement

## Model Resolution and Forwarding

### Direct Forwarding (Lines 3465-3480)

```java
String destId = contextMap.get(DEST_INST_ID_KEY);
if (destId != null) {
    if (!instanceId.equals(destId)) {
        if (directClient != null) {
            logger.info("Forwarding mis-directed request for model " + modelId + " to inst " + destId);
            return forwardInvokeModel(destId, modelId, remoteMeth, args);
        }
    }
}
```

**Purpose**: Handle requests with explicit destination instance ID
- Used for cross-cluster routing
- Handles mis-directed requests within cluster

### forwardInvokeModel Implementation (Lines 4130-4150)

```java
protected Object forwardInvokeModel(String destId, String modelId, Method remoteMeth, Object... args) {
    destinationInstance.set(destId);
    try {
        return remoteMeth.invoke(directClient, ObjectArrays.concat(modelId, args));
    } catch (Exception e) {
        // Handle forwarding failures
        if (t.getCause() instanceof ServiceUnavailableException) {
            throw new ModelNotHereException(destId, modelId);
        }
    } finally {
        destinationInstance.remove();
    }
}
```

## Local-Only Processing (Lines 3502-3519)

```java
if (local) { // HIT_ONLY flag set
    CacheEntry<?> ce = getFromCache(modelId, lastUsedTime);
    if (ce == null) {
        throw new ModelNotHereException(instanceId, modelId);
    }
    try {
        return invokeLocalModel(ce, method, args, vModelId);
    } catch (ModelLoadException mle) {
        // Handle local invocation failure
    }
}
```

**Purpose**: Direct local model execution without routing
- Used for cache hit confirmations
- Throws `ModelNotHereException` if model not available locally

## Exclusion and Filtering System

### Cache Hit Excludes (Lines 3561-3570)
- **Explicit excludes**: From method parameters
- **Cache hit exclude string**: Comma-separated instance IDs
- **Accumulated excludes**: Failed attempts in current request

### Cache Miss Excludes (Lines 3779-3787)
- **Model record failures**: Instances with recent load failures
- **Explicit excludes**: Passed in parameters  
- **Cache hit excludes**: Converted from previous attempts

## Error Handling and Retry Logic

### Failure Types Tracked:
- **ModelLoadException**: Model loading failures
- **ModelNotHereException**: Model not available on instance
- **InternalException**: System/configuration errors
- **Resource exhausted**: Capacity/memory limits

### Retry Mechanism (Lines 3595, 3834):
- **MAX_ITERATIONS**: 12 total attempts per request
- **Exclusion accumulation**: Failed instances excluded from subsequent attempts
- **Resource exhaustion limit**: MAX_RES_EXHAUSTED = 3

## Key Routing Decisions Summary

1. **Local vs Remote**: Based on source balancing and model availability
2. **Cache Hit Routing**: Prefer least loaded instance with model
3. **Cache Miss Placement**: Use constraint-aware placement algorithm  
4. **Failure Handling**: Comprehensive retry with exclusion tracking
5. **Direct Forwarding**: Support for explicit destination routing

The routing strategy balances performance (minimize hops), load distribution (avoid hotspots), and reliability (handle failures gracefully) while respecting deployment constraints and capacity limitations.