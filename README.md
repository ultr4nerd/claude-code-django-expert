# Claude Code Django Expert

Django 6.x expert toolkit for [Claude Code](https://code.claude.com). Makes Claude Code an expert Django developer with modern best practices from Two Scoops of Django, Twelve-Factor App, and Cookiecutter Django.

## What's Included

| Component | Count | Description |
|---|---|---|
| `CLAUDE.md` | 1 | Project instructions — Django stack, 12-factor, conventions |
| Rules | 8 | Auto-activate for models, views, serializers, tests, templates, settings, migrations, security |
| Skills | 8 | `/skill:django-new-app`, `django-new-model`, `django-new-api`, `django-review`, `django-debug`, `django-migration-check`, `django-security-audit`, `django-performance` |
| Agents | 3 | `@django-reviewer`, `@django-tester`, `@django-debugger` |
| Hooks | 2 | Auto-ruff on save, migration safety checks |
| Knowledge | 1 | Modernized Two Scoops of Django guide (~4,800 lines) |

## Quick Setup

Use the **setup-django-expert** skill in Claude Code:

```
/skill:setup-django-expert
```

Or manually copy files to your project:

```bash
# Clone this repo
git clone https://github.com/ultr4nerd/claude-code-django-expert.git /tmp/django-expert

# Copy to your project
cp /tmp/django-expert/CLAUDE.md your-project/
cp -r /tmp/django-expert/.claude your-project/
cp -r /tmp/django-expert/knowledge your-project/

# Clean up
rm -rf /tmp/django-expert
```

## Where Files Go

| Component | Per-project | Global (all projects) | Auto-loads? |
|---|---|---|---|
| CLAUDE.md | `./CLAUDE.md` | `~/.claude/CLAUDE.md` | Yes, always |
| Rules | `.claude/rules/*.md` | `~/.claude/rules/*.md` | Yes (path-filtered ones only on matching files) |
| Skills | `.claude/skills/*/SKILL.md` | `~/.claude/skills/*/SKILL.md` | No — only on `/skill:` |
| Agents | `.claude/agents/*.md` | `~/.claude/agents/*.md` | No — only on `@agent` |
| Hooks | `.claude/settings.json` | `~/.claude/settings.json` | Yes, always |

> **Priority**: Project > Global. Project-level files override global ones.

## Django Version

Targets **Django 6.0.3** (March 2026) with:
- CSP middleware (built-in)
- Template Partials (`{% partialdef %}`)
- Background Tasks framework
- `GeneratedField` and `db_default`
- Composite Primary Keys
- Async ORM (mature)
- `LoginRequiredMiddleware`

## License

MIT
