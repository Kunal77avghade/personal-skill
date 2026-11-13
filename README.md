# personal-skill

Personal [Claude Code](https://claude.com/claude-code) skills that teach Claude how I write code.

| Skill | What it covers |
|---|---|
| [`kunal-coding-style`](kunal-coding-style/SKILL.md) | How code looks: guard-first shape, compact helpers, reuse, comments, logging, per-language conventions (TypeScript/NestJS, Python, Go). |
| [`kunal-best-practices`](kunal-best-practices/SKILL.md) | How a change is made safely: planning, design for failure, error classes and bounded retries, async cleanup, security, compatibility, durable workflows, testing, PRs. |

The two are meant to be loaded together: the style skill wins on shape, the best-practices
skill wins on behaviour.

## Install

Clone straight into the Claude Code user skills folder:

```bash
git clone https://github.com/Kunal77avghade/personal-skill.git ~/.claude/skills
```

If `~/.claude/skills` already exists, clone elsewhere and copy or symlink the skill folders in.

## Maintaining

- Each skill is a folder with a `SKILL.md` (YAML front matter `name` + `description`, then the body).
- Skills stay generic: no employer, client or internal project names, paths or ticket ids.
  The pre-commit hook in `hooks/` enforces this against a local, untracked term list:

  ```bash
  git config core.hooksPath hooks
  # one term per line (case-insensitive regex); this file is never committed
  $EDITOR .git/info/forbidden-terms
  ```
