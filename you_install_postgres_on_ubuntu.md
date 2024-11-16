# What happens when... you install PostgreSQL 16 on Ubuntu

This document attempts to answer in detail what happens when you run `sudo apt install postgresql-16` on Ubuntu. We'll examine everything from command processing to having a running PostgreSQL server.

## Command Processing

When you hit Enter after typing `sudo apt install postgresql-16`:

1. The shell processes the command:
   - Tokenizes input into: `sudo`, `apt`, `install`, `postgresql-16`
   - Finds `sudo` in PATH (typically `/usr/bin/sudo`)
   - `sudo` reads `/etc/sudoers` for permission verification
   - If authorized, spawns new process with elevated privileges

2. `sudo` executes `apt`:
   - Creates new process via `fork()`
   - Replaces process image via `execve("/usr/bin/apt", args, env)`
   - Parent shell waits via `waitpid()`
   - Child process (apt) inherits stdout/stderr, allowing output to terminal

## APT Package Resolution

1. Repository Configuration:
   ```bash
   # APT reads sources from
   /etc/apt/sources.list
   /etc/apt/sources.list.d/*.sources
   
   # Example source entry
   Types: deb
   URIs: http://apt.postgresql.org/pub/repos/apt/
   Suites: noble-pgdg
   Components: main
   Signed-By: /usr/share/postgresql-common/pgdg/apt.postgresql.org.gpg
   ```

2. Package Metadata Resolution:
   ```bash
   # APT downloads and verifies
   http://apt.postgresql.org/pub/repos/apt/dists/noble-pgdg/InRelease
   http://apt.postgresql.org/pub/repos/apt/dists/noble-pgdg/main/binary-amd64/Packages.gz
   
   # InRelease file contains
   Origin: PostgreSQL
   Label: PostgreSQL
   Suite: noble-pgdg
   Version: 16.2-1
   Codename: noble-pgdg
   Date: Thu, 14 Mar 2024
   Architectures: amd64 arm64
   Components: main
   Description: PostgreSQL APT Repository
   ```

3. Dependency Resolution:
   ```plaintext
   postgresql-16
   ├── postgresql-client-16 (= 16.2-1)
   │   ├── libpq5 (>= 16.0)
   │   └── ssl-cert
   ├── postgresql-common (>= 241)
   │   ├── postgresql-client-common
   │   ├── ucf
   │   └── lsb-base
   └── system libraries
       ├── libc6 (>= 2.33)
       ├── libgssapi-krb5-2
       ├── libicu66
       ├── libldap-2.4-2
       ├── libpam0g
       ├── libssl3
       └── libsystemd0
   ```

## Package Download and Verification

1. Package Download:
   ```bash
   # Downloads to
   /var/cache/apt/archives/
   ├── postgresql-16_16.2-1.pgdg24.04+1_amd64.deb
   ├── postgresql-client-16_16.2-1.pgdg24.04+1_amd64.deb
   └── postgresql-common_241.pgdg24.04+1_all.deb
   ```

2. Package Verification:
   ```bash
   # Steps:
   1. Verify GPG signature on InRelease
   2. Check SHA256 sums from InRelease
   3. Verify package signatures
   4. Check package dependencies
   ```

## Installation Process

1. Pre-installation:
   ```bash
   # Create system user/group
   useradd --system \
           --home /var/lib/postgresql \
           --shell /bin/bash \
           postgres
   
   # Create directories
   mkdir -p /etc/postgresql/16/main
   mkdir -p /var/lib/postgresql/16/main
   mkdir -p /var/log/postgresql
   ```

2. Package Extraction:
   ```bash
   # Extract package contents
   dpkg-deb -x postgresql-16_16.2-1.deb /
   
   # Main directories:
   /etc/postgresql/16/          # Config files
   /usr/lib/postgresql/16/      # Binaries
   /usr/share/postgresql/16/    # Shared files
   /var/lib/postgresql/16/      # Data directory
   ```

3. Database Cluster Creation:
   ```bash
   # pg_createcluster execution
   1. Initialize data directory (initdb)
   2. Generate configuration files
   3. Set up system tables
   4. Configure authentication
   ```

## System Service Setup

1. Systemd Configuration:
   ```bash
   # Create service files
   /lib/systemd/system/postgresql@.service    # Service template
   /lib/systemd/system/postgresql.service     # Main service
   
   # Enable service
   systemctl enable postgresql@16-main
   systemctl start postgresql@16-main
   ```

2. Process Hierarchy:
   ```plaintext
   systemd
   └── postgres (postmaster)
       ├── postgres: logger process
       │   └── Writes to /var/log/postgresql/postgresql-16-main.log
       ├── postgres: checkpointer process
       │   └── Flushes dirty buffers to disk
       ├── postgres: background writer
       │   └── Writes dirty shared buffers
       ├── postgres: WAL writer
       │   └── Writes WAL buffers to disk
       ├── postgres: autovacuum launcher
       │   └── Cleans up dead tuples
       ├── postgres: stats collector
       │   └── Collects database statistics
       └── postgres: logical replication launcher
           └── Handles logical replication
   ```

## Resource Allocation

1. Memory Allocation:
   ```plaintext
   # For 1GB RAM system:
   shared_buffers      = 128MB   # 25% reduced for small RAM
   work_mem           = 4MB     # Per operation memory
   maintenance_work_mem = 64MB    # For maintenance operations
   wal_buffers        = 4MB     # WAL writing buffer
   ```

2. Disk Organization:
   ```plaintext
   /var/lib/postgresql/16/main/
   ├── base/                    # Databases
   │   ├── 1/                  # template1
   │   ├── 13067/              # template0
   │   └── 13068/              # postgres
   ├── global/                 # Shared objects
   ├── pg_wal/                # Write-ahead logs
   ├── pg_xact/               # Transaction status
   └── postgresql.conf        # Configuration
   ```

## Database Initialization

1. System Catalog Creation:
   ```sql
   # Creates system databases
   CREATE DATABASE template0;
   CREATE DATABASE template1;
   CREATE DATABASE postgres;
   
   # Creates system tables
   CREATE TABLE pg_class;    # Relation catalog
   CREATE TABLE pg_attribute; # Column catalog
   CREATE TABLE pg_proc;     # Function catalog
   ```

2. User Authentication Setup:
   ```bash
   # pg_hba.conf configuration
   local   all   postgres               peer
   local   all   all                    peer
   host    all   all     127.0.0.1/32   md5
   ```

## Final State

After installation completes:

1. PostgreSQL server is:
   - Running as service (postgresql@16-main)
   - Listening on port 5432
   - Managing shared memory segments
   - Processing client connections
   - Writing to WAL files
   - Performing background maintenance

2. System is configured with:
   - Default databases created
   - System catalog populated
   - PostgreSQL superuser account
   - Basic security settings
   - Logging enabled

The installation process involves multiple stages, each performed by different components:
- APT handles package management
- dpkg handles package extraction
- pg_createcluster initializes the database
- systemd manages the service
- postmaster manages the running instance

On a system with 1 vCPU, 1GB RAM, and 24GB SSD:
- Installation takes 2-5 minutes
- Uses ~100MB disk initially
- Consumes ~200MB RAM when running
- Uses minimal CPU at idle (<1%)