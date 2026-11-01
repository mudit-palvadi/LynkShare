# LynkShare

LynkShare is a privacy-first, login-free file transfer system. A sender picks
files, shares a QR code or link, and the recipient receives them — over a
direct WebRTC connection when possible, or through encrypted Cloudflare R2
storage when a direct path is not available. The service coordinates the
transfer; it never sees plaintext file contents.

This repository is the **meta-repo** for the three components. They are git
submodules, not copies, so each component keeps its own history, issues,
releases and CI.

## Components

| Path | Repository | Stack | Responsibility |
|---|---|---|---|
| [`client/`](https://github.com/mudit-palvadi/lynk-client) | `lynk-client` | Flutter / Dart | Android sender and receiver, QR scanning, deep links, local encryption, P2P and R2 transfer paths. |
| [`backend/`](https://github.com/mudit-palvadi/lynk-backend) | `lynk-backend` | FastAPI / Python | Transfer and receiver sessions, presigned R2 URLs, WebSocket signaling, TURN credentials, rate limits and Redis-backed state. |
| [`web/`](https://github.com/mudit-palvadi/lynk-web) | `lynk-web` | HTML / CSS / vanilla JS | Browser sender and receiver, landing page, deep-link association files and Cloudflare Worker assets. |

## Clone

```bash
git clone --recurse-submodules https://github.com/mudit-palvadi/LynkShare.git
cd LynkShare
```

If you cloned without submodules:

```bash
git submodule update --init --recursive
```

To update every component to the commits pinned by this repository:

```bash
git submodule update --init --recursive
```

To pull the latest `main` of each component instead, run `git submodule update --remote --merge`
only when you intend to move the parent repository's pins.

## Local development

Each component has its own setup guide:

- [Client setup](https://github.com/mudit-palvadi/lynk-client#run-it-locally)
- [Backend setup](https://github.com/mudit-palvadi/lynk-backend#local-environment-config)
- [Web setup](https://github.com/mudit-palvadi/lynk-web)

Run your own backend, Redis instance and Cloudflare R2 bucket. Do not point a
local build at the project's production API or storage. The client fails fast
when `API_BASE_URL` is missing rather than falling back to a hosted service.

A typical local stack is:

1. Redis on port `6379` with a password and persistence disabled.
2. The FastAPI backend on `http://localhost:8000`, with your own R2
   credentials in its `.env`.
3. The Flutter client with `API_BASE_URL` set to
   `http://10.0.2.2:8000/api/v1/` on an Android emulator, or
   `http://localhost:8000/api/v1/` on web, desktop or a physical device after
   `adb reverse tcp:8000 tcp:8000`.
4. The website served locally, or the deployed Worker.

The backend's signaling rooms are process-local, so it must run with a single
Uvicorn worker. Scaling out requires Redis-backed fan-out.

## Transfer paths

1. **Direct (P2P):** the peers exchange SDP and ICE candidates over an
   authenticated WebSocket, then transfer over an encrypted `RTCDataChannel`.
   TURN relays the connection when a direct path cannot be established.
2. **Cloud fallback:** if the data channel does not open, the sender uploads
   ciphertext to R2 through presigned URLs and the receiver downloads it with a
   short-lived link. The AES key stays in the URL fragment and never reaches
   the backend.

## Security

- Files are encrypted on the sending device with AES-256-CBC before upload.
- The key travels in the link fragment, which browsers do not send to servers.
- Signaling and transfer access are scoped to short-lived session tokens.
- TURN credentials are short-lived HMAC credentials minted for a live transfer.
- The backend stores metadata and wrapped keys only; it does not proxy file
  bytes.

See [the client security policy](https://github.com/mudit-palvadi/lynk-client/blob/main/SECURITY.md)
for private vulnerability reporting.

## CI

Each component runs its own GitHub Actions workflow on every push to `main`
and every pull request:

- Client: `flutter analyze` and `flutter test`
- Backend: `ruff check` and `pytest`
- Web: `node --test`

## License

The LynkShare client is licensed under the GNU General Public License v3.0.
The backend and web repositories do not currently declare a license; check
each repository before redistributing or reusing their code.
