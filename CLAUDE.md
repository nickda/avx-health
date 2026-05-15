# avx-health

Public skill repo. Single artifact: `SKILL.md`. No build step, no deps.

## Dev workflow

```bash
# Test locally: symlink into skills dir
ln -sf "$(pwd)/SKILL.md" ~/.claude/skills/avx-health/SKILL.md
```

## Gotchas

- Public repo (github.com/nickda/avx-health). No internal info in SKILL.md.
- Skill version in frontmatter (`version: x.y.z`) must be bumped on every substantive change.
- `depends-on: []`, `feeds-into: [avx-tshoot]` — keep these accurate.
- Demo config endpoint: `POST https://platform.mcp.aviatrix.com/auth/demo-config` (rate-limited 10 req/hr per IP).
