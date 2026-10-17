# Ravanfall-Bot
# RAVENFALL Discord Bot README

## 1. Bot Overview

RAVENFALL Bot is a multi-server Discord operations bot for roleplay/community management. It manages ranch, camp, hideout, shop, cattle, delivery, moonshine, still-part, LOA, clan payout, activity tracking, DM tools, timezone configuration, command access control, backups, logs, and setup workflows.

The bot is designed for multi-guild deployment. Runtime data is stored per Discord server under:

```text
data/<guild_id>/guild_databases/
```

The bot uses SQLite as the source of truth for guild configuration, command access, timezone settings, feature state, and operational logs. Some human-readable JSON mirrors may exist, but SQLite is the real backend.

## 2. Critical Security Notes

Never commit or share `.env` publicly. The `.env` file contains the Discord bot token and production control settings. If the token is leaked, immediately regenerate it in the Discord Developer Portal and restart the bot with the new token.

Do not use global command sync casually. Global sync affects every server where the bot exists and can leave stale commands around. Normal production sync should use joined or database/registered guild sync only.

Do not enable debug logs in production unless you are actively diagnosing a problem. Debug mode may expose IDs, paths, and internal state in terminal logs.

## 3. Required Discord Bot Permissions

The bot needs these Discord permissions for normal operation:

| Permission | Why it is needed |
|---|---|
| View Channels | Read configured log/menu channels. |
| Send Messages | Send reports, menus, warnings, and setup responses. |
| Embed Links | Send formatted report/menu embeds. |
| Attach Files | Send output files if needed. |
| Read Message History | Backfill logs and scan historical messages. |
| Use Slash Commands | Required for all `/` commands. |
| Manage Messages | Delete/replace old menu/status messages where supported. |
| Manage Webhooks | Create/remove feature webhooks for delivery, moonshine, still-part, etc. |
| Manage Roles | Needed only for LOA roles and role-menu assignment/removal features. |
| Members Intent | Required for role checks, member resolution, payout attribution, and menu permission checks. |
| Message Content Intent | Required because many features parse log message text. |

## 4. Permission and Access Model

### 4.1 Bot Owner / Control Guild

`BOT_ADMIN_GUILD_ID` defines the only Discord server where `/bot-admin` should appear and be usable. This is the owner/control server.

`/bot-admin` is not meant for normal client servers. It controls setup visibility, feature group locks, server blocking, command visibility, and administrative diagnostics.

### 4.2 Global Setup Load Switch

`SETUP_COMMANDS_ENABLED=true` allows setup cogs to load. This does not automatically expose setup commands to every server. Per-server access is still controlled by `/bot-admin setup-commands-enable` and command group unlocks.

### 4.3 Command Groups

Commands are grouped and locked/unlocked per guild. If a group is locked, its commands are removed from that guild's slash-command tree during sync.

| Group | Default | Purpose / Roots |
|---|---:|---|
| `core` | Always enabled | `/bot-admin`; control server only. |
| `setup` | Locked | Feature setup commands such as `/setup-shop`, `/setup-cattle`, `/setup-loa`, `/setup-moonshine`, `/setup ranch-chores`, `/timezone`. |
| `config` | Locked | Shared server config and price-log commands: `/setup-server-config`, `/setup-price-log`. |
| `menus` | Enabled | User-facing menu/panel roots where present. |
| `camp` | Locked | `/camp`, `/today`, `/delivery`, `/dm`. |
| `ranch` | Enabled | `/ranch`, `/cattle`, `/cattle-ranch`, ranch chores, ranch cooldown. |
| `hideout` | Locked | `/still-part`, `/moonshine`, `/brewing`, `/brewing-shack`. |
| `shop` | Locked | `/shop`. |
| `loa` | Enabled | `/loa`, LOA management tools. |
| `clan` | Enabled | `/clan`. |
| `join` | Enabled | Join-related commands if present. |
| `debug` | Locked | Developer/debug commands. |
| `basics` | Locked | Fallback for unmapped commands; locked to prevent command leaks. |

### 4.4 Practical Permission Rules

Bot-owner/admin commands are controlled by `/bot-admin` and should only be available in the configured control guild.

Setup commands generally require Discord Administrator permission and require setup access to be enabled for that guild.

Operational commands depend on the command group being unlocked for the guild. Some features also enforce feature-specific role access, such as shop owner roles, LOA roles, ranch IDs, hideout IDs, and configured menu roles.

## 5. Recommended Initial Setup Order

Use this order for a new server. Skipping steps causes broken command visibility or missing config errors.

1. Invite the bot with the required permissions and intents.
2. Confirm the guild is registered by the bot.
3. In the control guild, run `/bot-admin server-list` and verify the server exists.
4. Enable setup for the target server:
   ```text
   /bot-admin setup-commands-enable guild_id:<server_id>
   ```
5. Unlock the needed command groups:
   ```text
   /bot-admin group-unlock guild_id:<server_id> group:config
   /bot-admin group-unlock guild_id:<server_id> group:setup
   /bot-admin group-unlock guild_id:<server_id> group:camp
   /bot-admin group-unlock guild_id:<server_id> group:ranch
   /bot-admin group-unlock guild_id:<server_id> group:hideout
   /bot-admin group-unlock guild_id:<server_id> group:shop
   ```
6. Sync commands using `!syncslash guild` or `!syncslash current` if commands do not appear.
7. Configure timezone:
   ```text
   /timezone set timezone_name:India
   ```
8. Configure shared entities:
   ```text
   /setup-server-config add-ranch
   /setup-server-config add-camp
   /setup-server-config add-hideout
   /setup-shop config
   ```
9. Upload/configure the price log:
   ```text
   /setup-price-log upload
   ```
10. Configure feature menus/webhooks for the systems you use.
11. Lock setup/config groups again when setup is complete if normal admins should not change configuration.

## 6. Prefix Maintenance Commands

These are owner-only prefix commands, not slash commands.

| Command | Purpose | Permission |
|---|---|---|
| `!botstatus` | Shows system health, loaded/failed extensions, sync status, timezone, and runtime. | Bot owner only. |
| `!testalert` | Sends a test owner alert through configured owner alert channels. | Bot owner only. |
| `!backupdb` | Runs a manual SQLite database backup. | Bot owner only; requires `DB_BACKUP_ENABLED=true`. |
| `!cleanlogs old 7` | Deletes old rotated log files based on retention. | Bot owner only. |
| `!cleanlogs all` | Deletes rotated logs and truncates active logs. | Bot owner only. |
| `!cleanbackups old 30` | Deletes old database backup folders. | Bot owner only. |
| `!cleanbackups all` | Deletes all backup folders. | Bot owner only. |
| `!maintenanceclean` | Runs normal log and backup cleanup. | Bot owner only. |
| `!guilds` | Lists guild registry entries. | Bot owner only. |
| `!cleancommands guild` | Clears guild slash-command cache. | Bot owner only; requires `COMMAND_CACHE_CLEAN_ENABLED=true`. |
| `!syncslash` / `!syncslash guild` | Syncs slash commands to configured guild targets. | Bot owner only; requires `SYNC_COMMANDS_ENABLED=true`. |
| `!syncslash current` | Syncs current server only. | Bot owner only. |
| `!syncslash global` / `!syncslash all` | Global sync operations. | Bot owner only; blocked unless `GLOBAL_SYNC_COMMANDS_ENABLED=true`. |

## 7. Slash Command Reference

### 7.1 `/bot-admin` — Bot Owner Control

Access: control guild only. Used by bot owner/admin staff.

| Command | Purpose |
|---|---|
| `/bot-admin command-list` | Show command visibility/status. |
| `/bot-admin group-list` | List command groups and lock states. |
| `/bot-admin group-lock` | Lock a command group for a guild. |
| `/bot-admin group-unlock` | Unlock a command group for a guild. |
| `/bot-admin log-webhook-set` | Configure admin log webhook. |
| `/bot-admin server-block` | Block a server from using non-core commands. |
| `/bot-admin server-list` | List known servers. |
| `/bot-admin server-remove` | Remove/deactivate server record. |
| `/bot-admin server-unblock` | Unblock a server. |
| `/bot-admin setup-commands-enable` | Enable setup commands for a guild. |
| `/bot-admin setup-commands-disable` | Disable setup commands for a guild. |
| `/bot-admin setup-info` | Show setup status/instructions. |

### 7.2 `/setup-server-config` — Shared Ranch/Camp/Hideout/Shop Config

Access: Administrator; requires setup/config access.

| Command | Purpose |
|---|---|
| `/setup-server-config add-camp` | Register a camp ID and camp log channel. |
| `/setup-server-config add-ranch` | Register a ranch ID and ranch log channel. |
| `/setup-server-config add-hideout` | Register a hideout ID and hideout channel. |
| `/setup-shop config` | Register a shop ID and shop channel. |
| `/setup-server-config list` | List configured shared entities. |
| `/setup-server-config remove` | Remove a shared entity config. |
| `/setup-server-config set-mode` | Set entity mode, usually `single` or `multi`. |

### 7.3 `/setup-price-log` — Price Log Configuration

Access: Administrator; requires setup/config access.

| Command | Purpose |
|---|---|
| `/setup-price-log status` | Show this server's price-log status. |
| `/setup-price-log upload` | Upload server-specific `log_price.txt`. |
| `/setup-price-log preview` | Preview saved price log. |
| `/setup-price-log delete` | Delete saved price log. |
| `/setup-price-log migrate-global` | Migrate legacy global price log if needed. |

### 7.4 `/timezone` — Server Timezone

Access: setup/admin configuration.

| Command | Purpose |
|---|---|
| `/timezone set` | Set server timezone by country/common name or IANA name. |
| `/timezone get` | Show current server timezone. |
| `/timezone now` | Show current server-local time. |
| `/timezone list` | List common supported timezone aliases. |
| `/timezone reload` | Reload timezone config from storage. |
| `/timezone reset` | Reset timezone configuration. |

### 7.5 `/setup-shop` — Shop Setup

Access: Administrator; requires setup enabled.

| Command | Purpose |
|---|---|
| `/setup-shop config` | Configure shop access and owner role/channel mapping. |
| `/setup-shop config-view` | View saved shop config. |
| `/setup-shop config-remove` | Remove shop config. |
| `/setup-shop menu` | Send the shop menu panel. |
| `/setup-shop menu-remove` | Remove saved shop menu panel. |

### 7.6 `/shop` — Shop Operations

Access: `shop` group must be unlocked. User must have a configured shop owner role unless admin/system path allows otherwise.

| Command | Purpose |
|---|---|
| `/shop check-low-stock` | Show low stock items. |
| `/shop stock` | Show current shop stock. |
| `/shop ledger` | Show latest shop ledger amount. |
| `/shop product-pricelist` | Show product selling/buying price list. |
| `/shop month-report` | Show monthly shop report. |
| `/shop sold-to-shop` | Show sold-to-shop report. |
| `/shop purchase-demand` | Show top sold-to-shop demand items. |
| `/shop sale-demand` | Show top customer sale demand items. |
| `/shop tax-paid` | Log/handle tax paid action. |
| `/shop set-tax-reminder` | Set tax reminder. |
| `/shop remove-tax-reminder` | Remove tax reminder. |
| `/shop config-view` | View shop config/access info. |

Shop menu buttons include Check Low Stock Items, Shop Stock, Check Ledger, Product Price List, Month Report, Sold to Shop, Purchase Demand, and Sale Demand.

### 7.7 `/camp` — Camp Operations

Access: `camp` group must be unlocked.

| Command | Purpose |
|---|---|
| `/camp inventory-deposit` | Report camp inventory deposits. |
| `/camp inventory-withdraw` | Report camp inventory withdrawals. |
| `/camp ledger` | Show camp ledger. |
| `/camp material` | Show camp material report. |
| `/camp material-set` | Configure/update material values. |
| `/camp supplies-mission` | Log or report supply missions. |
| `/camp delivery-report` | Show camp delivery report. |
| `/camp add-live-performance` | Add camp live performance board. |
| `/camp remove-live-performance` | Remove camp live performance board. |

### 7.8 `/ranch` — Ranch Operations

Access: `ranch` group must be unlocked.

| Command | Purpose |
|---|---|
| `/ranch inventory` | Show ranch inventory. |
| `/ranch milk-and-egg` | Show milk and egg report. |
| `/ranch cattle-purchase` | Show cattle purchase report. |
| `/ranch cattle-sale` | Show cattle sale report. |
| `/ranch cattle-sale-by-member` | Show cattle sale grouped by member. |
| `/ranch cash-ledger` | Show ranch cash ledger. |
| `/ranch tax` | Show ranch tax report. |
| `/ranch summary` | Show ranch summary. |
| `/ranch hiring-activity` | Show ranch hiring/activity report. |
| `/ranch live-performance-add` | Add ranch live performance board. |
| `/ranch live-performance-list` | List ranch live performance boards. |
| `/ranch live-performance-remove` | Remove ranch live performance board. |

### 7.9 `/setup ranch-chores` — Ranch Chores Setup

Access: Administrator/setup.

| Command | Purpose |
|---|---|
| `/setup ranch-chores menu` | Create/send ranch chores panel. |
| `/setup ranch-chores menu-remove` | Remove ranch chores panel. |
| `/setup ranch-chores list` | List ranch chores panels/data. |
| `/setup ranch-chores data-remove` | Remove saved ranch chores data. |

### 7.10 Ranch Chores Runtime

Access: `ranch` group and configured panel. The runtime view handles ranch chores button interactions. Environment controls define max value, reduction amount, loop interval, red threshold, and message rotation.

Important environment keys:

```text
RANCH_CHORES_MAX_VALUE
RANCH_CHORES_REDUCE_BY
RANCH_CHORES_INTERVAL_MINUTES
RANCH_CHORES_RED_BELOW
```

### 7.11 `/setup-ranch-cooldown` — Ranch Cooldown Setup

Access: Administrator/setup.

| Command | Purpose |
|---|---|
| `/setup-ranch-cooldown menu` | Create/send cooldown status menu. |
| `/setup-ranch-cooldown menu-list` | List cooldown menus. |
| `/setup-ranch-cooldown menu-remove` | Remove cooldown menu. |
| `/setup-ranch-cooldown menu-data-remove` | Remove saved cooldown menu data. |
| `/setup-ranch-cooldown menu-backfill` | Backfill cooldown state from logs. |

### 7.12 Ranch Cooldown Runtime

Access: configured cooldown menu and ranch log parsing.

The cooldown system tracks egg/milk, herding, and ranch chore cooldowns. It refreshes menus periodically and can backfill recent ranch logs on startup.

Important environment keys:

```text
RANCH_EGG_MILK_COOLDOWN_SECONDS
RANCH_HERDING_COOLDOWN_SECONDS
RANCH_CHORE_COOLDOWN_SECONDS
RANCH_COOLDOWN_BACKFILL_LIMIT
RANCH_COOLDOWN_MAX_MEMBER_ROWS
```

### 7.13 `/setup-cattle` — Cattle Setup

Access: Administrator/setup.

| Command | Purpose |
|---|---|
| `/setup-cattle menu` | Create cattle menu panel. |
| `/setup-cattle menu-remove` | Remove cattle menu panel. |
| `/setup-cattle menu-data-list` | List saved cattle menu data. |
| `/setup-cattle menu-data-remove` | Remove saved cattle menu data. |
| `/setup-cattle menu-setup-list` | List cattle setup records. |
| `/setup-cattle menu` | Create/add cattle webhook. |
| `/setup-cattle webhook-list-view` | View cattle webhook configs. |
| `/setup-cattle webhook-remove` | Remove cattle webhook config. |

### 7.14 `/cattle` — Cattle Herding

Access: `ranch` group must be unlocked.

| Command | Purpose |
|---|---|
| `/cattle herd` | Start herd for yourself. |
| `/cattle herd-member` | Start herd for another member. |
| `/cattle queue` | Queue cattle. |
| `/cattle queue-view` | View current cattle queue. |
| `/cattle available` | View available cattle/herd state. |
| `/cattle release` | Release/remove herd state. |
| `/cattle sold` | Mark cattle as sold. |
| `/cattle sold-removal` | Remove sold record. |
| `/cattle sold-report` | Show sold cattle report. |
| `/cattle status` | Show cattle status. |
| `/cattle reset-all` | Reset cattle state. |
| `/cattle lock-animal` | Lock an animal. |
| `/cattle unlock-animal` | Unlock an animal. |
| `/cattle blocklist-view` | View blocklist. |
| `/cattle blocklist-remove` | Remove blocklist entry. |
| `/cattle warning-report` | Show warning report. |

### 7.15 `/cattle-ranch` — Ranch-Scoped Cattle Commands

Access: `ranch` group must be unlocked.

| Command | Purpose |
|---|---|
| `/cattle-ranch available` | View available state for a ranch. |
| `/cattle-ranch queue-view` | View ranch queue. |
| `/cattle-ranch sold-report` | Show ranch sold report. |
| `/cattle-ranch status` | Show ranch cattle status. |
| `/cattle-ranch reset` | Reset ranch cattle state. |
| `/cattle-ranch blocklist-view` | View ranch blocklist. |
| `/cattle-ranch blocklist-remove` | Remove ranch blocklist entry. |
| `/cattle-ranch warning-report` | Show ranch warning report. |

### 7.16 `/delivery` — Delivery Mission Logger

Access: `camp` group must be unlocked.

| Command | Purpose |
|---|---|
| `/delivery add` | Add successful delivery mission. |
| `/delivery missed` | Add missed delivery mission. |
| `/delivery failed` | Add failed delivery mission. |
| `/delivery remove` | Remove delivery log. |
| `/delivery report` | Show delivery report. |
| `/delivery missed-report` | Show missed delivery report. |
| `/delivery failed-report` | Show failed delivery report. |
| `/delivery reset` | Reset delivery logs. |

### 7.17 `/setup-delivery` — Delivery Setup

Access: Administrator/setup.

| Command | Purpose |
|---|---|
| `/setup-delivery menu` | Create delivery menu. |
| `/setup-delivery menu-remove` | Remove delivery menu. |
| `/setup-delivery menu` | Add delivery webhook. |
| `/setup-delivery webhook-list-view` | View delivery webhooks. |
| `/setup-delivery webhook-remove` | Remove delivery webhook. |

### 7.18 `/today` — Today Activity

Access: `camp` group must be unlocked.

| Command | Purpose |
|---|---|
| `/today camp-activity` | Show configured camp activity. |
| `/today ranch-activity` | Show configured ranch activity. |
| `/today run` | Run activity report by configured source. |

### 7.19 `/setup-today-config` — Today Activity Setup

Access: Administrator/setup.

| Command | Purpose |
|---|---|
| `/setup-today-config add-camp` | Add camp activity source. |
| `/setup-today-config add-ranch` | Add ranch activity source. |
| `/setup-today-config list-sources` | List configured activity sources. |
| `/setup-today-config remove-camp` | Remove camp activity source. |
| `/setup-today-config remove-ranch` | Remove ranch activity source. |

### 7.20 `/still-part` — Still Part Mission Logger

Access: `hideout` group must be unlocked.

| Command | Purpose |
|---|---|
| `/still-part add` | Add still-part mission. |
| `/still-part add-missed` | Add missed still-part mission. |
| `/still-part remove` | Remove still-part log. |
| `/still-part report` | Show still-part report. |
| `/still-part transactions` | Show transaction report. |
| `/still-part reset-all` | Reset still-part logs. |
| `/still-part view-shack` | View shack/hideout data. |

### 7.21 `/setup-still-part` — Still Part Setup

Access: Administrator/setup.

| Command | Purpose |
|---|---|
| `/setup-still-part menu` | Create still-part menu. |
| `/setup-still-part menu-remove` | Remove still-part menu. |
| `/setup-still-part menu-data-list` | List menu data. |
| `/setup-still-part menu-hideout-config` | Configure menu hideout mapping. |
| `/setup-still-part menu-hideout-data-list` | List hideout menu mappings. |
| `/setup-still-part menu-hideout-list-remove` | Remove hideout menu mapping. |
| `/setup-still-part menu` | Add still-part webhook. |
| `/setup-still-part webhook-list-view` | View still-part webhooks. |
| `/setup-still-part webhook-remove` | Remove still-part webhook. |

### 7.22 `/moonshine` — Moonshine Logger

Access: `hideout` group must be unlocked.

| Command | Purpose |
|---|---|
| `/moonshine add` | Add moonshine log. |
| `/moonshine add-missed` | Add missed moonshine log. |
| `/moonshine delete` | Delete moonshine log. |
| `/moonshine report` | Show moonshine report. |
| `/moonshine transaction-report` | Show moonshine transaction report. |
| `/moonshine reset-all` | Reset moonshine logs. |

### 7.23 `/brewing` and `/brewing-shack` — Brewing Controls

Access: `hideout` group must be unlocked.

| Command | Purpose |
|---|---|
| `/brewing moonshine` | Start moonshine brew timer. |
| `/brewing moonshine-stop` | Stop moonshine brew timer. |
| `/brewing mash` | Start mash timer. |
| `/brewing mash-stop` | Stop mash timer. |
| `/brewing alcohol` | Start alcohol timer. |
| `/brewing alcohol-stop` | Stop alcohol timer. |
| `/brewing stop-all` | Stop all brewing timers. |
| `/brewing resets-all` | Reset all brewing state. |
| `/brewing-shack moonshine-stop` | Stop moonshine timer for a shack. |
| `/brewing-shack mash-stop` | Stop mash timer for a shack. |
| `/brewing-shack alcohol-stop` | Stop alcohol timer for a shack. |

### 7.24 `/setup-moonshine` and `/setup-moonshine-brewed`

Access: Administrator/setup.

| Command | Purpose |
|---|---|
| `/setup-moonshine menu` | Create moonshine menu. |
| `/setup-moonshine menu-remove` | Remove moonshine menu. |
| `/setup-moonshine menu-data-list` | List menu data. |
| `/setup-moonshine menu-data-remove` | Remove menu data. |
| `/setup-moonshine menu-setup-list` | List setup records. |
| `/setup-moonshine webhook-add` | Add moonshine webhook. |
| `/setup-moonshine webhook-list-view` | View moonshine webhooks. |
| `/setup-moonshine webhook-remove` | Remove moonshine webhook. |
| `/setup-moonshine-brewed menu` | Create brewed-status menu. |
| `/setup-moonshine-brewed menu-remove` | Remove brewed-status menu. |

### 7.25 `/loa` — Leave of Absence System

Access: `loa` group must be unlocked.

| Command | Purpose |
|---|---|
| `/loa active` | Show active LOA records. |
| `/loa pending` | Show pending LOA requests. |
| `/loa cancel` | Cancel LOA request. |
| `/loa return` | Mark return from LOA. |

LOA approval/deny actions are handled by persistent buttons in the LOA view/menu.

### 7.26 `/setup-loa` — LOA System Setup

Access: Administrator/setup.

| Command | Purpose |
|---|---|
| `/setup-loa menu-config` | Configure report channel, LOA role, bench role, and review ping role. |
| `/setup-loa menu` | Create LOA request menu. |
| `/setup-loa menu-remove` | Remove LOA request menu. |

### 7.27 `/setup-loa-mgt` — LOA Management Role Menu

Access: Administrator/setup.

| Command | Purpose |
|---|---|
| `/setup-loa-mgt menu` | Create LOA management role menu. |
| `/setup-loa-mgt menu-info` | Show LOA management menu info. |
| `/setup-loa-mgt menu-remove` | Remove LOA management role menu. |

### 7.28 `/clan` — Clan Reports and Payouts

Access: `clan` group must be unlocked.

| Command | Purpose |
|---|---|
| `/clan income-generated-calculations` | Calculate generated income. |
| `/clan member-pay-input` | Add member payment entry. |
| `/clan member-pay-remove` | Remove member payment entry. |
| `/clan member-payout-report` | Build member payout report. |
| `/clan payout-view` | View saved payout report. |
| `/clan performance` | Show clan performance report. |

### 7.29 `/dm` — DM Sender

Access: `camp` group must be unlocked. Intended for administrators or authorized operators.

The DM sender cog provides direct-message tools through the `/dm` root and modal workflows. Use it carefully; bulk or repeated DMs can annoy users and may hit Discord limits.

## 8. Environment Configuration Notes

Important `.env` keys:

| Key | Purpose |
|---|---|
| `DISCORD_BOT_TOKEN` | Discord bot token. Keep secret. |
| `BOT_ADMIN_GUILD_ID` | Owner/control server where `/bot-admin` is allowed. |
| `OWNER_ALERTS_ENABLED` | Enables critical owner alerts. |
| `OWNER_ALERT_CHANNEL_ID` | Channel for owner alerts. |
| `SETUP_COMMANDS_ENABLED` | Global setup cog load switch. |
| `SYNC_COMMANDS_ENABLED` | Allows manual slash sync commands. |
| `SYNC_ON_STARTUP` | Sync commands automatically on startup. |
| `COMMAND_SYNC_SCOPE` | Sync target scope such as `joined`, `database`, `guild`, `admin`, or `all`. |
| `GLOBAL_SYNC_COMMANDS_ENABLED` | Safety gate for global sync/cleanup. |
| `LOG_TO_FILE` | Enables rotating file logs. |
| `DB_BACKUP_ENABLED` | Enables automatic SQLite backups. |
| `SHOP_DEMAND_TOP_LIMIT` | Controls top item count for shop demand reports. |
| `LIVE_PERFORMANCE_COMMANDS_ENABLED` | Enables live performance features. |
| `BOT_DEBUG_LOGS` | Enables verbose debug logs. |

Avoid duplicate keys in `.env`. If the same key appears twice, the later value may override the earlier one depending on loading behavior and runtime environment. This creates misleading configuration.

## 9. Price Log Notes

The bot expects a per-server price log at:

```text
data/<guild_id>/guild_databases/log_price.txt
```

Use `/setup-price-log upload` to store it correctly.

The price file is used by ranch, camp, delivery, cattle, moonshine, and shop-style calculations. If reports show zero, wrong payouts, or missing prices, check the price log first.

## 10. Operational Notes

Use per-server shared config before feature setup. For example, do not configure Still Part before the hideout ID exists in `/setup-server-config`.

Do not leave setup/config groups unlocked permanently unless you trust server administrators to change bot configuration.

Use `!botstatus` after deployment. Failed cogs, bad sync state, missing guild registry entries, or missing timezone config will show there.

When a command does not appear, the usual causes are: command group locked, setup disabled, command sync not run, bot missing guild access, or stale Discord command cache.

## 11. FAQ

### Why does `/bot-admin` not show in my normal server?

Because `/bot-admin` is restricted to `BOT_ADMIN_GUILD_ID`. That is intentional. Normal servers should not see bot-owner controls.

### Why do setup commands not show?

Setup commands require all of these:

1. `SETUP_COMMANDS_ENABLED=true` in `.env`.
2. The setup cog must load successfully.
3. The guild must have setup enabled through `/bot-admin setup-commands-enable`.
4. The `setup` and/or `config` command group must be unlocked.
5. Slash commands must be synced after changes.

### Why do operational commands not show?

The command group is probably locked for that guild, or the bot has not synced commands after a group change. Use `/bot-admin group-list`, unlock the needed group, then run `!syncslash guild` or `!syncslash current`.

### Why is a report empty?

Usually one of these is true:

- The wrong channel was configured.
- The date range is wrong.
- The feature is reading a different guild ID or config ID.
- The price log is missing or incomplete.
- The bot lacks Read Message History or Message Content Intent.
- The expected log format changed.

### Why are setup commands visible but failing?

Visibility and permission are separate. The command may appear because the group is unlocked, but it can still fail if the user lacks Administrator permission, if setup is disabled for the guild, or if required shared config is missing.

### Why is Discord returning Missing Access during sync?

The guild registry may contain stale servers where the bot is no longer present. Use `!guilds` and the bot status output to identify stale registry entries. The app code tries to avoid syncing unreachable guilds, but stale IDs still cause confusion if manually configured.

### Should I use global sync?

No, not for normal use. Use guild sync. Global sync is only for maintenance and must be explicitly unlocked with `GLOBAL_SYNC_COMMANDS_ENABLED=true`.

### Why are there duplicate or stale commands in Discord?

Discord may still have old command cache entries. Use `!cleancommands guild` only when `COMMAND_CACHE_CLEAN_ENABLED=true`, then run a normal sync again.

### Why does shop access say I do not have a configured shop owner role?

The shop config maps owner roles to shop IDs/channels. Your Discord role must match the configured owner role for that shop. Admin permissions do not automatically mean the role mapping is correct.

### Why does Still Part say the hideout ID is not configured?

Still Part depends on shared hideout config. Run:

```text
/setup-server-config add-hideout hideout_id:<id> hideout_channel:#channel
```

Then rerun the Still Part setup.

### Why does cattle/ranch cooldown behave oddly after restart?

Backfill and recovery settings control startup reconstruction. Check:

```text
CATTLE_RUN_RECOVERY_ON_START
CATTLE_RANCH_LOG_BACKFILL_ENABLED
RANCH_COOLDOWN_BACKFILL_ON_STARTUP
RANCH_COOLDOWN_BACKFILL_LIMIT
```

### Why are logs too noisy?

Set:

```text
BOT_DEBUG_LOGS=false
LOG_LEVEL=info
TERMINAL_COLORS_ENABLED=true
```

Debug logs should only be enabled while diagnosing issues.

### Why are database backups not running?

Check:

```text
DB_BACKUP_ENABLED=true
DB_BACKUP_INTERVAL_HOURS=24
DB_BACKUP_DIR=backups
```

Then use `!backupdb` to test manual backup.

### What should I do after changing command files?

Restart the bot, check terminal output for failed cogs, then run a guild sync. Do not use global sync unless you intentionally want to publish commands globally.

## 12. Minimum Production Checklist

Before running in production:

- [ ] Bot token is private and not leaked.
- [ ] `BOT_ADMIN_GUILD_ID` is correct.
- [ ] `SETUP_COMMANDS_ENABLED=true` only if setup cogs are needed.
- [ ] `GLOBAL_SYNC_COMMANDS_ENABLED=false` unless doing maintenance.
- [ ] `BOT_DEBUG_LOGS=false`.
- [ ] `LOG_TO_FILE=true` if persistent logs are wanted.
- [ ] `DB_BACKUP_ENABLED=true` if automatic backups are wanted.
- [ ] Required Discord intents are enabled in the Developer Portal.
- [ ] Required bot permissions are granted in the invite/server role.
- [ ] Command groups are locked/unlocked intentionally.
- [ ] Each guild has timezone configured.
- [ ] Each feature has shared config before feature setup.
- [ ] Price log is uploaded per guild.
- [ ] `!botstatus` shows healthy startup state.

## 13. Maintenance Discipline

This bot has many cogs and multiple setup/runtime layers. Do not make random edits in production. Change one feature at a time, restart, check failed extensions, sync guild commands, test, then move to the next feature.

When something breaks, first check:

1. Terminal startup output.
2. `!botstatus`.
3. Whether the command group is unlocked.
4. Whether the setup/config command exists in that guild.
5. Whether the required shared config exists.
6. Whether the guild-specific SQLite database exists.
7. Whether the bot has Discord permissions and intents.

Guessing wastes time. The bot already has enough diagnostics; use them.
## Merged setup commands

The setup flow now uses one command per feature instead of separate menu/config and webhook commands:

```text
/setup-shop config shop_id: role: shop_channel: shop_name:
/setup-cattle menu ranch_id: ranch_role: channel:
/setup-delivery menu channel:
/setup-still-part menu shack_id: channel:
/setup-ranch-ledger setup-menu role: ranch_id: channel:
/setup-clan member-pay-payment-menu ranch_id: channel:
```

The old separate shop, cattle, delivery, Still Part, Ranch Ledger webhook-add, and Clan deduction-webhook-add commands are no longer registered.

For Clan, `/setup-clan member-pay-payment-menu-remove ranch_id:` now removes both the member payment menu and its deduction webhook configuration; there is no separate deduction webhook removal command.



## Business Ticket System (Merged)

The bot now includes the Business Support Ticket system from the separate test build.

### Commands
- `/setup-business-ticket config` — configure optional business roles, ticket category, and transcript channel. Access is controlled by the `business-ticket` command group.
- `/setup-business-ticket menu` — post the Business Support Tickets menu in the current channel.
- `/setup-business-ticket clear` — remove an individual business/category/transcript configuration.

Configured buttons supported: Farmer, Illegal Trader, Doctor, Mining, Lumber, Blacksmith, Gunsmith, Horse Trainer, Distillery, Moonshine, and General Crafts.

Each configured business button opens an Order Details modal, creates a private ticket in the configured category, pings the configured business role, and provides Claim/Close controls. Closed tickets can be transcribed to the configured transcript channel.

Business-ticket configuration is stored in `data/business_config.json` and is intentionally separate from the existing general ticket system database/service to avoid collisions.


## Business Ticket SQL persistence
Business Ticket runtime state is stored in SQLite per guild at `data/<guild_id>/guild_databases/business_tickets.sqlite3`. SQL is the source of truth for claim state, order-complete state, added members, opener, business type, and closed-ticket metadata. Discord channel topics remain a compatibility/display mirror for existing tickets.
