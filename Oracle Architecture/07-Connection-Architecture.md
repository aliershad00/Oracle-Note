# Oracle Architecture: Connection Architecture - How Users Connect

## Introduction: Client Meeting Server

When you connect to Oracle, it's not a direct connection to data files. Instead:

- **Client** = Your application (laptop, mobile, web app)
- **Server** = Oracle database server (running background processes)
- Between them = **Connection process** manages the relationship

Think of it like calling a restaurant: You don't talk directly to the chef, but to the receptionist who coordinates everything.

---

## Connection Models

### Model 1: Local Connection (Same Machine)

```
User → Application → Local Oracle Listener
       (same computer)

Example:
├── You run SQL*Plus on the database server
├── Connects directly to Oracle
├── No network involved
└── Fastest connection (local IPC)
```

### Model 2: Remote Connection (Different Machine)

```
User Application       Oracle Server
(Your laptop)          (Database server)
     ↓                      ↓
   SQL*Plus ←→ Network ←→ Oracle Listener
                         (Port 1521)
```

### Real-World Analogy: Phone vs In-Person

```
Local Connection = Talking to someone in the same room
├── Direct communication
├── No intermediary needed
├── Fastest response

Remote Connection = Calling someone on phone
├── Need phone network
├── Signal goes through wires
├── Slight delay
└── Listener acts like receptionist
```

---

## Oracle Listener

### What is the Listener?

The **Listener** is a **background process** that:

- Listens for incoming connection requests
- Runs on the database server
- Accepts connections on a specific port (default: 1521)
- Spawns server processes for each connection

### How Listener Works

```
Step 1: Listener Starting
├── Oracle Listener process started
├── Listens on Port 1521 (TCP/IP)
├── Broadcasts database is available
└── Ready for connections

Step 2: Remote User Connection Request
├── User application: sqlplus scott@PROD
├── Application creates connection string
├── Sends to listener at server:port (proddb:1521)

Step 3: Listener Receives Request
├── Listener checks request
├── Verifies database name (PROD)
├── User credentials not validated here
└── Listener accepts request

Step 4: Listener Creates Server Process
├── Listener spawns new server process
├── Server process created for this user
├── Assigns a unique Process ID (PID)
└── Server process becomes user's connection handler

Step 5: User Connected
├── User now talks to server process
├── Listener returns to listening for new requests
├── User authenticated by server process
├── SQL execution can begin

Result: User has exclusive server process ✓
```

### Listener Commands

```bash
# Start listener
lsnrctl start

# Stop listener
lsnrctl stop

# Check listener status
lsnrctl status

# View listener configuration
lsnrctl show all

# Monitor connections in real-time
lsnrctl services
```

---

## Client-Side Components

### 1. Oracle Client

```
What is Oracle Client?
├── Software installed on user's machine
├── Provides connection tools (SQL*Plus, SQLDeveloper, etc.)
├── Handles network protocol
└── Communicates with server via Listener

Installation:
├── Full Client: All tools, drivers, libraries (~500 MB)
├── Instant Client: Minimal, just connection drivers (~50 MB)
└── ODAC: Microsoft tools for Oracle on Windows

Location Examples:
├── Windows: C:\oracle\product\19c\client_1\
├── Linux: /u01/oracle/product/19c/client/
└── Mac: /Users/username/oracle/client/
```

### 2. Connection String (TNS)

```
What is Connection String?

Simple Format:
server_name:port/database_name
Example: proddb:1521/PROD

Or Using Net Service Name:
PROD

The Connection String tells client:
├── Which server to connect to
├── Which port to use
├── Which database to open
└── Any special parameters
```

### 3. TNS Names

```
TNS = Transparent Network Substrate

TNS File Location:
├── Windows: %ORACLE_HOME%\network\admin\tnsnames.ora
├── Linux: $ORACLE_HOME/network/admin/tnsnames.ora

TNS File Content Example:

PROD =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = proddb.company.com)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = PROD)
    )
  )

DEV =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = devdb.company.com)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = DEV)
    )
  )

Usage:
sqlplus scott/tiger@PROD
→ Oracle looks up PROD in tnsnames.ora
→ Connects to proddb.company.com:1521/PROD
```

---

## Server-Side Components

### 1. Oracle Listener Process

```
Listener Role:
├── Network interface for database
├── Accepts incoming connections
├── Creates server processes
├── Manages connection requests
└── Routes requests to correct database

Listener Responsibilities:
├── Start/stop services
├── Monitor connection activity
├── Log connection attempts
├── Handle redirects
└── Security: Screen bad requests
```

### 2. Server Process

```
What is a Server Process?

Definition:
├── Dedicated process for one user session
├── Created when user connects
├── Destroyed when user disconnects
├── Private to that user

Real-World Analogy:
Restaurant example:
├── Listener = Host/Receptionist
├── Server Process = Waiter assigned to your table
├── Waiter takes your order (SQL command)
├── Waiter brings results (result set)
└── Dedicated to you while dining

Server Process Responsibilities:
├── Parse incoming SQL statements
├── Execute queries
├── Fetch data from SGA
├── Return results to client
├── Manage user session state
├── Enforce security
├── Log audit trails
└── Handle connection termination
```

### 3. User Process vs Server Process

```
Comparison:

User Process (Client-side):
├── Runs on user's machine
├── Displays results to user
├── Sends SQL to server
├── Minimal resources
└── May be GUI or command-line

Server Process (Server-side):
├── Runs on database server
├── Executes SQL
├── Manages memory (PGA)
├── Controls database access
├── Returns results to user
└── Heavier resource consumption
```

---

## Connection Architecture Detailed Flow

### Step-by-Step Connection Process

```
Time 0:00 - User Initiates Connection
├── User types: sqlplus scott/tiger@PROD
├── TNS Resolver looks up "PROD" in tnsnames.ora
└── Gets server address: proddb.company.com:1521

Time 0:01 - Client Sends Connection Request
├── Network transmission to listener
├── Request packet contains:
│   ├── Database name: PROD
│   ├── Service name: PROD
│   └── Protocol: TCP/IP

Time 0:02 - Listener Receives Request
├── Listener checks if database PROD is registered
├── Listener checks if port 1521 is available
├── Listener verifies request is valid
└── Listener accepts connection

Time 0:03 - Listener Creates Server Process
├── Fork/spawn new server process
├── Assign unique Process ID (PID)
├── Initialize PGA for new process
├── Register process in process table

Time 0:04 - Listener Returns Address
├── Listener sends server process address to client
├── Client now has server process location
├── Listener returns to waiting for new connections

Time 0:05 - Direct Connection Established
├── Client now connects directly to server process
├── Bypasses listener (listener no longer needed)
├── User not connected to listener directly

Time 0:06 - Authentication
├── Server process receives username/password
├── Checks user credentials
├── If using OS authentication: Different process
├── Validates privileges
└── Connection established ✓

Time 0:07 - Session Started
├── Server process allocates PGA memory
├── Session parameters set
├── Environment variables initialized
├── User can now submit SQL ✓

Result: User has dedicated server process ✓
```

### Visual Architecture

```
Client Machine                         Server Machine

┌─────────────────┐                ┌──────────────────┐
│   User          │                │  Oracle Instance │
│   (Developer)   │                │                  │
└────────┬────────┘                └──────────────────┘
         │
         │ (1) Connection Request
         │ (sqlplus scott@PROD)
         ├────────────────────────→ Listener Process
         │                         Port 1521
         │
         │ (2) Listener Creates    (Creates Server
         │     Server Process       Process #4521)
         │ (3) Returns Address
         ├←────────────────────────┤
         │
         │ (4) Direct Connection
         ├────────────────────────→ Server Process #4521
         │ (SQL Queries)           (PGA: 200 MB)
         │
         │ ← Results
         ├────────────────────────┤
         │
         │ (5) Query Results
         │ (Displayed on Client)
         │
```

---

## Authentication Methods

### 1. Database Authentication

```
User Password Stored in Database:
├── User/password in Oracle dictionary
├── Stored (hashed) in SYS.USER$ table
├── Server process verifies password
└── Standard connection method

Example:
sqlplus scott/tiger@PROD
├── Username: scott
├── Password: tiger
├── Database: PROD
└── Server validates password against SYS.USER$

Pros: Centralized, standard method
Cons: Password sent over network (use encryption)
```

### 2. Operating System Authentication

```
User Password Managed by OS:
├── Windows: Use Windows login credentials
├── Linux: Use OS user credentials
├── No Oracle password needed
├── More secure (leverages OS security)

Example (Linux):
sqlplus / as sysdba
├── Uses logged-in OS user
├── No password prompt
├── "as sysdba" = connect as administrator
└── Very secure ✓

How It Works:
├── OS authenticates user
├── Oracle trusts OS authentication
├── No password transmission needed
└── Safer for admin accounts
```

### 3. External Authentication (LDAP)

```
User Managed by Company Directory:
├── LDAP/Active Directory
├── Single sign-on (SSO)
├── Enterprise authentication

Example:
sqlplus user@PROD
├── User credentials checked against LDAP
├── No separate Oracle password
├── Company-wide management
└── Enterprise security ✓

Advantages:
├── Centralized user management
├── Single login for all systems
├── Compliance easier
└── Disabled user: Auto-removed from all systems
```

---

## Session State

### What Gets Stored in Session?

```
When user connects, session stores:

1. User Information
   ├── Username
   ├── User privileges
   ├── Roles assigned
   └── Authentication method

2. Environmental Variables
   ├── Date format
   ├── Decimal format
   ├── Language
   ├── Timezone
   └── NLS settings

3. System Variables
   ├── Database name
   ├── Instance name
   ├── Current user
   ├── Session ID
   └── Serial number

4. Work Areas (in PGA)
   ├── Sort area (for ORDER BY)
   ├── Hash area (for hash joins)
   ├── Cursors (open SQL statements)
   └── Context area

5. Transaction State
   ├── Current transaction info
   ├── Undo segment assigned
   ├── Locks held
   └── Dirty blocks modified
```

### Query: View Current Session

```sql
-- View your current session
SELECT SID, SERIAL#, USERNAME, STATUS, TYPE
FROM V$SESSION
WHERE SID = (SELECT SID FROM V$MYSTAT);

-- View all sessions (admin)
SELECT SID, SERIAL#, USERNAME, STATUS, OSUSER, MACHINE
FROM V$SESSION
ORDER BY USERNAME;

-- Kill a session (admin)
ALTER SYSTEM KILL SESSION 'SID,SERIAL#';
```

---

## Connection Pooling

### Problem: Too Many Connections

```
Application Server with 1000 users:

Without Pooling:
├── 1000 users = 1000 server processes
├── Each process: 200 MB PGA
├── Total memory: 1000 × 200 MB = 200 GB ✗
├── Database overwhelmed
└── Server runs out of memory

With Pooling:
├── Connection pool: 100 connections
├── Reused among 1000 users
├── Available connections: ~100
├── Each user gets connection when needed
├── Connection returned to pool when done
├── Total memory: 100 × 200 MB = 20 GB ✓
```

### How Connection Pooling Works

```
Connection Pool (Server-side or Client-side):

Pool Maintenance:
├── Initial connections: 10
├── Minimum connections: 10
├── Maximum connections: 100
├── Idle timeout: 5 minutes

Example Workflow:

T=0:00 - User 1 requests connection
├── Pool gives available connection #5
└── User 1 executes query

T=0:05 - User 2 requests connection
├── Pool gives available connection #7
└── User 2 executes query

T=0:10 - User 1 finishes
├── Returns connection #5 to pool
├── Connection not destroyed
└── Waiting for next user

T=0:15 - User 3 requests connection
├── Pool reuses connection #5
├── Connection already initialized
└── User 3 executes query (FAST)

Benefit: Fast connection reuse ✓
```

---

## Real-World Scenario: Company Bank Database

### Morning Scenario

```
Time 8:00 AM - System Starting

Step 1: Database Server Starts
├── Oracle instance starts
├── SGA allocated: 50 GB
├── Background processes started
└── Database mounted and opened ✓

Step 2: Listener Starts
├── Listener process starts
├── Listens on port 1521
├── Announces availability
└── Ready for connections ✓

Time 8:30 AM - Employees Start Connecting

Employee 1 (Teller): Connects to PROD database
├── Application: Teller terminal
├── Client sends connection request
├── Listener creates server process #1001
├── Employee authenticated
├── PGA allocated: 200 MB
└── Ready for transactions ✓

Employee 2 (Manager): Connects to PROD database
├── Application: Manager workstation
├── Client sends connection request
├── Listener creates server process #1002
├── Manager authenticated with OS auth
├── PGA allocated: 200 MB
└── Ready for reporting ✓

Employee 3-50: Similar connections
├── Each gets own server process
├── Each has own PGA
├── Total server processes: 50
├── Total PGA usage: 50 × 200 MB = 10 GB

Situation: 50 server processes running
├── Listener still listening for new connections
├── Memory: SGA 50 GB + PGA 10 GB = 60 GB
└── System performing well ✓

Time 12:00 PM - Peak Activity

100 employees connected:
├── 100 server processes running
├── 100 × 200 MB = 20 GB PGA
├── All transactions active
├── Some waiting in queue
└── System at 70 GB memory usage

Time 6:00 PM - Employees Disconnect

Employee 1 disconnects:
├── Issues: DISCONNECT;
├── Server process #1001 terminates
├── PGA memory freed: 200 MB
└── Connection closed ✓

Mass Disconnect:
├── 95 of 100 employees disconnect
├── 95 server processes terminate
├── 95 × 200 MB = 19 GB freed
├── Only 5 employees still connected
└── System returns to baseline ✓

Time 9:00 PM - Overnight

System Status:
├── Most employees gone
├── 2-3 background processes running
├── Database idle
├── Listener still listening
├── Ready for night operations or batch jobs
```

---

## Connection Troubleshooting

### Common Connection Problems

```
Problem 1: "Connection Refused"
├── Cause: Listener not running
├── Solution: lsnrctl start

Problem 2: "Unknown Service Name"
├── Cause: TNS not configured
├── Solution: Add entry to tnsnames.ora

Problem 3: "ORA-12514"
├── Cause: Service not registered with listener
├── Solution: Check listener.ora configuration

Problem 4: "Invalid Username/Password"
├── Cause: Wrong credentials
├── Solution: Verify username and password

Problem 5: "Connection Timeout"
├── Cause: Network issues or listener slow
├── Solution: Check network, restart listener

Problem 6: "Too Many Processes"
├── Cause: Processes parameter exceeded
├── Solution: Kill idle sessions or increase PROCESSES parameter
```

---

## Key Takeaways

1. **Listener** = Receptionist accepting connections
2. **Server Process** = Dedicated waiter for each user
3. **Connection string** = Address to connect to
4. **TNS** = Configuration for easy connections
5. **Authentication** = Verify user identity
6. **Session state** = User environment and context
7. **Connection pooling** = Share connections among users
8. **One user = One server process** = Dedicated resources
9. **Listener created once** at startup, serves all connections
10. **Connection is stateful** = Session info preserved
