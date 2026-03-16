# NanoClaw

Personal Claude assistant. See [README.md](README.md) for philosophy and setup. See [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) for architecture decisions.

## Quick Context

Single Node.js process with skill-based channel system. Channels (WhatsApp, Telegram, Slack, Discord, Gmail) are skills that self-register at startup. Messages route to Claude Agent SDK running in containers (Linux VMs). Each group has isolated filesystem and memory.

## Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/registry.ts` | Channel registry (self-registration at startup) |
| `src/ipc.ts` | IPC watcher and task processing |
| `src/router.ts` | Message formatting and outbound routing |
| `src/config.ts` | Trigger pattern, paths, intervals |
| `src/container-runner.ts` | Spawns agent containers with mounts |
| `src/task-scheduler.ts` | Runs scheduled tasks |
| `src/db.ts` | SQLite operations |
| `groups/{name}/CLAUDE.md` | Per-group memory (isolated) |
| `container/skills/agent-browser.md` | Browser automation tool (available to all agents via Bash) |

## Skills

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time installation, authentication, service configuration |
| `/customize` | Adding channels, integrations, changing behavior |
| `/debug` | Container issues, logs, troubleshooting |
| `/update-nanoclaw` | Bring upstream NanoClaw updates into a customized install |
| `/qodo-pr-resolver` | Fetch and fix Qodo PR review issues interactively or in batch |
| `/get-qodo-rules` | Load org- and repo-level coding rules from Qodo before code tasks |

## Working Rules

### Code Review Before Implementation

Before implementing ANY code change (skill, feature, bug fix, or refactor):

1. Read all files that will be modified
2. Present the plan with exact diffs or a clear description of what will change
3. Wait for explicit user approval before writing any code
4. Only then proceed with implementation

This applies to every change, no matter how small.

### Code Standards

Always write clean, readable, and maintainable code:

**Naming**
- Use descriptive names — functions and variables must reveal intent (`processIncomingMessage`, not `handleMsg`)
- Boolean variables/functions use is/has/can prefix (`isConnected`, `hasPermission`)

**Functions**
- Each function does one thing only — if it needs a comment to explain what it does, split it
- Keep functions short (ideally under 20 lines)
- Prefer pure functions; avoid side effects unless necessary

**Structure**
- No magic numbers or strings — extract to named constants in `src/config.ts`
- No deep nesting — use early returns to reduce indentation
- No dead code — remove unused variables, imports, and functions

**TypeScript**
- Always type function parameters and return values explicitly
- Prefer `interface` over `type` for object shapes
- Never use `any` — use `unknown` and narrow the type, or define a proper interface

**Error handling**
- Catch errors at the boundary, not inside every function
- Log errors with structured context (`logger.error({ err, jid }, 'message')`)
- Never swallow errors silently

**Tests**
- New logic must have unit tests (Vitest)
- Tests live beside the source file (`foo.ts` → `foo.test.ts`)

### Update CLAUDE.md and Memory After Every Implementation

After any skill is applied or feature is implemented, always:

1. Update `CLAUDE.md` if the change affects architecture, key files, available skills, or workflow
2. Update the relevant memory files in `/home/milka/.claude/projects/-home-milka-nanoclaw/memory/` if the change affects project context, user preferences, or reference information
3. Update `MEMORY.md` index if new memory files were added

Do not consider a task complete until documentation and memory are current.

## Development

Run commands directly—don't tell the user to run them.

```bash
npm run dev          # Run with hot reload
npm run build        # Compile TypeScript
./container/build.sh # Rebuild agent container
```

Service management:
```bash
# macOS (launchd)
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # restart

# Linux (systemd)
systemctl --user start nanoclaw
systemctl --user stop nanoclaw
systemctl --user restart nanoclaw
```

## Troubleshooting

**WhatsApp not connecting after upgrade:** WhatsApp is now a separate channel fork, not bundled in core. Run `/add-whatsapp` (or `git remote add whatsapp https://github.com/qwibitai/nanoclaw-whatsapp.git && git fetch whatsapp main && (git merge whatsapp/main || { git checkout --theirs package-lock.json && git add package-lock.json && git merge --continue; }) && npm run build`) to install it. Existing auth credentials and groups are preserved.

## Container Build Cache

The container buildkit caches the build context aggressively. `--no-cache` alone does NOT invalidate COPY steps — the builder's volume retains stale files. To force a truly clean rebuild, prune the builder then re-run `./container/build.sh`.
