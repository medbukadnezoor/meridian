# Meridian

Autonomous Meteora DLMM liquidity management for Solana.

Meridian is a configurable agent runner for finding Meteora DLMM pools, opening liquidity positions, monitoring live risk, and closing positions through deterministic safety rules plus LLM-assisted review.

## Branch Guide

This public repository uses `experimental` as the active public mainline.

- `experimental`: current public branch to clone and run if you want the latest Meridian code.
- `nanocap-v1`: separate nanocap research branch with smaller-cap presets and extra mean-reversion safety experiments.
- legacy/default branches: older upstream-style code; use them only for historical comparison.

Live owner configuration, wallets, API keys, backups, and exact production sizing are intentionally not included. Start from `user-config.example.json`, copy it to `user-config.json`, and use your own wallet, RPC, keys, thresholds, and risk limits.

## What It Does

- Screens Meteora DLMM pools using configurable market, liquidity, holder, fee, organic-score, and warning filters.
- Ranks candidate pools before LLM review, including persisted signal snapshots and Darwin-style performance feedback.
- Opens and manages DLMM positions through the Meteora DLMM SDK.
- Tracks open-position PnL, fees, range state, active-bin evidence, and close reasons.
- Applies deterministic risk exits before or alongside LLM management decisions.
- Sends Telegram cycle reports and owner-approved command responses when configured.
- Keeps research and shadow checks read-only until they are explicitly promoted.

## Current Safety Features

The active codebase includes several live-trading hardening layers:

- Candidate lease guard: manual or automated deploys must match a fresh same-process candidate lease before execution.
- Screening threshold guard: stale or below-threshold pools are rejected before deploy.
- Stop-loss policy: soft confirmation, hard stop, early-dump handling, and emergency direct-close paths are configurable.
- Rolling drawdown exit: closes positions that give back too much from a recent peak.
- Supertrend loss exit: confirmed bearish trend loss can escalate to an urgent direct close.
- Relay retry evidence: relay aborts and retry behavior are recorded for diagnosis.
- PnL snapshot logging: compact JSONL records help inspect MAE/MFE, false stops, and late exits.
- Active-bin oracle recorder: shadow-only active-bin and velocity evidence collection for future rug/fast-drop exits.
- Single-sided bid/ask guard: available on the nanocap branch to enforce SOL-only mean-reversion deploy shape.

The bot is still trading software. These checks reduce specific known failure modes; they do not remove market, execution, liquidity, RPC, or model risk.

## Architecture

Meridian runs scheduled loops:

| Loop | Purpose |
|---|---|
| Screening | Pull pool candidates, apply deterministic filters, rank candidates, and ask the screening model whether to deploy. |
| Management | Review open positions, PnL, fees, range state, memory, and close/redeploy opportunities. |
| Fast risk checks | Apply deterministic stop, early-dump, rolling drawdown, and urgent-exit paths without waiting for normal LLM cycles when configured. |
| Research/shadow | Record evidence for proposed filters without changing live behavior. |

Primary data sources:

- `@meteora-ag/dlmm` SDK for DLMM pool and position interactions.
- Solana RPC for chain reads and transactions.
- Pool discovery/screening APIs for candidate metadata.
- Meteora/LP position PnL APIs where configured.
- Optional external providers for enrichment, chat, and research.

LLM calls use OpenAI-compatible chat APIs by role. You can route screening, management, and general chat to different providers through config. Local router experiments can also use [CLIProxyAPI](https://github.com/mario-andreschak/CLIProxyAPI), but production routing should be verified with your own endpoint and keys before enabling live decisions.

## Requirements

- Node.js 20 recommended.
- Solana RPC endpoint.
- Solana trading wallet.
- OpenAI-compatible LLM provider key or local compatible router.
- Optional Telegram bot token and owner allowlist.

## Services And Keys

Minimum live setup:

| Service | Key or Config | Required | What Happens If Missing |
|---|---|---|---|
| Solana RPC | `RPC_URL` or `rpcUrl` | Yes | Chain reads and transactions fail. |
| Trading wallet | `WALLET_PRIVATE_KEY` or `walletKey` | Yes for live | Bot can run read-only/status poorly, but cannot deploy or close. |
| LLM provider | role API keys such as `DEEPSEEK_API_KEY`, `OPENROUTER_API_KEY`, or `LLM_API_KEY` | Yes for agent decisions | Screening/management chat calls fail; deterministic exits can still run where no LLM is needed. |
| Telegram | `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`, `TELEGRAM_ALLOWED_USER_IDS` | No | Bot runs without Telegram control or notifications. |

Optional services:

| Service | Key or Config | Toggle | Behavior |
|---|---|---|---|
| Agent Meridian relay | `agentMeridianApiUrl`, `publicApiKey`, `lpAgentRelayEnabled` | `lpAgentRelayEnabled` | If enabled, open-position reads try Agent Meridian first. Normal non-urgent closes may try relay zap-out first. If relay fails before submit, the bot falls back. |
| LPAgent direct | `LPAGENT_API_KEY` | key presence | Used as an open-position/PnL fallback after relay failure, or as supplemental PnL data after Meteora discovery. If missing, this layer is skipped. |
| Meteora APIs | none | always used | Final open-position fallback is Meteora portfolio plus Meteora DLMM PnL APIs. Pool search/discovery also uses Meteora endpoints. |
| GMGN enrichment | `GMGN_API_KEY` | key presence | Adds top-holder/trader risk signals for screening and Darwin context. If missing or failing, GMGN returns `null` and screening continues. |
| OKX enrichment | `OKX_API_KEY`, `OKX_SECRET_KEY`, `OKX_PASSPHRASE`, optional `OKX_PROJECT_ID` | key presence | If OKX keys are present, direct OKX signed requests are used. Without keys, the bot may try Agent Meridian OKX enrichment and public OKX-style calls; failures are logged as unavailable and screening continues. |
| Discord signal candidates | Agent Meridian API config | `useDiscordSignals`, `discordSignalMode` | Disabled by default. If enabled, fetch failures are logged and normal discovery continues unless you configure signal-only behavior. |
| HiveMind | `hiveMindUrl`, `hiveMindApiKey`, `agentId`, `hiveMindPullMode` | non-empty URL/key | Disabled in the public example. If configured, shares/pulls aggregate lessons and presets. |
| Jupiter | `JUPITER_API_KEY` | key presence | Optional for swap-related helpers; the code has a public fallback key/path, but serious live use should provide your own. |

Open-position source order:

1. If `lpAgentRelayEnabled=true`, try Agent Meridian relay.
2. If relay fails and `LPAGENT_API_KEY` exists, try LPAgent.io direct.
3. Fall back to Meteora portfolio and Meteora DLMM PnL APIs.
4. Returned positions are filtered by on-chain wallet owner before management uses them.

Close path order:

1. Urgent closes skip relay and use the local close-liquidity-first path.
2. Non-urgent closes may try Agent Meridian relay zap-out when `lpAgentRelayEnabled=true`.
3. If relay fails before submit, the bot falls back to local Meteora close plus swap/autoswap helpers where configured.

The public `user-config.example.json` keeps relay, HiveMind, and live-size assumptions off by default so a fresh clone does not depend on owner-only services.

## Setup

Clone the active branch:

```bash
git clone -b experimental https://github.com/medbukadnezoor/meridian.git
cd meridian
npm install
```

Create local secrets and config:

```bash
cp .env.example .env
cp user-config.example.json user-config.json
```

Edit `.env` with your own secrets. Do not commit `.env`, private keys, raw auth headers, shell history, or live backup configs.

Edit `user-config.json` with your own thresholds and risk limits. Keep `dryRun` enabled until you have verified candidate selection, deploy construction, close paths, and Telegram permissions in your own environment.

## Common Config Areas

Most runtime behavior is controlled from `user-config.json`:

| Area | Examples |
|---|---|
| Wallet/RPC | `walletKey`, `rpcUrl`, `dryRun` |
| Sizing | `deployAmountSol`, `maxDeployAmount`, `maxPositions`, `minSolToOpen` |
| Screening | market-cap range, TVL, volume, holders, organic score, warnings, bin step, fee/TVL thresholds |
| LLM routing | `screeningModel`, `managementModel`, `generalModel`, base URLs, API keys, fallback models |
| Entry style | strategy preset, chart-indicator preset, intervals, single-sided deploy guards |
| Exits | stop loss, hard stop, early dump, take profit, trailing profit, rolling drawdown, trend-loss exits |
| Logging | PnL snapshots, API activity, decision logs, active-bin oracle files |

Use the example config as a schema guide, not as a recommendation for capital size or thresholds.

## Running

Dry-run first:

```bash
npm run dev
```

Live mode should only be used after you understand the code paths, config, wallet permissions, and failure modes:

```bash
npm start
```

For long-running use, run under a process manager such as PM2 or systemd. Keep each bot instance in its own directory, branch, wallet, logs, and config.

## Commands

When the interactive prompt is enabled:

| Command | Description |
|---|---|
| `/status` | Refresh wallet and open-position state. |
| `/candidates` | Run the screening pipeline and show current candidates. |
| `/thresholds` | Show active screening thresholds and recent performance memory. |
| `/autoresearch` | Show shadow research trials. |
| `/learn` | Study candidate pools and update local lessons where supported. |
| `/stop` | Graceful shutdown. |

Telegram can expose similar control if configured, but inbound commands require explicit chat and user allowlists.

## Logs And Evidence

Useful local artifacts include:

- `logs/api-activity-YYYY-MM-DD.jsonl`
- `logs/decision-log-YYYY-MM-DD.jsonl`
- `logs/pnl-snapshots-YYYY-MM-DD.jsonl`
- `logs/active-bin-oracle-YYYY-MM-DD.jsonl`
- `pool-memory.json`
- `state.json`

Do not publish production logs unless you have scrubbed wallet addresses, position addresses, auth-bearing URLs, private API responses, and strategy-sensitive configuration.

## Verification

Before pushing or deploying code changes, run the repository verifier:

```bash
node scripts/verify-patches.js
```

Some features also have focused verifiers under `scripts/verify-*.js`. Run the focused verifier for the feature you changed, then run the full verifier.

## Nanocap Branch

The `nanocap-v1` branch is a separate research lane for smaller-cap mean-reversion behavior. It may contain stricter deploy-shape guards, different screening defaults, additional runtime proof scripts, and higher-churn experiments than `experimental`.

Use it only if you specifically want to study that lane. Keep its wallet, config, and process isolated from any main bot instance.

## Privacy And Security

- Never commit `.env`, `user-config.json`, private keys, API keys, raw auth headers, Telegram tokens, VPS backups, or live wallet material.
- Prefer `env:NAME` config references for API keys where supported.
- Treat logs as sensitive until scrubbed.
- Rotate any key or token that has ever been pasted into a public place.
- Public examples intentionally omit exact live thresholds, wallet addresses, and production sizing.

## Disclaimer

This software is provided as-is, with no warranty. Autonomous liquidity management and memecoin/nanocap trading can lose funds quickly. Start in dry-run mode, use small size, inspect every configured exit path, and never deploy capital you cannot afford to lose. This is not financial advice.
