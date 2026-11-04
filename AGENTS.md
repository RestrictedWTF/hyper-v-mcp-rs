# Hyper-V MCP agent instructions

## Scope

This Rust MCP server manages Hyper-V through a persistent Windows PowerShell process.
Read [README.md](README.md) before changing tools, configuration, or guest behavior.
The server requires Windows, Hyper-V, `powershell.exe`, and an elevated process.
Guest tools also require a compatible running guest and valid guest credentials.

## Repository map

| Path | Responsibility |
| --- | --- |
| [src/main.rs](src/main.rs) | Logging, elevation checks, and stdio startup. |
| [src/server.rs](src/server.rs) | MCP routing and resource integration. |
| [src/tool.rs](src/tool.rs) | Typed tool contracts, schema conversion, and registration. |
| [src/tools](src/tools) | Hyper-V management and guest operations. |
| [src/resources.rs](src/resources.rs) | Host and virtual machine resource reads. |
| [src/sidecar.rs](src/sidecar.rs) | PowerShell transport, request matching, timeouts, and restart. |
| [sidecar/hyperv_sidecar.ps1](sidecar/hyperv_sidecar.ps1) | PowerShell request execution and JSON responses. |
| [src/ps_escape.rs](src/ps_escape.rs) | PowerShell string escaping. |
| [src/config.rs](src/config.rs) | Configuration and credential resolution. |
| [src/dpapi.rs](src/dpapi.rs) | Windows credential encryption. |
| [tests/sidecar_integration.rs](tests/sidecar_integration.rs) | Sidecar transport and timeout recovery tests. |

## Implementation rules

- Implement tools through `HyperVTool` and the existing registration mechanism.
- Register each new tool module in [src/tools/mod.rs](src/tools/mod.rs).
- Preserve typed inputs, outputs, tool names, and schema compatibility.
- Validate names, paths, and required fields before invoking PowerShell.
- Use `escape_ps_string` for values inserted into PowerShell single-quoted strings.
- Keep guest script text inside the explicit guest-command operation.
- Preserve request identifiers, response matching, timeout behavior, and transport recovery.
- Preserve structured PowerShell errors, including category and error identifier.
- Do not add retries for management failures without accounting for commands that may already have changed state.
- Preserve credential precedence: explicit input, saved virtual machine credential, then configured default.
- Keep saved passwords encrypted through DPAPI.
- Do not log credentials, decrypted passwords, or generated commands containing passwords.
- Keep diagnostics on standard error and reserve standard output for MCP.
- Retain the elevation check. Do not change the host configuration to bypass a prerequisite.

## Validation

Run commands from the repository root on Windows.

```sh
cargo check --locked --all-targets
cargo test --locked
cargo build --locked --release
```

Run [sidecar integration tests](tests/sidecar_integration.rs) when changing transport or timeout handling.
These tests execute PowerShell.
Add escaping regressions for changes involving quotes, paths, or multiline guest scripts.
Check tool discovery and schemas when adding or changing a tool.

Use a disposable virtual machine for integration tests that change Hyper-V state.
Do not modify unrelated machines, switches, disks, replication, or saved credentials during verification.
Verify the resulting host or guest state and clean up only resources created by the test.
An environment without Hyper-V cannot establish that management operations work.

## Reporting and changes

- Report Windows, PowerShell, Hyper-V availability, elevation, and the tests actually run.
- Keep credential stores, configuration files, virtual disks, captures, and build output out of commits.
- Keep setup and public tool descriptions synchronized with the implementation.
- Use Conventional Commits. Push or publish only when the task authorizes it.
