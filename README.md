# Hyper-V MCP

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-4c566a" alt="License: MIT"></a>
  <a href="https://doc.rust-lang.org/edition-guide/rust-2021/index.html"><img src="https://img.shields.io/badge/Rust-2021-4c566a?logo=rust&amp;logoColor=white" alt="Rust 2021"></a>
</p>

Hyper-V MCP exposes Hyper-V management through Model Context Protocol tools.
The Rust server runs commands through a persistent Windows PowerShell process.
The server communicates with MCP clients over standard input and standard output.

## Features

- Clients can inspect and manage virtual machines, storage, switches, and replication.
- The server provides typed inputs and outputs for its registered tools.
- Clients can read host inventory through MCP resources.
- PowerShell Direct tools run guest commands and transfer files through VMBus.
- The credential store encrypts saved passwords through Windows DPAPI.
- The server restarts its PowerShell process after transport failures and timeouts.

## Requirements

- Windows with Hyper-V enabled.
- Windows PowerShell 5.1 or later, available as `powershell.exe`.
- An elevated process with administrative privileges.
- Rust for building the server.

Guest operations also require a running guest that supports PowerShell Direct.
The guest must accept the supplied credentials.

## Build

```powershell
cargo build --locked --release
```

Cargo writes the server to `target/release/hyperv-mcp.exe`.

## MCP configuration

Configure your MCP client to start the executable.
Clients that use an `mcpServers` configuration can use this entry:

```json
{
  "mcpServers": {
    "hyper-v": {
      "command": "C:\\path\\to\\hyperv-mcp.exe"
    }
  }
}
```

Use the absolute path to your executable.
The MCP client must start the server with administrative privileges.
The server checks elevation during startup and exits if the check fails.

## Resources

The server reads these resources on request:

| URI | Data |
| --- | --- |
| `hyperv://vms/inventory` | Virtual machine names, identifiers, and states. |
| `hyperv://networks/topology` | Virtual switches and adapter descriptions. |
| `hyperv://host/info` | Host name, processor count, memory capacity, and virtual machine paths. |

## Tools

Tool names use the `hyperv_<verb>_<noun>` pattern.
The [tool directory](src/tools) contains each implementation and its input fields.
The server lists registered tools and their schemas through MCP.

| Area | Examples |
| --- | --- |
| Virtual machines | `hyperv_get_vm`, `hyperv_start_vm`, `hyperv_stop_vm`, `hyperv_new_vm` |
| Virtual disks | `hyperv_get_vhd`, `hyperv_new_vhd`, `hyperv_resize_vhd`, `hyperv_test_vhd` |
| Networking | `hyperv_get_vm_switch`, `hyperv_new_vm_switch`, `hyperv_connect_vm_network_adapter` |
| Checkpoints | `hyperv_checkpoint_vm`, `hyperv_get_vm_snapshot`, `hyperv_restore_vmsnapshot` |
| Replication | `hyperv_enable_vm_replication`, `hyperv_start_vm_failover` |

The available operations depend on the Hyper-V version and host configuration.
The [Microsoft cmdlet reference](https://learn.microsoft.com/en-us/powershell/module/hyper-v/) describes the underlying commands.

## Guest tools

Each guest operation creates a PowerShell Direct session and closes it after the operation.

| Tool | Operation |
| --- | --- |
| `hyperv_invoke_guest_command` | Runs a PowerShell script in the guest. |
| `hyperv_copy_to_guest` | Copies a host file into the guest. |
| `hyperv_copy_from_guest` | Copies a guest file to the host. |
| `hyperv_register_vm_credential` | Saves credentials for a named virtual machine. |

The server resolves guest credentials in this order:

1. Credentials supplied in the tool call.
2. Credentials saved for the virtual machine.
3. The configured default credential.

The operation fails if the server cannot resolve credentials.

## Configuration

The server reads configuration from `%APPDATA%\hyperv-mcp\config.json`.
The server stores virtual machine credentials in `%APPDATA%\hyperv-mcp\credentials.json`.
Set `HYPERV_MCP_CONFIG_DIR` to use another directory.

The configuration accepts these fields:

| Field | Purpose |
| --- | --- |
| `default_credential.username` | The server uses this default guest username. |
| `default_credential.password_encrypted` | The server decrypts this Base64-encoded DPAPI value. |
| `powershell_direct_timeout_seconds` | The server uses this operation timeout, with a default of 30 seconds. |

Use `hyperv_register_vm_credential` to store credentials for a virtual machine.
The tool encrypts the password before writing the credential store.

## Error handling

The server returns structured errors for failed commands, invalid inputs, and transport failures.
The server retries once after a response mismatch or a dead PowerShell process.
The server restarts PowerShell after a timeout and returns the timeout error.
The client must inspect the returned error before deciding whether to repeat an operation.

## Source structure

| Path | Responsibility |
| --- | --- |
| [`src/main.rs`](src/main.rs) | Checks elevation and starts MCP transport. |
| [`src/server.rs`](src/server.rs) | Routes tools and resources. |
| [`src/sidecar.rs`](src/sidecar.rs) | Manages PowerShell requests, timeouts, and restart. |
| [`sidecar/hyperv_sidecar.ps1`](sidecar/hyperv_sidecar.ps1) | Executes requests and returns JSON. |
| [`src/config.rs`](src/config.rs) | Loads configuration and resolves credentials. |
| [`src/dpapi.rs`](src/dpapi.rs) | Encrypts and decrypts stored passwords. |
| [`src/tools`](src/tools) | Defines the management and guest tools. |

## Checks

```powershell
cargo test --locked
```

Tests that require Hyper-V or administrative privileges need a suitable Windows host.

## License

The [MIT license](LICENSE) covers this project.
