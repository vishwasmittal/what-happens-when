# What happens when you interrupt a PSQL query?

What happens when you interrupt a PostgreSQL query by pressing `Ctrl + C` in psql shell or press stop button in a your DB viewer tool?

## Overview

When you press `Ctrl + C` during a long-running query, PostgreSQL doesn't immediately terminate the connection. Instead, it implements a sophisticated cancellation mechanism allowing safe query interruption, ensures data integrity, reusing the main client-server connection.

## The Initial Connection

When psql first connects to PostgreSQL, the server provides two crucial pieces of information:

```c
ConnStatusResponse {
    backend_pid: 12345,          // Example PID
    cancel_secret_key: 0xABCD    // Random 32-bit key
}
```

It is stored in the connection object `PGconn` along with other info.

```c
typedef struct {
    int32 pid;           // Backend process ID
    int32 cancel_key;    // Secret cancellation key
    ...
} PGconn;
```

## The Cancellation Process

### 1. Client Side (PSQL)

When you press `Ctrl + C`, psql checks the process is in critical section and notifies the server accordingly.

> NOTE: A critical section is a piece of code that must execute atomically (without interruption) to maintain consistency and prevent race conditions, such as when modifying shared data or performing operations that must complete fully or not at all.

```sh
# Case 1: Not in critical section:
## Set the flag and notify the server immediately.
[Ctrl+C] -> [SIGINT Handler] -> [Sets cancel_pressed = true] 
                                          ↓
                                  [Sends cancel immediately]


# Case 2: During a critical Section:
## 1. Set the flag and return.
[Ctrl+C] -> [SIGINT Handler] -> [Sets cancel_pressed = true] -> [Returns immediately]

## 2. Critical Section Continues...

## 3. Notify server after critical section ends.
[Critical section ends] -> [Checks cancel_pressed] -> [Sends cancel if flag was set]
```

The common problems prevented:
```
Without Protection:
[Interrupt] -> [Immediate Cancel] -> [Inconsistent State]
                                    - Leaked resources
                                    - Corrupted display
                                    - Wrong terminal settings
                                    - Incomplete operations

With Protection:
[Interrupt] -> [Set Flag] -> [Complete Operation] -> [Process Cancel]
                             - Clean resource cleanup
                             - Consistent display
                             - Proper terminal state
                             - Complete operations
```

Example:
```c
// Without protection
\copy users TO 'users.csv'
[Ctrl+C]
// Result: Partially written file, terminal in wrong state

// With protection
\copy users TO 'users.csv'
[Ctrl+C]
// Result: Operation either completes or doesn't start,
// terminal state preserved, file not corrupted
```

### 2. Establishing a Cancel Connection

The interesting part is that psql doesn't use the existing connection to send the cancellation request. Instead, it creates a new, temporary connection specifically for cancellation:

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

The flow looks like this:

```
Normal Connection:
[psql] ←————————————————————→ [postgres backend]
  ↓                                  ↑
  ↓                                  |
  ↓    Cancel Connection:            |
  └———→ [postmaster] ——[SIGINT]——————┘
```

### 3. Server Side Processing

The PostgreSQL server processes the cancellation request through multiple stages:

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
    
    // Wake up backend process
    SetLatch(&MyProc->procLatch);
    
    // Don't exit - check flag at safe points
}
```

## Notes

1. **Unique Cancel Key**: Each connection gets a random 32-bit key
2. **Separate Connection**: Cancellation requires a new authenticated connection
3. **Process Validation**: Both PID and key must match
4. **Limited Scope**: Only affects the specific backend process
5. **Data Integrity**: The backend process checks for cancellation at safe points, ensuring no data corruption
6. **Connection Preservation**: The original connection remains intact
7. **Clean Cleanup**: Resources are properly released

## Complete Cancellation Flow

1. User presses `Ctrl + C` in psql
2. PSQL catches SIGINT
3. New connection opened to server
4. Cancel request sent with PID and key
5. Postmaster validates request
6. Backend process receives SIGINT
7. Backend sets interrupt flag
8. Query cancellation at next safe point
9. Error message returned to client
10. Original connection maintained
