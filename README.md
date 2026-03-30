# Customised Virtual File System — C++ Systems Project

![C++](https://img.shields.io/badge/C++-17-blue)
![License](https://img.shields.io/badge/License-MIT-green)
![Type](https://img.shields.io/badge/Type-Systems%20Project-red)
![Storage](https://img.shields.io/badge/Storage-In--Memory-yellow)
![Feature](https://img.shields.io/badge/Feature-Linux%20VFS%20Simulation-orange)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)

A custom in-memory Virtual File System (VFS) implemented in C++ that simulates the core internals of a Linux/Unix filesystem. The system replicates the architecture of real operating system file management — including an Incore Inode Table, a System-Wide Open File Table, and a per-process User Area (UAREA) with a User File Descriptor Table — all operating entirely in RAM without any disk I/O.

---

## Problem Statement

Understanding how a real Linux filesystem works internally — how inodes are managed, how file descriptors map to open file entries, and how the kernel tracks per-process file state — is difficult without hands-on implementation. This project builds that entire architecture from scratch in C++, simulating all core Linux VFS data structures and exposing them through familiar shell-style commands, giving a clear, working model of how OS-level file management actually functions.

---

## Project Structure

```
Customised_Virtual_File_System/
│
├── main.cpp                    # Entry point — command loop and dispatcher
├── inode.h / inode.cpp         # Incore Inode Table — metadata for all files and directories
├── filetable.h / filetable.cpp # System-Wide Open File Table — active open file entries
├── uarea.h / uarea.cpp         # UAREA — per-process User File Descriptor Table
├── directory.h / directory.cpp # Directory structure — hierarchical name-to-inode mapping
├── commands.h / commands.cpp   # All CVFS command implementations (ls, mkdir, create, etc.)
│
└── README.md
```

---

## Core Architecture

The CVFS is modelled on the three-tier kernel file management architecture used in Unix/Linux:

```
 User Process
 ┌─────────────────────────────────┐
 │  UAREA — User File Descriptor   │
 │  Table (per process)            │
 │  fd[0], fd[1], fd[2] ...        │
 └──────────────┬──────────────────┘
                │  each fd points to
                ▼
 ┌─────────────────────────────────┐
 │  System-Wide Open File Table    │
 │  (shared across all processes)  │
 │  offset, mode, ref count, ...   │
 └──────────────┬──────────────────┘
                │  each entry points to
                ▼
 ┌─────────────────────────────────┐
 │  Incore Inode Table             │
 │  (one entry per unique file)    │
 │  size, type, permissions, data  │
 └─────────────────────────────────┘
```

All three layers are maintained in memory. No disk reads or writes occur at any point.

---

## Key Components

### 1. Incore Inode Table
The Inode Table is the heart of the VFS. Every file and directory has one inode entry containing all of its metadata.

| Field | Description |
|-------|-------------|
| Inode Number | Unique identifier for each file/directory |
| File Type | Regular file or directory |
| File Size | Current size in bytes |
| Permissions | Read / Write / Execute flags |
| Reference Count | Number of active open file table entries pointing to this inode |
| Data Block | In-memory buffer holding the file's actual content |
| Link Count | Number of directory entries pointing to this inode |
| Timestamps | Created / Modified timestamps |

### 2. System-Wide Open File Table (File Table)
Maintained globally across the entire VFS. Each entry is created when a file is opened and removed when all references are closed.

| Field | Description |
|-------|-------------|
| File Table Index | Unique slot in the open file table |
| Inode Pointer | Reference to the corresponding incore inode |
| Current Offset | Read/write position cursor within the file |
| Open Mode | Read-only, write-only, or read-write |
| Reference Count | Number of file descriptors (from UAREA) pointing to this entry |

### 3. UAREA — User File Descriptor Table
Simulates the per-process file descriptor table maintained in the kernel's User Area for each running process.

| Field | Description |
|-------|-------------|
| File Descriptor (fd) | Integer index returned to the user on open |
| File Table Pointer | Points to the corresponding entry in the System-Wide File Table |
| fd[0], fd[1], fd[2] | Standard stdin, stdout, stderr (reserved, not actively used) |

---

## Supported Operations

| Command | Syntax | Description |
|---------|--------|-------------|
| `mkdir` | `mkdir <dirname>` | Create a new directory |
| `ls` | `ls` | List all files and directories in the current location |
| `create` | `creat <filename>` | Create a new regular file (allocates inode) |
| `open` | `open <filename> <mode>` | Open a file; returns a file descriptor |
| `close` | `close <fd>` | Close an open file descriptor |
| `read` | `read <fd> <bytes>` | Read N bytes from a file at the current offset |
| `write` | `write <fd> <data>` | Write data to a file at the current offset |
| `stat` | `stat <filename>` | Display inode metadata for a file |
| `help` | `help` | Display all available commands |
| clear | 'clear' | It is used to clear the shell
| `exit` | `exit` | Quit the VFS shell |

---

## How It Works — Step by Step

**Step 1 — Initialisation**
The VFS initialises the root directory with inode number 0, sets up the Incore Inode Table, clears the System-Wide File Table, and resets the UAREA file descriptor array.

**Step 2 — File / Directory Creation**
When `create` or `mkdir` is called, a new inode is allocated from the Incore Inode Table and a directory entry is added mapping the name to the new inode number.

**Step 3 — Opening a File**
`open` looks up the file's inode in the Incore Inode Table, creates a new entry in the System-Wide Open File Table (with offset = 0 and the specified mode), and returns a file descriptor (fd) by adding the File Table entry reference into the UAREA.

**Step 4 — Read / Write**
`read` and `write` use the fd to find the File Table entry, which holds the current offset and inode pointer. Data is read from or written to the inode's in-memory data buffer, and the offset is advanced accordingly.

**Step 5 — Closing a File**
`close` removes the fd from the UAREA, decrements the reference count in the File Table entry, and if the count reaches zero, removes the File Table entry and decrements the inode's reference count.

**Step 6 — Deletion**
`rm` removes the directory entry and decrements the inode's link count. If the link count and reference count both reach zero, the inode is freed from the Incore Inode Table and its memory is released.

---

## Data Flow Summary

```
User Command: open("notes.txt", READ)
        │
        ▼
Directory Lookup → finds inode number (e.g., inode #4)
        │
        ▼
Incore Inode Table → fetches inode #4 metadata
        │
        ▼
System-Wide File Table → creates new entry { inode #4, offset=0, mode=READ, refcount=1 }
        │
        ▼
UAREA → assigns next free fd (e.g., fd=3), stores pointer to File Table entry
        │
        ▼
Returns fd=3 to user
```

---

## How to Compile and Run

```bash
# Clone the repository
git clone https://github.com/<your-username>/Customised_Virtual_File_System.git
cd Customised_Virtual_File_System

# Compile with g++
g++ CVFS.cpp -o CVFS

# Run the CVFS shell
./CVFS
```

---

## Example Session

```
CVFS Shell $ mkdir documents
Directory 'documents' created successfully.

CVFS Shell $ cd documents
Changed directory to: documents

CVFS Shell $ create report.txt
File 'report.txt' created. Inode #3 allocated.

CVFS Shell $ open report.txt 2
File opened. File Descriptor: 3

CVFS Shell $ write 3 HelloWorld
10 bytes written to fd 3.

CVFS Shell $ read 3 10
Data: HelloWorld

CVFS Shell $ stat report.txt
Inode   : 3
Type    : Regular File
Size    : 10 bytes
Links   : 1
Refs    : 1
Mode    : Read-Write

CVFS Shell $ close 3
File descriptor 3 closed.

CVFS Shell $ exit
Thank You for Using This Application...
```

---

## Key Concepts Covered

- Unix/Linux three-tier file management architecture (UAREA → File Table → Inode Table)
- Incore Inode allocation, reference counting, and deallocation
- System-wide Open File Table management with shared offset tracking
- Per-process file descriptor table simulation (UAREA)
- Hierarchical directory structure with name-to-inode mapping
- File operations: open, close, read, write, stat, unlink
- Reference count-based inode lifecycle management
- In-memory buffer management for file content storage
- Command-line shell loop with argument parsing in C++

---

## Tech Stack

- C++ 17
- Standard Template Library (STL) — `vector`, `map`, `string`, `unordered_map`
- Object-Oriented Design — classes for Inode, FileTableEntry, UAREA, and Directory
- No external libraries — pure C++ implementation

---

## Author

**Laxmi Rahul Rathod**

Built as a C++ Systems Project to understand and replicate the internal architecture of a Linux Virtual File System — including incore inode management, the system-wide open file table, and the per-process UAREA file descriptor table — using pure in-memory data structures.
