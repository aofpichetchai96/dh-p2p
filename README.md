# Dahua P2P Camera Manager

A self-hosted manager that turns Dahua-compatible P2P cameras/NVRs (Dahua, KBVision, Amcrest, Lorex, Annke and other Easy4IPCloud-derived brands) into local RTSP streams — with a web dashboard, a REST API, and support for **multiple cameras running concurrently**, each accepting **multiple simultaneous RTSP client connections**.

What started as a single-connection command-line proof of concept has grown into a small always-on service: it holds one P2P/PTCP tunnel open per camera, multiplexes as many local TCP (RTSP) clients as you want through it, and exposes everything through a browser UI and a token-authenticated API so it can be automated or embedded in other systems (e.g. an NVR/VMS, a home automation hub, a monitoring dashboard).

## Motivation

The Dahua P2P protocol is used for remote access to Dahua devices, and is commonly implemented by Dahua-family apps such as [gDMSS Lite](https://play.google.com/store/apps/details?id=com.mm.android.direct.gdmssphoneLite) on Android or [SmartPSS](https://dahuawiki.com/SmartPSS) / [KBiVMS](https://kbvisiongroup.com/support/download-center.html) on Windows.

In my specific scenario, I have a KBVision CCTV system. Although I can access the cameras using the KBiVMS client, I primarily use non-Windows platforms and wanted a normal RTSP stream I could pull into any player, NVR, or automation pipeline. That meant reimplementing the Dahua P2P/PTCP protocol myself — and, since I ended up with several cameras/NVRs behind the same cloud relay, wrapping it in a small manager service rather than running one throwaway CLI process per camera.

## Key features

- **Multi-camera manager** – define any number of cameras, each with its own serial number, credentials, and local RTSP port; start/stop tunnels independently from a web UI or the API.
- **Concurrent RTSP clients per camera** – multiple players/consumers can connect to the same camera's local port at the same time; each client gets its own multiplexed "realm" inside the single PTCP/UDP session held with the device.
- **Web dashboard** (`static/index.html`) – login screen, a Cameras tab (add/edit/delete, start/stop, live status badges, one-click copy of the RTSP URL) and an API Tokens tab for issuing/revoking long-lived tokens.
- **REST API** – the same operations as the dashboard, protected by either a session token (from `/api/login`) or a long-lived API token, so cameras can be managed programmatically.
- **Auto-start** – cameras flagged `auto_start` are connected automatically when the service boots.
- **Brand-aware handshake** – the P2P cloud endpoint and app credentials (`main_server` / `app_username` / `app_userkey`) are configurable per brand instead of hardcoded, since many vendors white-label the same Easy4IPCloud-style backend under different names.
- **JSON file storage** – no database; configuration and state live in plain JSON files next to the binary, making it easy to back up, edit by hand, or provision via config management.

## Project layout

- Rust implementation (the actively developed manager service):
  - `src/main.rs` – process entry point: loads config, auto-starts cameras, boots the Axum web server.
  - `src/config.rs` – JSON-backed config/persistence for the app config, brands, cameras and API tokens.
  - `src/web.rs` – Axum router: login, brand/camera/token CRUD, tunnel start/stop/status, static file serving.
  - `src/tunnel.rs` – `TunnelManager`: owns one async task per camera, tracks status (`Stopped`/`Starting`/`Running`/`Error`), and accepts local TCP connections that get bridged into the PTCP session.
  - `src/dh.rs` – the P2P handshake (DHGET/DHPOST signaling against the cloud service) and PTCP session bring-up.
  - `src/process.rs` – the reader/writer tasks that shuttle bytes between local TCP clients and the PTCP session.
  - `src/ptcp.rs` – the PTCP (PhonyTCP) packet/session implementation.
  - `static/index.html` – the single-page web dashboard, served directly by the Axum server.
  - `Cargo.toml` / `Cargo.lock` – Rust dependencies (Axum, Tokio, serde, etc.).
- Data files (created at runtime, next to the binary — see [Configuration files](#configuration-files)):
  - `config.json`, `brands.json`, `cameras.json`, `tokens.json`
- Python implementation (kept as a lightweight reference/prototype, not actively developed further):
  - `main.py` – main script
  - `helpers.py` – helper functions
  - `requirements.txt` – Python dependencies
- Others:
  - `dh-p2p.lua` – Wireshark dissector for the Dahua P2P protocol, useful if you need to reverse-engineer additional message types.

## How it works

```mermaid
graph LR
  Browser[Web Dashboard]
  API[API client / automation]
  Server[[dh-p2p manager]]
  Cloud[Easy4IPCloud-style P2P service]
  Device1[Camera/NVR A]
  Device2[Camera/NVR B]
  RTSP1[RTSP Client 1]
  RTSP2[RTSP Client 2]

  Browser -- REST/JSON --> Server
  API -- REST/JSON --> Server
  Server -- PTCP/UDP --> Cloud
  Cloud -- notifies --> Device1
  Cloud -- notifies --> Device2
  Server <-. PTCP/UDP tunnel .-> Device1
  Server <-. PTCP/UDP tunnel .-> Device2
  RTSP1 -- TCP --> Server
  RTSP2 -- TCP --> Server
```

For each camera the manager:

1. Looks up the camera's `brand` in `brands.json` to get the P2P cloud endpoint and app credentials.
2. Performs the P2P handshake described in [Protocol description](#protocol-description) and establishes a PTCP session with the device over a single UDP socket.
3. Binds a local TCP listener on the camera's `local_port`.
4. For every TCP connection accepted on that port (e.g. an RTSP client), opens a new PTCP "realm" and relays bytes in both directions — so several RTSP clients can watch the same camera concurrently through one P2P/PTCP tunnel.
5. Sends a PTCP heartbeat every 5 seconds to keep the tunnel alive, independent of whether any client is currently connected.

The `TunnelManager` tracks each camera's task and its `Stopped` / `Starting` / `Running` / `Error(reason)` status, which is what the dashboard polls to render live status badges.

## Getting started (Rust)

```bash
# Build and run
cargo run --release
```

On first run the server creates sensible defaults if the JSON files don't exist yet (see below), then starts listening at `http://localhost:8080` (or whatever `web_port` you configure).

1. Open the dashboard in a browser and log in with the credentials from `config.json` (defaults to `admin` / `admin` — **change these**).
2. Add a brand entry (see [`brands.json`](#brandsjson)) for your camera's P2P cloud provider — the dashboard's camera form currently has hardcoded suggested defaults per brand name, but the brand itself must exist in `brands.json` (via the `/api/brands` endpoint) for the tunnel to resolve it.
3. Add a camera from the **Cameras** tab with its serial number, brand, device credentials, RTSP port (usually `554`) and the local port you want to expose it on.
4. Click **Start** (or enable `auto_start` so it connects automatically next time the service boots).
5. Stream it, e.g. with `ffplay`:

```bash
ffplay -rtsp_transport tcp -i "rtsp://[username]:[password]@127.0.0.1:[local_port]/cam/realmonitor?channel=1&subtype=0"
```

The dashboard shows the ready-to-use RTSP URL per camera once its tunnel is `Running`, and copies it to the clipboard on click.

## Configuration files

All files live in the working directory the binary is run from and are read/written on every change (no restart required). None of them are checked into git — treat them as local, sensitive runtime state.

### `config.json`

Admin login for the dashboard/API and the port the web server listens on.

```json
{
  "username": "admin",
  "password": "admin",
  "web_port": 8080
}
```

### `brands.json`

One entry per P2P cloud provider/brand. `main_server`, `app_username` and `app_userkey` are the signaling endpoint and app credentials used during the handshake in `src/dh.rs` — several vendors (Dahua, KBVision, Amcrest, Lorex, Annke, ...) reuse the same `www.easy4ipcloud.com:8800` backend under their own app identity.

```json
[
  {
    "id": "…uuid…",
    "name": "Dahua",
    "main_server": "www.easy4ipcloud.com:8800",
    "app_username": "…",
    "app_userkey": "…"
  }
]
```

A camera's `brand` field must match a `name` here, or the tunnel will fail to start with `Brand '<name>' not found in brands.json`. Managed via `/api/brands` (see [REST API](#rest-api)).

### `cameras.json`

```json
[
  {
    "id": "…uuid…",
    "name": "Front door",
    "brand": "Dahua",
    "serial": "ABC1234567890",
    "username": "admin",
    "password": "…device password…",
    "port": 554,
    "local_port": 1554,
    "auto_start": true
  }
]
```

- `serial` – the device's P2P serial number (same one used in gDMSS/SmartPSS/KBiVMS).
- `username` / `password` – the device's own RTSP credentials (used to build the RTSP URL shown by the dashboard; not the app/cloud credentials).
- `port` – the device's RTSP port, normally `554`.
- `local_port` – the local TCP port this camera's tunnel listens on.
- `auto_start` – if `true`, the tunnel is started automatically on service boot.

### `tokens.json`

Long-lived API tokens for programmatic access, independent of the dashboard's login session.

```json
[
  {
    "id": "…uuid…",
    "name": "Home Assistant",
    "token": "dhp2p_…",
    "expires_at": "2027-01-01T00:00:00Z",
    "enabled": true
  }
]
```

`expires_at` is optional (`null` = never expires); disabling or deleting a token immediately revokes it.

## Authentication

Every route under `/api` except `POST /api/login` requires an `Authorization: Bearer <token>` header, where `<token>` is either:

- a session token returned by `POST /api/login` (kept in memory only, lost on restart), or
- an API token from `tokens.json` that is `enabled` and not past its `expires_at`.

## REST API

Base path: `/api`

| Method | Path | Description |
| --- | --- | --- |
| POST | `/login` | Exchange `config.json` username/password for a session token. |
| GET | `/brands` | List configured P2P cloud brands. |
| POST | `/brands` | Create a brand. |
| PUT | `/brands/:id` | Update a brand. |
| DELETE | `/brands/:id` | Delete a brand. |
| GET | `/cameras` | List cameras. |
| GET | `/cameras/all` | List cameras enriched with their current RTSP URL (empty if not running). |
| POST | `/cameras` | Create a camera. |
| PUT | `/cameras/:id` | Update a camera. |
| DELETE | `/cameras/:id` | Delete a camera (stops its tunnel first). |
| POST | `/cameras/:id/start` | Start a camera's tunnel. |
| POST | `/cameras/:id/stop` | Stop a camera's tunnel. |
| GET | `/tunnels` | Status (`stopped`/`starting`/`running`/`error: …`) and RTSP URL for every camera. |
| GET | `/tokens` | List API tokens. |
| POST | `/tokens` | Create an API token (the raw token is only ever returned on creation). |
| PUT | `/tokens/:id` | Update a token's name/expiry/enabled state. |
| DELETE | `/tokens/:id` | Delete a token. |

Anything not matching `/api/*` falls back to serving `static/` (the dashboard).

## Known limitations

- No "Brands" tab in the dashboard yet — brands currently need to be created via the `/api/brands` endpoint directly (e.g. with `curl`) before a camera referencing that brand name can start.
- Config/state files are plain, unencrypted JSON, including device passwords and API tokens — protect the working directory and don't commit these files.
- The web server has no built-in TLS; put it behind a reverse proxy if exposing it beyond localhost/LAN.
- Devices that require authentication when creating the P2P channel (HTTP 403 during handshake) are detected but not yet supported (see `src/dh.rs`).
- `relay_mode` (falling back to a fully relayed connection when direct P2P fails) is implemented in the handshake but not currently exposed as a per-camera option.

## Python implementation

The Python implementation is the original, simpler approach used for drafting and reverse-engineering the protocol. It only handles a single connection/session at a time and is not the actively maintained path for the multi-camera manager described above, but it remains useful for quickly testing a single device or studying the protocol flow, since it's more linear and easier to read top-to-bottom than the async Rust implementation.

### Setup

```bash
# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run
python main.py [CAMERA_SERIAL]

# Stream (e.g. with ffplay) rtsp://[username]:[password]@127.0.0.1/cam/realmonitor?channel=1&subtype=0
ffplay -rtsp_transport tcp -i "rtsp://[username]:[password]@127.0.0.1/cam/realmonitor?channel=1&subtype=0"
```

### Python usage

To use the script with a device that requires authentication when creating a channel, use the `-t 1` option.

When running in `--debug` mode or when the `--type` > 0, the `USERNAME` and `PASSWORD` arguments are mandatory. Additionally, make sure that `ffplay` is in the system path when debug mode is enabled.

```text
usage: main.py [-h] [-u USERNAME] [-p PASSWORD] [-d] serial

positional arguments:
  serial                Serial number of the camera

options:
  -h, --help            show this help message and exit
  -d, --debug           Enable debug mode
  -t TYPE, --type TYPE  Type of the camera
  -u USERNAME, --username USERNAME
                        Username of the camera
  -p PASSWORD, --password PASSWORD
                        Password of the camer
```

### Limitations

- Single threaded, so only one client can connect at a time
- Polling based, so it's inefficient and inflexible
- Not fully implemented (e.g. only simplex keep-alive, no mulpile connections, etc.)
- Work better with `ffplay` and `-rtsp_transport tcp` option
- Still unstable, can crash at any time

## Protocol description

For reverse engineering the protocol, I used [Wireshark](https://www.wireshark.org/) and [KBiVMS V2.02.0](https://kbvisiongroup.com/support/download-center.html) as a client on Windows. Using `dh-p2p.lua` dissector, you can see the protocol in Wireshark easier.

For RTSP client, either [VLC](https://www.videolan.org/vlc/) or [ffplay](https://ffmpeg.org/ffplay.html) can be used for easier control of the signals.

### Overview

```mermaid
graph LR
  App[[This service]]
  Service[P2P cloud service, e.g. Easy4IPCloud]
  Device[Camera/NVR]
  App -- 1 --> Service
  Service -- 2 --> Device
  App <-. 3 .-> Device
```

The Dahua P2P protocol initiates with a P2P handshake. This process involves locating the device using its Serial Number (SN) via a third-party service (commonly Easy4IPCloud, or a white-labeled equivalent configured per brand in `brands.json`):

1. The service queries the cloud to retrieve the device's status and IP address, authenticating each request with the brand's `app_username`/`app_userkey`.
2. The cloud service then communicates with the device to prepare it for connection.
3. Finally, the manager establishes a direct (or relayed) connection with the device.

```mermaid
graph LR
  Device[Camera/NVR]
  App[[This service]]
  Client1[RTSP Client 1]
  Client2[RTSP Client 2]
  Clientn[RTSP Client n]
  Client1 -- TCP --> App
  Client2 -- TCP --> App
  Clientn -- TCP --> App
  App <-. UDP\nPTCP protocol .-> Device
```

Following the P2P handshake, the manager begins listening for TCP (RTSP) connections on the camera's configured local port. Upon a client's connection, it initiates a new realm within the PTCP protocol. Essentially, the manager serves as a tunnel between each client and the device, facilitating communication through PTCP encapsulation — all realms for a given camera share the same underlying PTCP session/UDP socket.

### P2P handshake

```mermaid
sequenceDiagram
  participant A as This service
  participant B as P2P cloud service
  participant C1 as P2P Server
  participant C2 as Relay Server
  participant C3 as Agent Server
  participant D as Camera/NVR

  A->>B: /probe/p2psrv
  B-->>A: ;
  A->>B: /online/p2psrv/{SN}
  B-->>A: p2psrv info

  A->>C1: /probe/device/{SN}
  C1-->>A: ;

  A->>C1: /info/device/{SN}
  C1-->>A: [randsalt]

  A->>B: /online/relay
  B-->>A: relay info

  A->>B: /device/{SN}/p2p-channel (*)

  par
    A->>C2: /relay/agent
    C2-->>A: agent info + token
    A->>C3: /relay/start/{token}
    C3-->>A: ;
  end

  B-->>A: device info

  A->>B: /device/{SN}/relay-channel + agent info

  C3-->>A: Server Nat Info!
  A->>C3: PTCP SYN
  A->>C3: PTCP request sign
  C3-->>A: PTCP sign

  A->>D: PTCP handshake (*)
```

_Note_: Both connections marked with `(*)` and all subsequent connections to the device must use the same UDP local port.

### PTCP protocol

PTCP (PhonyTCP) is a proprietary protocol developed by Dahua. It serves the purpose of encapsulating TCP packets within UDP packets, enabling the creation of a tunnel between a client and a device behind a NAT.

Please note that official documentation for PTCP is not available. The information provided here is based on reverse engineering.

### PTCP packet header

The PTCP packet header is a fixed 24-byte structure, as outlined below:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             magic                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             sent                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             recv                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             pid                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             lmid                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             rmid                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- `magic`: A constant value, `PTCP`.
- `sent` and `recv`: Track the number of bytes sent and received, respectively.
- `pid`: The Packet ID.
- `lmid`: The Local ID.
- `rmid`: The Local ID of previously received packet.

### PTCP packet body

The packet body varies in size (0, 4, 12 bytes or more) based on the packet type. Its structure is as follows:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      type       |                     len                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             realm                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             padding                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- `type`: Specifies the packet type.
- `len`: The length of the `data` field.
- `realm`: The Realm ID of the connection. Each concurrent RTSP client sharing a camera's tunnel gets its own randomly generated realm ID, multiplexed over the same PTCP session.
- `padding`: Padding bytes, always set to 0.
- `data`: The packet data.

Packet types:

- Special:
  - Empty body
  - `0x00`: SYN, the body is always 4 bytes `0x00030100`.
- Realm:
  - `0x10`: TCP data, where `len` is the length of the TCP data.
  - `0x11`: Binding port request.
  - `0x12`: Connection status, where the data is either `CONN` or `DISC`.
- Common (with `realm` set to 0):
  - `0x13`: Heartbeat, where `len` is always 0.
  - `0x17`
  - `0x18`
  - `0x19`: Authentication.
  - `0x1a`: Server response after `0x19`.
  - `0x1b`: Client response after `0x1a`.

## Acknowledgments

This project has been inspired and influenced by the following projects and people:

- [mcw0/PoC](https://github.com/mcw0/PoC): The foundational structure for the handshake and the PTCP protocol.
- [@p2p-sys](https://github.com/p2p-sys): The idea of inverting the STUN protocol.
