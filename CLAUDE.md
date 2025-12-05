# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Local Development Setup (macOS ARM64)

Docker images are x86_64 only. Run natively on Apple Silicon:

```bash
# 1. Start PostgreSQL in Docker
docker run --name postgres -p 5432:5432 -e POSTGRES_PASSWORD=postgres -d postgres:15

# 2. Build and initialize database
cd tinode-db
go build -tags postgres
./tinode-db -config=../server/tinode.conf

# 3. Download web client (optional)
cd ../server
git clone https://github.com/tinode/webapp.git static

# 4. Run server
cd ../server
go build -tags postgres
./server -config=tinode.conf
```

Server runs at http://localhost:6060/, gRPC at :16060

### Reset database with sample users
```bash
cd tinode-db
./tinode-db -config=../server/tinode.conf -reset -data=data.json
```
Creates test users: alice/alice123, bob/bob123, carol/carol123, etc.

## Current Configuration (tinode.conf)

**Enabled features:**
- WebRTC video/voice calls (using Google STUN servers)
- FCM push notifications (credentials: `zeroqc-dc611-firebase-adminsdk-*.json`)
- PostgreSQL database adapter

**Server endpoints:**
- HTTP/WebSocket: :6060
- gRPC: :16060

## Project Overview

Tinode is an open-source, federated instant messaging server written in pure Go. It provides a complete messaging platform with support for WebSocket, long polling, and gRPC protocols.

## Build Commands

Build server with a specific database backend (choose one):
```bash
go build -tags mysql ./server
go build -tags postgres ./server
go build -tags mongodb ./server
go build -tags rethinkdb ./server
```

Build with all adapters:
```bash
go build -tags "mysql rethinkdb mongodb postgres" ./server
```

Build database initializer:
```bash
go build -tags mysql ./tinode-db
```

Build with version stamp:
```bash
go build -tags mysql -ldflags "-X main.buildstamp=$(date -u '+%Y%m%dT%H:%M:%SZ')" ./server
```

## Testing

Run all server tests:
```bash
go test ./server/...
```

Run specific test packages:
```bash
go test -v ./server/drafty/
go test -v ./server/media/
go test -v ./server/ringhash/
go test -v ./server/db/common/
```

Database-specific tests require tags:
```bash
go test -tags mongodb ./server/db/mongodb/tests
```

## Linting

```bash
golangci-lint run --build-tags mysql,rethinkdb,mongodb ./server/...
```

Linter config: `server/.golangci.yml` - max line length 140, cyclomatic complexity 10.

## Architecture

### Core Components (server/)

- **Hub** (`hub.go`) - Central coordinator managing all topics and routing messages
- **Topics** (`topic.go`) - Independent message routers; types: `me`, `fnd`, p2p, group (`grp*`), `sys`
- **Sessions** (`session.go`) - Client connections; supports multiple simultaneous sessions per user
- **Cluster** (`cluster.go`, `cluster_leader.go`) - Sharded clustering with ring hash distribution

### Protocol Handlers (server/)

- `hdl_websock.go` - WebSocket connections
- `hdl_longpoll.go` - Long polling fallback
- `hdl_grpc.go` - gRPC with Protocol Buffers
- `hdl_files.go` - Out-of-band file transfers

### Pluggable Backends

**Authentication** (`server/auth/`): anon, basic, token, code, rest

**Databases** (`server/db/`): MySQL, PostgreSQL, MongoDB, RethinkDB - all implement the adapter interface in `adapter.go`

**Media Storage** (`server/media/`): filesystem (fs), Amazon S3

**Push Notifications** (`server/push/`): FCM, Tinode Push Gateway (tnpg), stdout (testing)

### Wire Protocol

Client messages: `{hi}`, `{acc}`, `{login}`, `{sub}`, `{leave}`, `{pub}`, `{get}`, `{set}`, `{del}`, `{note}`

Server messages: `{data}`, `{ctrl}`, `{meta}`, `{pres}`, `{info}`

Defined in `server/datamodel.go` for JSON, `pbx/model.proto` for gRPC.

### Key Directories

- `pbx/` - Protocol Buffer definitions and generated code
- `tn-cli/` - Python command-line client
- `tinode-db/` - Database initialization utility
- `chatbot/` - Example chatbot implementations
- `docker/` - Docker deployment configurations
- `monitoring/exporter/` - Prometheus/InfluxDB metrics exporter

## Configuration

Main config: `server/tinode.conf` - contains extensive inline documentation for all options.

Key settings: listen addresses (default 6060/16060), database adapter, media handler, push backends, auth backends, TLS/ACME.

## Proto Generation

After modifying `pbx/model.proto`:
```bash
cd pbx && ./go-generate.sh   # Go bindings
cd pbx && ./py-generate.sh   # Python bindings
```

## Documentation

- `docs/API.md` - Complete wire protocol documentation
- `docs/drafty.md` - Message formatting specification (Drafty format)
- `INSTALL.md` - Installation and deployment guide
