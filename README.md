# Claude Code Django Expert

Django 6.x expert toolkit for [Claude Code](https://code.claude.com). Makes Claude Code an expert Django developer with modern best practices from Two Scoops of Django, Twelve-Factor App, and Cookiecutter Django.

## Setup (2 steps)

### Step 1: Install the skill globally (once)

```bash
mkdir -p ~/.claude/skills/setup-django-expert

curl -sL https://raw.githubusercontent.com/ultr4nerd/claude-code-django-expert/main/.claude/skills/setup-django-expert/SKILL.md \
  -o ~/.claude/skills/setup-django-expert/SKILL.md
```

This installs a single file on your machine. It adds zero extra tokens to your sessions.

### Step 2: Run it in your Django project

Open Claude Code in your project and run:

```
/setup-django-expert
```

The skill handles everything:
- Clones this repo temporarily
- Copies rules, skills, agents, and hooks into your project's `.claude/` directory
- **If you already have a `CLAUDE.md`**, appends Django instructions at the end without deleting anything
- **If you already have `.claude/settings.json`** with hooks, merges them without duplicating
- Asks if you want the Two Scoops knowledge base (~132KB)
- Cleans up the temp clone

Repeat step 2 in every Django project where you need it.

## What gets installed

| Component | Count | Description |
|---|---|---|
| `CLAUDE.md` | 1 | Project instructions — Django stack, 12-factor, conventions |
| Rules | 8 | Auto-activate for models, views, serializers, tests, templates, settings, migrations, security |
| Skills | 8 | `/django-new-app`, `django-new-model`, `django-new-api`, `django-review`, `django-debug`, `django-migration-check`, `django-security-audit`, `django-performance` |
| Agents | 3 | `@django-reviewer`, `@django-tester`, `@django-debugger` |
| Hooks | 2 | Auto-ruff on `.py` save, destructive migration protection |
| Knowledge | 1 | Modernized Two Scoops of Django guide (~4,800 lines) — optional |

## Safe for existing projects

The skill **never overwrites** your existing configuration:

- **CLAUDE.md** — If it exists, Django instructions are **appended** at the end. If it already has a Django section, only that section is updated.
- **Rules** — Only adds Django rules (`models.md`, `views.md`, etc.). Your rules like `code-style.md` or `frontend.md` are untouched.
- **Skills/Agents** — Only adds `django-` prefixed ones. Your other skills and agents remain intact.
- **settings.json** — Merges Django hooks with existing ones. Your ESLint, Prettier, etc. hooks are preserved.

## Zero impact on non-Django projects

If you installed the global skill (step 1) but you're in a FastAPI, JS, or Go project:
- The `setup-django-expert` skill exists but **does not run automatically** — only when you invoke it
- No extra tokens are added to your session
- Django skills and agents don't load unless you explicitly invoke them

## Usage after installation

```
/django-new-app          → Scaffold a new Django app
/django-new-model        → Create a new model with best practices
/django-new-api          → Create a DRF API endpoint
/django-review           → Code review
/django-debug            → Debug an issue
/django-migration-check  → Verify migrations before commit
/django-security-audit   → Security audit
/django-performance      → Performance optimization

@django-reviewer         → Code review agent
@django-tester           → Test generation agent
@django-debugger         → Debugging agent
```

## Django version

Target: **Django 6.0.3** (March 2026) with:
- CSP middleware (built-in)
- Template Partials (`{% partialdef %}`)
- Background Tasks framework
- `GeneratedField` and `db_default`
- Composite Primary Keys
- Async ORM (mature)
- `LoginRequiredMiddleware`

## Updating

To update the toolkit in a project, run `/setup-django-expert` again. It will replace Django rules, skills, and agents with the latest version from the repo, and update the Django section of your CLAUDE.md.

## License

MIT
