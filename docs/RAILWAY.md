# Railway deployment

This repository is ready for Railway, but **nothing has been deployed**. You configure the Railway project and add your own secrets.

## 1. Create the project

1. Sign in at [Railway](https://railway.app/) and choose **New Project → Deploy from GitHub Repo**.
2. Select this private repository (`Lovelycrxshx/kuro-discord-builder`) and the branch you want to deploy.
3. Railway detects `package.json`. The committed `railway.json` sets:
   - Build: `npm run build` (dependencies are installed by Railway's install step)
   - Start: `npm run start`
   - Health check: `GET /health`

No `Dockerfile` is required (Railway's Nixpacks builder is used).

## 2. Add environment variables

In the Railway service → **Variables**, add:

| Variable | Required | Value |
| --- | --- | --- |
| `DISCORD_TOKEN` | Yes | Your bot token. Paste it directly in Railway; do **not** commit it. |
| `DISCORD_CLIENT_ID` | Yes | Discord application ID. |
| `DISCORD_GUILD_ID` | No | Test server ID to register commands only there. Leave unset for global commands. |
| `LOG_CHANNEL_ID` | No | Private audit channel ID. Leave unset to disable Discord mirroring. |
| `PORT` | No | Railway injects this automatically; the app defaults to `3000` locally. |
| `ACTION_LOG_PATH` | No | Defaults to `data/actions.jsonl`. Railway's filesystem is ephemeral — stdout is the durable log. |

Use `.env.example` as the placeholder reference only. Never put real values in a committed file.

## 3. Register slash commands once

Command registration is intentionally NOT run at service startup. Run it once from your machine (with the same `DISCORD_CLIENT_ID` and `DISCORD_TOKEN`) after installing dependencies:

```bash
cp .env.example .env   # fill the same values you used in Railway
npm ci
npm run register
```

With `DISCORD_GUILD_ID` set, commands appear immediately in that server. Without it, Discord can cache global commands for up to an hour. If you use the Railway CLI, `railway run npm run register` also works and reads your Railway variables.

Re-run `npm run register` after any command name or option changes.

## 4. Deploy and verify

1. Railway builds and starts the service. Watch **Deployments → Build Logs** and **Runtime Logs**.
2. Confirm a JSON line like `{"event":"health_server.listening","port":...}`.
3. Confirm `bot.ready` with your bot user ID.
4. Hit `/health` on the Railway-provided domain (or check the deployment health status). It returns:

   ```json
   { "status": "ok", "uptimeSeconds": 12, "guilds": 1, "pendingApplyRequests": 0 }
   ```

5. In Discord, run `/plan`, then `/apply` as the owner, confirm with `APPLY`, and review the audit JSON lines.

## 5. Operational notes

- **Use exactly one replica.** Pending owner confirmations live in memory for 10 minutes. Multiple replicas would send a button to one instance and the modal to another. Set the service replica count to `1`.
- **Logs:** stdout JSON lines are the durable audit source in Railway. `data/actions.jsonl` is useful locally but disappears on redeploy. For a second copy in Discord, set `LOG_CHANNEL_ID` to a private channel.
- **Restart policy:** `restartPolicyType: ON_FAILURE` with up to 10 retries is configured in `railway.json`. Manual restarts clear pending confirmations; users simply run `/apply` again.
- **Secrets:** rotate `DISCORD_TOKEN` immediately if it is ever exposed. Railway variable changes trigger a restart.
- **Permissions after invite:** keep the bot below your own highest role, above `@everyone`, and never grant Administrator. See [SETUP.md](SETUP.md#3-invite-with-minimum-permissions).
- **Rollback:** use Railway's **Deployments → Redeploy** on a previous successful build. Code changes alone never mutate Discord servers; only a confirmed `/apply` does.

## Local parity

To reproduce the Railway runtime locally:

```bash
npm ci
npm run build
npm run start
```

The health server listens on `0.0.0.0:$PORT` and the bot logs the same structured JSON as in Railway.
