# WARP.md
使用中文交流

This file provides guidance to WARP (warp.dev) when working with code in this repository.

Repository: xiaozhi-esp32-server — a multi-module backend for ESP32-based voice AI devices. Core modules live under main/:
- xiaozhi-server: Python realtime engine (WebSocket + HTTP) for ASR/LLM/TTS/VAD/VLLM, plugins, OTA.
- manager-api: Java Spring Boot admin API (MySQL + Redis) providing configuration, users/devices, OTA metadata.
- manager-web: Vue 2 admin console.
- manager-mobile: uni-app (Vue 3 + Vite) mobile admin.

1) Common commands

Python core (main/xiaozhi-server)
- Install (pip):
  pip install -r main/xiaozhi-server/requirements.txt
- Run server (checks ffmpeg/libopus; HTTP for OTA/vision; WS for device IO):
  python main/xiaozhi-server/app.py
- Performance tester (interactive menu over subtools):
  python main/xiaozhi-server/performance_tester.py
- Docker (server-only):
  docker compose -f main/xiaozhi-server/docker-compose.yml up -d
- Docker (full stack):
  docker compose -f main/xiaozhi-server/docker-compose_all.yml up -d

Java admin API (main/manager-api)
- Dev run (hot reload via Spring Boot):
  mvn -f main/manager-api spring-boot:run
- Build jar (skip tests as in pom):
  mvn -f main/manager-api clean package -DskipTests
- Run tests:
  mvn -f main/manager-api test -DskipTests=false
- Run a single test class or method:
  mvn -f main/manager-api -Dtest=MyTest test
  mvn -f main/manager-api -Dtest=MyTest#should_do_something test

Web admin (main/manager-web, Vue 2)
- Install deps:
  npm --prefix main/manager-web install
- Dev server:
  npm --prefix main/manager-web run serve
- Production build:
  npm --prefix main/manager-web run build
- Bundle analysis:
  npm --prefix main/manager-web run analyze

Mobile admin (main/manager-mobile, uni-app + Vue 3 + Vite, pnpm)
- Install deps:
  pnpm -C main/manager-mobile install
- Dev (H5 default):
  pnpm -C main/manager-mobile dev
- Build (H5 default):
  pnpm -C main/manager-mobile build
- Platform-specific dev/build examples:
  pnpm -C main/manager-mobile dev:mp-weixin
  pnpm -C main/manager-mobile build:app-android
- Lint and fix:
  pnpm -C main/manager-mobile lint
  pnpm -C main/manager-mobile lint:fix
- Type check:
  pnpm -C main/manager-mobile type-check

Docker images (local build)
- Server image:
  docker build -f Dockerfile-server -t xiaozhi-esp32-server:server_local .
- Web+API image:
  docker build -f Dockerfile-web -t xiaozhi-esp32-server:web_local .

CI/CD
- Pushing a tag vX.Y.Z triggers GitHub Actions to build and push multi-arch images to ghcr.io as:
  ghcr.io/<org>/xiaozhi-esp32-server:server_<version>, server_latest
  ghcr.io/<org>/xiaozhi-esp32-server:web_<version>, web_latest
  Secret required: TOKEN (packages:write).

2) High-level architecture and flows

System overview
- ESP32 devices communicate with the Python core over WebSocket for realtime audio uplink and TTS downlink. Control/status messages are JSON text over the same WS.
- The Python core exposes an HTTP server used for OTA firmware delivery and selected tools (e.g., vision explain endpoint). It loads configuration from a local YAML and/or pulls dynamic config from the admin API.
- The admin API (Spring Boot) persists users/devices/config to MySQL and uses Redis for caching; it also manages OTA metadata and TTS timbres. It powers both the web and mobile admin UIs.
- The web admin (Vue 2) and the mobile admin (uni-app) call the admin API’s REST endpoints to manage the system (models, keys, parameters, devices, users).

Key ports and endpoints (default)
- WebSocket (device IO): ws://<host>:8000/xiaozhi/v1/
- HTTP (server utilities): http://<host>:8003/xiaozhi/ota/, http://<host>:8003/mcp/vision/explain
- Admin API + Console (full stack docker): http://<host>:8002 (reverse-served by the web image)

Provider/plugin model (from CLAUDE.md and code organization)
- Python core uses a provider pattern for AI capabilities under core/providers/: ASR, LLM, TTS, VAD, VLLM, Intent, Memory.
- Providers implement common interfaces; selection is via configuration. This enables mixing local models (e.g., FunASR) and hosted APIs (e.g., OpenAI/Zhipu/Doubao/Alibaba).
- Tooling and integrations include: IoT control, MCP client/server/endpoint, and server-side plugins. Functional plugins reside under plugins_func/functions/ and are hot-loaded/registered at startup, enabling LLM function-call execution paths.

Configuration flow and precedence
- Primary local config path for the Python core: main/xiaozhi-server/data/.config.yaml (user overrides) over main/xiaozhi-server/config.yaml (defaults).
- For full-stack setups, the Python core can pull config from the admin API (manager-api). The admin API exposes parameters (including server.secret). The Python core reads manager-api.url and manager-api.secret from .config.yaml; if present and valid, it fetches and applies server-side configuration and can hot-swap providers.
- The Python core auto-derives an auth_key (JWT) from manager-api.secret when present; otherwise it generates a UUID for securing select HTTP endpoints.
- Web admin environment: main/manager-web/.env* (VUE_APP_API_BASE_URL points to the API mount, default “/xiaozhi”). Adjust for non-default API hosts during local dev.

Data and runtime notes
- Models: the default local ASR expects a SenseVoiceSmall model file at main/xiaozhi-server/models/SenseVoiceSmall/model.pt (mounted in docker examples). Without it, configure ASR to use a hosted API instead.
- OTA: the Python core’s HTTP server serves firmware via /xiaozhi/ota/ (single-service) or, in full-stack docker, the web+API service proxies the OTA path.
- Logs/health: the WebSocket server logs the active WS URL at startup; do not “open” this URL in a browser. Use the test/test_page.html under the Python module for local WS experiments.

3) How to run specific tasks

- Start only the Python core for device testing:
  python main/xiaozhi-server/app.py
  Ensure ffmpeg and libopus are available (docker image installs them; for local, install via OS package manager or conda as per docs/Deployment*.md).

- Start full stack with docker compose (recommended for integrated testing):
  docker compose -f main/xiaozhi-server/docker-compose_all.yml up -d
  Then open the admin console at http://127.0.0.1:8002 and create the first user (super admin).
  After entering model API keys in Model Config, restart the server container if the core needs to pick up new provider settings.

- Point an ESP32 device at your instance:
  WebSocket: ws://<your-host-ip>:8000/xiaozhi/v1/
  OTA: http://<your-host-ip>:8003/xiaozhi/ota/ (server-only) or http://<your-host-ip>:8002/xiaozhi/ota/ (full stack)

- Run Java API tests selectively:
  mvn -f main/manager-api -Dtest=SomeControllerTest test
  mvn -f main/manager-api -Dtest=SomeControllerTest#methodName test

- Build production artifacts locally:
  docker build -f Dockerfile-server -t xiaozhi-esp32-server:server_local .
  docker build -f Dockerfile-web -t xiaozhi-esp32-server:web_local .

4) Important references pulled into this guide

- README.md and README_en.md: deployment matrix, supported providers, demo links, and testing tools. See docs/ for detailed deployment and integration guides (Deployment.md, Deployment_all.md, performance_tester.md, homeassistant-integration.md, mcp-* docs, voiceprint-integration.md, etc.).
- CLAUDE.md: succinct overview of module boundaries and the provider/tool systems; key dev commands for each module align with the scripts and manifests in this repo.
- CI workflow (.github/workflows/docker-image.yml): tag-driven multi-arch builds for server and web images published to ghcr.io; requires TOKEN secret.

5) Gotchas and environment specifics

- On fresh local (non-docker) setups, ensure ffmpeg and libopus are installed before running the Python core. Docs include conda-based steps for Windows users.
- For full-stack docker, set manager-api.secret into the Python core’s data/.config.yaml and use the service DNS name http://xiaozhi-esp32-server-web:8002/xiaozhi inside the compose network so the core can pull config.
- Default ports: 8000 (WS), 8003 (HTTP for OTA/vision), 8002 (admin console/API). Adjust firewalls accordingly.
- Some performance tester subtools require that you first place your model/API keys into data/.config.yaml; only configured providers are exercised.

