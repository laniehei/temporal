# Proposal: Enhanced DeepHealthCheck Diagnostics

## Summary

Extend the `DeepHealthCheck` API to return diagnostic details about **which service** is unhealthy and **why**, while preserving the existing `HealthState` enum for backward compatibility.

## Motivation

Currently, when fault detection in saas-control-plane triggers a cell failover, we only know that the cell is unhealthy—not which component failed or why. This makes debugging difficult and limits our ability to:

1. Surface actionable information in alerts
2. Track failure patterns over time
3. Make smarter failover decisions (e.g., soft-reject vs hard failover)

The internal health signals (RPC latency, persistence error ratio, etc.) exist but aren't exposed in the API response.

## Current Flow (Before)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        CURRENT: Fault Detection Flow                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  saas-control-plane                                                              │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ MonitorHealthStatus Activity                                               │ │
│  │                                                                            │ │
│  │   every 10s:                                                               │ │
│  │     resp = adminClient.DeepHealthCheck()                                   │ │
│  │                           │                                                │ │
│  │                           ▼                                                │ │
│  │     if resp.State == NOT_SERVING || DECLINED_SERVING:                      │ │
│  │         consecutiveFailureCount++                                          │ │
│  │                                                                            │ │
│  │     if consecutiveFailureCount > threshold:                                │ │
│  │         return HealthStatusOutage  ───────────────────┐                    │ │
│  │                                                       │                    │ │
│  └───────────────────────────────────────────────────────│────────────────────┘ │
│                                                          │                      │
│                                                          ▼                      │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ AutoFailoverCluster Workflow                                               │ │
│  │                                                                            │ │
│  │   Log: "cellID unhealthy, triggerReason: DeepHealthCheck unhealthy         │ │
│  │         (NOT_SERVING) count (3) exceeded threshold (3)"                    │ │
│  │                                     │                                      │ │
│  │   ❌ No info about:                 │                                      │ │
│  │      • Which service failed         │                                      │ │
│  │      • Why it failed                ▼                                      │ │
│  │      • Actual metrics          Trigger Failover                            │ │
│  │                                                                            │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  temporal server                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ Frontend: AdminHandler.DeepHealthCheck()                                   │ │
│  │                                                                            │ │
│  │   hosts = getAllHistoryHosts()                                             │ │
│  │   for each host:                                                           │ │
│  │       state = historyClient.DeepHealthCheck(host)  ────┐                   │ │
│  │                                                        │                   │ │
│  │   aggregate states into single HealthState             │                   │ │
│  │                           │                            │                   │ │
│  │                           ▼                            │                   │ │
│  │   return DeepHealthCheckResponse{                      │                   │ │
│  │       State: SERVING | NOT_SERVING | DECLINED_SERVING  │                   │ │
│  │   }                                                    │                   │ │
│  │   ❌ Per-host details discarded                        │                   │ │
│  │   ❌ Failure reasons discarded                         │                   │ │
│  │                                                        │                   │ │
│  └────────────────────────────────────────────────────────│───────────────────┘ │
│                                                           │                     │
│  ┌────────────────────────────────────────────────────────│───────────────────┐ │
│  │ History: Handler.DeepHealthCheck()                     ▼                   │ │
│  │                                                                            │ │
│  │   // Check 1: gRPC health (graceful shutdown)                              │ │
│  │   if grpcHealth != SERVING:                                                │ │
│  │       return DECLINED_SERVING  ❌ reason lost                              │ │
│  │                                                                            │ │
│  │   // Check 2: RPC latency                                                  │ │
│  │   if rpcLatency > threshold:                                               │ │
│  │       return NOT_SERVING  ❌ latency value lost                            │ │
│  │                                                                            │ │
│  │   // Check 3: RPC error ratio                                              │ │
│  │   if rpcErrorRatio > threshold:                                            │ │
│  │       return NOT_SERVING  ❌ error ratio lost                              │ │
│  │                                                                            │ │
│  │   // Check 4: Persistence latency                                          │ │
│  │   if persistenceLatency > threshold:                                       │ │
│  │       return NOT_SERVING  ❌ latency value lost                            │ │
│  │                                                                            │ │
│  │   // Check 5: Persistence error ratio                                      │ │
│  │   if persistenceErrorRatio > threshold:                                    │ │
│  │       return NOT_SERVING  ❌ error ratio lost                              │ │
│  │                                                                            │ │
│  │   return SERVING                                                           │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## Proposed Flow (After)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       PROPOSED: Fault Detection Flow                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  saas-control-plane                                                              │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ MonitorHealthStatus Activity                                               │ │
│  │                                                                            │ │
│  │   every 10s:                                                               │ │
│  │     resp = adminClient.DeepHealthCheck()                                   │ │
│  │                           │                                                │ │
│  │                           ▼                                                │ │
│  │     if resp.State == NOT_SERVING || DECLINED_SERVING:                      │ │
│  │         consecutiveFailureCount++                                          │ │
│  │         ✅ Store resp.ServiceDetails for diagnostics                       │ │
│  │                                                                            │ │
│  │     if consecutiveFailureCount > threshold:                                │ │
│  │         return HealthStatusOutage + FailureDetails{                        │ │
│  │             ServiceDetails: resp.ServiceDetails  ◄── NEW                   │ │
│  │         }                                                                  │ │
│  │                                                       │                    │ │
│  └───────────────────────────────────────────────────────│────────────────────┘ │
│                                                          │                      │
│                                                          ▼                      │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ AutoFailoverCluster Workflow                                               │ │
│  │                                                                            │ │
│  │   ✅ Log: "cellID unhealthy"                                               │ │
│  │      • Service: history                                                    │ │
│  │      • Failed hosts: [host1, host3]                                        │ │
│  │      • Checks failed:                                                      │ │
│  │        - PERSISTENCE_LATENCY: 850ms (threshold: 500ms)                     │ │
│  │        - RPC_ERROR_RATIO: 0.15 (threshold: 0.10)                           │ │
│  │                                     │                                      │ │
│  │   ✅ Build HealthReport with:       │                                      │ │
│  │      • Checks[]: detailed failures  ▼                                      │ │
│  │      • FailoverAction: hard    Trigger Failover                            │ │
│  │                                                                            │ │
│  │   ✅ Persist to Cell.Health.HealthReport                                   │ │
│  │                                                                            │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  temporal server                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ Frontend: AdminHandler.DeepHealthCheck()                                   │ │
│  │                                                                            │ │
│  │   hosts = getAllHistoryHosts()                                             │ │
│  │   hostDetails = []                                                         │ │
│  │   for each host:                                                           │ │
│  │       resp = historyClient.DeepHealthCheck(host)                           │ │
│  │       hostDetails.append(resp)  ◄── NEW: collect details                   │ │
│  │                                                                            │ │
│  │   aggregateState = aggregate(hostDetails)                                  │ │
│  │                           │                                                │ │
│  │                           ▼                                                │ │
│  │   return DeepHealthCheckResponse{                                          │ │
│  │       State: aggregateState,           // backward compatible              │ │
│  │       ServiceDetails: [{               // NEW: optional diagnostics        │ │
│  │           Service: "history",                                              │ │
│  │           State: NOT_SERVING,                                              │ │
│  │           HostDetails: [{                                                  │ │
│  │               Address: "host1:7234",                                       │ │
│  │               State: NOT_SERVING,                                          │ │
│  │               Checks: [{                                                   │ │
│  │                   Type: PERSISTENCE_LATENCY,                               │ │
│  │                   State: NOT_SERVING,                                      │ │
│  │                   Value: 850.0,                                            │ │
│  │                   Threshold: 500.0                                         │ │
│  │               }]                                                           │ │
│  │           }]                                                               │ │
│  │       }]                                                                   │ │
│  │   }                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ History: Handler.DeepHealthCheck()                                         │ │
│  │                                                                            │ │
│  │   checks = []                                                              │ │
│  │   overallState = SERVING                                                   │ │
│  │                                                                            │ │
│  │   // Check 1: gRPC health (graceful shutdown)                              │ │
│  │   checks.append({Type: GRPC_HEALTH, State: grpcState})                     │ │
│  │   if grpcState != SERVING: overallState = DECLINED_SERVING                 │ │
│  │                                                                            │ │
│  │   // Check 2: RPC latency                                                  │ │
│  │   latency = rpcHealthSignal.AverageLatency()                               │ │
│  │   checks.append({Type: RPC_LATENCY, Value: latency, Threshold: t})         │ │
│  │   if latency > threshold: overallState = NOT_SERVING                       │ │
│  │                                                                            │ │
│  │   // Check 3: RPC error ratio                                              │ │
│  │   errRatio = rpcHealthSignal.ErrorRatio()                                  │ │
│  │   checks.append({Type: RPC_ERROR_RATIO, Value: errRatio, Threshold: t})    │ │
│  │   if errRatio > threshold: overallState = NOT_SERVING                      │ │
│  │                                                                            │ │
│  │   // Check 4: Persistence latency                                          │ │
│  │   pLatency = persistenceSignal.AverageLatency()                            │ │
│  │   checks.append({Type: PERSISTENCE_LATENCY, Value: pLatency, Threshold: t})│ │
│  │   if pLatency > threshold: overallState = NOT_SERVING                      │ │
│  │                                                                            │ │
│  │   // Check 5: Persistence error ratio                                      │ │
│  │   pErrRatio = persistenceSignal.ErrorRatio()                               │ │
│  │   checks.append({Type: PERSISTENCE_ERROR_RATIO, Value: pErrRatio, ...})    │ │
│  │   if pErrRatio > threshold: overallState = NOT_SERVING                     │ │
│  │                                                                            │ │
│  │   return DeepHealthCheckResponse{                                          │ │
│  │       State: overallState,                                                 │ │
│  │       Checks: checks  ◄── NEW: all check results with values               │ │
│  │   }                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## Proposed Proto Changes

### New Enum: HealthCheckType

```protobuf
// Generic health check types that can be used across services
enum HealthCheckType {
    HEALTH_CHECK_TYPE_UNSPECIFIED = 0;

    // Common (all services)
    HEALTH_CHECK_TYPE_GRPC_HEALTH = 1;        // gRPC health server status
    HEALTH_CHECK_TYPE_RPC_LATENCY = 2;        // RPC latency threshold
    HEALTH_CHECK_TYPE_RPC_ERROR_RATIO = 3;    // RPC error ratio threshold

    // History-specific (10-19)
    HEALTH_CHECK_TYPE_PERSISTENCE_LATENCY = 10;      // DB latency threshold
    HEALTH_CHECK_TYPE_PERSISTENCE_ERROR_RATIO = 11;  // DB error ratio threshold

    // Frontend-specific (20-29)
    HEALTH_CHECK_TYPE_HISTORY_HOST_AVAILABILITY = 20;   // % of history hosts healthy
    HEALTH_CHECK_TYPE_MATCHING_HOST_AVAILABILITY = 21;  // % of matching hosts healthy

    // Matching-specific (30-39)
    HEALTH_CHECK_TYPE_TASK_QUEUE_BACKLOG = 30;  // Task queue depth
    // Reserved for future matching checks
}
```

### New Messages

```protobuf
// Individual health check result
message HealthCheck {
    HealthCheckType type = 1;
    HealthState state = 2;       // Result of this specific check
    double value = 3;            // Actual observed value (optional)
    double threshold = 4;        // Threshold that was exceeded (optional)
    string message = 5;          // Human-readable detail (optional)
}

// Health details for a single host
message HostHealthDetail {
    string address = 1;
    HealthState state = 2;
    repeated HealthCheck checks = 3;
}

// Health details for a service (history, frontend, matching)
message ServiceHealthDetail {
    string service = 1;                    // "history", "frontend", "matching"
    HealthState state = 2;                 // Aggregated state for this service
    repeated HostHealthDetail hosts = 3;   // Per-host details
}
```

### Updated AdminService Response

```protobuf
message DeepHealthCheckResponse {
    HealthState state = 1;  // Unchanged - backward compatible

    // NEW: Optional diagnostic details
    repeated ServiceHealthDetail services = 2;
}
```

### Updated HistoryService Response

```protobuf
message DeepHealthCheckResponse {
    HealthState state = 1;  // Unchanged - backward compatible

    // NEW: Individual check results
    repeated HealthCheck checks = 2;
}
```

## Benefits

1. **Backward Compatible**: Existing clients continue to work unchanged
2. **Actionable Alerts**: Know exactly which service/host/check failed
3. **Debugging**: Actual values vs thresholds visible in logs and dashboards
4. **Extensible**: Easy to add checks for frontend/matching without proto changes
5. **HealthReport Integration**: Maps cleanly to saas-control-plane's `HealthReport` model

## Implementation Plan

### Phase 1: History Service
1. Update `DeepHealthCheckResponse` proto with `checks` field
2. Modify `Handler.DeepHealthCheck()` to populate check results
3. No change to aggregation logic at frontend (yet)

### Phase 2: Frontend Aggregation
1. Update `AdminService.DeepHealthCheckResponse` proto with `services` field
2. Modify `healthCheckerImpl.Check()` to collect and expose host details
3. Include service-level aggregation

### Phase 3: Frontend/Matching Services
1. Add health checking for frontend service
2. Add health checking for matching service
3. Use same `HealthCheckType` enum with service-specific values

## Related Work

- saas-control-plane PR #12203: `HealthReport` struct for persisting diagnostics
- saas-control-plane PR #11952: Hysteresis for fault detection counter reset
- JIRA CGSCE-514: Incident Autodetection Epic
