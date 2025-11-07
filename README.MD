# MSC MP — Server and Scripts Guide

A complete tutorial for setting up and running the server, building scripts, and an API reference. For the client mod README, see the repository root/release.

## Quick Start (Server)
- Unpack the server to `MSC-MP-Server/`.
- Run `MSC.MP.Server.exe` once — it will generate `server.cfg`.
- Stop the server, edit `server.cfg`, and start it again.
- Check status: `http://<ip>:<http_port>/` (returns JSON with server name and player count).

## Requirements
- Windows.
- .NET 8 Runtime (to run), .NET 8 SDK (to build from source).

## Server Layout
```
MSC-MP-Server/
├── MSC.MP.Server.exe
├── MSC.MP.Server.dll
├── server.cfg              # server config (created on first run)
└── scripts/                # script DLLs (optional)
    └── SampleMode.dll      # example script (if you build from source)
```

## Configuration — server.cfg
```
name=My Cool Server   # server name (shown in HTTP JSON)
port=26015            # game TCP port
http_port=26016       # HTTP status port
```
- Edit and restart the server to apply.
- You can override the TCP port via CLI: `MSC.MP.Server.exe 26020`.

## Running the Server
- Double‑click `MSC.MP.Server.exe`.
- Or from PowerShell in the server folder:
```
.\MSC.MP.Server.exe
```

## HTTP Status
- URL: `http://<ip>:<http_port>/`
- Example response:
```
{"name":"My Cool Server","players":3}
```

## Build from Source
### Server
```
dotnet build "MSC-MP/Server/MSC.MP.Server.csproj" -c Release -v minimal
```
Artifacts: `MSC-MP/Server/bin/Release/`

### Scripts (SampleMode example)
```
dotnet build "MSC-MP/Server/scripts/SampleMode/SampleMode.csproj" -c Release -v minimal
```
Copy the produced DLL to `MSC-MP-Server/scripts/` and restart the server.

## Writing a Script
A script is a .NET 8 Class Library (DLL) implementing `MSC.MP.Abstractions.IServerScript`.

### `IServerScript` Callbacks
- `void OnServerStart(IServerApi api)` — server startup, provides `api`.
- `void OnPlayerConnect(int id, string name)` — player connected.
- `void OnPlayerDisconnect(int id, string name)` — player disconnected.
- `bool OnPlayerText(int id, string name, string msg)` — incoming text; return `false` to suppress default broadcast.
- `bool OnPlayerCommand(int id, string name, string cmd, string args)` — `/cmd args`; return `true` if handled.

Minimal example:
```
using MSC.MP.Abstractions;

public class MyMode : IServerScript
{
    private IServerApi _api;

    public void OnServerStart(IServerApi api)
    {
        _api = api;
        _api.SetServerName("My Server");
    }

    public void OnPlayerConnect(int id, string name)
    {
        _api.MsgTo(id, $"Welcome, {name}!", "#ffffff");
    }

    public void OnPlayerDisconnect(int id, string name) { }

    public bool OnPlayerText(int id, string name, string msg)
    {
        return true; // allow default broadcast
    }

    public bool OnPlayerCommand(int id, string name, string cmd, string args)
    {
        if (string.Equals(cmd, "id", System.StringComparison.OrdinalIgnoreCase))
        {
            _api.MsgTo(id, $"ID: {id}", "#aaaaaa");
            return true;
        }
        return false;
    }
}
```

## API Reference (`IServerApi`)
- Messages
  - `void MsgAll(string message, string color = null)` — send to all.
  - `void MsgTo(int id, string message, string color = null)` — send to a single player.
- Lookup/Info
  - `int? FindIdByName(string name)` — find player ID by name (case‑insensitive).
  - `int GetPlayerCount()` — number of players online.
  - `void SetServerName(string name)` — set server name (overrides `server.cfg`, reflected in HTTP/`server_info`).
- Map (debug primitives)
  - `int MapSpawn(string shape, float x, float y, float z, float sx, float sy, float sz, string color = null)` — spawn, returns `mapId`.
  - `void MapClear(int mapId)` — remove object by `mapId`.
  - `void MapClearAll()` — remove all objects.
- Persistent State
  - `void SetState(string key, string value)` — save and broadcast `state_set`, writes to `state.txt`.
  - `void RemoveState(string key)` — remove and broadcast `state_remove`.
  - `string GetState(string key)` — fetch from memory.

Color is HTML hex (e.g., `"#rrggbb"` or `"#rrggbbaa"`).

## Protocol (for context)
- Server → Client: `welcome`, `server_info`, `join`, `leave`, `chat`, `pos`, `map_spawn`, `map_clear`, `map_clear_all`, `state_set`, `state_remove`.
- Client → Server: `hello` (name), `chat` (messages/commands), `pos` (position/angles).

## Debugging
- Server console: `[id] name connected/disconnected`, `<name> msg`.
- HTTP errors: `HTTP status endpoint error: ...`.
- `state.txt`: persistent key/value store near the exe.

## Releases
For the server package, include: `MSC.MP.Server.exe`, `MSC.MP.Server.dll`, `server.cfg`, and the `scripts/` folder (if used).

## License
All rights reserved.
