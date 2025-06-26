# KTP - Custom Transport Protocol over UDP

**Emulating End-to-End Reliable Flow Control over UDP**

---

## 📁 Project Structure

The codebase is organized as follows:

- **initksocket.c**  
  Initializes shared memory and sets up the environment for socket creation and management.

- **ksocket.h**  
  Contains data structure definitions, constants, and function prototypes used throughout the socket library.

- **ksocket.c**  
  Implements all core socket operations: creation, binding, sending, receiving, and closure. This file also includes the sliding window protocol and error-handling logic.

---

## 📦 Data Structures

### 1. Sender & Receiver Window Structures (`swindow`, `rwindow`)

**Purpose**: Implements a sliding window mechanism for efficient packet transmission and acknowledgment tracking.

**Key Fields**:
- `size`: Window size (max number of unacknowledged messages).
- `base`: Starting sequence number of the active window.
- `next_seq`:  
  - Sender: next sequence number to assign.  
  - Receiver: next expected in-order sequence.
- `seq_nums[256]`:  
  - Sender: `-1` for acknowledged or unsent messages.  
  - Receiver: `-1` for unexpected packets; positive values for valid ones.

---

### 2. Socket Management Entry (`SM_entry`)

**Purpose**: Manages the state of a socket and its associated buffers in shared memory.

**Components**:

- **Connection Info**:
  - `is_free`: Marks availability of the entry.
  - `process_id`: Process using the socket.
  - `udp_sockfd`: Underlying UDP socket file descriptor.

- **Address Info**:
  - `src_addr`: Local IP and port.
  - `dest_addr`: Remote IP and port.

- **Send Buffer**:
  - `send_buf[10][512]`, `send_len[10]`
  - `send_count`: Count of unacknowledged messages.
  - `send_tail`: Index to append new messages.

- **Receive Buffer**:
  - `recv_buf[10][512]`, `recv_len[10]`, `flags[10]`
  - `recv_head`: Index to read next message.
  - `recv_count`, `recv_tail`: Helpers to simplify logic.

- **Window Managers**:
  - `swnd`, `rwnd`: Sender and receiver windows.
  - `timers[10]`: Stores send timestamps.
  - `nospace`: True if receive buffer is full.
  - `socket_request`, `bind_request`: Ensure one-time setup.

- **Acknowledgement & Statistics**:
  - `last_ack_seq`: Last acknowledged sequence number.
  - `total_transmissions`: Counts all transmissions (including retransmissions).

---

## 🚀 Core Functionalities

### 1. `k_socket()`: Socket Initialization
- Allocates a free `SM_entry` from shared memory.
- Initializes buffers and window parameters.
- Validates socket type and configures defaults.

---

### 2. `k_sendto()`: Data Transmission
- Assigns sequence numbers using `swindow`.
- Sends messages and tracks ACKs.
- Handles retransmissions using timers and timeouts.

---

### 3. `k_recvfrom()`: Data Reception
- Buffers and reorders out-of-order packets.
- Updates `expected_seq` when packets arrive in order.
- Delivers in-sequence packets to the application layer.

---

### 4. `k_bind()` and `k_close()`: Binding and Cleanup
- Binds socket to local and remote addresses.
- Cleans up resources and resets state on closure.

---

### 5. `dropMessage()`: Packet Loss Simulation
- Simulates packet drops with configurable probability.
- Enables testing of retransmission logic and reliability.

---

## 🧠 Shared Memory & Threads

### Shared Memory
- Shared area stores all `SM_entry` structures and their associated buffers.
- Enables concurrent multi-process access and control.

### Threads
- **Receiver Thread**: Continuously receives packets and processes them.
- **Sender Thread**: Manages timeouts and retransmissions.
- **Garbage Collector**: Frees unused socket resources.

---

## 📊 Transmission Statistics

| Probability (P) | Original Transmissions | Total Transmissions | Avg Transmissions per Packet |
|-----------------|------------------------|---------------------|-------------------------------|
| 0.05            | 201                    | 239                 | 1.19                          |
| 0.10            | 201                    | 285                 | 1.42                          |
| 0.15            | 201                    | 317                 | 1.58                          |
| 0.20            | 201                    | 355                 | 1.77                          |
| 0.25            | 201                    | 414                 | 2.06                          |
| 0.30            | 201                    | 492                 | 2.45                          |
| 0.35            | 201                    | 518                 | 2.58                          |
| 0.40            | 201                    | 535                 | 2.66                          |
| 0.45            | 201                    | 548                 | 2.73                          |
| 0.50            | 201                    | 675                 | 3.36                          |

---

## 🧪 Example Usage

To test the full protocol stack:

```bash
make clean
make
./initksocket
./user2 127.0.0.1 5001 127.0.0.1 5002 output.txt
./user1 127.0.0.1 5002 127.0.0.1 5001 input.txt
