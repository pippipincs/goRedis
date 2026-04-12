# GoRedis

A lightweight, Redis-compatible in-memory key-value store built from scratch in Go. Implements the [RESP (Redis Serialization Protocol)](https://redis.io/docs/reference/protocol-spec/) for wire-level compatibility with standard Redis clients.

## Features

- **RESP protocol** — speaks the same wire format as Redis, so existing Redis clients can connect out of the box
- **Concurrent TCP server** — handles multiple client connections using Go channels for connection lifecycle and command dispatch
- **Thread-safe storage** — `sync.RWMutex`-backed key-value store for safe concurrent reads and writes
- **Custom Go client** — included client library with RESP serialization

## Supported Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `SET` | `SET key value` | Store a key-value pair |
| `GET` | `GET key` | Retrieve the value for a key |
| `HELLO` | `HELLO value` | Handshake/ping |

## Getting Started

### Prerequisites

- Go 1.24+

### Build & Run

```bash
make build   # compiles to ./bin/goredis
make run     # builds and starts the server on :5001
```

Or run directly:

```bash
go build -o bin/goredis .
./bin/goredis --listenAddr :5001
```

### Connect with redis-cli

```bash
redis-cli -p 5001
> SET hello world
> GET hello
"world"
```

### Connect with the Go client

```go
import "goredis/client"

c, _ := client.New("localhost:5001")
defer c.Close()

c.Set(context.TODO(), "foo", 42)
val, _ := c.Get(context.TODO(), "foo")
fmt.Println(val)
```

## Architecture

```
Client (TCP) ──► Accept Loop ──► Peer (read loop + RESP parsing)
                                        │
                                        ▼
                                  Message Channel
                                        │
                                        ▼
                                   Server Loop ──► KV Store (RWMutex)
                                   (select)
                                        │
                                   ┌────┼────┐
                                   ▼    ▼    ▼
                                 msg  add   del
                                      peer  peer
```

- **Accept loop** — listens for new TCP connections, spawns a goroutine per connection
- **Peer** — wraps a connection; runs a read loop that parses RESP frames into commands
- **Server loop** — single goroutine multiplexes commands, peer joins, and peer disconnects via channels
- **KV store** — mutex-protected `map[string][]byte`

## Testing

```bash
go test -v ./...
```

Includes a concurrent integration test that spins up the server and validates correctness across 10 simultaneous clients.

## Project Structure

```
├── main.go          # server entrypoint and core server logic
├── peer.go          # per-connection peer handling and RESP parsing
├── proto.go         # command types, RESP parsing, and serialization helpers
├── keyval.go        # thread-safe in-memory key-value store
├── client/
│   └── client.go    # Go client library
├── server_test.go   # integration tests
└── Makefile
```
