# Distributed Job Execution System

![Java](https://img.shields.io/badge/Language-Java_17-blue?logo=java)
![License](https://img.shields.io/badge/License-MIT-green)

This project implements a **distributed job execution platform** with a central coordinator server and a pool of worker nodes. Clients submit computation jobs (bytecode) with a declared memory requirement, and the system schedules and executes them across available secondary servers using memory-aware scheduling.

---

## Goal

To build a concurrent and distributed system where:
- Clients can register and authenticate securely;
- Clients submit jobs (binary programs) with a memory requirement to a central server;
- The server queues jobs and dispatches them to worker nodes based on available memory;
- Clients can query the current system status (pending jobs and total available memory);
- Worker nodes execute jobs in parallel and return results to the originating client.

---

## Architecture

```
Client(s) ──── TCP (port 1234) ────► Main Server ──── TCP ────► Secondary Server A (port 12345)
                                          │                    ► Secondary Server B (port 12346)
                                          │                    ► Secondary Server C (port 12347)
                                          │
                                    Job Queue (memory-aware scheduling)
```

### Components

| Component | Description |
|-----------|-------------|
| **Main Server** | Central coordinator on port `1234`. Handles client auth, manages job queue, dispatches jobs to worker nodes. |
| **Secondary Server** | Worker node on a configurable port. Receives jobs from the main server, executes them using `sd23.jar`, and returns results. |
| **Client** | CLI application. Supports register/login, job submission (with memory size), and server status queries. |
| **cmd (shared)** | Common utilities: `Connection`, `Message`, `Job`, `Memory` — used across all components. |

### Key Design Details

- **Memory-aware scheduling**: secondary servers are stored in a max-heap sorted by available memory; the job manager always picks the most available worker.
- **Round-robin with quantum**: the job queue uses a quantum-based policy to prevent starvation, cycling through jobs until they reach execution eligibility.
- **Priority re-queuing**: jobs that cannot be served yet (due to memory constraints) are re-queued with priority signalling via `Condition` variables.
- **Multiplexed connections**: clients use a `Demultiplexer` to handle concurrent responses over a single TCP connection, identified by message tags.
- **Thread safety**: all shared state is guarded with `ReentrantLock` / `ReentrantReadWriteLock`.
- **Password security**: user passwords are stored as SHA-1 hashes.

---

## Tech Stack

- Java 17
- Raw TCP sockets (no framework)
- `sd23.jar` — job execution library (provided)
- `java.util.concurrent` — locks, conditions, thread management

---

## Directory Structure

```
src/
├── server/               # Main server (coordinator)
│   ├── Main.java
│   ├── Server_Protocol.java
│   ├── Server.java
│   ├── ClientHandler.java
│   ├── JobManager.java
│   ├── JobList.java
│   ├── JobOrder.java
│   ├── JobExecute.java
│   ├── SSQueue.java
│   ├── SSdata.java
│   └── UserInfoManagement.java
├── SecundaryServer/      # Worker nodes
│   ├── Main.java
│   ├── Secundary_Server.java
│   └── SSJobExecute.java
├── cliente/              # Client application
│   ├── Main.java
│   ├── Client.java
│   ├── Demultiplexer.java
│   ├── Task4.java
│   └── Task5.java
├── cmd/                  # Shared utilities
│   ├── Connection.java
│   ├── Message.java
│   ├── Job.java
│   └── Memory.java
├── sd23.jar              # Job execution dependency
├── mainServerStarter.sh
├── secundaryServerStarter.sh
└── tester4ubuntu.sh
```

---

## Running the System

### Prerequisites

- Java 17+
- `sd23.jar` present in the `src/` directory

### 1. Compile all sources

```bash
cd src
javac -cp sd23.jar SecundaryServer/*.java cmd/*.java server/*.java cliente/*.java
```

### 2. Start the Secondary Servers

Each secondary server requires a unique port number:

```bash
java -cp sd23.jar:. SecundaryServer.Main 12345 &
java -cp sd23.jar:. SecundaryServer.Main 12346 &
java -cp sd23.jar:. SecundaryServer.Main 12347 &
```

Or use the provided script (Linux):

```bash
bash secundaryServerStarter.sh
```

### 3. Start the Main Server

Pass the IP and port of each secondary server as pairs of arguments:

```bash
java -cp sd23.jar:. server.Main 127.0.0.1 12345 127.0.0.1 12346 127.0.0.1 12347
```

Or use the provided script (Linux):

```bash
bash mainServerStarter.sh
```

### 4. Start the Client

```bash
java -cp sd23.jar:. cliente.Main
```

The client connects to `127.0.0.1:1234` and presents a menu to register, log in, submit jobs, or query server status.

### 5. Run the Load Tester (Linux)

Spawns 5 concurrent client sessions, each submitting a job:

```bash
bash tester4ubuntu.sh
```

---

## Client Message Protocol

| Code | Direction | Meaning |
|------|-----------|---------|
| `1` | C → S | Authenticate (login) |
| `2` | C → S | Register |
| `3` | C → S | Submit job |
| `4` | S → C | Login failed |
| `5` | S → C | Login successful |
| `6` | S → C | Register failed (username taken) |
| `7` | S → C | Register successful |
| `10` | S → C | Job rejected (exceeds max memory) |
| `12` | C ↔ S | Status request / response |

---

## Contributors

Developed by students from the University of Minho as part of the **Distributed Systems** curricular unit — Group 11.

| Name | Univ. ID |
|------|----------|
| [Gonçalo Marinho](https://github.com/gmarinhog165) | A90969 |
| [Henrique Vaz](https://github.com/Vaz7) | A95533 |

---

## License

This project is licensed under the **MIT License**.
