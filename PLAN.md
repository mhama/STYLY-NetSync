# Plan: Fix Issue #249 - Unity Client Slow Joiner Problem

## Problem Summary

The Unity client sometimes fails to reach the "ready" state when the NetSync server starts after the client app launches. This is caused by the ZeroMQ "slow joiner" problem in the PUB-SUB pattern.

**Root Cause:**
1. Client connects and sends a handshake via DEALER socket
2. Server receives handshake, assigns client number, and broadcasts Device ID Mapping via PUB socket
3. The SUB socket subscription takes time to propagate across the network
4. If the broadcast happens before the subscription is active, the message is lost
5. Client never receives Device ID Mapping, so `_clientNo` stays 0 and `IsReady` never becomes true

**Key Observation:**
Looking at the server code (`server.py:840-841`), ID mappings are only broadcast when `is_new_client` is True. Subsequent transforms from the same client don't trigger re-broadcasts, so even with the client's periodic transform sends, the ID mapping won't be resent.

## Proposed Solution: Dual-Layer Fix

Implement both client-side and server-side fixes for robustness:

### 1. Server-Side: Periodic ID Mapping Re-broadcast (Primary Fix)

**Location:** `STYLY-NetSync-Server/src/styly_netsync/server.py`

**Changes:**
- Track when each client joined (`client_join_time`)
- In the `_periodic_loop()`, periodically re-broadcast ID mappings for rooms that have clients who joined within the last N seconds (e.g., 5 seconds)
- Re-broadcast interval: every 500ms for the first 5 seconds after a new client joins

**Implementation Details:**
```python
# New fields in NetSyncServer.__init__():
self.room_new_client_time: dict[str, float] = {}  # Tracks when last new client joined each room
self.ID_MAPPING_REBROADCAST_DURATION = 5.0  # Re-broadcast for 5 seconds after new client
self.ID_MAPPING_REBROADCAST_INTERVAL = 0.5  # Re-broadcast every 500ms

# In _handle_client_transform(), when is_new_client:
self.room_new_client_time[room_id] = current_time

# In _periodic_loop(), add new check:
# Re-broadcast ID mappings for rooms with recent new clients
for room_id, join_time in list(self.room_new_client_time.items()):
    if current_time - join_time < self.ID_MAPPING_REBROADCAST_DURATION:
        if current_time - last_id_rebroadcast >= self.ID_MAPPING_REBROADCAST_INTERVAL:
            self._broadcast_id_mappings(room_id)
    else:
        del self.room_new_client_time[room_id]  # Clean up
```

### 2. Client-Side: Handshake Retry with Timeout (Secondary Fix)

**Location:** `STYLY-NetSync-Unity/Packages/com.styly.styly-netsync/Runtime/NetSyncManager.cs`

**Changes:**
- Track when the connection was established (`_connectionEstablishedTime`)
- If `_clientNo` is still 0 after a timeout period (e.g., 3 seconds), trigger a warning log
- The existing periodic transform send will continue naturally, which combined with the server-side fix will resolve the issue

**Implementation Details:**
```csharp
// New fields:
private float _connectionEstablishedTime;
private const float HandshakeTimeoutWarning = 3f;
private bool _hasLoggedHandshakeTimeout = false;

// In OnConnectionEstablished():
_connectionEstablishedTime = Time.time;
_hasLoggedHandshakeTimeout = false;

// In Update(), add check:
if (HasServerConnection && !HasHandshake && !_hasLoggedHandshakeTimeout)
{
    if (Time.time - _connectionEstablishedTime > HandshakeTimeoutWarning)
    {
        Debug.LogWarning("[NetSyncManager] Handshake timeout - still waiting for Device ID Mapping. This may indicate the 'slow joiner' issue.");
        _hasLoggedHandshakeTimeout = true;
    }
}
```

## Alternative Approaches Considered

### A. Request-Response via DEALER Socket
- Add a new message type `MSG_REQUEST_ID_MAPPING`
- Client sends request via DEALER, server responds via ROUTER
- More reliable but requires more protocol changes
- **Decision:** Not implementing in this fix - the periodic re-broadcast is simpler and sufficient

### B. Client-Side Disconnect/Reconnect
- If no ID mapping received within timeout, disconnect and reconnect
- Adds complexity and potential disruption
- **Decision:** Not implementing - periodic re-broadcast should be sufficient

## Implementation Steps

### Step 1: Server-Side Changes
1. Add `room_new_client_time` tracking dict to `NetSyncServer.__init__()`
2. Add constants for re-broadcast duration and interval
3. Update `_handle_client_transform()` to record new client join time
4. Update `_periodic_loop()` to re-broadcast ID mappings for rooms with recent new clients
5. Add logging for re-broadcast events (debug level)

### Step 2: Client-Side Changes
1. Add `_connectionEstablishedTime` field to track connection time
2. Add handshake timeout warning logging
3. Reset timeout tracking on connection established and connection error

### Step 3: Testing
1. Run pytest on the Python server
2. Verify compilation in Unity Editor
3. Test scenario: Start client first, then server, verify client becomes ready
4. Test scenario: Normal startup (server first), verify no regression

### Step 4: Code Quality
1. Run `black src/ tests/` for Python formatting
2. Run `ruff check src/ tests/` for Python linting
3. Run `mypy src/` for Python type checking
4. Verify no Unity compilation errors

## Files to Modify

1. `STYLY-NetSync-Server/src/styly_netsync/server.py`
   - Add re-broadcast tracking and logic

2. `STYLY-NetSync-Unity/Packages/com.styly.styly-netsync/Runtime/NetSyncManager.cs`
   - Add handshake timeout warning logging

## Risks and Mitigations

1. **Risk:** Increased network traffic from re-broadcasts
   - **Mitigation:** Limited to 5 seconds after new client joins, 500ms intervals (max ~10 extra broadcasts)

2. **Risk:** Thread safety issues in server periodic loop
   - **Mitigation:** Use existing locking patterns for room data access

3. **Risk:** Client timeout warning might fire unnecessarily on slow networks
   - **Mitigation:** 3-second timeout is generous; warning is non-blocking

## Success Criteria

- Client successfully reaches "ready" state when server starts after client
- No regression in normal startup flow
- All existing tests pass
- No new warnings or errors in normal operation
