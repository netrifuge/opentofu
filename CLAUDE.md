# OpenTofu - Guide for AI Assistants

This document provides a comprehensive overview of the OpenTofu codebase structure, development workflows, and key conventions for AI assistants working on this project.

## Table of Contents

1. [Project Overview](#project-overview)
2. [Codebase Structure](#codebase-structure)
3. [Architecture & Key Concepts](#architecture--key-concepts)
4. [Development Workflow](#development-workflow)
5. [Testing Conventions](#testing-conventions)
6. [Code Style & Linting](#code-style--linting)
7. [Common Patterns](#common-patterns)
8. [Key Packages Reference](#key-packages-reference)
9. [Build & Test Commands](#build--test-commands)
10. [Contributing Guidelines](#contributing-guidelines)
11. [Important Notes for AI Assistants](#important-notes-for-ai-assistants)

---

## Project Overview

**OpenTofu** is an open-source infrastructure-as-code tool forked from Terraform, licensed under the Mozilla Public License v2.0 (MPL-2.0). It allows users to define, provision, and manage infrastructure safely and efficiently using a declarative configuration language (HCL).

### Key Features
- **Infrastructure as Code**: Declarative configuration syntax for infrastructure management
- **Execution Plans**: Preview changes before applying them
- **Resource Graph**: Parallel execution based on dependency analysis
- **Change Automation**: Automated infrastructure changes with minimal human intervention
- **State Encryption**: Built-in encryption support for state files
- **Provider Plugin System**: Extensible architecture for cloud providers and services

### Technology Stack
- **Language**: Go 1.25.3 (see `.go-version`)
- **Configuration DSL**: HCL (HashiCorp Configuration Language)
- **Type System**: cty (go-cty)
- **Build System**: Go modules + Makefile
- **CI/CD**: GitHub Actions
- **License**: MPL-2.0

---

## Codebase Structure

### Root Directory Layout

```
opentofu/
├── cmd/                    # Command-line entry points
│   └── tofu/              # Main CLI application
├── internal/              # Core implementation (55+ packages)
│   ├── addrs/             # Address types (resources, modules, etc.)
│   ├── backend/           # Backend implementations (local, remote state)
│   ├── command/           # CLI command implementations
│   ├── configs/           # Configuration loading and parsing
│   ├── dag/               # Directed Acyclic Graph implementation
│   ├── encryption/        # State encryption functionality
│   ├── lang/              # Expression evaluation and functions
│   ├── plans/             # Plan generation and storage
│   ├── providers/         # Provider plugin interfaces
│   ├── provisioners/      # Provisioner interfaces
│   ├── states/            # State management
│   └── tofu/              # Core OpenTofu engine
├── contributing/          # Contributing guidelines and FAQ
├── docs/                  # Internal documentation
├── rfc/                   # Request For Comments documents
├── scripts/               # Build and development scripts
├── testing/               # Test fixtures and helpers
├── tools/                 # Development tools
├── version/               # Version information
└── website/               # Documentation website

```

### Key Directories

#### `internal/tofu/` - Core Engine
The heart of OpenTofu, containing:
- Graph building and walking logic
- Context management for operations
- Resource and provider handling
- Plan and apply execution
- State transformations
- Mock implementations for testing

#### `internal/command/` - CLI Commands
Each user-facing command (`plan`, `apply`, `init`, etc.) has its own implementation file:
- `command/plan.go` - `tofu plan`
- `command/apply.go` - `tofu apply`
- `command/init.go` - `tofu init`
- Follows pattern: `<command>.go` with corresponding `<command>_test.go`

#### `internal/backend/` - State Backends
Backend implementations for state storage:
- `local/` - Local filesystem backend (also executes operations)
- `remote-state/` - Remote backends (S3, GCS, Azure, Consul, Kubernetes, PostgreSQL, etc.)
- Each backend implements `backend.Backend` interface

#### `internal/configs/` - Configuration Loading
HCL configuration parsing and validation:
- Loads and validates `.tf` files
- Module resolution
- Schema definitions
- Expression parsing (deferred until graph walk)

#### `internal/providers/` - Provider Interface
Plugin protocol definitions for providers:
- `providers.Interface` - Main provider contract
- Provider schema handling
- Resource CRUD operations
- Plugin communication via gRPC

---

## Architecture & Key Concepts

### Request Flow

```
User Command (tofu plan/apply)
    ↓
CLI Command Handler (internal/command)
    ↓
Backend Selection & Operation Creation
    ↓
Configuration Loader (internal/configs)
    ↓
Context Creation (internal/tofu.Context)
    ↓
Graph Builder (operation-specific)
    ↓
Graph Walk (parallel execution)
    ↓
Vertex Evaluation (per resource/module)
    ↓
Provider Interactions
    ↓
State Updates
```

### Core Components

#### 1. Context (`internal/tofu/context.go`)
- **Purpose**: Central orchestrator for all OpenTofu operations
- **Key Methods**:
  - `Plan()` - Generate execution plan
  - `Apply()` - Apply changes
  - `Validate()` - Validate configuration
- **Thread Safety**: Uses `states.SyncState` for concurrent access

#### 2. Graph System (`internal/tofu/graph.go`, `internal/dag/`)
- **DAG-based**: Directed Acyclic Graph for dependency management
- **Parallel Execution**: Walks graph concurrently where possible
- **Transformers**: Modular graph construction via `GraphTransformer` interface
- **Key Transforms**:
  - `ConfigTransformer` - Adds vertices from configuration
  - `StateTransformer` - Adds vertices from state
  - `ReferenceTransformer` - Creates dependency edges
  - `ProviderTransformer` - Associates resources with providers

#### 3. State Management (`internal/states/`)
- **State Representation**: `states.State` object
- **State Managers**: `statemgr.Full` interface
  - `statemgr.Filesystem` - Local state files
  - Remote implementations for backends
- **Concurrency**: `states.SyncState` wrapper for thread-safe access
- **Serialization**: JSON format (stored in backends)

#### 4. Provider System (`internal/providers/`)
- **Plugin Architecture**: gRPC-based plugin system
- **Protocol**: Plugin protocols in `internal/tfplugin5/` and `internal/tfplugin6/`
- **Factory Pattern**: `providers.Factory` for provider instantiation
- **Schema**: Dynamic schema from `GetProviderSchema` RPC

#### 5. Expression Evaluation (`internal/lang/`)
- **HCL-based**: Uses `hcl.Expression` and `hcl.Body`
- **Value System**: `cty.Value` for type-safe values
- **Scope**: `lang.Scope` provides evaluation context
- **References**: `lang.References()` analyzes dependencies
- **Functions**: Built-in functions available to expressions

#### 6. Configuration (`internal/configs/`)
- **Root**: `configs.Config` represents entire configuration tree
- **Loader**: `configload.Loader` handles module loading
- **Deferred Parsing**: Some expressions parsed during graph walk
- **Module Tree**: Recursive structure of root + child modules

---

## Development Workflow

### Environment Setup

1. **Install Prerequisites**:
   - Go 1.25.3 (see `.go-version`)
   - Git
   - Optional: Docker (for integration tests)
   - IDE: VSCode or GoLand/IntelliJ recommended

2. **Optional**: Use DevContainer
   - `.devcontainer.json` provided
   - Works with VSCode Remote Containers or GoLand

### Building OpenTofu

```bash
# Standard build
go build ./cmd/tofu

# Build with version tag
go build -ldflags "-X main.version=$(git describe --tags --always --dirty)" -o tofu ./cmd/tofu

# Build without -dev suffix
go build -ldflags "-w -s -X 'github.com/opentofu/opentofu/version.dev=no'" -o tofu ./cmd/tofu

# Enable experimental features
go build -ldflags "-w -s -X 'main.experimentsAllowed=yes'" -o tofu ./cmd/tofu

# Using Makefile
make build
```

### Running Tests

```bash
# All tests
go test ./...

# Specific package
go test ./internal/tofu
go test ./internal/command

# With coverage
go test -coverprofile=coverage.out -covermode=atomic ./...
go tool cover -html=coverage.out -o coverage.html

# Using Makefile
make test
make test-with-coverage

# Verbose output
go test -v ./internal/tofu
```

### Code Generation

```bash
# Generate code (except protobuf)
go generate ./...

# Generate protobuf stubs (requires protoc)
make protobuf
```

### Linting

```bash
# Run golangci-lint
make golangci-lint

# Or directly
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.6.0 run --timeout 60m ./...
```

### License Checking

```bash
# Check dependencies for compatible licenses
export GITHUB_TOKEN=<your-token>
make license-check
```

### Debugging

**VSCode Configuration** (`.vscode/launch.json`):
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "tofu plan",
      "type": "go",
      "request": "launch",
      "mode": "debug",
      "program": "${workspaceFolder}/cmd/tofu",
      "env": {"TF_LOG": "trace"},
      "args": ["plan"]
    }
  ]
}
```

**Debugging Tools**:
- Interactive debugger: `dlv` (delve)
- Debug script: `scripts/debug-opentofu` (remote debugging on port 2345)
- Data inspection: `github.com/davecgh/go-spew` (already imported)

---

## Testing Conventions

### Test File Organization

**Pattern**: `*_test.go` files alongside source files

```
internal/tofu/
├── context.go
├── context_test.go          # Unit tests
├── context_plan_test.go     # Plan-specific tests
├── context_plan2_test.go    # Additional plan tests
├── context_apply_test.go    # Apply-specific tests
└── testdata/                # Test fixtures
    ├── plan/
    │   └── main.tf
    └── apply/
        └── main.tf
```

### Test Types

#### 1. Unit Tests
- **Location**: Same package as code
- **Purpose**: Test individual functions/types
- **Example**: `internal/addrs/output_value_test.go`

#### 2. Integration Tests
- **Location**: Backend integration tests in `internal/backend/remote-state/`
- **Purpose**: Test with real external services
- **Flags**: Environment variables like `TF_S3_TEST=1`, `TF_PG_TEST=1`

#### 3. Acceptance Tests
- **Flag**: `TF_ACC=1`
- **Purpose**: Test interactions with external services
- **Usage**: `TF_ACC=1 go test ./internal/initwd`

#### 4. E2E Tests
- **Location**: `internal/cloud/e2e/`
- **Purpose**: Complete workflow testing
- **Example**: `apply_auto_approve_test.go`

### Common Testing Patterns

#### Table-Driven Tests (Most Common)
```go
func TestParseAddress(t *testing.T) {
    tests := map[string]struct {
        input   string
        want    Address
        wantErr string
    }{
        "simple": {
            input: "aws_instance.example",
            want:  Address{Type: "aws_instance", Name: "example"},
        },
        "invalid": {
            input:   "invalid",
            wantErr: "invalid address format",
        },
    }

    for name, tc := range tests {
        t.Run(name, func(t *testing.T) {
            got, err := ParseAddress(tc.input)
            // assertions...
        })
    }
}
```

#### Subtests with `t.Run()`
```go
func TestResource(t *testing.T) {
    t.Run("create", func(t *testing.T) {
        // test creation
    })
    t.Run("update", func(t *testing.T) {
        // test updates
    })
}
```

#### Helper Functions
```go
func testContext2(t *testing.T, opts *ContextOpts) *Context {
    t.Helper()  // Mark as helper for accurate error reporting
    // setup and return context
}
```

### Mock Objects

#### MockProvider (`internal/tofu/provider_mock.go`)
```go
p := new(tofu.MockProvider)
p.GetProviderSchemaResponse = &providers.GetProviderSchemaResponse{
    ResourceTypes: map[string]providers.Schema{
        "test_resource": {Block: &configschema.Block{...}},
    },
}
p.PlanResourceChangeFn = func(req providers.PlanResourceChangeRequest) {
    // custom implementation
}
```

#### MockHook (`internal/tofu/hook_mock.go`)
```go
hook := new(tofu.MockHook)
hook.PreApplyReturn = tofu.HookActionContinue
// Use in context
```

### Test Helpers

**Location**: `internal/*/testing.go` files

**Common Helpers**:
- `testModule(t, name)` - Load module from testdata
- `testModuleInline(t, sources)` - Create inline configuration
- `testContext2(t, opts)` - Create test context
- `simpleMockProvider()` - Basic provider mock
- `TestLocal(t)` - Create local backend for testing
- `TestLocalProvider(t, b, name, schema)` - Add provider to local backend

**Assertion Helpers**:
```go
// In various test files
assertNoErrors(t, diags)
assertNoDiagnostics(t, diags)
assertDiagnosticsMatch(t, got, want)

// Using cmp package
if diff := cmp.Diff(want, got, CmpOptionsForTesting); diff != "" {
    t.Error("wrong result:\n" + diff)
}
```

### Integration Tests (Makefile)

```bash
# List all integration tests
make list-integration-tests

# Specific backend tests
make test-s3         # AWS S3 backend (requires AWS credentials)
make test-gcp        # GCP backend (requires gcloud auth)
make test-pg         # PostgreSQL (uses Docker)
make test-consul     # Consul (uses Docker)
make test-kubernetes # Kubernetes (uses KinD)
make test-azure      # Azure (see runbook in code)

# Run all integration tests
make integration-tests

# Cleanup after tests
make integration-tests-clean
```

---

## Code Style & Linting

### Configuration

**Linter**: golangci-lint v2.6.0 (`.golangci.yml`)

**Excluded Paths**:
- `internal/ipaddr/` - Legacy frozen code
- `internal/legacy/` - Backward compatibility code
- `internal/states/statefile/version*_upgrade.go` - State migrations
- `website/` - Documentation

**Static Checks**: `staticcheck` with selective rules
- Disabled: `QF1008`, `ST1003`, `ST1005`, `ST1012`, `ST1016`

### Conventions

#### File Headers
All Go files must include copyright headers:
```go
// Copyright (c) The OpenTofu Authors
// SPDX-License-Identifier: MPL-2.0
// Copyright (c) 2023 HashiCorp, Inc.
// SPDX-License-Identifier: MPL-2.0

package packagename
```

#### Naming Conventions
- **Packages**: Lowercase, single word (e.g., `tofu`, `configs`, `addrs`)
- **Files**: Snake_case (e.g., `context_plan.go`, `resource_provider.go`)
- **Test Files**: `*_test.go` suffix
- **Mock Files**: `*_mock.go` suffix
- **Generated Files**: Often `*_string.go` for stringer output

#### Code Organization
- **Interfaces First**: Define interfaces before implementations
- **Group by Functionality**: Related types and functions together
- **Test Helpers**: Separate `testing.go` files in packages

---

## Common Patterns

### 1. Context Pattern
```go
// Creating a context
ctx := testContext2(t, &ContextOpts{
    Providers: providers,
    Config:    config,
})

// Using context for operations
plan, diags := ctx.Plan(context.Background(), state, opts)
```

### 2. Diagnostic Pattern
```go
var diags tfdiags.Diagnostics

// Add diagnostics
diags = diags.Append(tfdiags.Sourceless(
    tfdiags.Error,
    "Summary",
    "Detail message",
))

// Check for errors
if diags.HasErrors() {
    return diags
}
```

### 3. Graph Transform Pattern
```go
type MyTransformer struct {
    Config *configs.Config
}

func (t *MyTransformer) Transform(g *Graph) error {
    // Add vertices
    g.Add(vertex)

    // Add edges
    g.Connect(dag.BasicEdge(from, to))

    return nil
}
```

### 4. Address Pattern
```go
// Resource instance address
addr := addrs.Resource{
    Mode: addrs.ManagedResourceMode,
    Type: "aws_instance",
    Name: "example",
}.Instance(addrs.IntKey(0))  // aws_instance.example[0]

// Module instance address
modAddr := addrs.RootModuleInstance.Child("child", addrs.NoKey)
```

### 5. State Access Pattern
```go
// Thread-safe state access
state := states.NewState()
syncState := states.NewSyncState(state)

// Read
syncState.Lock()
resource := state.Resource(addr)
syncState.Unlock()

// Write
syncState.Lock()
state.SetResourceInstanceCurrent(addr, obj, providerAddr)
syncState.Unlock()
```

### 6. Provider Factory Pattern
```go
providers := map[addrs.Provider]providers.Factory{
    addrs.NewDefaultProvider("aws"): func() (providers.Interface, error) {
        return &mockProvider, nil
    },
}
```

---

## Key Packages Reference

### Core Packages

| Package | Purpose | Key Types/Functions |
|---------|---------|-------------------|
| `internal/tofu` | Core engine | `Context`, `Graph`, `GraphTransformer` |
| `internal/command` | CLI commands | `PlanCommand`, `ApplyCommand`, etc. |
| `internal/configs` | Configuration loading | `Config`, `Resource`, `Module` |
| `internal/states` | State management | `State`, `ResourceInstance`, `SyncState` |
| `internal/plans` | Plan generation | `Plan`, `Change`, `ResourceInstanceChange` |
| `internal/providers` | Provider interface | `Interface`, `Schema`, `Factory` |
| `internal/backend` | Backend implementations | `Backend`, `Enhanced`, `Operation` |
| `internal/addrs` | Address types | `Resource`, `ModuleInstance`, `AbsResource` |
| `internal/lang` | Expression evaluation | `Scope`, `References`, `EvalBlock` |
| `internal/dag` | Graph algorithms | `AcyclicGraph`, `Walk` |
| `internal/tfdiags` | Diagnostics | `Diagnostics`, `Sourceless` |

### Utility Packages

| Package | Purpose |
|---------|---------|
| `internal/copy` | Deep copying utilities |
| `internal/didyoumean` | Suggestion algorithms |
| `internal/flock` | File locking |
| `internal/httpclient` | HTTP client configuration |
| `internal/logging` | Logging utilities |
| `internal/terminal` | Terminal I/O handling |
| `internal/encryption` | State encryption |

### Testing Packages

| Package | Purpose |
|---------|---------|
| `internal/e2e` | End-to-end tests |
| `internal/moduletest` | Module testing framework |
| `testing/` | Test fixtures |

---

## Build & Test Commands

### Makefile Targets

```bash
# Building
make build                    # Build tofu binary with git version
make generate                 # Run go generate
make protobuf                 # Generate protobuf stubs

# Testing
make test                     # Run all unit tests
make test-with-coverage       # Generate coverage report
make list-integration-tests   # List available integration tests

# Integration Tests
make test-s3                  # S3 backend tests
make test-pg                  # PostgreSQL backend tests
make test-consul              # Consul backend tests
make test-kubernetes          # Kubernetes backend tests
make test-gcp                 # GCP backend tests
make integration-tests        # Run all integration tests

# Cleanup
make test-pg-clean            # Clean PostgreSQL test environment
make test-consul-clean        # Clean Consul test environment
make test-kubernetes-clean    # Clean Kubernetes test environment
make integration-tests-clean  # Clean all integration test environments

# Linting & Checking
make golangci-lint           # Run linter
make license-check           # Check dependency licenses

# Dependencies
make deps                    # Install development dependencies

# Help
make help                    # Show all available targets
```

### Go Commands

```bash
# Testing
go test ./...                              # All tests
go test -v ./internal/tofu                 # Verbose output
go test -run TestSpecificTest ./...        # Specific test
go test -coverprofile=coverage.out ./...   # With coverage

# Building
go build ./cmd/tofu                        # Basic build
go build -race ./cmd/tofu                  # With race detector
GOOS=linux GOARCH=amd64 go build ./cmd/tofu  # Cross-compile

# Code Generation
go generate ./...                          # Generate code

# Formatting
go fmt ./...                               # Format code
gofmt -s -w .                             # Simplify and format

# Module Management
go mod tidy                                # Clean up dependencies
go mod vendor                              # Vendor dependencies
go mod download                            # Download dependencies
```

---

## Contributing Guidelines

### Issue Workflow

1. **Check Labels**:
   - `accepted` + `help wanted` - Ready for contribution
   - `pending-decision` - Awaiting maintainer decision
   - `needs-rfc` - Requires RFC before implementation
   - `good first issue` - Suitable for new contributors

2. **Comment on Issue**: Express interest and get assigned
3. **Wait for Assignment**: Prevents duplicate work
4. **Communicate Timeline**: Set expectations

### Development Process

1. **Understand the Issue**: Clarify requirements if needed (Slack: `#dev-general`)
2. **Write Tests**: All PRs need adequate test coverage
3. **Update CHANGELOG**: Add entry to `CHANGELOG.md`
4. **Sign Commits**: Use `git commit -s` for DCO compliance
5. **Submit PR**: Complete checklist in PR template
6. **Code Review**: Address maintainer feedback

### RFC Process

For complex features requiring detailed design:

1. **Copy Template**: `rfc/yyyymmdd-template.md` to `rfc/${isodate}-${title}.md`
2. **Fill Template**: Provide technical details
3. **Submit PR**: Link to related issues
4. **Discussion**: Community reviews and discusses
5. **Approval**: Requires majority Core Team approval
6. **Tracking Issue**: Created after approval

**See**: `rfc/README.md` for full process

### Copyright & DCO

**CRITICAL RULES**:
- **Write code yourself** - Do not copy from external sources without permission
- **Disable AI assistants** - LLMs may emit BSL-licensed Terraform code
- **Sign commits**: `git commit -s` adds DCO sign-off
- **Match Git identity**: Ensure `user.name` and `user.email` match GitHub
- **Never copy from Terraform**: BSL license incompatible with MPL-2.0
- **Co-authored-by**: Add for any code not written by you

**Violations** will immediately disqualify PRs and may bar future contributions.

### Backporting

1. Ensure `main` and target version branch are up-to-date
2. Create backport branch: `git checkout -b backports/ISSUE_NUMBER`
3. Cherry-pick commit: `git cherry-pick -s COMMIT_ID`
4. Update changelog
5. Submit PR to version branch

---

## Important Notes for AI Assistants

### Code Generation Restrictions

⚠️ **CRITICAL**: Due to copyright concerns with the BSL-licensed Terraform:

1. **DO NOT use LLM coding assistants** for generating OpenTofu code
2. **DO NOT copy code** from Terraform repository or PRs
3. **DO NOT emit code** that may be from BSL-licensed sources
4. **ALWAYS write from scratch** based on OpenTofu's existing patterns
5. **DOCUMENT sources** when adapting code from within OpenTofu

### Safe Assistance

✅ **AI assistants CAN help with**:
- Analyzing existing OpenTofu code
- Explaining architecture and patterns
- Suggesting test strategies
- Reviewing code for bugs/improvements
- Navigating the codebase
- Understanding HCL and Go patterns
- Debugging issues

❌ **AI assistants should NOT**:
- Generate new feature implementations
- Suggest code that might come from Terraform
- Auto-complete based on external training data
- Copy patterns from Terraform documentation

### Best Practices for AI-Assisted Development

1. **Analyze First**: Study existing OpenTofu patterns before suggesting changes
2. **Reference Examples**: Point to existing code in OpenTofu as templates
3. **Explain Reasoning**: Help developers understand *why*, not just *what*
4. **Encourage Testing**: Always suggest corresponding test cases
5. **Cite Sources**: Reference specific files and line numbers in OpenTofu
6. **Respect DCO**: Remind about commit sign-off requirements

### Working with This Codebase

**Start Here**:
1. Read `docs/architecture.md` for system understanding
2. Study `contributing/DEVELOPING.md` for development setup
3. Review `contributing/FAQ.md` for common questions
4. Explore test files to understand patterns
5. Use `make help` to see available commands

**Common Tasks**:
- **Adding a command**: See `internal/command/` for examples
- **Modifying graph**: Study `internal/tofu/graph*.go` and transformers
- **Changing state**: Review `internal/states/` and `states.SyncState` usage
- **Provider changes**: Look at `internal/providers/` interfaces
- **Backend work**: Examine `internal/backend/` implementations

**Testing Requirements**:
- Unit tests for all new code
- Table-driven tests for multiple scenarios
- Mock providers for integration testing
- Update testdata fixtures as needed
- Integration tests for backend changes

### Language Constraints

**HCL Integration**:
- HCL library is external (`github.com/hashicorp/hcl`)
- Cannot modify HCL without upstream changes
- Use `hcl.Expression` and `hcl.Body` as black boxes
- Expression evaluation happens via `lang.Scope`

**Cty Type System**:
- Type system is external (`github.com/zclconf/go-cty`)
- All values use `cty.Value`
- Cannot extend type system directly
- Work within existing type constraints

### Community

- **Slack**: [opentofu.org/slack](https://opentofu.org/slack) - `#dev-general` for questions
- **Meetings**: Weekly community meetings (Wednesdays 12:30 UTC)
- **TSC**: Technical Steering Committee (bi-weekly Tuesdays 4pm UTC)
- **Issues**: GitHub Discussions for questions, GitHub Issues for bugs/features

---

## Quick Reference

### File Locations

| What | Where |
|------|-------|
| Main CLI entry | `cmd/tofu/main.go` |
| Commands | `internal/command/*.go` |
| Core engine | `internal/tofu/*.go` |
| Configuration | `internal/configs/*.go` |
| State | `internal/states/*.go` |
| Providers | `internal/providers/*.go` |
| Backends | `internal/backend/*/*.go` |
| Tests | `internal/*/**/*_test.go` |
| Test fixtures | `internal/*/testdata/` |
| Documentation | `docs/*.md` |
| RFCs | `rfc/*.md` |

### Common Commands

```bash
# Build and run
make build && ./tofu version

# Test package
go test -v ./internal/tofu

# Test specific function
go test -v -run TestContext2Plan ./internal/tofu

# Debug with trace logging
TF_LOG=trace ./tofu plan

# Format and lint
go fmt ./... && make golangci-lint

# Update dependencies
go get -u ./... && go mod tidy
```

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `TF_LOG` | Logging level (TRACE, DEBUG, INFO, WARN, ERROR) |
| `TF_LOG_PATH` | Log file path |
| `TF_ACC` | Enable acceptance tests |
| `TF_S3_TEST` | Enable S3 integration tests |
| `TF_PG_TEST` | Enable PostgreSQL integration tests |
| `TF_CONSUL_TEST` | Enable Consul integration tests |
| `TF_K8S_TEST` | Enable Kubernetes integration tests |

---

## Version Information

- **Go Version**: 1.25.3 (see `.go-version`)
- **License**: MPL-2.0
- **Repository**: https://github.com/opentofu/opentofu
- **Website**: https://opentofu.org
- **Documentation**: https://opentofu.org/docs

---

## Additional Resources

- **Architecture**: `docs/architecture.md` - Detailed architecture overview
- **Glossary**: `docs/glossary.md` - OpenTofu terminology
- **State Encryption**: `docs/state_encryption.md` - Encryption implementation
- **Diagnostics**: `docs/diagnostics.md` - Error handling patterns
- **Planning Behaviors**: `docs/planning-behaviors.md` - Plan generation details
- **Tracing**: `docs/tracing.md` - Distributed tracing support

---

*This document was generated to assist AI assistants in understanding and working with the OpenTofu codebase. For human contributors, please see `CONTRIBUTING.md` and `contributing/DEVELOPING.md`.*

*Last Updated: 2025-11-18*
