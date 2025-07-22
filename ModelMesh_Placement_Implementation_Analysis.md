# ModelMesh Model Placement Implementation Analysis

*Based exclusively on current codebase examination*

## Core Placement Algorithm (PLACEMENT_ORDER Comparator)

**File**: `src/main/java/com/ibm/watson/modelmesh/ModelMesh.java`  
**Lines**: 4646-4703

The primary placement algorithm uses a `Comparator<Entry<String, InstanceRecord>>` with the following **exact priority order**:

```java
// Line 4646: PLACEMENT_ORDER comparator implementation
1. isShuttingDown() - Prefer non-shutting-down instances
2. Instance version comparison with saturation check  
3. isFull() status - Prefer non-full instances
4. For full instances: prefer one with least recently used models
5. Model count - Prefer instances with fewer models
6. Remaining capacity - Prefer instances with more space
7. For non-full instances: prefer one with least recently used models
8. Additional tie-breakers (loading threads, capacity, request rate, instance ID, location, zone, labels)
```

## Full Instance Definition

**File**: `src/main/java/com/ibm/watson/modelmesh/ModelMesh.java`  
**Lines**: 4640-4642

```java
protected boolean isFull(long availableUnits) {
    return availableUnits < minSpaceUnits;
}
```
- Instance is "full" when remaining capacity < `minSpaceUnits`
- `minSpaceUnits` is a configurable threshold value

## Load Balancing Implementation (CacheMissForwardingLB)

**File**: `src/main/java/com/ibm/watson/modelmesh/ModelMesh.java`  
**Lines**: 4757-4950+

### Selection Process:
1. **Filter candidates** using constraints and exclusions (line 4793)
2. **Get best instance** from sorted cluster state (line 4806)
3. **Build candidate shortlist** within "distance" tolerance from best (lines 4814+)
4. **Apply preferences** for specific model types (lines 4817-4849)
5. **Random selection** from qualified candidates (implementation continues beyond line 4950)

### Constraint Integration:
```java
// Lines 4789-4790: Type constraint filtering
final Set<String> constrainTo = typeConstraints != null ?
    typeConstraints.getCandidateInstances(exclude.modelType) : null;

// Lines 4817-4818: Preferred instance handling  
Set<String> prefer = typeConstraints != null ?
    typeConstraints.getPreferredInstances(exclude.modelType) : null;
```

## Type Constraints Implementation

**File**: `src/main/java/com/ibm/watson/modelmesh/TypeConstraintManager.java`  
**Lines**: 242-262

```java
// Line 242: Get allowed instances for model type
Set<String> getCandidateInstances(String type) {
    ModelTypeConstraints mtc = getTypeConstraints(type);
    return mtc != null ? mtc.allowedInstances : null; // null means "all"
}

// Line 248: Get preferred instances  
Set<String> getPreferredInstances(String type) {
    ModelTypeConstraints mtc = getTypeConstraints(type);
    return mtc != null ? mtc.preferredInstances : defaultPreferredInstances;
}

// Line 253: Get required labels
String[] getRequiredLabels(String type) {
    ModelTypeConstraints mtc = getTypeConstraints(type);
    return mtc != null ? mtc.requiredLabels : null;
}
```

## Version Handling in Placement

**File**: `src/main/java/com/ibm/watson/modelmesh/ModelMesh.java`  
**Lines**: 4660-4666

```java
if (vers1 != vers2) {
    // prefer newer version *unless* it's saturated
    if (vers1 > vers2) {
        if (!full1 || ir1.getLruTime() > minChurnAgeMs * 2) return -1;
    } else /* vers1 < vers2 */
        if (!full2 || ir2.getLruTime() > minChurnAgeMs * 2) return 1;
}
```
- Newer versions preferred unless they're saturated
- Saturation check: full AND LRU time > `minChurnAgeMs * 2`

## Capacity-Based Logic

**File**: `src/main/java/com/ibm/watson/modelmesh/ModelMesh.java`  

### Full vs Non-Full Handling:
```java
// Lines 4668-4669: Prefer non-full instances
if (full1 ^ full2) return full1 ? 1 : -1;

// Lines 4670-4674: For full instances, use LRU time
if (full1) {
    int oldestDiff = Long.compare(ir1.getLruTime(), ir2.getLruTime());
    if (oldestDiff != 0) return oldestDiff;
}

// Lines 4678-4680: For non-full, prefer more remaining space
int remDiff = Long.compare(rem2, rem1);
if (remDiff != 0) return remDiff;
```

### Model Count Priority:
```java
// Lines 4675-4677: Prefer fewer models for load distribution
int countDiff = ir1.getCount() - ir2.getCount();
if (countDiff != 0) return countDiff;
```

## Exclusion and Filtering

**File**: `src/main/java/com/ibm/watson/modelmesh/ModelMesh.java`  
**Lines**: 4763-4771

```java
return Iterators.filter(clusterState.iterator(), ent -> {
    final String iid = ent.getKey();
    if (constrainTo != null && !constrainTo.contains(iid)
        || exclude.isExcluded(iid) || !siMap.containsKey(iid)) {
        return false;
    }
    return excludeReplicaSets.isEmpty() || iid.length() < 7
           || !excludeReplicaSets.containsKey(iid.substring(0, 6));
});
```

### Exclusion Types:
- **Type constraints**: `constrainTo` set limitations
- **Explicit exclusions**: `exclude.isExcluded(iid)` 
- **Service map**: Must exist in `siMap`
- **Replica sets**: Exclude likely-replaced replica sets

## Tolerance-Based Selection

**File**: `src/main/java/com/ibm/watson/modelmesh/ModelMesh.java`  
**Comments**: Lines 4783-4786

```java
// Iterate over the cluster instances which are in order of most to least desirable
// in terms of placement location (see PLACEMENT_ORDER comparator).
// Break once we reach an appropriate "distance" from the most desirable instance.
// Those "seen" form a shortlist for the subsequent random choice.
```
- Creates shortlist of instances within tolerance "distance" from best
- Random selection from shortlist to avoid hotspots
- *Note: Exact tolerance calculation implementation extends beyond examined lines*

## Key Implementation Notes

1. **Ordering**: `clusterState` is maintained in `PLACEMENT_ORDER` for efficiency
2. **Thread Safety**: Uses `ThreadLocal<CacheMissExcludeSet>` for per-request state
3. **Fallback Logic**: If no candidates found with replica set exclusions, retries without them (lines 4798-4804)
4. **Instance Freshness**: Uses `getFreshInstanceRecord()` for local instance details (line 4810)

## Limitations of Analysis

*The following aspects require additional code examination beyond the scope reviewed:*
- Complete tolerance calculation logic for candidate shortlisting
- Exact random selection implementation from candidate pool  
- Full retry and failure handling mechanisms
- Detailed preference handling for non-simple cases
- Integration with leader election for cluster-wide operations