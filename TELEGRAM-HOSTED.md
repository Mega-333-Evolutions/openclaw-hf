# Telegram on Hugging Face

This deployment uses the existing Telegram extension with a Python **Telethon/MTProto** transport, not the old HTTP Bot API long-polling deployment. OpenClaw's agent and message-processing code remain in place.

## First setup

1. Add your own `TELEGRAM_API_ID`, `TELEGRAM_API_HASH`, and `TELEGRAM_BOT_TOKEN` as private Space secrets. Never commit tokens or session files.
2. Stop the same bot/session on other deployments; run one active instance.
3. Confirm that your configured model answers in the OpenClaw dashboard.
4. Send `/start` and a test message to your bot in Telegram.
5. DMs require **pairing** by default. Approve the user in OpenClaw's channel/pairing controls. The running container also provides `openclaw pairing --help`.
6. Confirm a real Telegram reply and a completed bucket checkpoint before restarting.

## Users and groups

`TELEGRAM_ALLOW_FROM` optionally accepts comma-separated numeric **user** IDs. Startup then sets `dmPolicy: "allowlist"` and `allowFrom` from that variable. Group IDs are not user IDs. Removing the variable does not automatically reset an already saved policy; edit the persisted channel configuration too.

Groups default to an **allowlist**. Add your bot to the group/channel, grant its required Telegram permissions, and configure the group/chat ID and group policy in OpenClaw. Mention requirements and Telegram bot privacy settings affect which messages arrive and trigger replies. Adding the bot alone does not grant OpenClaw access.

To intentionally allow all DMs, use OpenClaw's configuration controls to set `dmPolicy: "open"` and `allowFrom: ["*"]`. A configured `TELEGRAM_ALLOW_FROM` overrides that DM policy on startup. For all groups, use `groupPolicy: "open"` and review saved per-group restrictions and mention requirements. These broad policies are not the defaults.

The default Telegram account is bound to agent `main`; other saved configuration/bindings are retained. Additional accounts need their own appropriate credentials and policies.

## Persistence and integrations

Keep transport sessions/state in the OpenClaw state directory so native checkpoints include them. Do not mount a bucket over a live SQLite session/database; this template backs up local live state into verified archives.

When rotating a bot token, update the private Space secret and restart. For an invalidated session, use the supported reconnect procedure and logs; do not erase all state or bucket contents.

Mini Apps/web apps still require their own hosting, URLs, BotFather configuration and third-party credentials. This template does not automatically deploy external applications or remove Telegram permission requirements and limits.
