# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**yolo** is a Go CLI tool that generates new Golang web service projects with a standardized structure. It uses Go's `embed` directive to bundle templates and generates projects using Go's `text/template` package.

## Build and Install

```bash
# Install locally
go install github.com/jotiao/yolo@latest

# Build binary with version info
go build -ldflags "-X main.version=2.0.0 -X main.commit=$(git rev-parse HEAD) -X main.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)"

# Run directly
go run main.go init -name testproj -port 8080
```

## Architecture

### Entry Point

- `main.go`: CLI entry point with flag-based subcommands
  - `init`: Creates a new project (requires `-name`, optional `-port`, `-v`)
  - `version`: Displays build info (version, commit, date)
  - Build-time variables: `version`, `commit`, `date` injected via ldflags

### Core Package: `pkg/service`

- **`rander.go`**: Core project generation logic
  - `Rander` struct: Holds project name, port, and verbose flag
  - Uses functional options pattern (`WithVerbose`)
  - Main workflow: `InitDir()` → `InitPkg()` → `RunGoMod()`
  - `GenerateFile()`: Renders templates using `text/template`

- **`embed.go`**: Template file system
  - Embeds `pkg/service/templates/*` at compile time
  - Returns `fs.FS` for template access

- **`errors.go`**: Error handling
  - `YoloError` struct with error codes (E001-E007)
  - Error messages mapped in `errorMessages`
  - Implements `Unwrap()` for error chaining

### Generated Project Structure

Projects created by yolo follow this specific architecture:

```
<project>/
├── cmd/server.go           # HTTP server entry point (Gin framework)
├── pkg/
│   ├── config/             # YAML-based configuration with singleton pattern
│   ├── controller/         # HTTP handlers
│   ├── model/              # Database models
│   ├── router/             # URL routing with middleware
│   ├── service/            # Business logic layer
│   └── util/cm/            # Common utilities
├── etc/dev.yaml            # Configuration file
└── script/                 # Deployment scripts
```

### Template System

Templates use Go `text/template` syntax with access to `Rander` fields:
- `{{.ProjectName}}`: Project/module name
- `{{.ProjectPort}}`: Server port number

Templates are defined in `pkg/service/rander.go` in the `templates` slice within `InitPkg()`.

## Key Dependencies

Generated projects use:
- **gin-gonic/gin**: HTTP web framework
- **gin-contrib/cors**: CORS middleware
- **sirupsen/logrus**: Structured logging
- **gopkg.in/yaml.v2**: YAML configuration parsing
- **go-sql-driver/mysql**: MySQL driver (in model layer)

## Common Development Tasks

### Adding a New Template

1. Create `.tmpl` file in `pkg/service/templates/`
2. Add entry to `templates` slice in `pkg/service/rander.go`:
   ```go
   {"path/to/template.tmpl", "target/output/path"},
   ```
3. Use `{{.ProjectName}}` and `{{.ProjectPort}}` in template for dynamic values

### Adding a New CLI Command

1. Create flag set in `main.go` (see `initCommand` pattern)
2. Add case in `switch os.Args[1]` block
3. Implement handler function following `runInit()` pattern

### Testing Generated Projects

```bash
# Create test project
yolo init -name testapi -port 8080 -v

# Run generated server
cd testapi && go run cmd/server.go

# Test health endpoint
curl http://127.0.0.1:8080/v1/liveness
```

## Release Process

Releases are created via git tags:
```bash
git tag -a v2.0.0 -m "Release v2.0.0"
git push --tags
# Then create GitHub release via web interface
```

The project uses semantic versioning. Update version in `main.go` ldflags during build.
