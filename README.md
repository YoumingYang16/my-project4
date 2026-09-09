# Network: Custom File Transfer and Software-Defined Networking

A two-part Python networking coursework project that examines the path of data at two different layers.

- **Part I** implements a custom application-layer protocol for authenticated, concurrent, block-based file transfer over TCP.
- **Part II** builds a controlled Mininet topology and uses Ryu with OpenFlow 1.3 for MAC learning, flow installation, and transparent TCP redirection.

Together, the two parts connect byte-stream framing and concurrency at the application layer with packet matching and forwarding behaviour inside a software-defined network.

## Repository Structure

```text
.
├── Part-I/
│   ├── Codes/
│   │   ├── client.py
│   │   └── server.py
│   └── *.pdf                         # Part I coursework report
├── Part-II/
│   ├── Codes/
│   │   ├── networkTopo.py
│   │   ├── ryu_forward.py
│   │   └── ryu_redirect.py
│   └── *.pdf                         # Part II coursework report
└── README.md
```

## Part I: STEP File Transfer

### Protocol design

The client and server communicate through a custom message format referred to in the code as STEP. Each TCP message combines structured JSON metadata with an optional binary payload.

`make_packet` serialises outgoing metadata and payload information. `get_tcp_packet` reconstructs messages from the TCP byte stream using length-based framing, rather than assuming that one `send` corresponds to one `recv`.

The protocol defines message categories including:

- `AUTH`
- `FILE`
- `DATA`

Operations include:

- `LOGIN`
- `SAVE`
- `UPLOAD`
- `GET`
- `DOWNLOAD`
- `DELETE`
- `BYE`

Responses include status information so that failures can be reported to the caller instead of silently ignored.

### Upload sequence

```text
Client                         Server
  │ ---- LOGIN -------------> │ authenticate and issue token
  │ <--- token -------------- │
  │ ---- SAVE metadata -----> │ create upload plan
  │ <--- key/block plan ----- │
  │ == UPLOAD block 0 ======> │ write at offset 0
  │ == UPLOAD block 1 ======> │ write at calculated offset
  │ == ... in parallel ... => │ record received blocks
  │ ---- completion check --> │ size and MD5 verification
  │ <--- final result ------- │
```

The server returns a file key, block size, and total block count. The client divides the file into 20,480-byte blocks, creates independent upload tasks, and sends blocks concurrently. Each block request uses its index to calculate the write offset in the temporary server file.

### Concurrency and recovery

The client coordinates shared task state with locks. It tracks task indexes, completed work, failed blocks, progress, and final integrity values. Failed blocks can be retried up to the configured limit, and network operations use a timeout of 30 seconds.

The server records received blocks and only finalises the temporary file after the required blocks are present. It checks file size and computes an MD5 digest so that the client can compare the transferred file with the local source.

### Download and delete

The same framing logic supports metadata requests and binary downloads. The client can request a stored object, reconstruct the response, write the downloaded content, and compare its integrity value. A delete operation is also included for managed server-side files.

### Authentication and security boundary

The coursework login derives a password from the MD5 of the student identifier, and the client writes the issued session token to `client.log` during execution.

This is suitable only for demonstrating protocol state and authenticated requests in a controlled assignment. It is **not production credential management**:

- MD5 is not an appropriate password-hashing algorithm.
- MD5 transfer checks detect accidental corruption but do not prevent deliberate tampering.
- Session tokens should not be written to persistent logs in a real system.
- The protocol does not provide TLS transport security by itself.

For a production design, use TLS, a memory-hard password hash such as Argon2 or bcrypt, protected token storage, modern content authentication, strict filesystem isolation, and carefully bounded resource usage.

## Part II: Mininet, Ryu, and OpenFlow

### Topology

`networkTopo.py` creates a controlled emulated network with:

- one Open vSwitch instance in secure mode
- one client host
- two server hosts
- fixed MAC addresses
- host addresses `10.0.1.2`, `10.0.1.3`, and `10.0.1.5` on a `/24` network
- a remote OpenFlow controller connection

The script starts the topology and opens host terminals for interactive testing in the Mininet environment.

### Learning-switch controller

`ryu_forward.py` targets OpenFlow 1.3 and implements:

1. a table-miss rule that sends unmatched packets to the controller
2. source-MAC learning from `PacketIn` events
3. ARP flooding when the destination is not yet known
4. learned Layer 2 forwarding
5. IPv4 flow installation
6. a higher-priority rule for selected TCP SYN traffic

The controller uses a priority of 100 for the selected TCP SYN flow, priority 50 for normal IPv4 forwarding, and an idle timeout of 5 seconds for installed experiment flows.

### Transparent redirection

`ryu_redirect.py` extends the learning-switch behaviour. When it observes a TCP SYN addressed to Server 1, it can redirect the connection to Server 2 by rewriting the destination IP and MAC address.

The reverse path rewrites Server 2's source IP and MAC back to the Server 1 identity expected by the client. This produces a bidirectional mapping:

```text
Forward direction:
client -> Server 1 identity
          rewrite destination IP/MAC
       -> Server 2

Reverse direction:
Server 2 -> client
rewrite source IP/MAC to Server 1 identity
       -> client
```

The redirection depends on the controller having learned the switch port for Server 2. Hard-coded addresses are part of the controlled Mininet experiment and are not a dynamic service-discovery mechanism.

## Requirements

### Part I

- Python 3
- A client and server that can reach each other over TCP
- Read and write permission for the configured working directories

The Part I programs use Python's standard networking, threading, hashing, JSON, and filesystem facilities.

### Part II

- Linux environment suitable for Mininet
- Mininet
- Open vSwitch
- Ryu controller
- OpenFlow 1.3 support
- Root privileges where required by Mininet

Part II is intended for a controlled virtual-machine or lab environment rather than a general desktop Python setup.

## Running Part I

Open separate terminals for the server and client.

```bash
cd Part-I/Codes
python3 server.py
```

```bash
cd Part-I/Codes
python3 client.py
```

Before running, inspect the configuration constants and command-line handling in the current files for the server address, port, identifier, and working paths. Do not commit generated `client.log` files because they may contain session tokens.

### Suggested verification

1. Log in and confirm that an invalid login is rejected.
2. Upload a small text file and compare the reported MD5 values.
3. Upload a file large enough to use multiple blocks.
4. Interrupt or delay one block and observe retry behaviour.
5. Download the stored file and compare it byte-for-byte with the source.
6. Test deletion and confirm the expected server response.
7. Attempt an invalid block index or malformed request in an isolated test environment.

## Running Part II

The exact interface names and command syntax depend on the installed Mininet and Ryu versions. A typical workflow is:

### 1. Start a controller

```bash
ryu-manager Part-II/Codes/ryu_forward.py
```

For the redirection experiment:

```bash
ryu-manager Part-II/Codes/ryu_redirect.py
```

### 2. Start the topology

In another terminal:

```bash
sudo python3 Part-II/Codes/networkTopo.py
```

### 3. Verify forwarding

From the Mininet CLI or opened host terminals:

```bash
pingall
```

Run the intended client/server TCP traffic, then inspect controller logs and switch flows:

```bash
sudo ovs-ofctl -O OpenFlow13 dump-flows <switch-name>
```

Replace `<switch-name>` with the Open vSwitch name created by the topology.

### 4. Verify redirection

- Start the service on Server 2.
- Generate the selected TCP SYN traffic from the client toward Server 1.
- Confirm that the controller has learned the Server 2 port.
- Observe the installed forward and reverse flow entries.
- Verify from the client that replies retain the Server 1 identity expected by the connection.

### 5. Clean the environment

After the experiment:

```bash
sudo mn -c
```

Only run cleanup in the intended Mininet test environment.

## Engineering Lessons

- TCP is a byte stream, so application protocols must define message boundaries.
- Concurrent transfer requires coordination of task ownership, completion state, retries, and file offsets.
- Integrity checks are different from confidentiality and authentication.
- Reactive SDN controllers must handle incomplete knowledge, such as an unknown destination port.
- Transparent redirection requires consistent rewriting in both directions.
- Flow priority and timeout choices directly affect observed network behaviour.

## Limitations

- Part I's authentication and MD5 choices are coursework mechanisms, not production security.
- Session-token logging should be removed or redacted before real deployment.
- Resource limits and hostile-input hardening are not equivalent to those of a public file-transfer service.
- Part II uses fixed experimental IP and MAC addresses.
- Redirection assumes the required destination port has been learned.
- Five-second idle timeouts are useful for observation but do not model every connection lifecycle.
- The topology is an emulation and does not establish performance on physical production networks.

## Public Repository Notes

- Do not commit `client.log`, captured tokens, transferred personal files, Mininet packet captures, or generated credentials.
- Before publishing the coursework PDFs, check for student numbers, private marking comments, private email addresses, and institutional information that should not be public.
- Add a software licence only after confirming ownership and distribution rights for every included component and report.

## Academic Context

This repository was developed as networking coursework. It demonstrates protocol design, concurrent socket programming, controlled SDN experiments, and explicit reflection on security limitations; it is not presented as a production file-transfer or load-balancing service.
