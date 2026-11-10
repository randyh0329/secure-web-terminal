# secure-web-terminal — a browser terminal for Claude Code, behind Cloudflare Access

A free, open agent **skill** that walks Claude Code (or any skills-compatible agent) through
setting up a **self-hosted browser terminal for Claude Code**:

- `ttyd` + `tmux` running as `systemd --user` units, so sessions survive tab closes, disconnects and restarts
- published through a **Cloudflare Tunnel**, with **Cloudflare Access** as the only login
- loopback-only binds, verified at each step, plus a troubleshooting table for common failures
- optional: one tab per project behind a single hostname, and paste-a-screenshot-into-Claude-Code

The skill comes from building [webterm / cloud_cli](https://github.com/randyh0329/cloud_cli), a
multi-project web terminal for Claude Code.

## Install

**Claude Code (plugin marketplace):**

```
/plugin marketplace add randyh0329/secure-web-terminal
/plugin install secure-web-terminal@secure-web-terminal
```

**Claude Code (manual):**

```bash
git clone https://github.com/randyh0329/secure-web-terminal.git
mkdir -p ~/.claude/skills
cp -r secure-web-terminal/skills/secure-web-terminal ~/.claude/skills/
```

**Other agents:** `skills/secure-web-terminal/SKILL.md` is plain Markdown with YAML
frontmatter. Put it wherever your agent loads skills or instructions from.

## Use

Ask something like:

> Set up a web terminal so I can use Claude Code from my phone. My domain is example.com and only me@example.com should get in.

The skill makes the agent confirm the allowed identities first, create the Access policy
**before** publishing the hostname, and check that nothing listens on a public interface.

## Layout

```
skills/secure-web-terminal/
  SKILL.md                     single-session setup, verification, troubleshooting
  references/multi-project.md  one-hostname, tab-per-project design and its invariants
```

## License

MIT
