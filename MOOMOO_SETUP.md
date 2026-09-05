# Moomoo Setup Guide

This guide covers everything needed to run FriesTrader with Moomoo: installing
OpenD (the local gateway daemon), configuring the `moomoo-api-mcp` MCP server
for Claude Code, and connecting your account.

## Prerequisites

- A [Moomoo](https://www.moomoo.com) account with OpenAPI enabled
- [uv](https://github.com/astral-sh/uv) installed (`curl -LsSf https://astral.sh/uv/install.sh | sh`)
- [Claude Code](https://claude.ai/claude-code)

## 1. Enable Moomoo OpenAPI

1. Log in to [Moomoo OpenAPI portal](https://www.moomoo.com/OpenAPI)
2. Apply for OpenAPI access — this is a separate opt-in from your regular
   Moomoo account (approval is typically instant for retail accounts)
3. Note your trading password — you'll need it to unlock REAL account access

## 2. Install and Configure OpenD

**OpenD** is a locally-running gateway daemon that bridges the Moomoo API to
your machine. The MCP server talks to OpenD; OpenD talks to Moomoo's servers.

### Download

Download the correct OpenD binary for your OS from the
[Moomoo OpenD download page](https://www.moomoo.com/OpenAPI/download):
- **macOS**: `OpenD_Mac.dmg` or the `.tar.gz` archive
- **Windows**: `OpenD_Win.exe`
- **Linux**: `OpenD_Linux.tar.gz`

### Configure OpenD

OpenD uses an `OpenDConfig.xml` configuration file. A minimal working
configuration for US paper trading:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Config>
    <ip>127.0.0.1</ip>           <!-- Listen address -->
    <api_port>11111</api_port>   <!-- Port the MCP connects to -->
    <log_dir>./log</log_dir>
    <login_account>YOUR_MOOMOO_ACCOUNT_NUMBER</login_account>
    <login_pwd>YOUR_MOOMOO_LOGIN_PASSWORD</login_pwd>
    <trade_pwd>YOUR_MOOMOO_TRADE_PASSWORD</trade_pwd>
    <region>US</region>
    <lang>2</lang>               <!-- 1=CN, 2=EN -->
    <env>0</env>                 <!-- 0=production, 1=test -->
    <push_data_type>0</push_data_type>
</Config>
```

**Security note:** `OpenDConfig.xml` contains your Moomoo login and trade
passwords. Keep this file local and never commit it to any repository.

### Run OpenD

**macOS/Linux:**
```bash
cd /path/to/OpenD
./OpenD ./OpenDConfig.xml
```

**Windows:**
```powershell
OpenD.exe OpenDConfig.xml
```

OpenD runs in the foreground. Keep the terminal window open while trading
sessions are active. For automated unattended runs, use a process manager
(launchd on macOS, systemd on Linux, Task Scheduler on Windows) to start
OpenD before the Claude Code scheduled routines fire and shut it down after.

**Verify OpenD is running:** the startup log should show `Moomoo server connected`
and list your accounts. OpenD listens on `127.0.0.1:11111` by default.

## 3. Install the Moomoo MCP Server

The MCP server (`moomoo-api-mcp`) is the bridge between Claude Code and OpenD.
Install it via `uvx` (no permanent install required — `uvx` runs it on demand):

```bash
uvx moomoo-api-mcp
```

Or install permanently:
```bash
uv tool install moomoo-api-mcp
```

The server connects to OpenD at `127.0.0.1:11111` by default. Both OpenD and
the MCP server must be running for any Claude Code session to work.

## 4. Configure Claude Code

Add the MCP server to your Claude Code configuration. For cloud-hosted
scheduled routines, add it to the routine's MCP connections.

**Claude Code MCP config (`~/.claude/mcp_config.json` or similar):**
```json
{
  "mcpServers": {
    "moomoo": {
      "command": "uvx",
      "args": ["--refresh", "moomoo-api-mcp"],
      "env": {
        "MOOMOO_TRADE_PASSWORD": "YOUR_TRADE_PASSWORD_HERE",
        "MOOMOO_SECURITY_FIRM": "FUTUINC"
      }
    }
  }
}
```

**`MOOMOO_SECURITY_FIRM`** values (use the one matching your account's
regulatory entity):
- `FUTUINC` — Futu Inc. (US retail accounts, FINRA/SIPC)
- `FUTUSECURITIES` — Futu Securities (HK accounts)
- `FUTUSG` — Futu Singapore
- Check the Moomoo OpenAPI docs for your region

**Security note:** Do not commit `MOOMOO_TRADE_PASSWORD` to version control.
The `env` block in the config file must remain local. Use environment variables
or a secrets manager for production deployments.

## 5. Find Your Account ID

Moomoo uses numeric account IDs (e.g. `"281756479345015383"`) rather than
human-readable account numbers.

1. Start OpenD and the MCP server
2. Ask Claude to call `get_accounts()` — it returns a list of all your
   accounts (REAL and SIMULATE) with their `acc_id`, `trd_env`, `acc_type`
3. Copy the `acc_id` of the account you want to trade
4. Paste it into `risk_rules.json`'s `acc_id` field
5. Update `wash_sale_avoidance.linked_accounts` with all acc_ids you personally
   control (REAL accounts only for wash-sale purposes)

**SIMULATE accounts** have small numeric IDs (e.g. `"8377516"`) and don't
require trade password unlock. They're ideal for dry-run validation.

## 6. Set Up Your Watchlist

FriesTrader uses a watchlist in your Moomoo app as its primary screening universe.

1. Open the Moomoo app (desktop or mobile)
2. Create a new **watchlist** (custom security group) and name it — remember
   this exact name
3. Add the stocks you want screened daily to this watchlist
4. Paste the exact watchlist name into `risk_rules.json`'s `universe.watchlist_name`

Phase A calls `get_user_security_group()` and `get_user_security(group_name)` to
fetch the watchlist symbols each run. If the name doesn't match exactly, Phase A
will error and stop rather than silently using the wrong list.

## 7. Configure risk_rules.json

Fill in the fields that reference your account:

```json
{
  "acc_id": "YOUR_ACC_ID_HERE",         // from get_accounts()
  "trd_env": "SIMULATE",                // start with SIMULATE (paper trading)
  "moomoo_market": "US",                // US equities
  "starting_capital_usd": 500.00,       // your actual starting balance
  "universe": {
    "watchlist_name": "YOUR_WATCHLIST_NAME_HERE"
  },
  "wash_sale_avoidance": {
    "linked_accounts": ["YOUR_ACC_ID_HERE", "YOUR_OTHER_ACC_ID_HERE"]
  }
}
```

Review all other thresholds — the defaults are illustrative, not a recommendation.

## 8. Paper Trading (SIMULATE mode)

Moomoo's SIMULATE accounts are proper paper trading accounts with virtual
funds. They don't require trade password unlock and behave identically to
REAL accounts for API purposes.

- Start with `trd_env: "SIMULATE"` and `execution.mode: "dry_run"` in
  `risk_rules.json`
- Run at least `dry_run_min_cycles_before_live` cycles before considering
  going live
- Review `trade_log.jsonl` after each cycle — look at rejected candidates
  and stop-loss triggers, not just what "worked"

## 9. Going Live (REAL accounts)

When you're ready to switch to live trading:

1. Set `trd_env: "REAL"` in `risk_rules.json` (by hand, never by the agent)
2. Ensure `MOOMOO_TRADE_PASSWORD` is set in the MCP config
3. Verify `unlock_trade` succeeds in Phase B Step 0 — Phase B won't proceed
   with REAL orders if unlock fails
4. Keep `execution.mode: "dry_run"` until you've also satisfied
   `dry_run_min_cycles_before_live`
5. Only flip `execution.mode` to `"live"` yourself, by hand

**Note:** Moomoo REAL accounts may have additional requirements depending on
your region and regulatory entity (e.g. margin accounts may require additional
verification). Confirm your account is fully enabled for API trading before
switching.

## 10. Running the Scheduled Sessions

OpenD must be running before each Phase A and Phase B Claude Code session.
The MCP server starts automatically via `uvx` when Claude Code launches.

If running as cloud-hosted Claude Code routines:
- Ensure OpenD is running on a machine that's always on (a server or desktop
  that doesn't sleep during market hours)
- The Claude Code session itself runs in the cloud, but it connects to OpenD
  running locally — so OpenD must be accessible. Use a VPN or port-forwarding
  if needed, or run on a cloud VM.

For local runs, add an OpenD startup step to your cron/launchd config that
fires before the Claude Code sessions.

## Troubleshooting

**"OpenD not connected"**: OpenD is not running, or the MCP server can't reach
it on `127.0.0.1:11111`. Check that OpenD is running and has successfully
logged in (look for "Moomoo server connected" in its log output).

**`unlock_trade` failed**: wrong `MOOMOO_TRADE_PASSWORD`, or OpenD isn't
running in REAL mode. Verify the password in your MCP config matches your
Moomoo trade password exactly.

**"No matching account for acc_id"**: the `acc_id` in `risk_rules.json` doesn't
match any account returned by `get_accounts()`. Re-run `get_accounts()` and
copy the exact numeric string.

**"Watchlist not found"**: the `universe.watchlist_name` in `risk_rules.json`
doesn't match any security group in Moomoo. Check exact spelling and case —
`get_user_security_group()` returns the exact names; match one of those.

**Orders not filling**: in US markets, `order_type="NORMAL"` is a limit order.
If the market moves away before the fill, the order may expire unfilled at end
of day (`time_in_force="DAY"`). This is expected behavior — FriesTrader doesn't
chase fills.
