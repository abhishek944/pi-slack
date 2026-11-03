> **Moved to Astack**
> This repository is archived and read-only. Development is maintained in the [Astack monorepo](https://github.com/abhishek944/astack); see [services/pi-slack](https://github.com/abhishek944/astack/tree/main/services/pi-slack).

# pi-slack

A small background **Slack to Pi gateway service**. It runs as a normal Node process managed by
launchd (macOS) or systemd `--user` (Linux), keeps one Slack Socket Mode connection open, and maps one
Slack thread to one persistent native Pi session. Authorized teammates mention the bot in an enabled
channel or simply message it in a 1:1 DM; later messages in that thread continue the same session with
the full thread as context; other threads get independent sessions. Assistant output, including later
background turns, is posted back to the bound thread.

This package is no longer a Pi extension. It has no `pi` manifest, no extension entry and no slash
commands. Do not add it to Pi's `packages` setting or run `pi install`; remove any older checkout that
was installed there. It uses the installed Pi SDK as a normal dependency and the configured project's
normal Pi settings, packages, skills and prompts. It does not enable or disable First Mate, create
workers, use Herdr, run tasks, choose models, or change settings.

## Requirements

- Node.js 22+.
- The Pi SDK, which `npm ci` installs as a normal local dependency
  (`@earendil-works/pi-coding-agent` 1.4.0), so the service never depends on a host-provided peer.
- A Slack app with **Socket Mode** enabled, a bot token (`xoxb-…`) and an app-level token (`xapp-…`)
  with the `connections:write` scope. The bot needs `app_mentions:read`, `chat:write`,
  `channels:history` and `groups:history` so it can read thread history. For Slack DMs it also needs
  `im:history` and `mpim:history`. The optional `reactions:write` scope lets an accepted message get an
  immediate 👀 reaction; without it the service posts one short receipt message with `chat:write`
  instead. Subscribe the app to the `app_mention`, `message.im` and `message.mpim` bot events, install
  the app in the workspace, and invite the bot into the channels it should serve (all it can access
  when `enabledChannels` is empty). Slack never delivers `app_mention` for DMs or group DMs, so the
  service reads those from native `message` events: every authorized 1:1 DM message is a request, while
  a group DM still needs an explicit `<@bot-user-id>` mention. It does not use
  `message.channels`/`message.groups` events to trigger; untagged channel replies become context on a
  later mention. The bot can see only its own 1:1 DMs and the group DMs it belongs to, never other
  participants' private conversations. App creation is outside this package.

## Install and build

From source:

```bash
git clone https://github.com/abhishek944/pi-slack.git
cd pi-slack
npm ci --ignore-scripts
npm run build
```

`npm ci` fetches the pinned Pi SDK and Bolt into this package's `node_modules`; no lifecycle scripts
are needed. The compiled entry is `dist/service.js`. `dist/` and `node_modules/` stay untracked; rebuild
after cloning or updating. `package.json` is still `private: true`, so an accidental `npm publish` is
refused and there is no published package to install.

Normal Pi setup is separate. Configure Pi on this host as usual before starting the gateway:
authenticate, choose models, and install any project settings, packages, skills or prompts you want.
Thread sessions load those normal Pi resources for `projectCwd`; this repository does not bundle or
replace your Pi auth, models or settings.

## Configuration

Create `~/.config/pi-slack/config.json` (override the directory with `PI_SLACK_STATE_DIR` and the
file with `PI_SLACK_CONFIG`). `deploy/config.example.json` is a starting point. This is the gateway's
own service configuration, separate from Pi's agent directory. Thread sessions still load the host's
normal Pi auth, saved model settings, packages, extensions, skills and prompts for `projectCwd`; the
gateway does not disable or replace them.

```json
{
  "enabledChannels": [],
  "allowedUsers": ["U0123456789"],
  "projectCwd": "/absolute/path/to/project"
}
```

- `allowedUsers` is required and non-empty; replace the example with real Slack user IDs from the
  target workspace. Only those users are routed.
- `enabledChannels` is optional: `[]` (the default) answers in any channel the bot can access. List
  channel IDs to restrict it. Invalid values fail startup.
- `projectCwd` must be an absolute path chosen by the installer. It is the only working directory for
  new thread sessions; Slack text never selects a path. Existing bindings resume with the session's
  own recorded cwd.

Create both private files with restrictive permissions, then edit them locally. `cp -n` avoids
overwriting a live config if this step is repeated:

```bash
mkdir -p ~/.config/pi-slack
cp -n deploy/slack.env.example ~/.config/pi-slack/slack.env
cp -n deploy/config.example.json ~/.config/pi-slack/config.json
chmod 700 ~/.config/pi-slack
chmod 600 ~/.config/pi-slack/slack.env ~/.config/pi-slack/config.json
```

Slack tokens come from the private environment file `~/.config/pi-slack/slack.env`. The service
validates both tokens at startup, then deletes them from its own process environment before any Pi
session exists, so thread sessions and their shell tools cannot inherit them. Never put real tokens in
the plist, unit file, command line or this repository, and keep the env file mode 0600.

## Run manually

```bash
cd pi-slack
node --env-file="$HOME/.config/pi-slack/slack.env" dist/service.js
```

Node 22 reads the private env file, so token values never appear on the command line or in shell
history. `npm start` runs `node dist/service.js` and expects the tokens to already be in the
environment; the `--env-file` form above is safer for a manual run.

The service reads `~/.config/pi-slack/config.json`, connects once, and writes one JSON record per
line to stderr. `Ctrl-C` (SIGINT) and SIGTERM both stop it gracefully.

## Run with launchd (macOS, per-user LaunchAgent)

Files:

- `deploy/launchd/local.pi.slack-gateway.plist` — LaunchAgent template that runs an absolute Node
  binary directly with `--env-file=__HOME__/.config/pi-slack/slack.env` and
  `__CHECKOUT__/dist/service.js`, so secrets stay out of the plist and argv. `WorkingDirectory` is
  `__HOME__`, outside the checkout, which removes an unnecessary dependency on the checkout
  directory at startup. macOS may still require access to protected code or project paths.
- `deploy/launchd/run-service.sh` — optional wrapper for a foreground manual run; the LaunchAgent
  does not use it.

```bash
mkdir -p ~/Library/LaunchAgents
cp deploy/launchd/local.pi.slack-gateway.plist ~/Library/LaunchAgents/
# Edit the copy and replace all three placeholders with absolute paths:
#   __NODE_BIN__  run `command -v node` (Node 22)
#   __CHECKOUT__  the absolute path to the cloned repository
#   __HOME__      your absolute home directory
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.pi.slack-gateway.plist
launchctl print gui/$(id -u)/local.pi.slack-gateway | head -20
```

Start/restart/stop/status:

```bash
launchctl kickstart -k gui/$(id -u)/local.pi.slack-gateway   # restart
launchctl bootout gui/$(id -u)/local.pi.slack-gateway        # stop and unload
launchctl print gui/$(id -u)/local.pi.slack-gateway          # status
tail -f ~/Library/Logs/pi-slack.err.log ~/Library/Logs/pi-slack.out.log
```

The agent runs as the logged-in user, not root. If the checkout, Node binary or token file location
change, update the absolute paths in the plist before loading it.

## Run with systemd (Linux, user unit)

The unit assumes the repository is cloned at `~/pi-slack` (that is, `%h/pi-slack`). If your checkout
is elsewhere, edit `WorkingDirectory` and `ExecStart` in the copy before enabling it.

```bash
mkdir -p ~/.config/systemd/user
cp deploy/systemd/pi-slack.service ~/.config/systemd/user/pi-slack.service
# Edit the copy: adjust the checkout path and replace /usr/bin/node with the
# absolute Node 22 binary if Node is installed elsewhere (for example NVM).
systemctl --user daemon-reload
systemctl --user enable --now pi-slack.service
```

Start/restart/stop/status:

```bash
systemctl --user status pi-slack.service
systemctl --user restart pi-slack.service
systemctl --user stop pi-slack.service
systemctl --user disable pi-slack.service
journalctl --user -u pi-slack.service -f
```

The unit uses `EnvironmentFile=%h/.config/pi-slack/slack.env` and absolute paths (`%h` expands to the
user's home). `/usr/bin/node` is the template default; replace it with the real Node path if Node is
installed elsewhere. Keep the env file outside the repository and mode 0600. To keep the gateway
running without a login session, run `loginctl enable-linger "$USER"`; on some hosts this requires an
administrator policy. These Linux files are provided for adaptation and were not runtime-tested from
the macOS development host.

## Moving the gateway to another machine

- Run **one** active gateway listener per Slack app. If two machines connect with the same app tokens,
  Slack can split events between them and replies become unreliable. Stop the old service only when
  you are ready to cut over; do not stop a healthy local service merely to test a clone.
- A new host starts with fresh thread bindings and fresh native Pi sessions. Gateway state and Pi
  session histories are private and machine-specific. Do not copy `state.json`, Pi session files,
  logs or private config into this repository or blindly to another host. If you deliberately migrate
  them, do it separately with backups and reconciled absolute paths.
- Recreate the Slack tokens and gateway config privately on the new host. Never copy secrets into the
  repository or a service template.

## Lifecycle

- One Socket Mode connection is opened at startup; nothing connects per thread.
- SIGTERM or SIGINT stops intake, aborts active thread runs, clears the visible working status while
  Slack is still reachable, disposes every thread runtime (which emits `session_shutdown` so
  loops/monitors clean up), flushes bindings and then closes the Slack transport.
- launchd `KeepAlive` and systemd `Restart=on-failure` own crash restart. The service implements no
  scheduler, supervisor, installer or database.
- Persisted bindings and sessions resume lazily on the next message after a restart. Active runs,
  monitors and queued work do not automatically survive process death and are not replayed.
- Startup problems (missing or invalid config, missing tokens, unreadable state, Slack auth/start
  failure) produce one sanitized stderr record and a non-zero exit before the gateway serves.

## Behavior

- **Binding**: `(team, channel, root thread ts)` maps to one `SessionManager`/`AgentSession`. The binding
  stores the session file, session ID, cwd and accepted message identities so a restart resumes the
  same native history.
- **Authorization**: only user IDs in `allowedUsers` are routed. If `enabledChannels` is a non-empty
  list, messages must also come from those channels; an empty list means any channel the bot can
  access. Bot messages, edits/deletes and other subtypes are ignored. Channels invoke Pi through
  `app_mention` and require a real `<@bot-user-id>` token. Slack never sends `app_mention` for DMs, so
  native `message` events are used there: any nonempty message in a 1:1 DM is a request, while a group
  DM still requires the mention. Untagged channel replies and group-DM messages are read only as
  thread context by the next mention. The bot sees only its own 1:1 DMs and the group DMs it belongs
  to, never other participants' private conversations.
- **Dedupe**: each binding persists the exact Slack message timestamps already accepted. The duplicate
  event Slack sends for the same mention/message, and event retries, are delivered once. An unseen
  older message is **not** treated as a duplicate. There is no retention cap; the identity list grows
  with delivered messages.
- **Thread context**: every accepted message is queued in the thread's lane before any network work, so
  a slow history fetch cannot reorder requests. Inside that lane's turn the service reads the thread
  parent and all accessible replies through the message's own timestamp with `conversations.replies`,
  cursor pagination, `latest` plus `inclusive`, and excludes newer messages and the trigger itself. The
  snapshot is refreshed on every accepted message, including messages that reuse an existing Pi
  session. Each message keeps its Slack user/bot ID, supplied bot name when Slack provides one,
  timestamp and text in chronological order; names are never looked up. Earlier messages are quoted as
  discussion context that is explicitly not a new request. The current requester ID, question and an
  explicit `<@requester-id>` reply target follow separately, so the trigger text appears once.
- **History limits**: Slack returns only messages the app can access; deleted, retention-removed or
  inaccessible messages are omitted, messages without text or file names are skipped, and attachment
  or file contents are not downloaded. The prompt states these limits. There is no application history
  cap and no silent truncation; Pi's normal context management owns model-context limits. One native Pi
  session stays bound per workspace/channel/root thread.
- **Ordering and attribution**: each thread has one FIFO lane and one in-flight prompt. Different
  threads are independent.
- **Acknowledgement**: after authorization and exact-timestamp dedupe, the accepted message gets a
  👀 reaction immediately, before any history read or model work, so a message queued behind a slow
  run still shows a receipt. If Slack refuses the reaction (for example a missing `reactions:write`
  scope) one fixed "Received, Pi will reply" message is posted instead. While the run works, the
  service shows the assistant-thread status "Pi is working…" using the installed
  `assistant.threads.setStatus` method and the bot's normal `chat:write` access. If that method is
  refused in a conversation, the service still shows a working state with `chat:write`: it updates the
  existing receipt message and restores it afterwards, or, when no receipt message exists, posts one
  temporary working notice and removes it when the run stops. Visible status changes are serialized per
  thread, so an older clear cannot wipe a newer run's indicator. The status is refreshed before Slack's
  documented 2-minute expiry, re-asserted if a posted reply auto-clears it while the run continues, and
  cleared whenever no run is active: on settle, prompt rejection, a handled command that emits no agent
  events, open/history failures, and service stop. These calls are cosmetic: failures are logged in
  sanitized form and never drop the underlying request.
- **Output**: assistant text from each run is posted to the thread on `agent_end` (unless the run will
  retry) and any remainder on `agent_settled`. The session subscription lives for the thread runtime, so
  later extension-triggered turns are forwarded too. Tool output, prompts, logs and secrets are not
  forwarded.
- **Missing session file**: if a bound session file no longer exists, delivery pauses with a visible
  notice and the binding is kept. No replacement session is created; queued messages are delivered if
  the file is restored. A binding whose session file never materialized also blocks; resetting that
  binding is an explicit user action, not an automatic replacement.
- **History failures**: if the Slack history read fails (missing scope, lost channel access, deleted
  thread, rate limit), the service posts one concise sanitized notice, does not prompt Pi with partial
  context, keeps the message deduped so it is not replayed, and continues with the next queued
  message.
- **Failures**: a prompt rejected before acceptance and a terminal assistant/provider error or abort
  produce a concise, sanitized Slack notice. Raw diagnostics are never posted.
- **Dialogs**: a thread session has no terminal. Pi's documented headless default declines
  `select`/`confirm`/`input`/`editor`/custom UI requests (safe default), reports `ctx.hasUI` as false,
  and grants nothing. There is no Slack approval relay and no dialog notification to Slack.
- **Slash commands**: Slack messages are attributed text, not Pi commands. A Slack participant cannot
  run `/firstmate on` or other host commands through the gateway.
- **Settings and resources**: each thread gets its own `SettingsManager`, resource loader and event bus
  for the configured project. Normal saved settings, packages, extensions, skills and prompts load.
  The CLI built-in codemode, tool-search and MCP factories are added explicitly because a bare SDK
  session omits them. First Mate is respected as configured and never forced.
- **Project trust**: the project's saved trust decision is used; without one, `defaultProjectTrust`
  applies (`always` trusts, `ask`/`never` fail closed because a thread session cannot prompt). The
  `project_trust` extension event is not run for thread sessions.

## Headless limitations

- Thread sessions run in this service process and share its environment and real identity. There is no
  TUI, no interactive dialog, no terminal-only rendering and no approval relay.
- Worker/delegation backends that require a real interactive/Core host may not operate from a plain
  Node service. That is a backend-specific runtime limitation; this package does not add a Core
  adapter, fake terminal identity or worker scheduler, and does not change model or delegation policy.
- Extensions that only work with terminal rendering will see no UI (`ctx.hasUI === false`).
- The service uses Node's normal process environment and the configured project's Pi resources; it is
  not an OS security sandbox between threads.

## Security

- Run the service as the intended user, never root.
- Keep `~/.config/pi-slack/slack.env` and `config.json` private (`chmod 600`) and out of version
  control. Tokens are removed from the process environment after startup but remain in memory for the
  live connection.
- Bindings and accepted message identities are the only durable state, written atomically with mode
  0600 under the state directory. A corrupt state file is fatal rather than silently reset.
- Logs are JSON on stderr, and gateway state stores only thread bindings and accepted message
  identities; message text and credentials are omitted. Thread context is sent to the configured model
  as part of the normal Pi prompt and may be persisted in the native Pi session history. Protect Pi's
  agent/session files separately; they are outside this repository and are not part of gateway state.
  Sanitization redacts token-shaped strings from startup errors.
- No Slack, API, model or network calls are made by the static checks in this repository.

## Development

```bash
npm run typecheck
npm run build
```

No unit tests, mocks or prompt-based verification harness are included. The compiled service entry is
`dist/service.js`; deployment templates run it with an absolute Node path.
