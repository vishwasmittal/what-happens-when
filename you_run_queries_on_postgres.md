# [WIP] What happens when you run queries on PostgreSQL 16

## Introduction

This guide explains in detail what happens when you connect to PostgreSQL and execute queries. We'll examine every step from typing the psql command to getting results, including internal PostgreSQL processes, memory management, disk operations, and system interactions.

## Part 1: Connection and Authentication

### 1. Command Execution

When you type and execute:
```bash
psql postgres://username:password@localhost:5432/postgres?application_name=what-happens-when-client
```

The following steps occur:

1. **Shell Processing**
   - Your shell (e.g., bash) splits the command into tokens
   - Locates the psql executable in your PATH
   - Prepares to execute the program

2. **Connection String Parsing**
   ```
   Protocol  : postgres://
   Username  : username
   Password  : password
   Host     : localhost
   Port     : 5432
   Database : postgres
   Parameters: application_name=what-happens-when-client
   ```

3. **libpq Initialization**
   - libpq is PostgreSQL's C client library
   - It handles all client-server communication
   - Sets up default parameters:
   ```c
   typedef struct {
       char *pghost;     /* Host name */
       char *pghostaddr; /* Host IP */
       char *pgport;     /* Port number */
       char *pgdatabase; /* Database name */
       char *pguser;     /* Username */
       char *pgpassword; /* Password */
       /* ... other fields ... */
   } PGconn;
   ```

### 2. Client Parameter Setup

1. **Client Encoding**
   ```sql
   -- Default client_encoding is based on OS locale
   SHOW client_encoding;
   -- Typical output: UTF8
   
   -- Can be changed with:
   SET client_encoding = 'UTF8';
   ```
   - Determines how text is encoded between client and server
   - Common options:
     * UTF8 (default in most installations)
     * LATIN1
     * SQL_ASCII

2. **DateStyle Configuration**
   ```sql
   SHOW DateStyle;
   -- Typical output: ISO, MDY
   ```
   - Controls date/time format representation
   - Format: `output_format, input_format`
   - Common options:
     * ISO: 2024-01-17 15:30:00
     * German: 17.01.2024 15:30:00
     * SQL: 17/01/2024 15:30:00
     * Postgres: Wed 17 Jan 15:30:00 2024

### 3. Network Connection

1. **Socket Creation**
   ```c
   /* System call to create TCP socket */
   int sock = socket(AF_INET, SOCK_STREAM, 0);
   
   /* Set non-blocking mode */
   fcntl(sock, F_SETFL, O_NONBLOCK);
   ```

2. **Connection Process**
   ```c
   struct sockaddr_in addr;
   addr.sin_family = AF_INET;
   addr.sin_port = htons(5432);
   addr.sin_addr.s_addr = /* resolved IP */;
   
   connect(sock, (struct sockaddr *)&addr, sizeof(addr));
   ```

### 4. Authentication Flow

1. **pg_hba.conf Configuration**
   ```
   # TYPE  DATABASE    USER        ADDRESS          METHOD
   # Local connections
   local   all         all                          peer
   # IPv4 local connections:
   host    all         all         127.0.0.1/32     md5
   host    all         all         192.168.1.0/24   scram-sha-256
   # IPv6 local connections:
   host    all         all         ::1/128          md5
   ```
   - Each line specifies:
     * Connection type (local/host/hostssl/hostnossl)
     * Database name(s)
     * Username(s)
     * IP address range
     * Authentication method

2. **pg_authid Catalog**
   ```sql
   -- Structure of pg_authid (system catalog for roles)
   CREATE TABLE pg_authid (
       rolname          name,           -- Role name
       rolsuper         bool,           -- Superuser?
       rolinherit       bool,           -- Can inherit privileges?
       rolcreaterole    bool,           -- Can create roles?
       rolcreatedb      bool,           -- Can create databases?
       rolcanlogin      bool,           -- Can login?
       rolreplication   bool,           -- Can initiate streaming replication?
       rolbypassrls     bool,           -- Bypasses row level security?
       rolconnlimit     int4,           -- Connection limit (-1=no limit)
       rolpassword      text,           -- Password (possibly encrypted)
       rolvaliduntil    timestamptz     -- Password expiry time (null=never)
   );
   ```

3. **Authentication Process Flow**
   ```mermaid
   sequenceDiagram
       Client->>Server: StartupMessage(user, database)
       Server->>HBA: Check pg_hba.conf rules
       HBA->>Server: Authentication method
       Server->>Client: AuthenticationMD5Password
       Client->>Client: md5(md5(password + username) + salt)
       Client->>Server: PasswordMessage
       Server->>Authid: Verify against pg_authid
       Authid->>Server: Success/Failure
       Server->>Client: AuthenticationOk/Error
   ```

4. **Password Hashing Example**
   ```sql
   -- For MD5:
   -- 1. First hash: md5(password + username)
   -- 2. Second hash: md5(step1_result + random_salt)
   
   -- Example entry in pg_authid
   SELECT rolname, rolpassword FROM pg_authid WHERE rolname = 'myuser';
   -- Output: myuser    md53fc6d78d8d9d3d5a1d3d3d5a1d3d3d5a
   ```

### 5. Backend Process Creation

1. **Process Forking**
   ```c
   pid_t pid = fork();
   if (pid == 0) {
       /* Child process (backend) */
       InitPostmasterChild();
       /* ... */
   } else {
       /* Parent process (postmaster) */
       AddChildProcess(pid, BACKEND_TYPE);
   }
   ```

2. **Signal Handlers Setup**
   ```c
   static void
   BackendInitializeSignals(void)
   {
       /* Ignore all signals by default */
       sigfillset(&BlockSig);
       
       /* Unblock critical signals */
       sigdelset(&BlockSig, SIGQUIT);
       sigdelset(&BlockSig, SIGTERM);
       
       /* Set up specific handlers */
       pqsignal(SIGTERM, die);
       pqsignal(SIGQUIT, quickdie);
       pqsignal(SIGALRM, handle_sig_alarm);
   }
   ```


# Part 2: Memory Management and System Catalogs

## Memory Architecture

### 1. Memory Contexts

Memory contexts are hierarchical containers for memory allocations in PostgreSQL. They provide automatic memory cleanup when a context is deleted.

```c
/* Basic memory context structure */
typedef struct MemoryContextData {
    NodeTag     type;               /* Context type (identifies exact kind) */
    MemoryContextMethods *methods;  /* Management methods */
    MemoryContext parent;           /* Parent context, NULL if top */
    MemoryContext firstchild;       /* First child context */
    MemoryContext prevchild;        /* Previous child of same parent */
    MemoryContext nextchild;        /* Next child of same parent */
    char       *name;               /* Context name (for debugging) */
    bool        isReset;            /* True if context is reset */
    /* ... other fields ... */
} MemoryContextData;
```

Memory Context Hierarchy:
```
TopMemoryContext
├── MessageContext          (error messages)
├── CacheMemoryContext      (relation/catalog caches)
├── TopTransactionContext   (transaction-level storage)
│   └── CurrentTransactionContext
└── PortalContext          (query execution)
    └── ExecutorState
        ├── ExprContext    (expression evaluation)
        └── TupleTableSlot (tuple storage)
```

### 2. Resource Owner

Resource owners track ownership of resources like buffers, files, and relations:

```c
typedef struct ResourceOwnerData {
    ResourceOwner parent;           /* NULL if no parent */
    ResourceOwner firstchild;       /* Head of linked list of children */
    ResourceOwner nextchild;        /* Next child of same parent */
    ResourceOwner prev;             /* Previous resource owner */
    ResourceOwner next;             /* Next resource owner */
    char *name;                     /* Name (for debugging) */
    
    /* Arrays to track different resource types */
    Buffer *buffers;                /* Array of held buffers */
    int nbuffers;                   /* Number of buffers */
    File *files;                    /* Array of held files */
    int nfiles;                     /* Number of files */
    /* ... other resource arrays ... */
} ResourceOwnerData;
```

Example of resource tracking:
```c
/* When acquiring a buffer */
ResourceOwnerEnlargeBuffers(CurrentResourceOwner);
ResourceOwnerRememberBuffer(CurrentResourceOwner, buffer);

/* When releasing */
ResourceOwnerForgetBuffer(CurrentResourceOwner, buffer);
```

### 3. Shared Memory Setup

1. **Shared Memory Segments**
```c
typedef struct {
    int         num_entries;        /* Number of array entries */
    bool        cacheBlocks;        /* Are we caching block contents? */
    HASHCTL     hash_ctl;          /* Hash table control structure */
    char       *blocks;            /* Array of cached blocks */
    /* ... other fields ... */
} SharedBufferCache;
```

2. **Shared Buffer Access**
```c
/* Buffer descriptor in shared memory */
typedef struct BufferDesc {
    BufferTag   tag;              /* ID of page contained in buffer */
    int         buf_id;           /* Buffer's index number (from 0) */
    int         free_next;        /* Link in freelist chain */
    unsigned    refcount;         /* Number of pins on buffer */
    
    /* State data */
    pg_atomic_uint32 state;    /* State of buffer: clean/dirty/pinned */
    int         usage_count;      /* Clock sweep access count */
    
    /* Exclusive locking */
    pg_atomic_uint32 wait_backend_pid;  /* Backend waiting for pin */
    
    /* Buffer I/O state */
    pg_atomic_uint32 io_in_progress_lock; /* Counter of I/O operations */
} BufferDesc;
```

3. **Buffer Pool Initialization**
```sql
-- From postgresql.conf
shared_buffers = 128MB    # Default is usually 128MB
huge_pages = try         # Try to use huge pages if available
```

```c
/* Initialization code */
void
InitBufferPool(void)
{
    int         num_buffers;
    Size        block_size;
    
    /* Calculate number of buffers based on shared_buffers setting */
    num_buffers = shared_buffers / BLCKSZ;
    
    /* Allocate the buffer descriptors */
    BufferDescriptors = (BufferDesc *)
        ShmemInitStruct("Buffer Descriptors",
                       num_buffers * sizeof(BufferDesc),
                       &foundDescs);
                       
    /* Allocate the buffer blocks */
    BufferBlocks = (char *)
        ShmemInitStruct("Buffer Blocks",
                       num_buffers * BLCKSZ,
                       &foundBlocks);
}
```

## System Catalogs

### 1. Essential System Catalogs

1. **postgresql.conf Settings**
```ini
# Database Connections
max_connections = 100               # Maximum concurrent connections
superuser_reserved_connections = 3  # Connections reserved for superuser

# Memory Settings
shared_buffers = 128MB             # Memory for shared buffer cache
work_mem = 4MB                     # Memory for query operations
maintenance_work_mem = 64MB        # Memory for maintenance operations

# Write Ahead Log
wal_level = replica               # WAL level (minimal, replica, logical)
fsync = on                       # Forces synchronization of updates
synchronous_commit = on          # Wait for WAL write to disk

# Query Planning
random_page_cost = 4             # Cost of non-sequential page access
effective_cache_size = 4GB       # Estimate of disk cache size
```

2. **System Catalog Tables**

```sql
-- pg_database: Database information
CREATE TABLE pg_database (
    oid         oid,          -- Object identifier
    datname     name,         -- Database name
    datdba      oid,          -- Owner of the database
    encoding    int4,         -- Character encoding
    datcollate  name,         -- LC_COLLATE for this database
    datctype    name,         -- LC_CTYPE for this database
    datistemplate bool,       -- If true, can be TEMPLATE for CREATE DATABASE
    datallowconn bool,        -- If false, no one can connect to this DB
    datconnlimit int4,        -- Connection limit (-1 means no limit)
    datlastsysoid oid,        -- Last system OID in database
    datfrozenxid xid,        -- All transaction IDs before this are frozen
    datminmxid xid,         -- All multixact IDs before this are frozen
    dattablespace oid        -- The database's default tablespace
);

-- pg_class: Table, index, sequence, view information
CREATE TABLE pg_class (
    oid         oid,          -- Object identifier
    relname     name,         -- Name of table, index, view, etc.
    relnamespace oid,         -- The OID of the namespace containing this relation
    reltype     oid,         -- The OID of the data type that corresponds
    reloftype   oid,         -- The OID of the composite type this is a table of
    relowner    oid,         -- Owner of the relation
    relam       oid,         -- If an index, the access method used
    relfilenode oid,         -- Name of the on-disk file of this relation
    reltablespace oid,       -- The tablespace in which this relation is stored
    relpages    int4,        -- Size of the on-disk representation in pages
    reltuples   float4,      -- Number of live rows in the table
    relhasindex bool,        -- True if this is a table and it has (or recently had) any indexes
    relisshared bool,        -- True if this table is shared across all databases in the cluster
    relpersistence char,     -- p = permanent table, u = unlogged table, t = temporary table
    relkind     char,        -- r = ordinary table, i = index, S = sequence, v = view, etc.
    relnatts    int2,        -- Number of user columns
    relchecks   int2,        -- Number of CHECK constraints
    relhasrules bool,        -- True if table has (or once had) rules
    relhastriggers bool,     -- True if table has (or once had) triggers
    partbound   pg_node_tree -- Partition bound node tree if table is partitioned
);

-- pg_attribute: Column information
CREATE TABLE pg_attribute (
    attrelid    oid,         -- The table this column belongs to
    attname     name,        -- Column name
    atttypid    oid,         -- Data type of this column
    attlen      int2,        -- Copy of type's attlen
    attnum      int2,        -- Column number
    attndims    int4,        -- Number of dimensions, if an array type
    attcacheoff int4,        -- Always -1 in storage, but used in cache
    atttypmod   int4,        -- Type-specific data supplied at table creation
    attbyval    bool,        -- Whether type is passed by value
    attstorage  char,        -- Strategy for storing variable-length types
    attalign    char,        -- Alignment required for type
    attnotnull  bool,        -- Not null constraint
    atthasdef   bool,        -- Has a default value
    attidentity char,        -- If an identity column, generated always/by default
    attgenerated char,       -- If a generated column, 's' = stored
    attisdropped bool,       -- Column has been dropped
    attislocal  bool,        -- Column is local to relation (not inherited)
    attinhcount int4,        -- Number of inheritance ancestors
    attcollation oid,        -- Column collation
    attacl      aclitem[],   -- Column-level access privileges
    attoptions  text[],      -- Attribute-level options
    attfdwoptions text[]     -- FDW-specific options
);
```

### 2. Catalog Access Example

When backend starts, it reads various system catalogs:

```c
/* Initialize system catalog cache */
void
InitCatalogCache(void)
{
    /* Initialize cache for pg_database */
    CacheRegisterSysCache(DATABASEOID,
                         "pg_database",
                         DatabaseRelationId,
                         1,
                         DatabaseKeyIndexes);
                         
    /* Initialize cache for pg_class */
    CacheRegisterSysCache(RELOID,
                         "pg_class",
                         RelationRelationId,
                         1,
                         RelationKeyIndexes);
    
    /* And so on for other catalogs... */
}

/* Example of reading database info */
void
GetDatabaseInfo(Oid dbid)
{
    HeapTuple   tuple;
    Form_pg_database dbform;
    
    tuple = SearchSysCache1(DATABASEOID, ObjectIdGetDatum(dbid));
    if (!HeapTupleIsValid(tuple))
        elog(ERROR, "cache lookup failed for database %u", dbid);
        
    dbform = (Form_pg_database) GETSTRUCT(tuple);
    
    /* Now we can access database properties */
    printf("Database: %s, Owner: %u, Encoding: %d\n",
           NameStr(dbform->datname),
           dbform->datdba,
           dbform->encoding);
           
    ReleaseSysCache(tuple);
}
```

# Part 3: Query Processing and Planning

## Query Parsing

### 1. Tokenization Process

When you enter a SQL command, PostgreSQL first breaks it down into tokens. Let's see this with an example:

```sql
CREATE TABLE employee (
    id SERIAL PRIMARY KEY,
    name VARCHAR(256) NOT NULL,
    email TEXT UNIQUE,
    date_of_birth DATE,
    salary NUMERIC(10,2)
);
```

This gets tokenized into:
```c
/* Internal token representation */
typedef struct {
    TokenType type;
    int len;
    char *str;
} Token;

Tokens[] = {
    {T_CREATE, 6, "CREATE"},
    {T_TABLE, 5, "TABLE"},
    {T_IDENT, 8, "employee"},
    {T_OPEN_PAREN, 1, "("},
    {T_IDENT, 2, "id"},
    {T_IDENT, 6, "SERIAL"},
    {T_PRIMARY, 7, "PRIMARY"},
    {T_KEY, 3, "KEY"},
    {T_COMMA, 1, ","},
    {T_IDENT, 4, "name"},
    {T_IDENT, 7, "VARCHAR"},
    {T_OPEN_PAREN, 1, "("},
    {T_INTEGER, 3, "256"},
    {T_CLOSE_PAREN, 1, ")"},
    {T_NOT, 3, "NOT"},
    {T_NULL, 4, "NULL"},
    /* ... more tokens ... */
    {T_CLOSE_PAREN, 1, ")"},
    {T_SEMICOLON, 1, ";"}
};
```

### 2. Abstract Syntax Tree (AST)

The tokens are then parsed into an Abstract Syntax Tree. Here's a simplified representation:

```c
/* AST for CREATE TABLE */
CreateStmt {
    type: T_CreateStmt,
    relation: {
        type: T_RangeVar,
        relname: "employee",
        inh: true,
        relpersistence: 'p'
    },
    tableElts: [
        {
            type: T_ColumnDef,
            colname: "id",
            typename: {
                type: T_TypeName,
                names: ["pg_catalog", "serial"],
                typeOid: 23  /* int4 */
            },
            constraints: [
                {
                    type: T_Constraint,
                    contype: T_CONSTR_PRIMARY
                }
            ]
        },
        {
            type: T_ColumnDef,
            colname: "name",
            typename: {
                type: T_TypeName,
                names: ["pg_catalog", "varchar"],
                typeOid: 1043,  /* varchar */
                typemod: 260    /* length + 4 */
            },
            constraints: [
                {
                    type: T_Constraint,
                    contype: T_CONSTR_NOTNULL
                }
            ]
        },
        /* ... other columns ... */
    ],
    oncommit: ONCOMMIT_NOOP,
    tablespacename: NULL
}
```

### 3. Name and Permission Validation

PostgreSQL validates names and permissions through several steps:

```c
/* Name validation */
static void
CheckRelationName(const char *relationName, RangeVar *relation)
{
    /* Check name length */
    if (strlen(relationName) >= NAMEDATALEN)
        ereport(ERROR,
                (errcode(ERRCODE_NAME_TOO_LONG),
                 errmsg("relation name \"%s\" is too long",
                        relationName)));

    /* Check valid characters */
    for (const char *cp = relationName; *cp; cp++)
    {
        if (!isalnum((unsigned char) *cp) && *cp != '_')
            ereport(ERROR,
                    (errcode(ERRCODE_INVALID_NAME),
                     errmsg("relation name contains invalid characters")));
    }
}

/* Permission checking */
static void
CheckCreateTablePermissions(void)
{
    /* Must have CREATE privilege on namespace */
    if (!pg_namespace_aclcheck(namespaceId, GetUserId(),
                             ACL_CREATE) == ACLCHECK_OK)
        aclcheck_error(ACLCHECK_NO_PRIV, OBJECT_SCHEMA,
                      get_namespace_name(namespaceId));
}
```

## Transaction Management

### 1. Transaction Start

When a new transaction begins:

```c
/* Transaction structure */
typedef struct TransactionStateData {
    TransState   state;           /* state of the transaction */
    SubTransactionId subTransactionId;  /* current sub-transaction ID */
    XLogRecPtr  beginLoc;        /* WAL location of transaction start */
    TimestampTz startTimestamp;  /* timestamp of transaction start */
    Oid         databaseId;      /* database the transaction is in */
    Oid         userId;          /* user running the transaction */
    /* ... other fields ... */
} TransactionStateData;

/* Starting a transaction */
void
StartTransaction(void)
{
    /* Allocate transaction memory context */
    TopTransactionContext = AllocSetContextCreate(TopMemoryContext,
                                                "TopTransactionContext",
                                                ALLOCSET_DEFAULT_SIZES);
                                                
    /* Initialize transaction state */
    s = &TopTransactionStateData;
    s->state = TRANS_START;
    s->startTimestamp = GetCurrentTimestamp();
    s->databaseId = MyDatabaseId;
    s->userId = GetUserId();
    
    /* Get new transaction ID */
    s->transactionId = GetNewTransactionId(true);
    
    /* Set up transaction snapshot */
    PushActiveSnapshot(GetTransactionSnapshot());
}
```

### 2. Lock Management

Example of AccessExclusiveLock acquisition:

```c
/* Lock modes */
typedef enum LOCKMODE {
    NoLock,                     /* no lock */
    AccessShareLock,            /* SELECT */
    RowShareLock,              /* SELECT FOR UPDATE/SHARE */
    RowExclusiveLock,          /* UPDATE, DELETE, INSERT */
    ShareUpdateExclusiveLock,  /* VACUUM, ANALYZE, CREATE INDEX */
    ShareLock,                 /* CREATE INDEX CONCURRENTLY */
    ShareRowExclusiveLock,     /* like EXCLUSIVE MODE, but allows ROW SHARE */
    ExclusiveLock,             /* blocks ROW SHARE/SELECT...FOR UPDATE */
    AccessExclusiveLock        /* ALTER TABLE, DROP TABLE, TRUNCATE, REINDEX, CLUSTER */
} LOCKMODE;

/* Lock acquisition */
void
LockRelation(Relation relation, LOCKMODE lockmode)
{
    LockAcquireResult result;
    
    /* Try to acquire the lock */
    result = LockAcquire(&relation->rd_lockInfo.lockRelId,
                        lockmode,
                        false,  /* don't wait if can't acquire immediately */
                        false); /* don't report error if can't acquire */
                        
    if (result == LOCKACQUIRE_NOT_AVAIL)
    {
        /* Wait for lock */
        LockAcquire(&relation->rd_lockInfo.lockRelId,
                    lockmode,
                    true,   /* wait for lock */
                    false);
    }
}
```

## Query Planning

### 1. Query Planner Components

```c
/* Planner input */
typedef struct PlannedStmt {
    NodeTag     type;
    CmdType     commandType;    /* SELECT/INSERT/UPDATE/DELETE */
    uint64      queryId;        /* query identifier */
    Plan       *planTree;       /* tree of Plan nodes */
    List       *rtable;         /* list of RangeTblEntry nodes */
    List       *targetList;     /* list of TargetEntry nodes */
    /* ... other fields ... */
} PlannedStmt;

/* Example plan node */
typedef struct Plan {
    NodeTag     type;
    Cost        startup_cost;   /* cost before first tuple */
    Cost        total_cost;     /* total cost */
    double      plan_rows;      /* number of rows plan expected to emit */
    int         plan_width;     /* average row width in bytes */
    /* ... other fields ... */
} Plan;
```

### 2. Planning Process Example

For a simple SELECT query:
```sql
SELECT name, salary 
FROM employee 
WHERE salary > 50000 
ORDER BY salary DESC;
```

The planner generates:
```c
/* Simplified plan tree */
Sort (
    cost=25.41..25.42 rows=5 width=36
    Sort Key: salary DESC
    ->  Seq Scan on employee  (
        cost=0.00..25.40 rows=5 width=36
        Filter: (salary > 50000::numeric)
    )
)
```

### 3. Planner Skip for DDL

DDL (Data Definition Language) commands like CREATE TABLE bypass the planner because:
1. They have fixed execution paths
2. No optimization is possible/needed
3. They're handled directly by the command processing logic

```c
/* Command processing */
switch (nodeTag(parseTree))
{
    case T_CreateStmt:
        /* CREATE TABLE */
        ProcessCreateTableStmt(stmt, queryString);
        break;
    case T_SelectStmt:
        /* Regular SELECT - needs planning */
        planner_stmt = planner(stmt, ...);
        break;
}
```

## System Catalog Updates

### 1. pg_class Entry

When creating the employee table:

```sql
/* New row in pg_class */
INSERT INTO pg_class VALUES (
    nextval('pg_class_oid_seq'),  -- oid
    'employee',                    -- relname
    current_schema()::oid,         -- relnamespace
    /* ... other fields ... */
    'r',                          -- relkind (r = regular table)
    8,                            -- relnatts (number of columns)
    false,                        -- relhasindex (initially)
    true,                         -- relhasrules
    /* ... more fields ... */
);
```

### 2. pg_attribute Entries

For each column:

```sql
/* One row per column in pg_attribute */
INSERT INTO pg_attribute VALUES
(
    'employee'::regclass,         -- attrelid
    'id',                         -- attname
    'int4'::regtype,             -- atttypid
    4,                           -- attlen
    1,                           -- attnum
    0,                           -- attndims
    -1,                          -- attcacheoff
    -1,                          -- atttypmod
    true,                        -- attbyval
    'p',                         -- attstorage
    'i',                         -- attalign
    true,                        -- attnotnull
    true,                        -- atthasdef
    /* ... more fields ... */
),
/* ... entries for other columns ... */
;
```


# Part 4: Storage and Buffer Management

## Physical Storage Organization

### 1. Database Directory Structure

```bash
$PGDATA/
├── base/                          # Database directories
│   ├── 1/                        # Template1 database
│   │   ├── PG_VERSION
│   │   ├── 12345                 # Table file
│   │   ├── 12345_fsm            # Free Space Map
│   │   └── 12345_vm             # Visibility Map
│   └── 16384/                    # Our database (OID: 16384)
│       ├── PG_VERSION
│       ├── 24576                 # employee table file
│       ├── 24576_fsm            # Free Space Map for employee
│       └── 24576_vm             # Visibility Map for employee
├── global/                       # Cluster-wide tables
│   ├── pg_control               # Control file
│   ├── pg_filenode.map          # Filenode mapping
│   └── pg_internal.init         # Internal catalog init file
├── pg_wal/                      # Write Ahead Log
│   ├── 000000010000000000000001 # WAL segment file
│   └── 000000010000000000000002 # WAL segment file
└── pg_xact/                     # Transaction status
    └── 0000                     # Transaction status bits
```

### 2. Page Structure (8KB)

Every table file is divided into 8KB pages. Here's the detailed structure:

```c
/* Page header (24 bytes) */
typedef struct PageHeaderData {
    PageXLogRecPtr pd_lsn;      /* LSN: next byte after last byte of WAL record */
    uint16      pd_checksum;    /* CRC of page contents */
    uint16      pd_flags;       /* Flag bits */
    LocationIndex pd_lower;     /* Offset to start of free space */
    LocationIndex pd_upper;     /* Offset to end of free space */
    LocationIndex pd_special;   /* Offset to start of special space */
    uint16      pd_pagesize_version;
    TransactionId pd_prune_xid; /* Oldest unpruned XMAX, or zero if none */
    ItemIdData   pd_linp[1];    /* Beginning of line pointer array */
} PageHeaderData;

/* Item pointer (4 bytes) */
typedef struct ItemIdData {
    unsigned    lp_off:15,      /* Offset to tuple (from start of page) */
                lp_flags:2,     /* Status of item pointer */
                lp_len:15;      /* Length of tuple */
} ItemIdData;

/* Page layout example for employee table */
Page Layout (8192 bytes):
+--------------------------------+ 0x0000
| PageHeaderData (24 bytes)      |
+--------------------------------+ 0x0018
| ItemIdData Array               |
| (4 bytes per item)             |
+--------------------------------+ varies
| Free Space                     |
|                               |
+--------------------------------+ varies
| Tuple Data                     |
| (variable length)              |
+--------------------------------+ 0x2000
```

### 3. Tuple (Row) Structure

For our employee table:

```c
/* Heap tuple header (23 bytes minimum) */
typedef struct HeapTupleHeaderData {
    union {
        HeapTupleFields t_heap;
        DatumTupleFields t_datum;
    } t_choice;
    
    ItemPointerData t_ctid;      /* current TID of this or newer tuple */
    
    /* Size of tuple header varies */
    uint16      t_infomask2;     /* number of attributes + various flags */
    uint16      t_infomask;      /* various flag bits */
    uint8       t_hoff;          /* sizeof header incl. bitmap, padding */
    
    /* bitmap of NULLs starts here */
} HeapTupleHeaderData;

/* Example tuple layout for employee row */
Tuple Layout:
+---------------------------------+ 0x0000
| HeapTupleHeaderData (23 bytes)  |
+---------------------------------+ 0x0017
| Null Bitmap (1 byte)            |
+---------------------------------+ 0x0018
| Attribute Data                  |
| - id (4 bytes)                 |
| - name (variable)              |
| - email (variable/TOAST)       |
| - date_of_birth (4 bytes)      |
| - salary (variable)            |
+---------------------------------+ varies
```

## Buffer Management

### 1. Shared Buffer Pool

```c
/* Shared buffer configuration */
typedef struct {
    int32   NBuffers;           /* Number of shared buffers */
    int32   BlockSize;          /* Size of each buffer (usually 8KB) */
    bool    UseLocalBuffer;     /* Use local buffer for temp tables */
    /* ... other configuration ... */
} BufferDesc;

/* Buffer state bits */
#define BM_DIRTY       (1 << 0)    /* Buffer needs writing */
#define BM_VALID       (1 << 1)    /* Buffer contains valid data */
#define BM_TAG_VALID   (1 << 2)    /* Tag is assigned */
#define BM_IO_IN_PROGRESS (1 << 3) /* Buffer undergoing I/O */
#define BM_IO_ERROR    (1 << 4)    /* Previous I/O failed */
#define BM_JUST_DIRTIED (1 << 5)   /* Dirtied since last write */
```

### 2. Buffer Allocation Process

```c
/* Reading a page into buffer pool */
BufferDesc *
ReadBuffer(Relation relation, BlockNumber blockNum)
{
    BufferDesc *buf;
    bool found;
    
    /* Look up the buffer in shared hash table */
    buf = BufferTableLookup(relation, blockNum, &found);
    if (found) {
        /* Buffer is already in pool */
        IncrBufferRefCount(buf);
        return buf;
    }
    
    /* Need to read the block */
    buf = GetVictimBuffer();  /* Clock sweep algorithm */
    if (buf->flags & BM_DIRTY) {
        /* Write dirty buffer to disk first */
        FlushBuffer(buf);
    }
    
    /* Read the block from disk */
    smgrread(relation->rd_smgr, blockNum, buf->data);
    
    /* Set up buffer header */
    buf->tag.rnode = relation->rd_node;
    buf->tag.blockNum = blockNum;
    buf->flags = BM_VALID | BM_TAG_VALID;
    
    return buf;
}
```

### 3. TOAST (The Oversized-Attribute Storage Technique)

```c
/* TOAST pointer structure */
typedef struct {
    uint32      va_rawsize;     /* Original data size */
    uint32      va_extsize;     /* External saved size */
    Oid         va_valueid;     /* TOAST OID */
    uint32      va_toastrelid;  /* TOAST table OID */
} varatt_external;

/* TOAST strategies */
#define TOAST_PGLZ_STRATEGY   1
#define TOAST_LZ4_STRATEGY    2

/* Example of TOAST table structure for employee */
CREATE TABLE pg_toast.pg_toast_24576 (
    chunk_id      oid,          /* OID of owning TOAST value */
    chunk_seq     int4,         /* Sequence number within value */
    chunk_data    bytea         /* Actual data chunk */
);

/* TOAST value insertion */
void
toast_insert_or_update(Relation rel, HeapTuple newtup, HeapTuple oldtup)
{
    /* Check for TOAST-able attributes */
    if (HeapTupleHasExternal(newtup)) {
        /* For each TOAST-able attribute */
        for (i = 0; i < RelationGetNumberOfAttributes(rel); i++) {
            if (att[i]->attlen == -1) {  /* Variable length */
                if (VARSIZE_EXTERNAL(val) > TOAST_MAX_CHUNK_SIZE) {
                    /* Split into chunks and store in TOAST table */
                    chunked_datum = toast_save_datum(rel, val);
                    /* Replace original value with TOAST pointer */
                    SET_VARTAG_EXTERNAL(result, VARTAG_ONDISK);
                }
            }
        }
    }
}
```

### 4. Write-Ahead Log (WAL)

```c
/* WAL record header */
typedef struct XLogRecord {
    uint32      xl_tot_len;     /* Total length of record */
    TransactionId xl_xid;       /* Transaction ID */
    XLogRecPtr  xl_prev;        /* Previous record location */
    uint8       xl_info;        /* Flag bits */
    RmgrId      xl_rmid;        /* Resource manager ID */
    /* followed by backup blocks and main data */
} XLogRecord;

/* Example WAL record for tuple insertion */
typedef struct xl_heap_insert {
    OffsetNumber offnum;        /* inserted tuple's offset */
    uint8       flags;          /* flags */
    /* TUPLE DATA FOLLOWS AT END OF STRUCT */
} xl_heap_insert;

/* WAL writing process */
void
XLogInsert(RmgrId rmid, uint8 info, XLogRecData *rdata)
{
    /* Get current insert position */
    RecPtr = GetXLogInsertRecPtr();
    
    /* Construct record header */
    rechdr->xl_tot_len = totalsize;
    rechdr->xl_xid = GetCurrentTransactionId();
    rechdr->xl_prev = PrevRecPtr;
    rechdr->xl_info = info;
    rechdr->xl_rmid = rmid;
    
    /* Copy record to WAL buffer */
    CopyXLogRecordToWAL(rechdr, rdata);
    
    /* Update LSN */
    PageSetLSN(page, RecPtr);
    
    if (synchronous_commit) {
        /* Force WAL flush up to this record */
        XLogFlush(RecPtr);
    }
}
```

### 5. Data Flushing to Disk

```c
/* Background writer process */
void
BackgroundWriterMain(void)
{
    for (;;) {
        /* Wait for next iteration */
        WaitLatch(&MyProc->procLatch,
                  WL_LATCH_SET | WL_TIMEOUT | WL_POSTMASTER_DEATH,
                  BgWriterDelay);
                  
        /* Write some dirty buffers */
        num_written = BgBufferSync();
        
        /* Write a progress report */
        pgstat_report_bgwriter();
        
        if (got_SIGHUP) {
            /* Reload configuration */
            ProcessConfigFile(PGC_SIGHUP);
        }
    }
}

/* Checkpoint process */
void
CreateCheckPoint(bool shutdown)
{
    /* Write dirty buffers */
    FlushDirtyBuffers();
    
    /* Write checkpoint record */
    CheckPoint.redo = GetRedoRecPtr();
    CheckPoint.time = (pg_time_t) time(NULL);
    
    XLogInsert(RM_XLOG_ID, XLOG_CHECKPOINT_SHUTDOWN, &rdata);
    
    /* Flush WAL up to checkpoint record */
    XLogFlush(CheckPoint.redo);
    
    /* Update control file */
    UpdateControlFile();
}
```


# Part 5: Transaction Management and Locking

## Transaction Management

### 1. Transaction States and Structure

```c
/* Transaction states */
typedef enum TransState {
    TRANS_DEFAULT,              /* Idle state */
    TRANS_START,               /* Transaction starting */
    TRANS_INPROGRESS,         /* Transaction in progress */
    TRANS_COMMIT,             /* Transaction committing */
    TRANS_ABORT,              /* Transaction aborting */
    TRANS_PREPARED           /* Transaction prepared for 2PC */
} TransState;

/* Transaction structure */
typedef struct TransactionStateData {
    TransState   state;         /* Current transaction state */
    XidStatus   *statusFlags;   /* Status bits in pg_xact */
    XLogRecPtr   xactStartLSN;  /* LSN at transaction start */
    
    /* Transaction identification */
    TransactionId transactionId;  /* XID of current transaction */
    SubTransactionId subTransactionId;  /* Current subtransaction ID */
    
    /* Memory context management */
    MemoryContext curTransactionContext;  /* Current transaction memory */
    ResourceOwner curTransactionOwner;   /* Resource tracking */
    
    /* Transaction characteristics */
    bool     blockState;       /* True if waiting on a lock */
    char    *commandTag;       /* Command being executed */
    
    /* Parent/child relationships */
    TransactionStateData *parent;  /* Parent transaction if nested */
    int      nestingLevel;     /* Transaction nesting depth */
    
    /* MVCC snapshot data */
    Snapshot   snapshot;       /* Current MVCC snapshot */
    XidStatus  *snapshotXids;  /* Array of concurrent XIDs */
} TransactionStateData;
```

### 2. MVCC (Multi-Version Concurrency Control)

```c
/* Tuple visibility information */
typedef struct HeapTupleFields {
    TransactionId t_xmin;        /* Insert XID */
    TransactionId t_xmax;        /* Delete XID */
    union {
        CommandId   t_cid;       /* Insert or Delete command ID */
        TransactionId t_xvac;    /* VACUUM FULL XID */
    } t_field3;
} HeapTupleFields;

/* MVCC Snapshot structure */
typedef struct SnapshotData {
    SnapshotSatisfiesFunc satisfies;  /* Tuple visibility function */
    TransactionId xmin;        /* Earliest XID still active */
    TransactionId xmax;        /* First as-yet-unassigned XID */
    TransactionId *xip;        /* Array of active XIDs */
    int          xcnt;         /* Number of active XIDs */
    TransactionId *subxip;     /* Array of active sub-XIDs */
    int          subxcnt;      /* Number of active sub-XIDs */
    bool         suboverflowed; /* Sub-XID array overflowed */
    bool         takenDuringRecovery;  /* Taken during recovery */
    bool         copied;       /* Snapshot has been copied */
    CommandId    curcid;       /* Current command ID */
    uint32       active_count; /* Active transaction count */
    uint32       regd_count;   /* Registered snapshot count */
    TimestampTz whenTaken;     /* Timestamp when snapshot taken */
    XLogRecPtr   lsn;         /* Position in WAL when taken */
} SnapshotData;

/* Tuple visibility check example */
bool
HeapTupleSatisfiesSnapshot(HeapTuple tuple, Snapshot snapshot)
{
    TransactionId xmin = HeapTupleHeaderGetXmin(tuple->t_data);
    TransactionId xmax = HeapTupleHeaderGetXmax(tuple->t_data);
    
    /* Tuple inserted after snapshot? */
    if (TransactionIdFollowsOrEquals(xmin, snapshot->xmax))
        return false;
    
    /* Tuple deleted before snapshot? */
    if (TransactionIdPrecedes(xmax, snapshot->xmin))
        return false;
    
    /* Is inserting transaction visible? */
    if (TransactionIdIsCurrentTransactionId(xmin))
        return true;
    
    /* Check if xmin is in progress in snapshot */
    for (int i = 0; i < snapshot->xcnt; i++) {
        if (xmin == snapshot->xip[i])
            return false;  /* Transaction still in progress */
    }
    
    return true;  /* Tuple is visible */
}
```

### 3. Lock Management

```c
/* Lock modes */
typedef enum LockMode {
    NO_LOCK,                    /* No lock held or needed */
    ACCESS_SHARE_LOCK,          /* SELECT */
    ROW_SHARE_LOCK,            /* SELECT FOR UPDATE/SHARE */
    ROW_EXCLUSIVE_LOCK,        /* UPDATE, DELETE, INSERT */
    SHARE_UPDATE_EXCLUSIVE_LOCK, /* VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY */
    SHARE_LOCK,                /* CREATE INDEX */
    SHARE_ROW_EXCLUSIVE_LOCK,  /* Like EXCLUSIVE MODE, but allows ROW SHARE */
    EXCLUSIVE_LOCK,            /* Blocks ROW SHARE/SELECT ... FOR UPDATE */
    ACCESS_EXCLUSIVE_LOCK      /* ALTER TABLE, DROP TABLE, TRUNCATE, REINDEX */
} LockMode;

/* Lock tag (what is being locked) */
typedef struct LOCKTAG {
    uint32      locktag_field1; /* Database OID */
    uint32      locktag_field2; /* Relation OID */
    uint32      locktag_field3; /* Page number or other */
    uint32      locktag_field4; /* Tuple offset or other */
    uint32      locktag_type;   /* Lock type */
} LOCKTAG;

/* Lock request */
typedef struct LOCKREQUEST {
    LOCKTAG     locktag;        /* What to lock */
    LockMode    lockmode;       /* Lock mode */
    bool        granted;        /* Has lock been granted? */
    PGPROC     *proc;          /* Process holding/waiting for lock */
    int         pid;           /* Process ID */
} LOCKREQUEST;
```

### 4. Lock Implementation

```c
/* Lock acquisition */
LockAcquireResult
LockAcquire(LOCKTAG *locktag, LockMode lockmode, bool sessionLock,
            bool dontWait)
{
    LOCK       *lock;
    LOCKREQUEST *request;
    bool        found;
    
    /* Look up or create lock object */
    lock = (LOCK *) hash_search(LockMethodLockHash,
                               (void *) locktag,
                               HASH_ENTER,
                               &found);
                               
    /* Initialize new lock if needed */
    if (!found)
        InitLock(lock);
        
    /* Add lock request */
    request = (LOCKREQUEST *)
        SHMQueueNext(&lock->requests,
                    &lock->requests,
                    offsetof(LOCKREQUEST, chain));
                    
    /* Can we get the lock immediately? */
    if (LOCK_CONFLICTS(lock->granted, lockmode)) {
        if (dontWait)
            return LOCKACQUIRE_NOT_AVAIL;
            
        /* Wait for lock */
        WaitOnLock(lock, request);
    }
    
    /* Grant the lock */
    GrantLock(lock, request);
    return LOCKACQUIRE_OK;
}
```

### 5. Row-Level Locks (for UPDATE/DELETE)

```c
/* Row lock modes */
#define MultiXactStatusUpdate           0x01    /* Updating */
#define MultiXactStatusNoKeyUpdate      0x02    /* No key update */
#define MultiXactStatusShare            0x04    /* Sharing */
#define MultiXactStatusForKeyShare      0x08    /* For key share */

/* Row locking example for UPDATE */
static void
ExecLockRows(ModifyTableState *mtstate, EPQState *epqstate)
{
    ResultRelInfo *resultRelInfo;
    Relation    rel;
    TupleTableSlot *slot;
    
    /* Get tuple to lock */
    slot = ExecProcNode(outerPlanState(mtstate));
    
    /* Attempt to lock row version */
    tuple = heap_lock_tuple(rel,
                           tuple,
                           GetCurrentCommandId(true),
                           LockTupleExclusive,
                           LockWaitBlock,
                           false);
                           
    /* Handle concurrent update case */
    if (!tuple) {
        /* Row was concurrently updated */
        if (epqstate != NULL) {
            /* Re-evaluate with new row version */
            slot = EvalPlanQual(epqstate, rel, rowmark);
        }
    }
}
```

### 6. Deadlock Detection

```c
/* Deadlock detection data */
typedef struct DEADLOCK_INFO {
    PGPROC     *proc;          /* Blocked process */
    LOCK       *waitLock;      /* Lock being waited for */
    int         nWaitProcs;    /* Number of procs waiting on waitLock */
    PGPROC    **waitProcs;     /* Processes holding conflicting locks */
} DEADLOCK_INFO;

/* Deadlock checker */
bool
DeadLockCheck(PGPROC *proc)
{
    DEADLOCK_INFO *deadlock_state;
    int         nprocs;
    bool        deadlock_found = false;
    
    /* Initialize deadlock state */
    deadlock_state = InitDeadLockState();
    
    /* Check for cycles in wait-for graph */
    for (;;) {
        PGPROC     *next_proc;
        
        /* Get next process in chain */
        next_proc = GetBlockingProc(deadlock_state);
        if (next_proc == NULL)
            break;              /* No deadlock */
            
        if (next_proc == proc) {
            deadlock_found = true;  /* Found a cycle */
            break;
        }
        
        /* Add process to deadlock state */
        AddToDeadLockState(deadlock_state, next_proc);
    }
    
    if (deadlock_found) {
        /* Choose victim and abort its transaction */
        PGPROC *victim = ChooseDeadLockVictim(deadlock_state);
        LWLockAcquire(ProcArrayLock, LW_EXCLUSIVE);
        ProcArrayEndTransaction(victim, InvalidTransactionId);
        LWLockRelease(ProcArrayLock);
    }
    
    return deadlock_found;
}
```

### 7. Transaction Isolation Levels

```c
/* Isolation levels */
typedef enum {
    ISOLATION_DEFAULT,           /* Default level (READ COMMITTED) */
    ISOLATION_READ_COMMITTED,    /* Each statement gets new snapshot */
    ISOLATION_REPEATABLE_READ,   /* Transaction gets single snapshot */
    ISOLATION_SERIALIZABLE      /* Ensures serial execution */
} IsolationLevel;

/* Snapshot management for different isolation levels */
Snapshot
GetTransactionSnapshot(void)
{
    switch (XactIsoLevel) {
        case ISOLATION_READ_COMMITTED:
            /* Get fresh snapshot for each statement */
            return GetNewTransactionSnapshot();
            
        case ISOLATION_REPEATABLE_READ:
        case ISOLATION_SERIALIZABLE:
            if (FirstSnapshotSet)
                /* Reuse existing snapshot */
                return CurrentSnapshot;
            else {
                /* Get initial snapshot */
                CurrentSnapshot = GetNewTransactionSnapshot();
                FirstSnapshotSet = true;
                return CurrentSnapshot;
            }
            
        default:
            elog(ERROR, "unrecognized isolation level: %d",
                 XactIsoLevel);
            return NULL;
    }
}
```

### 8. Serializable Isolation Implementation

```c
/* Serializable transaction information */
typedef struct SERIALIZABLEXACT {
    TransactionId xid;          /* Transaction ID */
    int          pid;           /* Process ID */
    
    /* Conflict tracking */
    SHM_QUEUE    inConflicts;   /* List of incoming conflicts */
    SHM_QUEUE    outConflicts;  /* List of outgoing conflicts */
    
    /* Transaction status */
    bool         prepared;      /* True if transaction is prepared */
    bool         committed;     /* True if transaction committed */
    bool         failed;        /* True if must abort on commit */
} SERIALIZABLEXACT;

/* Conflict tracking for serializable transactions */
void
CheckTargetForSerializableConflict(Relation relation,
                                  ItemPointer tid)
{
    SERIALIZABLEXACT *sxact;
    
    if (MySXactGlobalXmin == InvalidTransactionId)
        return;  /* Not serializable */
        
    /* Check for conflicts with concurrent transactions */
    sxact = FindTargetSXact(relation, tid);
    if (sxact) {
        /* Record the conflict */
        RecordConflict(MySerializableXact, sxact);
        
        /* Check if conflict creates cycle */
        if (DetectSerializationConflict(MySerializableXact))
            ereport(ERROR,
                    (errcode(ERRCODE_T_R_SERIALIZATION_FAILURE),
                     errmsg("could not serialize access due to " 
                           "read/write dependencies among transactions")));
    }
}
```


# Part 6: Query Execution and Results Handling

## Query Executor

### 1. Executor Node Types

```c
/* Basic executor node structure */
typedef struct Node {
    NodeTag     type;          /* Node type identifier */
} Node;

/* Common fields for all plan nodes */
typedef struct Plan {
    NodeTag     type;
    Cost        startup_cost;   /* Cost before first tuple */
    Cost        total_cost;     /* Total cost */
    double      plan_rows;      /* Estimated number of result rows */
    int         plan_width;     /* Estimated average row width */
    List       *targetlist;     /* Target list (projection) */
    List       *qual;          /* Qual conditions (filtering) */
    struct Plan *lefttree;      /* Left tree */
    struct Plan *righttree;     /* Right tree */
    void       *extradata;      /* Extra node-specific data */
} Plan;

/* Common executor node states */
typedef struct PlanState {
    NodeTag     type;           /* Node type */
    Plan       *plan;           /* Associated plan node */
    EState     *state;          /* Shared executor state */
    ExprState  *qual;           /* Qual expr state */
    List       *targetlist;     /* Target list state */
    struct PlanState *lefttree; /* Left tree state */
    struct PlanState *righttree;/* Right tree state */
} PlanState;

/* Example node types for different operations */
typedef struct SeqScanState {
    ScanState   ss;             /* Inherited scan state */
    HeapScanDesc scandesc;      /* Scan descriptor for heap scan */
} SeqScanState;

typedef struct HashJoinState {
    JoinState   js;             /* Inherited join state */
    List       *hashclauses;    /* List of hash clauses */
    HashJoinTable hashtable;    /* Hash table for join */
    bool        hashclauses_computed; /* Hash clauses evaluated? */
} HashJoinState;
```

### 2. Executor State Management

```c
/* Executor state structure */
typedef struct EState {
    NodeTag     type;
    
    /* Transaction and snapshot management */
    TransactionId xactStartTimestamp;
    Snapshot    activeSnapshot;
    
    /* Memory management */
    MemoryContext per_tuple_context;
    MemoryContext estate_context;
    
    /* Result management */
    TupleTableSlot *es_result_relation_info;
    List       *es_rowMarks;
    
    /* Execution state */
    bool        es_instrument;    /* Performance instrumentation? */
    bool        es_finished;      /* Execution finished? */
    int         es_top_eflags;    /* Top-level execution flags */
    
    /* Resource management */
    ResourceOwner es_query_resource_owner;
    
    /* Statistics */
    struct Instrumentation *es_instrument_options;
} EState;

/* Tuple table slot - holds current tuple being processed */
typedef struct TupleTableSlot {
    NodeTag     type;
    bool        tts_isempty;     /* True if slot is empty */
    bool        tts_shouldFree;  /* Should free contents? */
    bool        tts_shouldFreeMin; /* Minimize memory usage? */
    bool        tts_slow;        /* Slow access path? */
    HeapTuple   tts_tuple;       /* Physical tuple */
    TupleDesc   tts_tupleDescriptor; /* Tuple descriptor */
    MemoryContext tts_mcxt;      /* Memory context */
    Buffer      tts_buffer;      /* Buffer containing tuple */
    int         tts_nvalid;      /* Number of valid values */
    Datum      *tts_values;      /* Array of values */
    bool       *tts_isnull;      /* Array of null flags */
} TupleTableSlot;
```

### 3. Query Execution Process

```c
/* Main executor entry point */
static TupleTableSlot *
ExecutePlan(EState *estate,
           PlanState *planstate,
           bool use_parallel_mode,
           CmdType operation,
           bool sendTuples,
           long count,
           ScanDirection direction)
{
    TupleTableSlot *slot;
    uint64      processed = 0;
    
    /* Initialize for execution */
    estate->es_processed = 0;
    estate->es_filtering = false;
    
    /* Execute plan */
    for (;;) {
        /* Get next tuple */
        slot = ExecProcNode(planstate);
        
        if (TupIsNull(slot)) {
            /* No more tuples */
            break;
        }
        
        processed++;
        
        /* Process the tuple */
        if (sendTuples) {
            /* Send to client */
            (void) ExecClearTuple(slot);
        }
        
        /* Check for count limit */
        if (count && processed >= count) {
            break;
        }
    }
    
    /* Return last slot processed */
    return slot;
}

/* Execute a single node */
TupleTableSlot *
ExecProcNode(PlanState *node)
{
    TupleTableSlot *result;
    
    switch (nodeTag(node)) {
        case T_SeqScanState:
            result = ExecSeqScan((SeqScanState *) node);
            break;
            
        case T_HashJoinState:
            result = ExecHashJoin((HashJoinState *) node);
            break;
            
        /* ... other node types ... */
        
        default:
            elog(ERROR, "unrecognized node type: %d",
                 (int) nodeTag(node));
            result = NULL;
            break;
    }
    
    return result;
}
```

### 4. Results Processing and Communication

```c
/* Protocol message types */
typedef enum {
    'T',    /* RowDescription */
    'D',    /* DataRow */
    'C',    /* CommandComplete */
    'Z',    /* ReadyForQuery */
    'E'     /* ErrorResponse */
} ProtocolMessageType;

/* Row description message */
static void
SendRowDescription(TupleDesc typeinfo, List *targetlist, int16 *formats)
{
    StringInfoData buf;
    int         natts = typeinfo->natts;
    int         i;
    
    /* Initialize buffer */
    pq_beginmessage(&buf, 'T');  /* RowDescription */
    
    pq_sendint16(&buf, natts);   /* # of fields */
    
    /* For each column */
    for (i = 0; i < natts; i++) {
        Form_pg_attribute att = TupleDescAttr(typeinfo, i);
        
        pq_sendstring(&buf, NameStr(att->attname));
        pq_sendint32(&buf, att->atttypid);
        pq_sendint16(&buf, att->atttypmod);
        pq_sendint16(&buf, formats[i]);
    }
    
    pq_endmessage(&buf);
}

/* Send a data row */
static void
SendDataRow(TupleTableSlot *slot, DestReceiver *dest)
{
    TupleDesc   typeinfo = slot->tts_tupleDescriptor;
    int         natts = typeinfo->natts;
    StringInfoData buf;
    int         i;
    
    /* Initialize data row message */
    pq_beginmessage(&buf, 'D');  /* DataRow */
    
    pq_sendint16(&buf, natts);   /* # of columns */
    
    /* For each column */
    for (i = 0; i < natts; i++) {
        Datum       value;
        bool        isnull;
        
        /* Get the attribute */
        value = slot_getattr(slot, i + 1, &isnull);
        
        if (isnull) {
            pq_sendint32(&buf, -1);  /* NULL marker */
            continue;
        }
        
        /* Convert and send value */
        convert_and_send_value(value,
                             TupleDescAttr(typeinfo, i)->atttypid,
                             &buf);
    }
    
    pq_endmessage(&buf);
}
```

### 5. Memory Management and Cleanup

```c
/* Memory context hierarchy for query execution */
typedef struct QueryContextData {
    MemoryContext queryContext;     /* Top query context */
    MemoryContext portalContext;    /* Portal-specific context */
    MemoryContext messageContext;   /* Message processing context */
    MemoryContext tupleContext;     /* Tuple processing context */
} QueryContextData;

/* Cleanup after query execution */
static void
ExecutorCleanup(QueryDesc *queryDesc)
{
    EState     *estate;
    
    estate = queryDesc->estate;
    
    /* Cleanup executor state */
    ExecEndPlan(queryDesc->planstate, estate);
    
    /* Free executor state */
    FreeExecutorState(estate);
    
    /* Release snapshot */
    if (estate->es_snapshot)
        UnregisterSnapshot(estate->es_snapshot);
        
    /* Release portal memory context */
    if (queryDesc->portalContext)
        MemoryContextDelete(queryDesc->portalContext);
}

/* Memory context reset */
static void
ResetExprContext(ExprContext *econtext)
{
    /* Reset per-tuple memory context */
    if (econtext->ecxt_per_tuple_memory != NULL)
        MemoryContextReset(econtext->ecxt_per_tuple_memory);
        
    /* Clear computed values */
    econtext->ecxt_tuple_is_virtual = false;
    econtext->ecxt_values_is_dirty = true;
}
```

### 6. Performance Monitoring and Instrumentation

```c
/* Instrumentation data structure */
typedef struct Instrumentation {
    instr_time  starttime;      /* Start time of execution */
    instr_time  counter;        /* Accumulated execution time */
    double      firsttuple;     /* Time for first tuple */
    double      tuplecount;     /* Number of tuples processed */
    bool        running;        /* True if node is running */
    
    /* Buffers */
    long        nblocks_hit;    /* Buffer hits */
    long        nblocks_read;   /* Buffer reads */
    long        nblocks_dirtied;/* Buffers dirtied */
    long        nblocks_written;/* Buffers written */
    
    /* WAL */
    long        wal_records;    /* WAL records generated */
    long        wal_bytes;      /* WAL bytes written */
} Instrumentation;

/* Node execution timing */
static void
InstrStartNode(Instrumentation *instr)
{
    if (instr->running)
        return;                 /* Already running */
        
    INSTR_TIME_SET_CURRENT(instr->starttime);
    instr->running = true;
}

static void
InstrStopNode(Instrumentation *instr, double nTuples)
{
    instr_time  endtime;
    
    if (!instr->running)
        return;                 /* Not running */
        
    INSTR_TIME_SET_CURRENT(endtime);
    
    INSTR_TIME_ACCUM_DIFF(instr->counter,
                         endtime,
                         instr->starttime);
                         
    if (nTuples >= 0)
        instr->tuplecount += nTuples;
        
    instr->running = false;
}

/* Execution statistics */
typedef struct PlanInstrumentation {
    /* Executor statistics */
    double      startup_cost;    /* Startup cost */
    double      total_cost;      /* Total cost */
    double      rows;            /* Number of rows */
    
    /* Time statistics */
    instr_time  startup_time;    /* Time before first tuple */
    instr_time  total_time;      /* Total execution time */
    
    /* Buffer statistics */
    BufferUsage bufusage;        /* Buffer usage stats */
    
    /* WAL statistics */
    WalUsage    walusage;        /* WAL usage stats */
} PlanInstrumentation;
```

[To be continued in Part 7: Special Operations and Extensions...]