# Delphi MCP Server

![Delphi](https://img.shields.io/badge/Delphi-11%2B-red)
![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux-lightgrey)
![License](https://img.shields.io/badge/license-MIT-blue)
![MCP](https://img.shields.io/badge/MCP-2026--07--28%20(dual--era)-green)

A Model Context Protocol (MCP) server implementation in Delphi, designed to integrate with Claude Code, Codex, and other MCP-compatible clients for AI-powered Delphi development workflows.

## Table of Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Transport Modes](#transport-modes)
- [Protocol Versions and Dual-Era Behaviour](#protocol-versions-and-dual-era-behaviour)
- [Using as a Library](#using-as-a-library)
- [Integration with Claude Code](#integration-with-claude-code)
- [Integration with Codex](#integration-with-codex)
- [Testing with MCP Inspector](#testing-with-mcp-inspector)
- [Available Example Tools](#available-example-tools)
- [Available Example Prompts](#available-example-prompts)
- [Available Example Resources](#available-example-resources)
- [Configuration](#configuration)
  - [Authentication](#authentication)
  - [Network and Security](#network-and-security)
- [License](#license)
- [Contributing](#contributing)
- [About GDK Software](#about-gdk-software)
- [Support](#support)
- [Commercial Support](#commercial-support)

## Features

- **Dual-era MCP**: Serves MCP 2026-07-28 (per-request `_meta`, `server/discover`) and the initialize-based revisions 2025-06-18 and 2025-11-25 on the same endpoint and the same stdio process; see [Protocol versions](#protocol-versions-and-dual-era-behaviour)
- **Dual Transport Support**: HTTP (Streamable HTTP with SSE) and STDIO (stdin/stdout)
- **Dual Response Mode**: Supports both JSON-RPC and Server-Sent Events in the same server
- **Tool System**: Extensible tool system with RTTI-based discovery and execution
- **Resource Management**: Modular resource system supporting various content types
- **Security**: `Origin` and `Host` validation against DNS rebinding on every request, loopback binding by default, CORS headers for browser clients, request size and nesting limits, opt-in bearer authentication with OAuth 2.1 resource-server discovery
- **Multi round-trip requests, streaming and subscriptions**: `InputRequiredResult` with signed `requestState`, progress and log notifications on the response stream, `subscriptions/listen` for change notifications
- **Native HTTP stack**: Indy HTTP server with keep-alive, no external runtime
- **Optional Parameters**: Support for optional tool parameters using custom attributes
- **Cross-Platform**: Supports Windows (Win32/Win64) and Linux (x64)

## Requirements

- Delphi 11 Alexandria or later. The source uses no language feature beyond inline variables (10.3) and no RTL unit newer than Delphi 11; development happens on Delphi 12 Athens, which is what the default project files target.
- Windows (Win32/Win64) or Linux (x64)
- No external dependencies (all required libraries included)

### Building with Delphi 11 Alexandria

`build.bat` and `build-tests.bat` compile the `.dpr` directly, so only `DELPHI_PATH` needs to point at your installation:

```bash
set DELPHI_PATH=C:\Program Files (x86)\Embarcadero\Studio\22.0
build.bat
```

For the IDE and for the Linux64 build, open `src/MCPServer.D11.dproj` (tests: `tests/MCPServerTests.D11.dproj`) instead of the Athens project files. They carry the same units and settings, write their output to a separate `D11` subdirectory, and exist because Alexandria will not load a `ProjectVersion 20.3` project file.

## Installation

### For Standalone Usage

1. Clone the repository:
```bash
git clone https://github.com/GDKsoftware/delphi-mcp-server.git
cd delphi-mcp-server
```

2. Build the project:

#### Windows Build
```bash
build.bat
```

Or specify configuration and platform:
```bash
build.bat Debug Win32
build.bat Release Win64
```

The script picks up the highest TaurusTLS version installed in the CatalogRepository of the Studio release that `DELPHI_PATH` points at. To build against a copy somewhere else, set `TAURUS_PATH` to its `Source` directory first:
```bash
set TAURUS_PATH=C:\path\to\TaurusTLS\Source
build.bat Release Win64
```

#### Linux Build

**Prerequisites:**
- Delphi Enterprise with Linux platform support
- PAServer running on Linux target machine
- Linux SDK configured in RAD Studio

From the batch file:
```bash
build.bat Release Linux64
```

Or from RAD Studio IDE:
1. Open MCPServer.dproj
2. Select Linux64 platform
3. Build

## Transport Modes

The server supports two transport modes:

### HTTP Transport (Default)

Start the server without arguments for HTTP transport with Server-Sent Events (SSE):

```bash
Win32\Debug\MCPServer.exe
```

The server will listen on `http://localhost:3000/mcp` by default (configurable via settings.ini).

**Use HTTP transport for:**
- Claude Code (SSE support)
- MCP Inspector
- Web-based clients
- Remote connections

### STDIO Transport

Start the server with `--stdio` flag for stdin/stdout communication:

```bash
Win32\Debug\MCPServer.exe --stdio
```

The server will:
- Read JSON-RPC messages from stdin, UTF-8, one per line, no byte-order mark
- Write JSON-RPC messages to stdout the same way
- Log diagnostic messages to stderr, never to stdout
- Answer `notifications/cancelled` by stopping the named request; it gets no response
- Send `notifications/progress` for a request that carries `_meta.progressToken`, before its response
- Exit within `[Server] MaxConcurrentRequests` worker threads' drain time (2 seconds by default) once stdin closes

**Use STDIO transport for:**
- Codex (OpenAI)
- Local MCP clients that use process spawning
- Automated testing and scripting

**Supported flag variants:** `--stdio`, `-stdio`, `/stdio`

By default requests are answered one at a time, in the order they arrive.
`[Server] MaxConcurrentRequests` in `settings.ini` raises the number of worker
threads for a client that issues concurrent requests over the same process; a
stdio server never writes `settings.ini` on its own, so this and the other
`[Server]` limits still need explicit configuration when they should differ
from the defaults.

A tool sees the request it is answering through `TMCPRequestContext.Current`:
`CheckCancelled` raises once the client cancels, `ReportProgress` sends a
`notifications/progress` when the request carries a progress token, and
`Log` sends a `notifications/message` when the request carries
`_meta.io.modelcontextprotocol/logLevel` and the message's level is at or
above it. See `test_tool_with_progress` and `test_logging_tool` in
`MCPServer.Tool.ContentSamples` for worked examples.

Over HTTP the same notifications reach the client on the response: when the
request accepts `text/event-stream` and a tool sends one, the response turns
into an SSE stream (chunked, `X-Accel-Buffering: no`) that carries the
notifications first and the JSON-RPC response as its last event. A request
that sends none is answered as before. A client that closes the stream
cancels the request: the server's next write to it fails and the tool sees
`IsCancelled`. That is the HTTP cancellation. A `notifications/cancelled`
naming the same request is answered `202` and dropped, because the tracker a
request consults is its own response stream and a notification always arrives
on a connection of its own; only over stdio, where every message shares one
channel, does the notification stop a running request.

### Change notifications (`subscriptions/listen`)

A modern client that wants to hear about changes opens a long-lived
`subscriptions/listen` request with a `notifications` filter
(`toolsListChanged`, `promptsListChanged`, `resourcesListChanged`,
`resourceSubscriptions`: a list of URIs). `TMCPSubscriptionsManager`
(`MCPServer.SubscriptionsManager`) answers with
`notifications/subscriptions/acknowledged` carrying the honoured filter and
keeps the stream open: over HTTP as an SSE response with a keep-alive comment
every 15 seconds, over stdio on a thread of its own so the worker threads stay
free. Every message on the subscription carries
`_meta.io.modelcontextprotocol/subscriptionId`, the JSON-RPC id of the
`subscriptions/listen` request. Closing the SSE stream, or sending
`notifications/cancelled` for that id over stdio, ends the subscription;
when the server stops (or stdin closes) it answers the request with a
completion result first.

Notifications are delivered synchronously on the thread that causes the
change, so a subscriber that stops reading can hold up that thread until its
socket buffer drains.

Assign the manager as `ChangeNotifier` of the tools, prompts and resources
managers, as `MCPServer.dpr` does, and the `tools`, `prompts` and `resources`
capabilities announce `listChanged` (and `resources.subscribe`) to modern
clients. `AddTool`, `RemoveTool`, `AddPrompt`, `RemovePrompt`, `AddResource`,
`RemoveResource` and `AddResourceTemplate` then notify the subscribed clients,
and `TMCPResourcesManager.ResourceUpdated(Uri)` reports a changed resource to
the clients that subscribed to that URI. Without a `ChangeNotifier` nothing is
announced and nothing is sent.

## Protocol Versions and Dual-Era Behaviour

The server decides per request which protocol era it is speaking; nothing is negotiated per connection and no session is minted.

| Request | Era | Served as |
|---|---|---|
| `params._meta` with `io.modelcontextprotocol/protocolVersion` | modern | `2026-07-28`. `clientCapabilities` is required (`-32602`); an unknown revision gets `-32022` with the supported list; `initialize`, `ping`, `logging/setLevel` and `resources/subscribe` do not exist in this era (`-32601`). |
| `initialize` without modern `_meta` | legacy | The requested revision when it is `2025-06-18` or `2025-11-25`, otherwise `2025-11-25`. The result carries `capabilities` and `serverInfo` only. |
| `server/discover` without `_meta` | modern, malformed | `-32602` |
| Anything else | legacy | The revision negotiated by `initialize` on this stdio process, the `MCP-Protocol-Version` header on HTTP, or `2025-11-25` when nothing is known. |

Modern results carry `resultType`, `_meta.io.modelcontextprotocol/serverInfo` and, on `server/discover`, `tools/list`, `resources/list`, `resources/templates/list` and `resources/read`, the cache hints `ttlMs` and `cacheScope`. Legacy results are unchanged. Client responses (`result` or `error` without `method`) are ignored.

Legacy clients do not get the features that 2026-07-28 deprecates or removes. `logging/setLevel`, `resources/subscribe` and `resources/unsubscribe` answer `-32601`, and the `initialize` result does not announce `logging` or `resources.subscribe`. The server sends no requests to the client (`sampling/createMessage`, `elicitation/create`, `roots/list`), so a request whose tool, resource or prompt needs input from the client answers `-32603`. Modern clients get these through `_meta["io.modelcontextprotocol/logLevel"]`, `subscriptions/listen` and the multi round-trip requests described under [Asking the client for input](#asking-the-client-for-input-multi-round-trip-requests).

Over HTTP, modern requests must carry `MCP-Protocol-Version`, `Mcp-Method` and, for `tools/call`, `resources/read`, `prompts/get` and `tasks/*`, `Mcp-Name` (the tool or prompt name, the resource URI or the task id; Base64 sentinel encoding accepted); a missing or different header is `400` with `-32020`. Modern protocol errors get `400`, an unknown method `404`; legacy requests get `200` for every JSON-RPC error, except `400` for an unknown `MCP-Protocol-Version` header. Notifications get `202` with an empty body. Every 4xx to a modern request carries a JSON-RPC error body, so dual-era clients can tell a modern server from a legacy one.

Handlers can read the era, the negotiated revision and the client's declared capabilities through `TMCPRequestContext.Current` (`MCPServer.RequestContext`) or by implementing `IMCPCapabilityManagerEx`, and can raise `EMCPError` (`MCPServer.Errors`) to send a specific JSON-RPC error code.

`settings.ini` keys: `[Server] Title`, `Description`, `WebsiteUrl` and `Instructions` fill `serverInfo` and `instructions`; `[Protocol] LenientModernPing` answers `ping` in the modern era anyway, `DiscoverListsLegacyVersions` also lists the legacy revisions in `server/discover`, and `DiscoverTtlMs` is the cache hint on `server/discover`.

`2025-03-26` is accepted on `initialize` but answered with `2025-11-25`; JSON-RPC batch arrays are rejected with `-32600`.

## Using as a Library

The Delphi MCP Server is designed to be used both as a standalone application and as a library for your own MCP server implementations. This section covers how to integrate it into your existing Delphi projects.

### Project Setup for Library Usage

#### Option 1: Git Submodule (Recommended)

```bash
# Add MCPServer as a submodule to your project
git submodule add https://github.com/GDKsoftware/delphi-mcp-server.git lib/mcpserver
git submodule update --init --recursive
```

#### Option 2: Direct Source Inclusion

Copy the `src` folder from MCPServer into your project and add the units to your uses clauses.

#### Delphi Project Configuration

1. **Search Paths**: Add the MCPServer source directories to your project search path:
   - `lib\mcpserver\src\Core`
   - `lib\mcpserver\src\Managers` 
   - `lib\mcpserver\src\Protocol`
   - `lib\mcpserver\src\Server`
   - `lib\mcpserver\src\Tools`
   - `lib\mcpserver\src\Resources`
   - `lib\mcpserver\src\Prompts`

2. **Required Units**: `MCPServer.Host` is the only unit a host needs; it pulls
   in the managers, the HTTP server and the stdio transport:
   ```pascal
   MCPServer.Host,              // TMCPServerHost, the library facade
   MCPServer.Types,             // Interfaces, protocol constants, schema attributes
   MCPServer.Registration       // TMCPRegistry, for self-registering tools
   ```
   Composing the managers by hand instead needs `MCPServer.Settings`,
   `MCPServer.ManagerRegistry`, `MCPServer.CoreManager`, the managers you want,
   and `MCPServer.IdHTTPServer` or `MCPServer.StdioTransport`.

### Library Integration

`TMCPServerHost` (`MCPServer.Host`) is the whole composition behind one class:
it builds the managers, owns the HTTP server and drives either transport.
`StartHttp` returns as soon as the server listens, so the host fits into an
application that has a message loop of its own:

```pascal
program YourMCPServer;

{$APPTYPE CONSOLE}

uses
  System.SysUtils,
  MCPServer.Host in 'lib\mcpserver\src\Server\MCPServer.Host.pas',
  YourProject.Tool.Custom in 'YourProject.Tool.Custom.pas';

begin
  const Host = TMCPServerHost.Create;
  try
    Host.Settings.Port := 3000;
    Host.AddTool(TCustomTool.Create);
    Host.StartHttp;

    Writeln('MCP Server running on port ', Host.BoundPort);
    Readln;

    Host.Stop;
  finally
    Host.Free;
  end;
end.
```

**What a host starts with.** Nothing. A fresh host publishes only the tools,
resources, resource templates and prompts it was handed through `AddTool`,
`AddResource` and `AddPrompt`, so two hosts in one process publish exactly what
each of them was given. `SeedFromGlobalRegistry := True` takes everything from
`TMCPRegistry` instead, which is what the standalone server does; set it before
the first call that builds the managers, or it raises
`EMCPConfigurationError`. `[Server] ExposeDiagnosticsResources` is honoured
either way: with it off, `server://status`, `logs://recent` and `logs://{level}`
never reach the lists.

**Settings.** `Create` reads no file at all and starts from the built-in
defaults, which `Host.Settings` then lets you change in code.
`Create(SettingsFile)` reads the `.ini` you name and writes none.
`Create(Settings)` takes a `TMCPSettings` you built yourself and keep owning.
Only the standalone server reads `settings.ini` from the executable's own
directory.

**Ports.** `Settings.Port := 0` asks the operating system for a free port and
`BoundPort` reports the one it gave. A loopback server takes that same port on
both its IPv4 and its IPv6 binding, so `localhost` reaches it whichever family
the client resolves first. `StartHttp` and `Stop` are both idempotent.

**Transports.** `RunStdio` blocks and is a console entry point only: the stdio
transport claims stdout and redirects the logger to stderr for the whole
process, so a GUI application must never call it. `RunStdioWith(Input, Output)`
runs the same dispatch over two streams, which is how a test drives one line in
and reads one line out.

Composing the managers by hand still works, and is what to do when you need a
manager the host does not build:

```pascal
var
  ManagerRegistry: IMCPManagerRegistry;
begin
  const Settings = TMCPSettings.Create;
  try
    ManagerRegistry := TMCPManagerRegistry.Create;
    ManagerRegistry.RegisterManager(TMCPCoreManager.Create(Settings));
    ManagerRegistry.RegisterManager(TMCPToolsManager.Create(False));
    ManagerRegistry.RegisterManager(TMCPResourcesManager.Create(False));

    const Server = TMCPIdHTTPServer.Create(nil);
    try
      Server.Settings := Settings;
      Server.ManagerRegistry := ManagerRegistry;
      Server.Start;

      Writeln('MCP Server running on port ', Server.BoundPort);
      Readln;

      Server.Stop;
    finally
      Server.Free;
    end;
  finally
    Settings.Free;
  end;
end.
```

#### Library checklist

- **Register before you start, or hand them over afterwards.** The parameterless `TMCPToolsManager.Create`, `TMCPResourcesManager.Create` and `TMCPPromptsManager.Create` read `TMCPRegistry` once, so a registration made after the managers exist is not picked up: register your tools, resources and prompts (normally from unit `initialization` sections) first. `Create(False)`, and a `TMCPServerHost` left at its default `SeedFromGlobalRegistry`, read the registry not at all and publish only what you hand them, which you can do at any time.
- **STDIO: keep stdout clean.** Everything on stdout must be an MCP message. `TMCPStdioTransport.Create` forces `TLogger.UseStdErr := True` and sets `TLogger.StdoutReserved`, so console logging goes to stderr and an attempt to switch it back is refused with a one-time warning. Never `Writeln` from tools, managers or resources; log through `TLogger`.
- **`server://status` is registered by default** by the unit initialization of `MCPServer.Resource.Server`, and reaches every manager that seeds from the registry. `TServerStatusResource.SetNamePrefix('myapp_')` renames it to `server://myapp_status`; call it before the managers are created. `[Server] ExposeDiagnosticsResources = false` keeps it, `logs://recent` and `logs://{level}` out of the lists altogether.
- **Error codes and protocol constants** live in `MCPServer.Types` (`JSONRPC_*`, `MCP_ERROR_*`, `MCP_PROTOCOL_VERSION_*`, `MCP_META_*`, and the header names `MCP_HEADER_SESSION_ID`, `MCP_HEADER_PROTOCOL_VERSION`, `MCP_HEADER_METHOD` and `MCP_HEADER_NAME`). The `JSONRPC_*` names in `MCPServer.JsonRpcProcessor` remain as aliases. `TMCPHeaderValue.Encode` (`MCPServer.HttpHeaders`) writes a value into a header the way `TryDecode` reads one back: a header-safe value passes through, anything else is wrapped in the `=?base64?...?=` sentinel.
- **Prompts and completion are optional managers**, registered the same way as tools and resources: `ManagerRegistry.RegisterManager(TMCPPromptsManager.Create)` and, if you want argument completion, `ManagerRegistry.RegisterManager(TMCPCompletionManager.Create(PromptsManager, ResourcesManager))` (it needs the concrete manager instances, not the `IMCPCapabilityManager` interface, to look prompts and resource templates up by name). The `prompts` and `completions` capabilities are only advertised when these managers are registered.

### Creating Custom Tools

```pascal
unit YourProject.Tool.Custom;

interface

uses
  MCPServer.Tool.Base,
  MCPServer.Types,
  MCPServer.Registration