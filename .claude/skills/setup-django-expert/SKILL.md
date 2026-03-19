# Setup Django Expert

Configures the current project with Django 6.x expert tools from the [claude-code-django-expert](https://github.com/ultr4nerd/claude-code-django-expert) repository.

## What this skill does

1. Downloads rules, skills, agents, hooks, and knowledge from GitHub
2. Installs them into the current project's `.claude/` directory
3. Appends Django instructions to the existing `CLAUDE.md` (or creates one)
4. Never overwrites existing non-Django configuration

## Instructions

When the user invokes this skill, follow these steps exactly:

### Step 1: Check current state

```bash
echo "=== Current project state ==="
[ -f CLAUDE.md ] && echo "CLAUDE.md: EXISTS ($(wc -l < CLAUDE.md) lines)" || echo "CLAUDE.md: NOT FOUND"
[ -d .claude ] && echo ".claude/: EXISTS" || echo ".claude/: NOT FOUND"
[ -d .claude/rules ] && echo "  rules: $(find .claude/rules -name '*.md' 2>/dev/null | wc -l)" || echo "  rules: 0"
[ -d .claude/skills ] && echo "  skills: $(find .claude/skills -name 'SKILL.md' 2>/dev/null | wc -l)" || echo "  skills: 0"
[ -d .claude/agents ] && echo "  agents: $(find .claude/agents -name '*.md' 2>/dev/null | wc -l)" || echo "  agents: 0"
[ -f .claude/settings.json ] && echo "  settings.json: EXISTS" || echo "  settings.json: NOT FOUND"
```

### Step 2: Clone the repo to a temp directory

```bash
git clone --depth 1 https://github.com/ultr4nerd/claude-code-django-expert.git /tmp/django-expert-setup
```

### Step 3: Install rules

Copy Django rules. These have unique names (`models.md`, `views.md`, etc.) — they won't conflict with existing rules like `code-style.md` or `frontend.md`.

```bash
mkdir -p .claude/rules
for rule in models views serializers tests templates settings migrations security; do
    cp /tmp/django-expert-setup/.claude/rules/${rule}.md .claude/rules/${rule}.md
    echo "✓ Rule: ${rule}.md"
done
```

### Step 4: Install skills

Copy Django skills. All prefixed with `django-` so no conflicts.

```bash
mkdir -p .claude/skills
for skill in django-new-app django-new-model django-new-api django-review django-debug django-migration-check django-security-audit django-performance; do
    mkdir -p .claude/skills/${skill}
    cp /tmp/django-expert-setup/.claude/skills/${skill}/SKILL.md .claude/skills/${skill}/SKILL.md
    echo "✓ Skill: ${skill}"
done
```

### Step 5: Install agents

Copy Django agents. All prefixed with `django-` so no conflicts.

```bash
mkdir -p .claude/agents
for agent in django-reviewer django-tester django-debugger; do
    cp /tmp/django-expert-setup/.claude/agents/${agent}.md .claude/agents/${agent}.md
    echo "✓ Agent: ${agent}"
done
```

### Step 6: Install/merge hooks (settings.json)

If `.claude/settings.json` exists, **merge** Django hooks into it without removing existing hooks or settings. If it doesn't exist, create it.

```python
import json
import os

DJANGO_HOOKS = {
    "hooks": {
        "PostToolUse": [
            {
                "matcher": "Write|Edit",
                "hooks": [
                    {
                        "type": "command",
                        "command": "file=\"$CLAUDE_FILE_PATH\"; if [ -n \"$file\" ] && echo \"$file\" | grep -qE '\\.py$'; then echo \"Running ruff on $file...\"; ruff check --fix --quiet \"$file\" 2>/dev/null; ruff format --quiet \"$file\" 2>/dev/null; echo \"Ruff check and format complete.\"; fi",
                        "timeout": 10000
                    }
                ]
            }
        ],
        "PreToolUse": [
            {
                "matcher": "Bash",
                "hooks": [
                    {
                        "type": "command",
                        "command": "cmd=\"$CLAUDE_TOOL_INPUT\"; if echo \"$cmd\" | grep -qE 'migrate\\b' && ! echo \"$cmd\" | grep -qE '(showmigrations|makemigrations|--plan|--check|sqlmigrate|--fake)'; then echo 'WARNING: About to run migrations. Verify migration files have been reviewed. Use --plan to preview first.'; fi; if echo \"$cmd\" | grep -qE 'flush\\b|reset_db'; then echo 'DANGER: This will destroy all data in the database!'; exit 1; fi",
                        "timeout": 5000
                    }
                ]
            }
        ]
    }
}

settings_path = ".claude/settings.json"

if os.path.exists(settings_path):
    with open(settings_path) as f:
        existing = json.load(f)
    
    if "hooks" not in existing:
        existing["hooks"] = {}
    
    for hook_type, hook_list in DJANGO_HOOKS["hooks"].items():
        if hook_type not in existing["hooks"]:
            existing["hooks"][hook_type] = []
        # Deduplicate
        existing_cmds = set()
        for h in existing["hooks"][hook_type]:
            for inner in h.get("hooks", []):
                existing_cmds.add(inner.get("command", "")[:80])
        for h in hook_list:
            is_new = all(
                inner.get("command", "")[:80] not in existing_cmds
                for inner in h.get("hooks", [])
            )
            if is_new:
                existing["hooks"][hook_type].append(h)
    
    with open(settings_path, "w") as f:
        json.dump(existing, f, indent=2)
        f.write("\n")
    print("✓ Hooks merged into existing settings.json")
else:
    with open(settings_path, "w") as f:
        json.dump(DJANGO_HOOKS, f, indent=2)
        f.write("\n")
    print("✓ settings.json created")
```

### Step 7: Handle CLAUDE.md (CRITICAL — complement, never replace)

**If CLAUDE.md exists**: Check if it already has Django instructions. If yes, update only the Django section. If no, **append** the Django instructions after the existing content.

**If CLAUDE.md does not exist**: Create it with the Django instructions.

```python
import os

DJANGO_MARKER = "# Django Project — Claude Code Instructions"

# Read Django CLAUDE.md from the cloned repo
with open("/tmp/django-expert-setup/CLAUDE.md") as f:
    django_content = f.read().strip()

claude_md_path = "CLAUDE.md"

if os.path.exists(claude_md_path):
    with open(claude_md_path) as f:
        existing = f.read()
    
    if DJANGO_MARKER in existing:
        # Replace only the Django section
        idx = existing.find(DJANGO_MARKER)
        before = existing[:idx].rstrip()
        parts = [p for p in [before, django_content] if p]
        with open(claude_md_path, "w") as f:
            f.write("\n\n".join(parts) + "\n")
        print("✓ CLAUDE.md — Django section updated (existing content preserved)")
    else:
        # Append
        with open(claude_md_path, "a") as f:
            f.write("\n\n" + django_content + "\n")
        print("✓ CLAUDE.md — Django instructions appended (existing content preserved)")
else:
    with open(claude_md_path, "w") as f:
        f.write(django_content + "\n")
    print("✓ CLAUDE.md — created")
```

### Step 8: Install knowledge base (optional)

Ask the user if they want to include the Two Scoops Modern Django knowledge base (~132KB). If yes:

```bash
mkdir -p knowledge
cp /tmp/django-expert-setup/knowledge/two-scoops-modern-django.md knowledge/
echo "✓ Knowledge base installed"
```

If the user says yes, also add this line to the end of CLAUDE.md if it's not already there:

```
For Django best practices, read knowledge/two-scoops-modern-django.md
```

### Step 9: Cleanup

```bash
rm -rf /tmp/django-expert-setup
echo "✓ Cleanup complete"
```

### Step 10: Summary

Show the user what was installed:

```bash
echo ""
echo "=== Django Expert Setup Complete ==="
echo ""
echo "Rules:    $(find .claude/rules -name '*.md' 2>/dev/null | wc -l)"
echo "Skills:   $(find .claude/skills -name 'SKILL.md' 2>/dev/null | wc -l)"
echo "Agents:   $(find .claude/agents -name '*.md' 2>/dev/null | wc -l)"
echo "Hooks:    $(cat .claude/settings.json 2>/dev/null | python3 -c 'import json,sys; d=json.load(sys.stdin); print(sum(len(v) for v in d.get("hooks",{}).values()))' 2>/dev/null || echo 0)"
echo ""
echo "Available skills:"
echo "  /skill:django-new-app          — Scaffold a new Django app"
echo "  /skill:django-new-model        — Create a new model with best practices"
echo "  /skill:django-new-api          — Create a DRF API endpoint"
echo "  /skill:django-review           — Code review"
echo "  /skill:django-debug            — Debug an issue"
echo "  /skill:django-migration-check  — Verify migrations before commit"
echo "  /skill:django-security-audit   — Security audit"
echo "  /skill:django-performance      — Performance optimization"
echo ""
echo "Available agents:"
echo "  @django-reviewer  — Code review agent"
echo "  @django-tester    — Test generation agent"
echo "  @django-debugger  — Debugging agent"
```
