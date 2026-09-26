# ZCode agent session learning — 2026-09-26: Enabled the reserved Discord bot provider in ZCode desktop (asar patch)

## Discovery
ZCode's Bots subsystem (bot-config.v3.json, ~/.zcode/v2/) ships with provider adapters: telegram (long-polling), webhook (generic HTTP), feishu/lark (Gateway WebSocket), weixin — and RESERVED SLOTS `discord:null, wecom:null` in the adapter registry `F={...}`. The renderer picker already lists Discord with an icon, flagged `implemented:!1`. So "enabling Discord" = building the missing adapter + ingress, not fighting the UI.

## Reversed architecture (host bundle, out/host/index.js)
- Provider interface: `{test, resolveName, syncCommands, send, sendTyping, acknowledgeCallback?, downloadAttachment, parseCallback}` — factories receive `{loadCredential}`.
- Ingress contract: any channel calls `processProviderCallback(provider, rawPayload)` → `Yd(provider, payload)` → `adapter.parseCallback(rawPayload)` → normalized `{botId, text, actor:{provider, botId, providerUserId, chatType, chatId, displayName, providerMessageId}}`. Replies flow OUT via `adapter.send(config, {providerUserId, text})` — set providerUserId = reply target.
- Telegram polls getUpdates; feishu runs per-bot Gateway WS loops with lock+reconcile (`createFeishuChannelRuntime`). A Discord adapter can self-manage its Gateway WS (identify op2, heartbeat op1/op10, intents 512|1024|32768=34304, DMs always ingested, guild messages only when bot mentioned).
- Wiring site: `X=uH({...}),we=pH({...}),ue=fH({summarizeCallbackPayload:SH,processProviderCallback:Yd})` — the closure holds `e.credentialService`, `n.readConfig()`, `re` (statusSink), `Yd`. Injected a discord reconcile tick (8s) right after `ue=fH({...});`.

## Patch method (repeatable)
1. Anchor on unique minified strings; assert count==1 before every replace.
2. Inject function definitions at MODULE SCOPE before the closure (`function up(e){`), NOT before mid-expression anchors like `let M,F={...}` (that produced `,function...` → SyntaxError; caught by node --check on a .mjs copy).
3. When replacing an i18n KEY, keep the anchor in the replacement — consuming a key leaves its orphaned closing quote + value (`"newkey":"v""` `:` backtick-value). Repair regex: `("k\.discord":"[^"]*")":(?=\`)` → `\1,"k.webhook":`.
4. node --check each patched chunk (copy to .mjs first) BEFORE repack.
5. Repack: `npx @electron/asar pack app app-new.asar --unpack "{**/prebuilds/darwin-arm64/**,**/sshcrypto.node}"` (brace glob — CLI keeps only the LAST --unpack flag; needs `**/` prefixes), verify: unpacked list == 3 natives, file-list parity vs original, 12/12 sha256 spot-checks via header offsets.
6. Helpers saved at ~/.zcode/scripts/: asar_unpacked_list.py, asar_spotcheck.py, patch_discord.py.

## ENVIRONMENT GOTCHA (important)
Files written by the Write tool into /tmp may be INVISIBLE to later Bash calls (and vanish) — /tmp diverges between the tools. Keep cross-tool scripts under ~/.zcode/scripts/ and extract trees under /tmp only via Bash.

## Discord-side requirements (user)
Bot token from discord.com/developers → Bot → Reset Token; enable "Message Content Intent" (privileged); invite bot (scopes=bot; Send Messages + Read Message History). DMs always trigger; in servers the bot answers @mentions.

## State
Patched asar INSTALLED (/Applications/ZCode.app, backup app.asar.bak-prediscord). Loads on next app restart. Discord bot creation in UI: Bots → new → Discord → paste token (create-dialog credential field is generic).
