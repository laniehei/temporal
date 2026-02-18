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
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         CURRENT: Fault Detection Flow                             │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│  saas-control-plane                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │ Activities.MonitorHealthStatus()                                             │  │
│  │ [internal/workflows/temporal/activities.go:1577]                             │  │
│  │                                                                              │  │
│  │   every CheckIntervalInSeconds (default 10s):                               │  │
│  │     resp, err = adminClient.DeepHealthCheck(ctx, &DeepHealthCheckRequest{})  │  │
│  │     [activities.go:1617]                                                     │  │
│  │                           │                                                  │  │
│  │                           ▼                                                  │  │
│  │     if err != nil:                                                           │  │
│  │       if isConnectionFailure(err): serverConnectionErrorCount++             │  │
│  │       [activities.go:1627-1628]                                              │  │
│  │                                                                              │  │
│  │     if resp.State == NOT_SERVING || DECLINED_SERVING:                       │  │
│  │       consecutiveFailureCount++  [activities.go:1637]                       │  │
│  │     else if resp.State == SERVING:                                           │  │
│  │       consecutiveFailureCount = 0; serverConnectionErrorCount = 0           │  │
│  │                                                                              │  │
│  │     if input.CheckCanaryConnection:                                         │  │
│  │       err = a.describeCanaryNamespace(clusterID, region, logger)            │  │
│  │       [activities.go:1651]                                                   │  │
│  │       if errCanaryConnection: canaryConnectionErrorCount++                  │  │
│  │                                                                              │  │
│  │     if consecutiveFailureCount > MaxFailureCount                            │  │
│  │        || serverConnectionErrorCount > MaxConnectionFailureCount            │  │
│  │        || canaryConnectionErrorCount > MaxFailureCount:                     │  │
│  │       return MonitorHealthStatusOutput{                                      │  │
│  │         HealthStatus: HealthStatusOutage,                                    │  │
│  │         FailureDetails: &MonitorHealthStatusFailureDetails{...}             │  │
│  │       }  ────────────────────────────────────────────┐                      │  │
│  └──────────────────────────────────────────────────────│──────────────────────┘  │
│                                                          │                        │
│                                                          ▼                        │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │ Workflows.AutoFailoverCluster()                                             │  │
│  │ [internal/workflows/xdc/fault_detection.go:137]                             │  │
│  │                                                                              │  │
│  │   Log: "cellID unhealthy, triggerReason: DeepHealthCheck unhealthy          │  │
│  │         (NOT_SERVING) count (3) exceeded threshold (3)"                     │  │
│  │                                     │                                        │  │
│  │   ❌ No info about:                 │                                        │  │
│  │      • Which service failed         │                                        │  │
│  │      • Why it failed                ▼                                        │  │
│  │      • Actual metrics          Trigger Failover                              │  │
│  │                                                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                   │
├───────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│  temporal server                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │ AdminHandler.DeepHealthCheck()                                              │  │
│  │ [service/frontend/admin_handler.go:260]                                     │  │
│  │                                                                              │  │
│  │   healthStatus, err = adh.historyHealthChecker.Check(ctx)                   │  │
│  │   [admin_handler.go:266]                                                     │  │
│  │                           │                                                  │  │
│  │                           ▼                                                  │  │
│  │   return &DeepHealthCheckResponse{State: healthStatus}                      │  │
│  │   [admin_handler.go:270]                                                     │  │
│  │   ❌ Per-host details discarded                                              │  │
│  │   ❌ Failure reasons discarded                                               │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                           │                                                       │
│                           ▼                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │ healthCheckerImpl.Check()                                                   │  │
│  │ [service/frontend/health_check.go:48]                                       │  │
│  │                                                                              │  │
│  │   resolver, err = h.membershipMonitor.GetResolver(h.serviceName)            │  │
│  │   [health_check.go:49]                                                       │  │
│  │                                                                              │  │
│  │   hosts = resolver.AvailableMembers()                                       │  │
│  │   [health_check.go:54]                                                       │  │
│  │                                                                              │  │
│  │   for each host (in parallel goroutines):                                   │  │
│  │     resp, err = h.healthCheckFn(ctx, host.GetAddress())                     │  │
│  │     [health_check.go:62]                                                     │  │
│  │     └── healthCheckFn is a closure set in NewAdminHandler():                │  │
│  │         args.HistoryClient.DeepHealthCheck(ctx,                              │  │
│  │           &DeepHealthCheckRequest{HostAddress: hostAddress})                 │  │
│  │         [admin_handler.go:178]                                               │  │
│  │                           │                                                  │  │
│  │                           ▼                                                  │  │
│  │   Aggregate results:     [health_check.go:73-101]                           │  │
│  │     failedHostCount / totalHosts + declinedCount / totalHosts               │  │
│  │       > hostFailurePercentage (0.50)? → NOT_SERVING                         │  │
│  │     declinedCount / totalHosts                                              │  │
│  │       > ensureMinimumProportionOfHosts(0.05, totalHosts)? → DECLINED        │  │
│  │     otherwise → SERVING                                                     │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                           │                                                       │
│                           ▼                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │ Handler.DeepHealthCheck()  (per history host)                               │  │
│  │ [service/history/handler.go:197]                                            │  │
│  │                                                                              │  │
│  │   // Check 1: gRPC health (graceful shutdown / hysteresis)                  │  │
│  │   status, err = h.healthServer.Check(ctx, &HealthCheckRequest{...})         │  │
│  │   [handler.go:202]                                                           │  │
│  │   if status.Status != SERVING:                                              │  │
│  │     return DECLINED_SERVING  ❌ reason lost                                 │  │
│  │                                                                              │  │
│  │   // Checks 2-3: History RPC health signals                                 │  │
│  │   rsp = h.checkHistoryHealthSignals()                                       │  │
│  │   [handler.go:211]                                                           │  │
│  │                                                                              │  │
│  │     // Check 2: h.historyHealthSignal.AverageLatency()                      │  │
│  │     //   > h.config.HealthRPCLatencyFailure() (500ms)                       │  │
│  │     //   [handler.go:230]                                                    │  │
│  │     //   → NOT_SERVING  ❌ latency value lost                               │  │
│  │                                                                              │  │
│  │     // Check 3: h.historyHealthSignal.ErrorRatio()                          │  │
│  │     //   > h.config.HealthRPCErrorRatio() (0.90)                            │  │
│  │     //   [handler.go:236]                                                    │  │
│  │     //   → NOT_SERVING  ❌ error ratio lost                                 │  │
│  │                                                                              │  │
│  │   // Check 4: Persistence latency                                           │  │
│  │   h.persistenceHealthSignal.AverageLatency()                                │  │
│  │     > h.config.HealthPersistenceLatencyFailure() (500ms)                    │  │
│  │   [handler.go:216-219]                                                       │  │
│  │     → NOT_SERVING  ❌ latency value lost                                    │  │
│  │                                                                              │  │
│  │   // Check 5: Persistence error ratio                                       │  │
│  │   h.persistenceHealthSignal.ErrorRatio()                                    │  │
│  │     > h.config.HealthPersistenceErrorRatio() (0.90)                         │  │
│  │   [handler.go:219]                                                           │  │
│  │     → NOT_SERVING  ❌ error ratio lost                                      │  │
│  │                                                                              │  │
│  │   return SERVING  [handler.go:224]                                          │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                   │
└───────────────────────────────────────────────────────────────────────────────────┘
```

## Proposed Flow (After)

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                        PROPOSED: Fault Detection Flow                              │
├───────────────────────────────────────────────────────────────────────────────────┤
│                                                                                    │
│  saas-control-plane                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ Activities.MonitorHealthStatus()                                              │  │
│  │ [internal/workflows/temporal/activities.go:1577]                              │  │
│  │                                                                               │  │
│  │   every CheckIntervalInSeconds (default 10s):                                │  │
│  │     resp, err = adminClient.DeepHealthCheck(ctx, &DeepHealthCheckRequest{})   │  │
│  │     [activities.go:1617]                                                      │  │
│  │                           │                                                   │  │
│  │                           ▼                                                   │  │
│  │     if resp.State == NOT_SERVING || DECLINED_SERVING:                        │  │
│  │       consecutiveFailureCount++                                              │  │
│  │       ✅ Store resp.Services for diagnostics  ◄── NEW                        │  │
│  │                                                                               │  │
│  │     if consecutiveFailureCount > MaxFailureCount:                            │  │
│  │       return MonitorHealthStatusOutput{                                       │  │
│  │         HealthStatus: HealthStatusOutage,                                     │  │
│  │         FailureDetails: &MonitorHealthStatusFailureDetails{                   │  │
│  │           ...,                                                                │  │
│  │           ServiceDetails: resp.Services  ◄── NEW                              │  │
│  │         }                                                                     │  │
│  │       }  ────────────────────────────────────────────┐                       │  │
│  └──────────────────────────────────────────────────────│───────────────────────┘  │
│                                                          │                         │
│                                                          ▼                         │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ Workflows.AutoFailoverCluster()                                              │  │
│  │ [internal/workflows/xdc/fault_detection.go:137]                              │  │
│  │                                                                               │  │
│  │   buildHealthReportFromFailureDetails(healthStatus.FailureDetails, now)       │  │
│  │   [fault_detection.go:264]                                                    │  │
│  │                                                                               │  │
│  │   ✅ Log: "cellID unhealthy"                                                 │  │
│  │      • Service: history                                                       │  │
│  │      • Failed hosts: [host1, host3]                                           │  │
│  │      • Checks failed:                                                         │  │
│  │        - PERSISTENCE_LATENCY: 850ms (threshold: 500ms)                        │  │
│  │        - RPC_ERROR_RATIO: 0.15 (threshold: 0.10)                              │  │
│  │                                     │                                         │  │
│  │   ✅ CreateCellHealthEvent(...)     │                                         │  │
│  │   [fault_detection.go:267-281]      ▼                                         │  │
│  │                                                                               │  │
│  │   ✅ w.invokeCellEntity(UpdateCellHealthRequest{HealthReport: ...})           │  │
│  │   [fault_detection.go:284-296]                                                │  │
│  │                                                                               │  │
│  │   ✅ Persist to Cell.Health.HealthReport                                      │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
├────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                    │
│  temporal server                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ AdminHandler.DeepHealthCheck()                                               │  │
│  │ [service/frontend/admin_handler.go:260]                                      │  │
│  │                                                                               │  │
│  │   healthStatus, err = adh.historyHealthChecker.Check(ctx)                    │  │
│  │   [admin_handler.go:266]                                                      │  │
│  │                           │                                                   │  │
│  │   ◄── CHANGE: also collect ServiceHealthDetail[] from Check()                │  │
│  │                           │                                                   │  │
│  │                           ▼                                                   │  │
│  │   return &DeepHealthCheckResponse{                                           │  │
│  │     State: healthStatus,                    // backward compatible            │  │
│  │     Services: [{                            // NEW: optional diagnostics      │  │
│  │       Service: "history",                                                     │  │
│  │       State: NOT_SERVING,                                                     │  │
│  │       Hosts: [{                                                               │  │
│  │         Address: "host1:7234",                                                │  │
│  │         State: NOT_SERVING,                                                   │  │
│  │         Checks: [{                                                            │  │
│  │           Type: PERSISTENCE_LATENCY,                                          │  │
│  │           State: NOT_SERVING,                                                 │  │
│  │           Value: 850.0,                                                       │  │
│  │           Threshold: 500.0                                                    │  │
│  │         }]                                                                    │  │
│  │       }]                                                                      │  │
│  │     }]                                                                        │  │
│  │   }                                                                           │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                           │                                                        │
│                           ▼                                                        │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ healthCheckerImpl.Check()                                                    │  │
│  │ [service/frontend/health_check.go:48]                                        │  │
│  │                                                                               │  │
│  │   resolver, err = h.membershipMonitor.GetResolver(h.serviceName)             │  │
│  │   [health_check.go:49]                                                        │  │
│  │                                                                               │  │
│  │   hosts = resolver.AvailableMembers()                                        │  │
│  │   [health_check.go:54]                                                        │  │
│  │                                                                               │  │
│  │   for each host (in parallel goroutines):                                    │  │
│  │     resp, err = h.healthCheckFn(ctx, host.GetAddress())                      │  │
│  │     [health_check.go:62]                                                      │  │
│  │     └── healthCheckFn is a closure set in NewAdminHandler():                 │  │
│  │         args.HistoryClient.DeepHealthCheck(ctx,                               │  │
│  │           &DeepHealthCheckRequest{HostAddress: hostAddress})                  │  │
│  │         [admin_handler.go:178]                                                │  │
│  │                           │                                                   │  │
│  │   ◄── CHANGE: collect resp into hostDetails[] instead of just counting       │  │
│  │                           │                                                   │  │
│  │                           ▼                                                   │  │
│  │   Aggregate + return ServiceHealthDetail with per-host breakdown             │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                           │                                                        │
│                           ▼                                                        │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ Handler.DeepHealthCheck()  (per history host)                                │  │
│  │ [service/history/handler.go:197]                                             │  │
│  │                                                                               │  │
│  │   checks = []                                                                │  │
│  │   overallState = SERVING                                                     │  │
│  │                                                                               │  │
│  │   // Check 1: gRPC health (graceful shutdown / hysteresis)                   │  │
│  │   status, err = h.healthServer.Check(ctx, &HealthCheckRequest{...})          │  │
│  │   [handler.go:202]                                                            │  │
│  │   checks.append({Type: GRPC_HEALTH, State: grpcState})                       │  │
│  │   if status.Status != SERVING: overallState = DECLINED_SERVING               │  │
│  │                                                                               │  │
│  │   // Checks 2-3: h.checkHistoryHealthSignals()  [handler.go:211]             │  │
│  │   // Check 2: h.historyHealthSignal.AverageLatency()  [handler.go:230]       │  │
│  │   latency = historyHealthSignal.AverageLatency()                             │  │
│  │   checks.append({Type: RPC_LATENCY, Value: latency, Threshold: t})           │  │
│  │   if latency > config.HealthRPCLatencyFailure(): overallState = NOT_SERVING  │  │
│  │                                                                               │  │
│  │   // Check 3: h.historyHealthSignal.ErrorRatio()  [handler.go:236]           │  │
│  │   errRatio = historyHealthSignal.ErrorRatio()                                │  │
│  │   checks.append({Type: RPC_ERROR_RATIO, Value: errRatio, Threshold: t})      │  │
│  │   if errRatio > config.HealthRPCErrorRatio(): overallState = NOT_SERVING     │  │
│  │                                                                               │  │
│  │   // Check 4: Persistence latency  [handler.go:216]                          │  │
│  │   pLatency = persistenceHealthSignal.AverageLatency()                        │  │
│  │   checks.append({Type: PERSISTENCE_LATENCY, Value: pLatency, Threshold: t})  │  │
│  │   if pLatency > config.HealthPersistenceLatencyFailure(): NOT_SERVING        │  │
│  │                                                                               │  │
│  │   // Check 5: Persistence error ratio  [handler.go:219]                      │  │
│  │   pErrRatio = persistenceHealthSignal.ErrorRatio()                           │  │
│  │   checks.append({Type: PERSISTENCE_ERROR_RATIO, Value: pErrRatio, ...})      │  │
│  │   if pErrRatio > config.HealthPersistenceErrorRatio(): NOT_SERVING           │  │
│  │                                                                               │  │
│  │   return DeepHealthCheckResponse{                                            │  │
│  │     State: overallState,                                                      │  │
│  │     Checks: checks  ◄── NEW: all check results with values                   │  │
│  │   }                                                                           │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
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
