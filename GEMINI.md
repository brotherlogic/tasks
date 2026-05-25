# GEMINI.md - Developer Guide for `tasks`

This document provides context, architecture guidelines, and key commands for AI coding assistants working on the `brotherlogic/tasks` repository.

## Project Overview

`tasks` is a Go-based gRPC microservice designed to act as a bi-directional synchronization bridge between a **Google Tasks** list and a **GitHub Repository**'s issues. It runs in a Kubernetes cluster and uses the custom `brotherlogic/pstore` Protobuf storage engine for persistence and state mapping.

## Tech Stack & Architecture

- **Language:** Go (1.26+)
- **Communication:** gRPC & Protobuf (`protoc-gen-go`, `protoc-gen-go-grpc`)
- **APIs:** Google Tasks API (OAuth2) & GitHub API (PAT / App Installation Token)
- **Database:** `github.com/brotherlogic/pstore` (Protobuf Storage)
- **Deployment:** Kubernetes (Dockerized container)

### Synchronization Flow & Rules

- **Execution Mode:** A background polling loop (runs periodically, e.g., every 5 minutes).
- **Direction:** Bi-directional sync.
- **Mapping Protocol:**
  - Google Task **Title** $\Leftrightarrow$ GitHub Issue **Title**.
  - Google Task **Notes** $\Leftrightarrow$ GitHub Issue **Body**.
  - Google Task **Completed** status $\Leftrightarrow$ GitHub Issue **Closed** status.
  - Automatically tag created GitHub Issues with a `google-tasks` label.
- **Persistence Model:** State mappings (`GoogleTaskID` $\Leftrightarrow$ `GitHubIssueNumber`, along with last synced hashes/timestamps) are defined as Protobuf messages and stored/retrieved from `pstore`.
- **Orphan/Delete Handling:** If an item is deleted on one side, log the event and mark the mapping as orphaned instead of deleting the other side, preventing accidental data loss.

## Configuration & Environment

The service is statically configured using environment variables supplied via Kubernetes secrets:

- `GITHUB_TOKEN`: GitHub Personal Access Token or App Installation Token.
- `GITHUB_REPO`: Target repository in `owner/repo` format (e.g., `brotherlogic/tasks`).
- `GOOGLE_CLIENT_ID`: Google OAuth2 client identifier.
- `GOOGLE_CLIENT_SECRET`: Google OAuth2 client secret.
- `GOOGLE_REFRESH_TOKEN`: Stored long-lived Google OAuth2 refresh token for authentication.
- `GOOGLE_TASKLIST_ID`: Specific Google Tasklist ID to sync.

## Common Development Commands

### Initialize Go Module
```bash
go mod init github.com/brotherlogic/tasks
go get google.golang.org/grpc
go get google.golang.org/protobuf
go get github.com/brotherlogic/pstore
```

### Generate Protobuf Code
```bash
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    proto/tasks.proto
```

### Run Tests
```bash
go test -v ./...
```

### Build & Run
```bash
go build -o tasks main.go
./tasks
```

## Directory Structure (Planned)

```
/
├── proto/
│   └── tasks.proto       # Protobuf message and service definitions
├── client/
│   ├── github.go         # GitHub API integration client
│   └── google.go         # Google Tasks API integration client
├── server/
│   └── server.go         # gRPC server implementation (optional/management)
├── sync/
│   └── engine.go         # Bi-directional sync execution engine
├── main.go               # Application entrypoint & initialization
├── GEMINI.md             # This instruction manual
└── README.md             # High-level developer readme
```

## Styling & Coding Conventions

- **Clean gRPC APIs:** Maintain strict separation between domain logic and gRPC delivery layers.
- **Context Handling:** Pass `context.Context` explicitly through all API calls, network calls, and database transactions.
- **Robust Error Handling:** Do not ignore errors. Proactively wrap errors using standard Go formatting (`fmt.Errorf("context: %w", err)`).
- **Graceful Shutdown:** The polling engine must capture termination signals (`SIGINT`, `SIGTERM`) and wait for the current synchronization pass to safely complete before exiting.
