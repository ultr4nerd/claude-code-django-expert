# Claude Code Django Expert

Django 6.x expert toolkit for [Claude Code](https://code.claude.com). Makes Claude Code an expert Django developer with modern best practices from Two Scoops of Django, Twelve-Factor App, and Cookiecutter Django.

## Setup (2 pasos)

### Paso 1: Instalar el skill globalmente (una sola vez)

```bash
mkdir -p ~/.claude/skills/setup-django-expert

curl -sL https://raw.githubusercontent.com/ultr4nerd/claude-code-django-expert/main/.claude/skills/setup-django-expert/SKILL.md \
  -o ~/.claude/skills/setup-django-expert/SKILL.md
```

Esto instala un solo archivo en tu máquina. No consume tokens extra en tus sesiones.

### Paso 2: Ejecutar en tu proyecto Django

Abre Claude Code en tu proyecto y ejecuta:

```
/skill:setup-django-expert
```

El skill se encarga de todo:
- Clona este repo temporalmente
- Copia rules, skills, agents y hooks a `.claude/` de tu proyecto
- **Si ya tienes un `CLAUDE.md`**, agrega las instrucciones Django al final sin borrar nada
- **Si ya tienes `.claude/settings.json`** con hooks, los mergea sin duplicar
- Te pregunta si quieres la knowledge base de Two Scoops (~132KB)
- Limpia el clon temporal

Repite el paso 2 en cada proyecto Django donde lo necesites.

## Qué se instala en tu proyecto

| Componente | Cantidad | Descripción |
|---|---|---|
| `CLAUDE.md` | 1 | Instrucciones del stack Django — 12-factor, convenciones, comandos |
| Rules | 8 | Se auto-activan al editar models, views, serializers, tests, templates, settings, migrations, security |
| Skills | 8 | `/skill:django-new-app`, `django-new-model`, `django-new-api`, `django-review`, `django-debug`, `django-migration-check`, `django-security-audit`, `django-performance` |
| Agents | 3 | `@django-reviewer`, `@django-tester`, `@django-debugger` |
| Hooks | 2 | Auto-ruff al guardar `.py`, protección contra migraciones destructivas |
| Knowledge | 1 | Guía modernizada Two Scoops of Django (~4,800 líneas) — opcional |

## Seguro para proyectos existentes

El skill **nunca sobreescribe** tu configuración existente:

- **CLAUDE.md** → Si existe, las instrucciones Django se **agregan al final**. Si ya tiene sección Django, solo actualiza esa sección.
- **Rules** → Solo agrega rules Django (`models.md`, `views.md`, etc.). Tus rules como `code-style.md` o `frontend.md` no se tocan.
- **Skills/Agents** → Solo agrega los prefijados con `django-`. Tus otros skills y agents quedan intactos.
- **settings.json** → Mergea los hooks Django con los existentes. Tus hooks de ESLint, Prettier, etc. se preservan.

## Cero impacto en proyectos no-Django

Si instalaste el skill global (paso 1) pero estás en un proyecto FastAPI, JS, o Go:
- El skill `setup-django-expert` existe pero **no se ejecuta automáticamente** — solo cuando tú lo invocas
- No se agregan tokens extra a tu sesión
- Las skills y agents de Django no se cargan a menos que los invoques explícitamente

## Uso después de instalar

```
/skill:django-new-app          → Scaffoldear una nueva app Django
/skill:django-new-model        → Crear un modelo con buenas prácticas
/skill:django-new-api          → Crear un endpoint DRF
/skill:django-review           → Code review
/skill:django-debug            → Debuggear un issue
/skill:django-migration-check  → Verificar migraciones antes de commit
/skill:django-security-audit   → Auditoría de seguridad
/skill:django-performance      → Optimización de performance

@django-reviewer               → Agente de code review
@django-tester                 → Agente de generación de tests
@django-debugger               → Agente de debugging
```

## Django Version

Target: **Django 6.0.3** (marzo 2026) con:
- CSP middleware (built-in)
- Template Partials (`{% partialdef %}`)
- Background Tasks framework
- `GeneratedField` y `db_default`
- Composite Primary Keys
- Async ORM (maduro)
- `LoginRequiredMiddleware`

## Actualizar

Para actualizar el toolkit en un proyecto, simplemente ejecuta `/skill:setup-django-expert` de nuevo. Reemplazará las rules, skills y agents Django con la versión más reciente del repo, y actualizará la sección Django del CLAUDE.md.

## License

MIT
