# What Happens When You Install PostgreSQL 16 on Ubuntu

This document explains in detail what happens when you run `sudo apt install postgresql-16` on Ubuntu. We'll examine everything from command processing to having a running PostgreSQL server.

> Note: Skip to the end for a quick overview.

## Command Processing

When you hit Enter after typing `sudo apt install postgresql-16`:

1. The shell processes the command:
   - Tokenizes input into: `sudo`, `apt`, `install`, `postgresql-16`
   - Locates `sudo` executable in PATH (typically `/usr/bin/sudo`)
   - Creates a child process to run this command:
     ```c
     pid = fork()
     if (pid == 0) {
        // Execute `/usr/bin/sudo` in child process
        execve("/usr/bin/sudo", 
           [
              "sudo",             // argv[0] - program name
              "apt",              // argv[1] - first argument
              "install",          // argv[2] - second argument
              "postgresql-16",    // argv[3] - third argument
              NULL               // argv[4] - array terminator
           ],
           env                    // Current environment variables
        );
     } else {
        // Parent process (bash)
        waitpid(pid, &status, 0);
     }
     ```

2. `sudo` verifies user permissions and elevates the process privileges to root:
   1. Reads main sudoers file `/etc/sudoers`:
      ```bash
      # User privilege specification
      root    ALL=(ALL:ALL) ALL

      # Allow members of group sudo to execute any command
      %sudo   ALL=(ALL:ALL) ALL

      # Example user entry
      username ALL=(ALL:ALL) ALL

      @includedir /etc/sudoers.d
      ```
   2. Reads additional configurations `/etc/sudoers.d/*`:
      ```bash
      # User rules for ubuntu
      ubuntu ALL=(ALL) NOPASSWD:ALL
      ```
   3. If the user has sufficient privileges to run this command, the system changes the UID (user ID) from current user (UID=1000) to root (UID=0), giving it root privileges for the duration of the process.

3. Sudo executes `apt`:
   ```c
   pid_t childPid = fork();
   if (childPid == 0) {
      // Execute `/usr/bin/apt` in child process
      execve("/usr/bin/apt",
         [
            "apt",              // argv[0] - program name
            "install",          // argv[1] - first argument
            "postgresql-16",    // argv[2] - second argument
            NULL               // argv[3] - array terminator
         ],
         cleaned_env           // Sanitized env vars with updated $USER (root)
      );
   } else {
      // Parent process (sudo)
      waitpid(childPid, &status, 0);
   }
   ```

**Note:**
1. Parent process waits for child process to finish with `waitpid()`
2. Child process inherits stdout/stderr, allowing output to terminal

### Process Hierarchy
```bash
# Before sudo:
bash (UID=1000)            # bash running with current user ID (1000)
└── sudo (UID=1000 → 0)    # sudo starts with current UID (1000) but escalates to root (0)

# After sudo:
bash (UID=1000)            # Bash running with current UID (1000)
└── sudo (UID=0)           # sudo running with root UID (0)
    └── apt (UID=0)        # apt starts with root UID (0)
```

## APT Package Resolution

To install PostgreSQL, APT (Advanced Package Tool) needs to:
1. Find where to download it from
2. Make sure the download source is trustworthy
3. Figure out what additional software (dependencies) is needed
4. Check if there's enough space
5. Prepare for download and ask for user input
6. Download the packages and verify checksums

### 1. Repository Configuration Processing

APT reads its "address book" of software sources. APT needs to know which servers have the software it needs to install.

1. Read sources configuration:
   ```yaml
   # Primary source file
   /etc/apt/sources.list.d/ubuntu.sources:

   Types: deb
   URIs: http://archive.ubuntu.com/ubuntu/
   Suites: noble noble-updates noble-backports
   Components: main universe restricted multiverse
   Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
   ```

2. Build repository URLs:
   ```bash
   # URL Pattern:
   {URI}/dists/{Suite}/{Component}/binary-{Architecture}/Packages.gz

   # Actual URLs:
   http://archive.ubuntu.com/ubuntu/dists/noble/main/binary-amd64/Packages.gz
   http://archive.ubuntu.com/ubuntu/dists/noble/universe/binary-amd64/Packages.gz
   http://archive.ubuntu.com/ubuntu/dists/noble-updates/main/binary-amd64/Packages.gz
   ...
   ```

### 2. Repository Authentication and Metadata

APT downloads a Release file and verifies it's really from Ubuntu. 

The InRelease file contains repository metadata (checksums) and is signed by a GPG key to ensure its authenticity. APT fetches this file from the repository and verifies its signature before trusting the metadata for package installation or upgrade.

1. Download and verify release files:
   ```bash
   # Download InRelease file
   # URL: http://archive.ubuntu.com/ubuntu/dists/{Suite}/InRelease
   curl http://archive.ubuntu.com/ubuntu/dists/noble/InRelease
   
   # Content excerpt of InRelease:
   -----BEGIN PGP SIGNED MESSAGE-----
   Hash: SHA512
   
   Origin: Ubuntu
   Label: Ubuntu Noble
   Suite: noble
   Version: 24.04
   Codename: noble
   Date: Thu, 14 Mar 2024 12:04:32 UTC
   Architectures: amd64 arm64 armhf i386
   Components: main restricted universe multiverse
   Description: Ubuntu Noble 24.04
   
   MD5Sum:
      262e2d...          7165069 main/binary-amd64/Packages
      61fdc2...          1808488 main/binary-amd64/Packages.gz
      d9a7b0...          1401160 main/binary-amd64/Packages.xz
   SHA1:
      b81dce...          7165069 main/binary-amd64/Packages
      41679a...          1808488 main/binary-amd64/Packages.gz
      cfb5fc...          1401160 main/binary-amd64/Packages.xz
   SHA256:
      8f6f71...          7165069 main/binary-amd64/Packages
      e0d7e4...          1808488 main/binary-amd64/Packages.gz
      2a6a19...          1401160 main/binary-amd64/Packages.xz
   -----BEGIN PGP SIGNATURE-----
   [signature data]
   -----END PGP SIGNATURE-----
   ```

2. Verify repository InRelease file signature:
   ```bash
   # Using Ubuntu's keyring
   gpgv \
      --keyring=/usr/share/keyrings/ubuntu-archive-keyring.gpg \            # Official Ubuntu public keys to sign package metadata
      /var/lib/apt/lists/archive.ubuntu.com_ubuntu_dists_noble_InRelease    # Location of InRelease file
   
   # If verification fails:
   E: Release file not valid yet (invalid for another Xh XXmin XXs)
   # or
   E: GPG error: http://archive.ubuntu.com noble InRelease: Invalid signature
   ```

### 3. Package List Processing

APT downloads and reads detailed information about all available packages to find what exactly it needs and its specifications.

1. APT downloads the `Packages.gz` to `/var/lib/apt/lists` and verifies the data integrity using checksums in InRelease file.

2. Parse `Packages.gz` repository downloaded above:
   ```yaml
   # Example package entry - like a product description:
   # Entry taken from http://archive.ubuntu.com/ubuntu/dists/noble/main/binary-amd64/Packages.gz
   Package: postgresql-16
   Architecture: amd64
   Version: 16.2-1ubuntu4
   Priority: optional
   Section: database
   Origin: Ubuntu
   Maintainer: Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com>
   Original-Maintainer: Debian PostgreSQL Maintainers <team+postgresql@tracker.debian.org>
   Bugs: https://bugs.launchpad.net/ubuntu/+filebug
   Installed-Size: 43800
   Provides: postgresql-16-jit-llvm (= 17), postgresql-contrib-16
   Depends: locales | locales-all, postgresql-client-16, postgresql-common (>= 252~), ssl-cert, tzdata, debconf (>= 0.5) | debconf-2.0, libc6 (>= 2.38), libgcc-s1 (>= 3.3.1), libgssapi-krb5-2 (>= 1.14+dfsg), libicu74 (>= 74.1-1~), libldap2 (>= 2.6.2), libllvm17t64, liblz4-1 (>= 0.0~r130), libpam0g (>= 0.99.7.1), libpq5 (>= 16~~), libselinux1 (>= 3.1~), libssl3t64 (>= 3.0.0), libstdc++6 (>= 5.2), libsystemd0, libuuid1 (>= 2.16), libxml2 (>= 2.7.4), libxslt1.1 (>= 1.1.25), libzstd1 (>= 1.5.5), zlib1g (>= 1:1.1.4)
   Recommends: sysstat
   Breaks: dbconfig-common (<< 2.0.22~)
   Filename: pool/main/p/postgresql-16/postgresql-16_16.2-1ubuntu4_amd64.deb
   Size: 15452168
   MD5sum: a0ddd38a3ce650de38cbd1fa4960861f
   SHA1: 55098507be06aa3bbdb7d0fef855338c6195e0fd
   SHA256: 7f25383e8f39ca99f4886b7b2cbb31c62534ac924e3a61da3f0e8e6456091e69
   SHA512: 181d452698e0895c5f54d916a4ee2f2bf6899175e9f5a7d0049723026db9360da0a3f4001f34e3015bc0e0d90d92a63eb161b50d62085cc2c87f0a559e317fe2
   Homepage: http://www.postgresql.org/
   Description: The World's Most Advanced Open Source Relational Database
   Task: postgresql-server
   Postgresql-Catversion: 202307071
   Description-md5: d7367291c15795e56aa78252604f9ced
   ```

### 4. Dependency Resolution

APT figures out what other software PostgreSQL needs to work properly.

1. Build dependency graph:
   ```bash
   postgresql-16
   ├── postgresql-client-16    # Needed to connect to database
   │   ├── libpq5             # Core PostgreSQL library
   │   └── ssl-cert           # For secure connections
   ├── postgresql-common      # Shared PostgreSQL utilities
   └── System libraries       # Basic system requirements
   ```

2. Version Resolution: Check the availability and details of the package `postgresql-16` in the APT repositories configured on the system:
   ```bash
   $ apt-cache policy postgresql-16
   # Output format
   postgresql-16:
      Installed: <installed_version>     # Installed version (if any)
      Candidate: <candidate_version>     # Version chosen to install (higher priority or version)
      Version table:                     # Lists all available versions of postgresql-16
         <version_num> <priority>        # Default priority is 500
            <priority> <repository_url> <distribution>/<repository_section> <architecture> <file>

   # Actual output
   postgresql-16:
      Installed: (none)
      Candidate: 16.4-0ubuntu0.24.04.2
      Version table:
         16.4-0ubuntu0.24.04.2 500
            500 http://ap-south-1.ec2.archive.ubuntu.com/ubuntu noble-updates/main amd64 Packages
         16.4-0ubuntu0.24.04.1 500
            500 http://security.ubuntu.com/ubuntu noble-security/main amd64 Packages
         16.2-1ubuntu4 500
            500 http://ap-south-1.ec2.archive.ubuntu.com/ubuntu noble/main amd64 Packages
   ```

### 5. Package Download Preparation

APT calculates total download size and checks if you have enough space.

1. Calculate requirements:
   ```bash
   # Create download queue
   Need to get 23.1 MB of archives.
   After installation: 85.3 MB will be used.
   
   Download Queue:
   1. postgresql-16          (16.2 MB)
   2. postgresql-client-16   (4.8 MB)
   3. postgresql-common      (2.1 MB)
   ```

2. Check free space:
   ```bash
   # Verify space in:
   /var/cache/apt/archives/     # Temporary download storage
   /var/lib/postgresql/         # Where PostgreSQL will live
   /etc/postgresql/            # Where settings will be stored
   ```

3. Prepare download locations:
   ```bash
   # Create/verify directories
   mkdir -p /var/cache/apt/archives/partial
   
   # Set up download resume markers
   touch /var/cache/apt/archives/partial/postgresql-16_16.2-1ubuntu1_amd64.deb.dl
   ```

At this point, APT has:
1. Verified repository authenticity
2. Located all required packages
3. Resolved dependencies
4. Calculated disk space requirements
5. Prepared for download

The next phase begins once the user confirms the installation prompt.

### 6. Package Download

1. Download packages:
   ```bash
   # Download to cache
   /var/cache/apt/archives/
   └── partial/                  # Temporary download location
       └── postgresql-16_16.2-1ubuntu1_amd64.deb
   
   # After download completion, move to
   /var/cache/apt/archives/postgresql-16_16.2-1ubuntu1_amd64.deb
   ```

2. Verify packages:
   ```bash
   # Check SHA256 sum against Packages file
   sha256sum postgresql-16_16.2-1ubuntu1_amd64.deb

   # Verify package signatures if available
   dpkg-sig --verify postgresql-16_16.2-1ubuntu1_amd64.deb
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
   /etc/postgresql/16/          # Configuration files
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
   shared_buffers       = 128MB   # 25% reduced for small RAM
   work_mem             = 4MB     # Per operation memory
   maintenance_work_mem = 64MB    # For maintenance operations
   wal_buffers          = 4MB     # WAL writing buffer
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

1. PostgreSQL Server State:
   - Running as service (postgresql@16-main)
   - Listening on port 5432
   - Managing shared memory segments
   - Processing client connections
   - Writing to WAL files
   - Performing background maintenance

2. System Configuration:
   - Default databases created
   - System catalog populated
   - PostgreSQL superuser account configured
   - Basic security settings applied
   - Logging enabled

The installation process involves multiple components:
- APT handles package management
- dpkg handles package extraction
- pg_createcluster initializes the database
- systemd manages the service
- postmaster manages the running instance

System Requirements and Performance:
- Installation time: 2-5 minutes on a system with 1 vCPU
- Initial disk usage: ~100MB
- Runtime memory usage: ~200MB
- Idle CPU usage: <1%



## TODO
- [ ] Add a sequence diagram.
- [ ] Review postgres preinst, installation, postinst steps and add more details.