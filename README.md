---
title: OpenClaw by Charlie
emoji: 🦞
sdk: docker
app_port: 7860
---

# OpenClaw on Hugging Face — by Charlie

Host your own OpenClaw dashboard and Telethon Telegram bot: clone this small repository, attach your own private bucket, add your secrets, and upload it to a Hugging Face Docker Space.

The Dockerfile pulls the existing [OpenClaw Telethon fork](https://github.com/ArnavSingh76533/openclaw-for-telethon). This repository contains hosting files, not another copy of OpenClaw. Every new installation starts with fresh private data. No accounts, tokens, models, chats, browser logins, or backups are included.

## Included features

- The pinned fork's OpenClaw dashboard, agents, model integrations, tools, and web interfaces.
- Python Telethon/MTProto Telegram communication connected to the existing OpenClaw integration.
- Chromium and the pinned Fortress engine, available for browser configuration.
- Verified native OpenClaw checkpoints and restoration using a private Hugging Face bucket.
- HTTP/WebSocket forwarding, process supervision, and Gateway health recovery.

OpenClaw remains a Node/TypeScript application; its Telegram transport runs in Python. Mini Apps and external integrations retain their own hosting, credentials, and setup requirements. These files do not add AGY/Antigravity or Claude Code.

## 1. Clone

```sh
git clone https://github.com/ArnavSingh76533/openclaw-on-hf-by-charlie-.git
cd openclaw-on-hf-by-charlie-
```

You do not need to clone the large application repository yourself.

## 2. Create your Space and bucket

1. Create a [new Space](https://huggingface.co/new-space) with **Docker**, a blank template, and preferably **Private** visibility. CPU hardware with 16 GB RAM matches the original deployment.
2. Create a **private** [Storage Bucket](https://huggingface.co/new-bucket), for example `YOUR_USERNAME/openclaw-state`.
3. In the Space settings, attach it as a **read-write** volume at exactly `/data/openclaw-backups`.
4. Use an empty bucket for fresh state and a dedicated bucket per installation. Do not let two Spaces write to the same checkpoint directory.

Startup deliberately fails if the bucket directory is missing. An ordinary local directory at that path is not a substitute: it disappears on container replacement. Keep live SQLite files on local disk; this bucket stores completed archives.

See Hugging Face's [Docker Space guide](https://huggingface.co/docs/hub/spaces-sdks-docker) and [attached volume documentation](https://huggingface.co/docs/hub/spaces-storage). Account storage allowances and hardware availability apply.

## 3. Add your environment values

In **Space Settings → Variables and secrets**, add these as **Secrets**:

| Secret | Value |
| --- | --- |
| `OPENCLAW_GATEWAY_TOKEN` | A new long random dashboard/Gateway password. Keep it for your first login. |
| `TELEGRAM_API_ID` | Your numeric application ID from [my.telegram.org](https://my.telegram.org). |
| `TELEGRAM_API_HASH` | Its matching application hash. |
| `TELEGRAM_BOT_TOKEN` | Your bot token from Telegram's verified `@BotFather`. |

Generate a Gateway token locally:

```sh
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

Optional values:

| Name | Type | Purpose |
| --- | --- | --- |
| `TELEGRAM_ALLOW_FROM` | Variable | Comma-separated numeric **user** IDs, e.g. `123456789,987654321`. Enables a DM allowlist; omit for pairing. |
| `OPENAI_API_KEY` | Secret | Only for intentional API-key OpenAI authentication; not needed for supported OAuth login. |
| `ANTHROPIC_API_KEY` | Secret | Only for intentional Anthropic API authentication. A Claude subscription is separate. |

The image already sets internal paths, ports, and the Python executable. The attached bucket provides storage access; no runtime `HF_TOKEN` is needed by this template.

`.env.example` is a checklist only. Neither Hugging Face nor this startup script loads it automatically. Enter values in the Space settings. Never commit a filled environment file, token, or Telegram session.

## 4. Upload to the Space

Install the official CLI and sign in locally with permission to write to your Space:

```sh
python -m pip install --upgrade huggingface_hub
hf auth login
hf upload YOUR_USERNAME/YOUR_SPACE . . --repo-type space --include "Dockerfile" --include "README.md" --include "start-telegram.mjs" --include "space-state.mjs" --include "space-state.test.mjs" --include "TELEGRAM-HOSTED.md" --include ".gitattributes" --include ".dockerignore"
```

Replace `YOUR_USERNAME/YOUR_SPACE`. This explicit file list uploads only eight hosting/documentation files. Git history and your local environment files are excluded. The [official upload command](https://huggingface.co/docs/huggingface_hub/guides/cli#upload-an-entire-folder) does not delete existing remote files, so start with a blank Space.

Alternatively, use your Space's **Files → Add file → Upload files** and upload those same eight files to the root. Do not upload `.git` or filled secrets files. The README YAML selects Docker and port 7860.

GitHub updates do not automatically update your Space. Pull this repository and repeat the upload command to deploy a later template change.

## 5. Connect, select a model, and test

Wait for the image build and then the runtime compilation. The source deployment took roughly **5–8 minutes for cold-start compilation**, after image building; your hardware/network can differ. Watch **Build logs**, then **Container logs**.

Open the Space's App or its direct `https://YOUR-SPACE.hf.space` address. Connect using your `OPENCLAW_GATEWAY_TOKEN`. If asked for a Gateway URL, use the app host with `wss://`.

Configure a model/account using the methods supported by this pinned OpenClaw version. An optional provider key alone does not guarantee a default model is selected. Complete any OAuth sign-in with your own account and test a dashboard reply. No subscription login or default paid model is preconfigured. Claude Code is not installed by this template.

Follow [TELEGRAM-HOSTED.md](TELEGRAM-HOSTED.md) to pair your user, permit groups, and test the bot. DMs default to pairing and groups to an allowlist.

## Browser configuration

Chromium is at `/usr/bin/chromium`; Fortress's launcher is `/home/node/fortress/tilion`. Select the executable and a managed browser profile in OpenClaw's browser settings, using headless mode and the container's required sandbox setting. Fresh state does not automatically select a browser profile.

Fortress may require activation with your own account. Keep activation files under `/home/node/.openclaw/fortress-license`, linked from `/home/node/.tilion`, and browser profiles inside `/home/node/.openclaw` so checkpoints can retain them. Site sign-ins and verification remain necessary; no browser guarantees access to every site. Gmail browser login alone does not configure email push notifications or monitoring.

## How it works

```text
Browser → Hugging Face HTTPS → proxy :7860 → Gateway 127.0.0.1:18789
                                             ├─ agents / models / tools / UI
                                             └─ Telegram adapter → Python Telethon → Telegram

Local /home/node/.openclaw → verified checkpoint → private bucket /data/openclaw-backups
Fresh container           ← verified restore    ← latest completed checkpoint
```

The image fetches pinned application commits, installs dependencies and browsers, and switches to UID 1000. Container startup compiles OpenClaw and launches `start-telegram.mjs`. That script restores state before reading configuration when local `openclaw.json` is absent. Existing local configuration survives supervisor restarts. Startup enables Telegram, adds its default account binding to agent `main` if absent, and preserves other configuration.

The proxy handles HTTP and WebSocket traffic. Child processes restart after exit. Every 20 seconds a health probe checks the Gateway; three consecutive failures trigger a Gateway restart without resetting state.

Backups use OpenClaw's native create/verify/restore commands with a **600-second timeout**. A first checkpoint is attempted after 15 seconds, then at 120-second intervals; configuration changes also trigger a debounced checkpoint. An archive is copied to the bucket, SHA-256 checked, and published by its manifest. The latest three completed generations are retained. Failed latest restores stop startup for investigation instead of silently resetting state or automatically choosing an older archive.

Only files included in the native OpenClaw state backup persist. Keep durable workspaces and managed browser profiles in the state directory or arrange separate storage. Packages installed interactively elsewhere disappear on rebuild; permanent dependencies belong in the Dockerfile.

Checkpoints are **not continuous replication**. Crashes can lose changes since the last completed checkpoint; large browser profiles can make that window longer than two minutes. Before a planned restart, wait for a recent `[space-state] Saved verified checkpoint` log. Do not erase bucket contents to fix a startup error.

The proxy retains the source deployment's loopback header handling and automatic device approval for Gateway token holders. Treat this as a single-owner installation: the token grants powerful access to agents, files, and tools. Keep the Space and bucket private and share the token only with trusted operators. Archives can contain credentials/browser sessions; hashes verify integrity, not encryption.

## Files and versions

| File | Role |
| --- | --- |
| `Dockerfile` | Installs the pinned application, Python transport and browsers; exposes port 7860. |
| `start-telegram.mjs` | Restores state, prepares configuration, supervises services and checkpoints. |
| `space-state.mjs` | Verified checkpoint creation, restore, retention and health watchdog. |
| `space-state.test.mjs` | Isolated persistence/watchdog tests with a fake backup CLI. |
| `TELEGRAM-HOSTED.md` | Telegram setup and troubleshooting. |
| `.env.example` | Placeholder-only environment checklist. |
| `.dockerignore`, `.gitignore`, `.gitattributes` | Restrict build context, exclude private/runtime files, preserve LF text. |

OpenClaw source: [`f0b4a9f7ce9e869f43998d75e5fc8cb7856ccb24`](https://github.com/ArnavSingh76533/openclaw-for-telethon/commit/f0b4a9f7ce9e869f43998d75e5fc8cb7856ccb24), with the two-file Telegram codec/test overlay from [`4fd651f5c82b98c06edd82f81655deb692a72376`](https://github.com/ArnavSingh76533/openclaw-for-telethon/commit/4fd651f5c82b98c06edd82f81655deb692a72376). Telethon `1.45.0`, Node 24 Bookworm, Bun `1.4.3`, checksum-verified Fortress `v153.0.8010.36`.

Updating the template does not automatically update OpenClaw. Review source pin changes carefully: the transport update-journaling hook is tied to its Telethon version. Dependencies retain their upstream licenses and terms. This is a community hosting template.

## Validation and troubleshooting

With Node 24:

```sh
node --check start-telegram.mjs
node --check space-state.mjs
node --test space-state.test.mjs
```

The persistence tests simulate the native CLI; they do not authenticate to Telegram, call a model, or prove a real cloud restore. Image builds also run the fork's Python transport tests and these persistence tests.

| Symptom | Check |
| --- | --- |
| Missing bucket / permission denied | Exact mount path, bucket permissions and read-write mode. |
| Starting / 502 / connecting | Runtime compilation and Gateway logs; wait for startup to finish. |
| Gateway secret rejected | This Space's Gateway token, not the bot token or another deployment's token. |
| Telegram connected but no answer | Model reply test, pairing, group/user policies and Telegram logs. |
| State seems reset | Verify the mount and completed manifests before making more changes. |
| Checkpoint failed | Bucket access, available storage, profile size and native backup logs. |
| Duplicate replies / session errors | Stop the same bot/session on other running deployments. |

On your own Space, verify a dashboard reply, a permitted Telegram reply, a completed checkpoint, and a planned restart/restore with a harmless saved setting.
