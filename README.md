# Reliable Data Proxy

A reliable data transfer implementation over UDP in C, with a Python proxy that emulates packet loss -- part of a computer networking project exploring Go-Back-N (GBN) and the three-way handshake over an unreliable channel.

## Overview

This project implements a client/server pair that transfers a file reliably over UDP using a custom packet protocol. Because UDP provides no delivery guarantees, the client and server implement:

- A **three-way handshake** (SYN / SYN-ACK / ACK) to establish a connection
- A **sliding window** with Go-Back-N (GBN) semantics and retransmission on timeout
- A **four-way connection teardown** (FIN / ACK exchange with 2-second FIN wait)

`rdproxy.py` sits between the client and server and randomly drops ~15% of packets (configurable) to simulate an unreliable network. The client and server must handle these losses purely through timeouts and retransmission.

## Packet Format

Each packet is 524 bytes total (12-byte header + 512-byte payload):

| Field | Type | Description |
|---|---|---|
| `seqnum` | `unsigned short` | Sequence number |
| `acknum` | `unsigned short` | Acknowledgment number |
| `syn` | `char` | SYN flag |
| `fin` | `char` | FIN flag |
| `ack` | `char` | ACK flag |
| `dupack` | `char` | Duplicate ACK flag |
| `length` | `unsigned int` | Payload length |
| `payload` | `char[512]` | Data |

Sequence numbers wrap modulo 25601.

## Components

| File | Description |
|---|---|
| `client.c` | Reads a file and transmits it to the server using a sliding window over UDP; handles SYN/ACK handshake and FIN teardown |
| `server.c` | Receives file data from the client, writes it to a numbered output file (e.g. `1.file`), and acknowledges each packet; handles connection setup and teardown |
| `rdproxy.py` | Asyncio UDP proxy that forwards packets between client and server with configurable random drop rate |
| `test_format.py` | Output format validation script |

## Tech Stack

- **Language:** C (client/server), Python 3 (proxy)
- **Transport:** POSIX UDP sockets (`SOCK_DGRAM`), non-blocking I/O via `fcntl`
- **Reliability mechanism:** Sliding window / Go-Back-N, 500 ms RTO, window size 10
- **Build:** GNU Make + GCC (`-Wall -Wextra`)

## Build and Run

```bash
make                          # builds server and client executables
```

Start the server:

```bash
./server <PORT> <ISN>
# example: ./server 5000 0
```

Start the proxy (in a separate terminal):

```bash
python3 rdproxy.py <SERVER_PORT> <PROXY_PORT> <DROP_RATE>
# example: python3 rdproxy.py 5000 9999 0.15
```

Send a file from the client through the proxy:

```bash
./client <HOSTNAME-OR-IP> <PROXY_PORT> <ISN> <FILENAME>
# example: ./client 127.0.0.1 9999 0 myfile.txt
```

The server writes received data to sequentially numbered files (`1.file`, `2.file`, ...) in the working directory.

To clean build artifacts:

```bash
make clean
```

