# System Architecture Overview

**Status:** Living document - last updated: 2026-05-21

## Overview
This document provides an architectural map of the Hermes agent environment, including dockerized services, local services (notably the locally hosted LLM), custom skills we've developed together, and their interactions. It is intended to be a living wiki document with trackable history.

## Host & Environment
- **Primary Host:** Windows 10/11 (Host OS)
- **WSL2:** Ubuntu-based distribution running under Windows Subsystem for Linux, providing the Linux environment for Hermes agent.
- **Filesystem:** Windows host filesystem mounted under `/mnt/` (e.g., `/mnt/c/Users/<username>`).
- **Docker:** Docker Desktop on Windows with WSL2 integration enabled, allowing Linux containers to be managed from within WSL and accessed via `localhost` or host IP.
- **Hermes Agent Base:** `/home/mataanek/.hermes/hermes-agent` (current working directory for agent operations).
- **User Home:** `/home/mataanek` (within WSL).

## Dockerized Services
The following services run in Docker containers managed via `docker-compose` (from `/home/mataanek/.hermes/webui-mvp/attachments/20260521_080859_9384c2/docker-compose.yml`):

| Service | Purpose | Ports | Notes |
|---------|---------|-------|-------|
| comfyui | Advanced AI image generation workflow | 8188:18188 | GPU-enabled, persistent models/outputs/custom_nodes |
| tei-embeddings | Text embedding generation service | 8088:80 | CPU-based BAAI/bge-small-en-v1.5 model |
| firecrawl-playwright-service | Web scraping service via Playwright | 3000 | Concurrent page limiting |
| firecrawl-redis | Redis cache for Firecrawl | - | Alpine Redis |
| firecrawl-rabbitmq | Message queue for Firecrawl | - | With management plugin |
| firecrawl-nuq-postgres | PostgreSQL for Firecrawl NuQ storage | - | Firecrawl-specific Postgres |
| firecrawl-api | Main Firecrawl API service | 3002:3002 | Depends on Redis, RabbitMQ, Postgres, Playwright |
| vibe-trading | Financial analysis & backtesting engine | 8899:8899 | Built from ./Vibe-Trading directory |
| vibe-trading-mcp | MCP server for vibe-trading | 8900:8900 | SSE transport on port 8900 |
| freshrss | RSS/Atom feed aggregator | 8081:80 | Central Europe timezone, cron schedule |
| news-worker | Processes FreshRSS data for Hermes knowledge | - | Worker that syncs to vault/news clippings |
| (Optional) llama-server | Local LLM serving via llama.cpp | 8080:8080 | GPU-enabled, commented out in compose |
| (Optional) hermes-agent | Hermes agent containerized | 9119:9119 | Dashboard on 9119, commented out |
| (Optional) llama-server-utility | Secondary LLM for utilities | 8081:8080 | Smaller model, commented out |
| (Optional) agent-zero | AgentZero AI framework | 40080:80 | Integrated with local LLM & ComfyUI |

*Note:* The Hermes agent itself is primarily executed natively in the WSL environment (via the `hermes` CLI) for rapid iteration and direct access to local resources (e.g., the locally hosted LLM via `nix_core.sh`). Docker containers are used for auxiliary services that benefit from isolation.

## Local Services (Non-Dockerized)
These services run directly on the WSL host (or are accessible via localhost) and are integral to the agent's operation:

| Service | Access Point | Purpose | Details |
|---------|--------------|---------|---------|
| **Local LLM Engine** | `http://localhost:8080` (or `0.0.0.0:8080`) | Provides text generation for the agent's LLM capabilities. | Started via `/home/mataanek/nix_core.sh` using `llama.cpp`'s `llama-server`. Supports multiple GGUF models (see script for model selection). GPU acceleration (CUDA) enabled via `ngl` flag. Logs to `/home/mataanek/llama_server.log`. |
| **Optional Embedding Engine** | `http://localhost:8090` (commented out in `nix_core.sh`) | Generates text embeddings for retrieval-augmented generation (RAG). | Uses `gte-Qwen2-1.5B-instruct` model when enabled. Logs to `/home/mataanek/llama_embed_server.log`. |
| **Hermes Gateway** | `http://localhost:PORT` (default from `hermes config`) | Exposes the Hermes agent as an HTTP API for external integrations (e.g., webhooks, custom frontends). | Started by `nix_core.sh`: `setsid $HERMES_BIN gateway > /home/mataanek/.hermes/logs/gateway.log 2>&1 &`. |
| **OpenRGB Controller** | Via `openrgb-control` skill | Controls PC RGB lighting (motherboard, peripherals) via OpenRGB daemon. | Requires OpenRGB service running on host. |
| **Voice Control** | Via `voice-control` skill | Wake word detection, speech-to-text, intent parsing, and text-to-speech for voice-driven interactions. | Uses local Whisper for STT and NeuTTS (or custom) for TTS. |
| **Netatmo Weather Station** | Via `netatmo-control` skill | Collects and stores weather data (temperature, humidity, CO2) from Netatmo devices for long-term trend analysis in the wiki. | Data appended to `wiki/health-data.md` or similar. |
| **Philips Hue Lights** | Via `hue-control` or `openhue` skills | Control Philips Hue lighting system (lights, scenes, groups). | Communicates with Hue Bridge via local network. |
| **Smart Home Integrations** | Via `smart-home` skill | Framework for integrating additional smart home devices (e.g., thermostats, switches) and exposing them to the agent. | Base skill for home automation orchestration. |
| **Health Data Extraction** | Via `health-data-extraction` skill | Pulls detailed daily health and workout data from Apple Health exports (via Health Auto Export) into a structured wiki for training planning and feedback. | Processes JSON exports from `/home/mataanek/Apple_Health_export/` or similar. |
| **Garmin Connect** | Via `garmin-connect` skill | Workflow for establishing and maintaining a persistent connection to Garmin Connect for health/fitness data retrieval. | Periodically syncs activity, sleep, heart rate data. |
| **Email (Himalaya)** | Via `himalaya` skill | IMAP/SMTP email client for reading, sending, and managing email from the terminal. | Configured for user's email accounts. |
| **Social Media (X/Twitter)** | Via `xitter` or `xurl` skills | Interact with X/Twitter: post, search, DM, media, bookmarks, etc. | Uses official X API credentials. |
| **Spotify** | Via `spotify` skill | Play, search, queue, manage playlists and devices on Spotify. | Requires Spotify Premium and OAuth token. |
| **Obsidian Vault** | Via `obsidian` skill | Read, search, and create notes in the user's Obsidian vault (located at `/home/mataanek/Obsidian/` or similar). | Enables agent to augment personal knowledge base. |
| **Notion** | Via `notion` skill | Interface with Notion API for pages, databases, and automation. | Requires integration token. |
| **Airtable** | Via `airtable` skill | Interact with Airtable bases via REST API. | Requires API key and base ID. |
| **Linear** | Via `linear` skill | Manage issues, projects, teams via GraphQL + curl. | Requires API key. |
| **Google Workspace** | Via `google-workspace` skill | Access Gmail, Calendar, Drive, Docs, Sheets via `gws` CLI or Python. | Uses service account or OAuth. |
| **Teams Meeting Pipeline** | Via `teams-meeting-pipeline` skill | Operate the Teams meeting summary pipeline via Hermes CLI — summarize meetings, inspect pipeline status, replay jobs, manage Microsoft Graph subscriptions. | Requires Microsoft 365 credentials. |
| **Band Wiki Update** | Via `band-wiki-update` skill | Standardized process for updating Mataanek's band wiki with new album/single information, lyrics, and metadata from mataanek.eu and raw source files. | Keeps the band's discography current. |
| **Whiskey Wiki Update** | Via `Whiskey Wiki Update` skill | Standardized process for adding new whiskey entries to Mataanek's Obsidian-style whiskey wiki. | Ensures consistency with existing entries. |
| **Maps & Geocoding** | Via `maps` skill | Geocode addresses, points of interest, routes, and timezones using OpenStreetMap/OSRM. | No API key needed. |
| **PDF OCR & Documents** | Via `ocr-and-documents` skill | Extract text from PDFs and scans (using `pymupdf`, `marker-pdf`). | Enables agent to process scanned documents. |
| **Nano-PDF** | Via `nano-pdf` skill | Edit PDF text/typos/titles via `nano-pdf` CLI. | For quick PDF corrections. |
| **PowerPoint** | Via `powerpoint` skill | Create, read, edit `.pptx` decks, slides, notes, templates. | For presentation automation. |

## Custom Skills Developed Together
Below are the skills we've created or significantly customized together for this specific environment (excluding standard Hermes/third-party skills):

- **Home Automation & IoT**: `smart-home`, `openrgb-control`, `voice-control`, `netatmo-control`, `hue-control`, `openhue`
- **Health & Personal Analytics**: `health-data-wiki-update`, `health-data-extraction`, `garmin-connect`
- **Communication & Media**: `freshrss-news-*`, `himalaya`, `xitter`, `xurl`, `spotify`, `youtube-content`
- **Productivity & Knowledge Management**: `notion`, `obsidian`, `airtable`, `linear`, `maps`, `nano-pdf`, `powerpoint`, `teams-meeting-pipeline`, `band-wiki-update`, `Whiskey Wiki Update`, `google-workspace`, `quick-note`
- **Software Development & Engineering**: `hermes-agent-skill-authoring`, `code-review`, `debugging-hermes-tui-commands`, `requesting-code-review`, `subagent-driven-development`, `systematic-debugging`, `test-driven-development`, `writing-plans`, `agent-soul-md-update`
- **MLOps & AI/ML**: `hermes-neutts-voice` (our custom voice cloning)
- **Financial Analysis & Trading**: `vibe-trading` (our complete finance toolkit)
- **Miscellaneous Utilities**: `rule-0-verification-protocol`, `yuanbao`, `dogfood`, `czech-corrector`

*Note:* This list focuses on skills we've created or significantly customized together. Standard skills like `claude-code`, `codex`, `github-*`, `ascii-art`, `architecture-diagram`, `baoyu-comic`, `baoyu-infographic`, `claude-design`, etc. are available but not included here as they represent third-party integrations rather than our joint development work.

## Skills Architecture Diagram
For a visual representation of how our custom skills interact with the core Hermes agent and external services, see the separate file: [skills-architecture.md](./skills-architecture.md)

## Local LLM Entry Point: `nix_core.sh`
The script `/home/mataanek/nix_core.sh` is the primary method for initializing the local AI backend and Hermes gateway. It performs the following steps:
1. Stops any existing `llama-server` and `hermes` processes.
2. Sets GPU environment variable (`export GGML_CUDA_NO_VMM=1`).
3. Defines paths to the model directory (`/home/mataanek/models`), llama.cpp binary, and Hermes binary.
4. Selects a GGUF model (default choice 7: `gemma-4-E4B-it-UD-Q8_K_XL.gguf`) based on argument or fallback.
5. Constructs the `llama-server` command with appropriate flags (GPU offloading, context size, chat template, sampling parameters, etc.) and starts it in the background, logging to `/home/mataanek/llama_server.log`.
6. (Optional) Starts an embedding server on port 8090 (currently commented out).
7. Waits for the LLM engine to bind to port 8080 (native Bash TCP check).
8. Starts the Hermes gateway in the background, logging to `/home/mataanek/.hermes/logs/gateway.log`.
9. Prints "NIX CORE HANDOVER SUCCESSFUL."

This script enables rapid switching between different local LLMs (e.g., Gemma, Qwen, Ministral) and ensures the Hermes agent has a responsive LLM backend for reasoning, code generation, and natural language understanding.

## Interactions & Data Flow
- **User → WebUI/Gateway:** The user interacts with the Hermes agent either through the web interface (WebUI) at `http://localhost:3000` or via direct API calls to the Hermes gateway (default port from `hermes config`).
- **Agent → LLM:** When the agent needs to generate text (for reasoning, responding, skill execution), it sends a request to the local LLM engine at `http://localhost:8080`.
- **Agent → Skills:** The agent loads and executes skills (both built-in and custom) to perform tasks. Skills may call internal tools (`web_search`, `terminal`, `execute_code`, etc.) or external APIs (e.g., Spotify, Notion, weather services).
- **Agent → Local Services:** Skills can directly invoke local services (e.g., `openrgb-control` to change lighting, `netatmo-control` to fetch weather, `obsidian` to read/write notes).
- **Data Persistence:** Interaction logs, wiki updates, skill outputs, and configuration changes are persisted under `/home/mataanek/.hermes/` (wiki, logs, audio cache, etc.).
- **Feedback Loops:** Outputs from skills (e.g., news digest, health data summary) are often written back to the wiki or shared with the user via the WebUI/gateway.

## Related Documentation
- [Hermes Agent Skill Authoring](../hermes-agent-skill-authoring.md) - for extending agent capabilities
- [Team Workflows](../team-agents.md) - for multi-agent coordination
- [Ideation Wiki](../ideation/) - for capturing and expanding ideas
- [Health Data Wiki Update](../health-data-wiki-update.md) - details on health data ingestion pipeline
- [Smart Home Integration Guide](../smart-home.md) - framework for adding new device types
- [Local LLM Operations](./nix_core.sh) - the script that starts the local LLM and gateway
- [Skills Architecture](./skills-architecture.md) - visual representation of custom skills interactions

## History & Updates
- **2026-05-21:** Major overhaul to include complete dockerized services list from compose.yml, filtered custom skills we developed together, added skills architecture diagram reference, and updated environment details.
- **2026-04-25:** Initial creation - outlined Dockerized services (WebUI, News Aggregator, Hermes Agent Core) and basic environment.
- **2026-03-15:** Added sections for smart home skills and voice control integration.
- **2026-02-01:** Documented Hermes WebUI setup and initial news aggregator.

*Next steps:*
- Add Mermaid architecture diagram showing service interactions (separate file).
- Detail inter-service communication mechanisms (e.g., Redis pub/sub, direct HTTP calls).
- Map out data storage locations (vector DBs for RAG, wiki file structure, logs).
- Document deployment and scaling considerations for production.
- Include diagrams for data flow in common workflows (e.g., "answer a question" → LLM → skill → external API → wiki update).

---\
*Automatically generated as part of architectural mapping initiative. To be refined iteratively.*