# CLAUDE.md - AI Assistant Guide for LND

## Project Overview

LND (Lightning Network Daemon) is a complete implementation of a [Lightning Network](https://lightning.network) node written in Go. It fully conforms to the Lightning Network specification (BOLTs) and supports multiple backend chain services including `btcd`, `bitcoind`, and `neutrino`.

**Repository:** github.com/lightningnetwork/lnd
**Language:** Go 1.24.9+
**License:** MIT

## Key Directories

```
lnd/
├── cmd/
│   ├── lnd/          # Main daemon entry point
│   ├── lncli/        # Command-line interface
│   └── commands/     # CLI command implementations
├── lnrpc/            # gRPC/REST API definitions (.proto files)
├── itest/            # Integration tests
├── channeldb/        # Channel state database
├── lnwallet/         # Wallet and transaction handling
├── htlcswitch/       # HTLC forwarding logic
├── routing/          # Payment routing and pathfinding
├── funding/          # Channel funding workflows
├── contractcourt/    # On-chain contract resolution
├── discovery/        # P2P gossip protocol
├── lnwire/           # Lightning Network wire protocol
├── brontide/         # Encrypted transport (BOLT 8)
├── watchtower/       # Watchtower server/client
├── autopilot/        # Automatic channel management
├── invoices/         # Invoice handling
├── sweep/            # UTXO sweeping logic
├── chainntnfs/       # Chain notification interfaces
├── kvdb/             # Key-value database abstraction (submodule)
├── sqldb/            # SQL database layer (submodule)
├── fn/               # Functional programming utilities (submodule)
├── tlv/              # Type-Length-Value encoding (submodule)
└── docs/             # Documentation
```

## Building and Running

```bash
# Build debug binaries
make build

# Build and install to $GOPATH/bin
make install

# Build release binaries
make release-install
```

## Testing

### Unit Tests

```bash
# Run all unit tests
make unit

# Run tests for a specific package
make unit pkg=routing

# Run a specific test case
make unit pkg=routing case=TestFindPath

# Run with debug logging
make unit-debug log="stdlog trace" pkg=routing case=TestFindPath timeout=10s

# Run with race detector
make unit-race

# Run with coverage
make unit-cover
```

### Integration Tests

```bash
# Build and run all integration tests
make itest

# Run specific integration test (use snake_case name)
make itest icase=channel_backup

# Run integration tests in parallel
make itest-parallel

# Run with specific database backend
make itest dbbackend=postgres
make itest dbbackend=sqlite
```

### Test Guidelines

- All new code must have accompanying tests (positive and negative paths)
- Bug fixes must include tests that trigger the bug
- Unit tests should use the `require` library from testify
- Table-driven tests or tests using the `rapid` library are preferred
- Keep unit tests fast (< 10 seconds for a single test case)
- Integration test logs are scanned for `[ERR]` lines - avoid these in normal operation

## Code Style

### Line Length

- **Maximum 80 characters** (tabs count as 8 spaces)
- Use string concatenation for long strings
- Log/error messages are an exception - minimize lines while staying under 80 chars

### Function Comments

Every function must have a comment that:
- Begins with the function name
- Is a complete sentence
- Explains purpose and assumptions

```go
// DeriveRevocationPubkey derives the revocation public key given the
// counterparty's commitment key, and revocation preimage derived via a
// pseudo-random-function.
func DeriveRevocationPubkey(commitPubKey *btcec.PublicKey,
	revokePreimage []byte) *btcec.PublicKey {
```

### Code Spacing

Segment code into logical stanzas with blank lines and comments:

```go
witness := make([][]byte, 4)

// When spending a p2wsh multi-sig script, add a nil stack element.
witness[0] = nil

// Ensure signatures appear in the correct order.
if bytes.Compare(pubA, pubB) == -1 {
    witness[1] = sigB
    witness[2] = sigA
} else {
    witness[1] = sigA
    witness[2] = sigB
}

// Add the preimage as the last witness element.
witness[3] = witnessScript

return witness
```

### Switch/Select Spacing

Add blank lines between cases:

```go
switch {
// Handle case a.
case a:
    <code>

case b:
    <code>

default:
    <code>
}
```

### Wrapping Function Calls

When wrapping, put closing parenthesis on its own line:

```go
// WRONG
value, err := bar(a,
    a, b, c)

// RIGHT
value, err := bar(
    a, a, b, c,
)

// PREFERRED for inline structs
response, err := node.AddInvoice(ctx, &lnrpc.Invoice{
    Memo:      "invoice",
    ValueMsat: int64(amount),
})
```

### Error/Log Formatting

Minimize lines for log and error messages:

```go
// WRONG
return fmt.Errorf(
    "this is a long error message with placeholder %d",
    len(things),
)

// RIGHT
return fmt.Errorf("this is a long error message with placeholder %d",
    len(things))
```

## Git Commit Messages

### Format

```
subsystem: short description (50 chars max)

More detailed explanatory text wrapped at 72 characters.
Explain the "what" and "why" of the change.

- Bullet points are okay
- Use present tense: "Fix bug" not "Fixed bug"
```

### Subsystem Prefixes

- Single package: `lnwallet: add new signing method`
- Multiple packages: `lnwallet+htlcswitch: refactor commitment handling`
- Widespread changes: `multi: update error handling`

### Commit Structure

- Small, atomic commits that build on their own
- Separate commits for: bug fixes, refactoring, file restructuring
- Sign commits with GPG (`git commit -S`)
- Use `[skip ci]` suffix for documentation-only changes

## Linting

```bash
# Run all linters (requires Docker)
make lint

# Format code
make fmt

# Check formatting
make fmt-check

# Lint proto files
make protolint
```

Key linting rules:
- Line length: 80 characters (custom `ll` linter, excludes log lines with `S` suffix)
- Tabs: 8 spaces
- Uses golangci-lint v2 with custom configuration

## Proto/RPC Development

```bash
# Compile protobuf definitions
make rpc

# Format proto files
make rpc-format

# Verify protos are up to date
make rpc-check
```

Proto files are in `lnrpc/` directory. New RPC methods need:
- Proto definition
- REST annotations
- Appropriate tags in comments for `lncli` commands

## Database

LND supports multiple database backends:
- **bbolt** (default): Embedded key-value store
- **etcd**: Distributed key-value store for clustering
- **postgres**: SQL database
- **sqlite**: SQL database (embedded)

```bash
# Run tests with specific backend
make unit dbbackend=sqlite
make itest dbbackend=postgres nativesql=1
```

### SQL Development

```bash
# Generate SQL models (requires Docker)
make sqlc

# Verify SQL code generation
make sqlc-check
```

## Submodules

Several packages are Go submodules with their own `go.mod`:
- `cert/`, `clock/`, `fn/`, `healthcheck/`, `kvdb/`, `queue/`
- `sqldb/`, `ticker/`, `tlv/`, `tor/`, `tools/`

To update a submodule:
1. Create PR for submodule changes
2. Wait for merge and new tag (e.g., `tor/v1.0.x`)
3. Create second PR bumping the submodule in root `go.mod`

## Log Levels

Available: `trace`, `debug`, `info`, `warn`, `error`, `critical`

**Important:** Only use `error` for internal errors that are never expected during normal operation. External events (RPC, chain backend) should not trigger `error` logs.

## Important Files

- `sample-lnd.conf` - Sample configuration with all options
- `.golangci.yml` - Linter configuration
- `Makefile` - Build and test commands
- `docs/code_contribution_guidelines.md` - Full contribution guide
- `docs/development_guidelines.md` - Detailed style guide

## Pull Request Checklist

- [ ] All CI checks pass
- [ ] Tests cover positive and negative (error) paths
- [ ] Bug fixes include regression tests
- [ ] Code follows 80-character line limit
- [ ] Function comments follow conventions
- [ ] Commits have proper subsystem prefixes
- [ ] New logging uses appropriate levels
- [ ] Release notes updated (in `docs/release-notes/`)

## Common Patterns

### Error Handling

```go
if err != nil {
    return fmt.Errorf("failed to do X: %w", err)
}
```

### Context Usage

Most operations accept `context.Context` for cancellation:

```go
func (s *Server) SendPayment(ctx context.Context, req *Request) (*Response, error) {
    // Check for cancellation
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }
    // ...
}
```

### Testing with require

```go
func TestSomething(t *testing.T) {
    result, err := DoSomething()
    require.NoError(t, err)
    require.Equal(t, expected, result)
}
```

## Security Considerations

- This is financial software - bugs can cause loss of funds
- Never commit secrets or credentials
- Be cautious with cryptographic code
- Report security issues to security@lightning.engineering
