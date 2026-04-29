# Codex CRM — Claude Code plugin

Three skills for interacting with the [Codex CRM](https://codex.gocodeable.com) lead database from a Claude Code session.

| Skill                | Purpose                                                |
|----------------------|--------------------------------------------------------|
| `codex-add-lead`     | Create a new lead. Asks for confirmation before write. |
| `codex-update-lead`  | Update **any** field on one or more leads — status, owner, tags, name, email, company, job title, etc. Mandatory confirmation; bulk operations on >100 leads ask twice. |
| `codex-query-leads`  | Read-only filter / search / lookup.                    |

## Prerequisites

1. A Codex account. Ask an admin to add you under **Settings → Manage Values → Owners** in Codex.
2. Network access to `https://codex.gocodeable.com`.
3. [Claude Code](https://claude.com/claude-code) installed.

## Configuration

Set these in your shell (`~/.zshrc` / `~/.bashrc`):

```bash
export CODEX_API_URL="https://codex.gocodeable.com"
export CODEX_USER_EMAIL="your.email@gocodeable.com"   # YOUR Codex login email
```

The skills authenticate via the `X-User-Email` header, so use your own email — anything you create or change is attributed to you.

## Install

In Claude Code, run:

```
/plugin marketplace add gocodeable/codex-claude-plugin
/plugin install codex@codex-marketplace
```

Then `/reload-plugins` (or restart Claude Code) and check `/plugin → Installed`.

## Usage

Just ask in plain English. Examples:

- *"How many qualified leads do we have at fintech companies?"* — triggers `codex-query-leads`
- *"Add John Doe at Acme as a new lead from LinkedIn"* — triggers `codex-add-lead`
- *"Mark John Doe as qualified"* — triggers `codex-update-lead`
- *"Change Sara's title to VP of Engineering"* — triggers `codex-update-lead`
- *"Assign all Series A leads from Apollo to Mike"* — triggers `codex-update-lead` (bulk)

All write operations show a summary and wait for explicit confirmation.

## Updating

```
/plugin marketplace update codex-marketplace
/plugin update codex@codex-marketplace
```

## License

MIT — see [LICENSE](./LICENSE).
