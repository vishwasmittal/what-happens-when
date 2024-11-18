# What happens when you interrupt a PSQL query?

This guide explains what happens when you interrupt a PostgreSQL query by pressing `Ctrl + C` in the psql shell or clicking the stop button in your database viewer tool.

> Note: Skip to the end for a quick overview.

## Introduction

When you interrupt a long-running query, PostgreSQL doesn't immediately terminate the connection. Instead, it implements a sophisticated cancellation mechanism that:
- Allows safe query interruption
- Ensures data integrity
- Preserves the main client-server connection

## Initial Connection Setup

When psql first connects to PostgreSQL, the server provides two crucial pieces of information:

```c
// Example
ConnStatusResponse {
    backend_pid: 12345,          // Example process ID
    cancel_secret_key: 0xABCD    // Random 32-bit key
}
```

This information is stored in the connection object `PGconn` along with other connection details:

```c
typedef struct {
    int32 pid;           // Backend process ID
    int32 cancel_key;    // Secret cancellation key
    ...
} PGconn;
```

## The Cancellation Process

### 1. Client-Side Handling (PSQL)

When you press `Ctrl + C`, psql first checks if the process is in a critical section before notifying the server.

> **Note**: A critical section is code that must execute atomically (without interruption) to maintain consistency and prevent race conditions. Examples include modifying shared data or performing operations that must either complete fully or not execute at all.

There are two possible scenarios:

```sh
# Scenario 1: Not in critical section
[Ctrl+C] -> [SIGINT Handler] -> [Sets cancel_pressed = true] 
                                          ↓
                                  [Sends cancel immediately]


# Scenario 2: During critical section
## Step 1: Set flag and return
[Ctrl+C] -> [SIGINT Handler] -> [Sets cancel_pressed = true] -> [Returns immediately]

## Step 2: Critical section continues

## Step 3: Server notification after critical section ends
[Critical section ends] -> [Checks cancel_pressed] -> [Sends cancel if flag was set]
```

This protection mechanism prevents common problems:
```
Without Protection:
[Interrupt] -> [Immediate Cancel] -> [Inconsistent State]
    Results in:
    - Resource leaks
    - Corrupted display
    - Incorrect terminal settings
    - Incomplete operations

With Protection:
[Interrupt] -> [Set Flag] -> [Complete Operation] -> [Process Cancel]
    Ensures:
    - Clean resource cleanup
    - Consistent display
    - Proper terminal state
    - Complete operations
```

Example:
```sql
-- Without protection
\copy users TO 'users.csv'
[Ctrl+C]
-- Result: Partially written file, incorrect terminal state

-- With protection
\copy users TO 'users.csv'
[Ctrl+C]
-- Result: Operation either completes or doesn't start,
-- terminal state remains correct, file integrity preserved
```

### 2. Cancel Connection Establishment

Instead of using the existing connection, psql creates a new, temporary connection specifically for cancellation:

```c
int PQrequestCancel(PGconn *conn)
{
    // Open new temporary connection
    int sock = connect_to_server(conn->pghost, conn->pghost);
    
    // Prepare cancellation packet
    CancelRequest pkt;
    pkt.header = 16;
    pkt.protocol = 80877102;  // Special cancel protocol ID
    pkt.pid = conn->be_pid;
    pkt.key = conn->cancel_key;
    
    // Send and immediately close
    send(sock, &pkt, sizeof(pkt), 0);
    close(sock);
}
```

The connection flow works as follows:

```
Normal Connection:
[psql] ←————————————————————→ [postgres backend]
  ↓                                  ↑
  ↓                                  |
  ↓    Cancel Connection:            |
  └———→ [postmaster] ——[SIGINT]——————┘
```

### 3. Server-Side Processing

The PostgreSQL server processes cancellation requests in multiple stages:

```c
// In postmaster (main server process)
void ProcessIncomingConnection(int port)
{
    int32 protocol;
    read(port, &protocol, 4);
    
    if (protocol == 80877102) {  // Cancel request
        CancelRequest request;
        read(port, &request, sizeof(request));
        
        // Validate credentials
        if (request.pid == backend_pid &&
            request.key == backend_key) {
            // Send SIGINT to specific backend process
            kill(request.pid, SIGINT);
        }
    }
}
```

The backend process then handles the interrupt:

```c
static void handle_sigint_backend(int sig)
{
    // Set cancellation flag
    InterruptPending = true;

    // Use Postgresql's latch objects for IPC to
    // wake up backend process that might be idle.
    SetLatch(&MyProc->procLatch);

    // For busy workers, latch.is_set & interrupt
    // flag are checked and processed at safe-points.
}
```

## Key Points

1. **Unique Cancel Key**: Each connection receives a random 32-bit key
2. **Separate Connection**: Cancellation requires a new authenticated connection
3. **Process Validation**: Both PID and key must match for security
4. **Limited Scope**: Cancellation affects only the specific backend process
5. **Data Integrity**: The backend process checks for cancellation only at safe points
6. **Connection Preservation**: The original connection remains active
7. **Clean Cleanup**: All resources are properly released

## Complete Cancellation Flow

1. User presses `Ctrl + C` in psql
2. PSQL catches SIGINT
3. New connection opens to server
4. Cancel request sent with PID and key
5. Postmaster validates request
6. Backend process receives SIGINT
7. Backend sets interrupt flag
8. Query cancels at next safe point
9. Error message returns to client
10. Original connection maintains its state