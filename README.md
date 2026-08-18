# Network 

This repository contains two CAN201 coursework components that demonstrate practical networking concepts with Python socket programming, Mininet, and the Ryu SDN framework.

## Overview

| Part | Topic | Main Technologies |
| --- | --- | --- |
| Part I | File transfer system using a custom STEP protocol | Python sockets, JSON, MD5, multithreading |
| Part II | Software-defined network traffic forwarding and redirection | Mininet, Open vSwitch, Ryu |

## Repository Structure

```text
.
|-- Part-I/
|   |-- Codes/
|   |   |-- client.py
|   |   `-- server.py
|   `-- Report/Report_Part_I.pdf
`-- Part-II/
    |-- Codes/
    |   |-- networkTopo.py
    |   |-- ryu_forward.py
    |   `-- ryu_redirect.py
    `-- Report/Report_Part_II.pdf
```

## Part I: File Transfer System

Part I implements a client/server file transfer application. It supports user authentication, file operations, block-based uploads, and multithreaded upload handling.

### Requirements

- Python 3
- `tqdm`

Install the Python dependency:

```bash
python -m pip install tqdm
```

### Run

Open two terminals in `Part-I/Codes`.

Start the server in the first terminal:

```bash
python server.py --ip 127.0.0.1 --port 1379
```

Start the client in the second terminal:

```bash
python client.py
```

The server accepts `--ip` and `--port` options. The default port is `1379`; use matching connection settings in the client when using a different server address or port.

## Part II: SDN Forwarding and Redirection

Part II defines a Mininet topology with two servers and one client, together with two Ryu controller applications:

- `ryu_forward.py` implements reactive Layer-2 forwarding and flow installation.
- `ryu_redirect.py` redirects TCP connection attempts intended for Server 1 to Server 2, including reverse-path address rewriting.

### Requirements

- Linux environment with Mininet and Open vSwitch
- Python 3
- Ryu SDN framework

### Run

Open a terminal in `Part-II/Codes` and start one controller application:

```bash
ryu-manager ryu_forward.py
```

For the redirection experiment, use this instead:

```bash
ryu-manager ryu_redirect.py
```

In a second terminal, launch the Mininet topology (administrator privileges are normally required):

```bash
sudo python3 networkTopo.py
```

The topology script opens the Mininet command-line interface and creates terminal windows for the hosts. Consult the reports for the experimental procedure and evaluation details.

## Reports

- [Part I report](Part-I/Report/Report_Part_I.pdf)
- [Part II report](Part-II/Report/Report_Part_II.pdf)

## Notes

These materials were created for academic coursework and are intended for learning and demonstration purposes.
