# Kit de Claude Code: un equipo de agentes con estándar de ingeniería

Este kit convierte a [Claude Code](https://claude.com/claude-code) en un **equipo de desarrollo
con reglas**: un orquestador que planifica y reparte trabajo, agentes especialistas que lo
ejecutan en paralelo, y un estándar escrito que todos respetan. Opcionalmente suma
[Codex CLI](https://github.com/openai/codex) como segundo ejecutor y revisor.

La IA no tiene la última palabra: cada cambio pasa por build, tests, linters y revisión antes de
darse por terminado. Nada llega a `git push` ni se ejecuta contra una base de datos sin que una
persona lo autorice.

Stack del estándar: **Python + FastAPI + PostgreSQL** (psycopg 3, SQL a mano) y
**React + TypeScript + Vite + Tailwind v4**, con CI en GitHub Actions y deploy en Railway y
Vercel.

## Cómo trabaja

```
                       ┌────────────────────┐
     pedido  ────────▶ │    orquestador     │  lee el perfil del proyecto,
                       │  (agente principal)│  fija el contrato y reparte
                       └─────────┬──────────┘
            ┌─────────────────┬──┴──────────────┬─────────────────┐
            ▼                 ▼                 ▼                 ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │backend-python│  │frontend-react│  │    Codex     │  │      qa      │
    │ API, SQL,    │  │ páginas,     │  │ tareas       │  │ navegador    │
    │ tests        │  │ hooks, tests │  │ acotadas     │  │ (Playwright) │
    └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
           └─────────────────┴───────┬─────────┴─────────────────┘
                                     ▼
              verificación del orquestador: build + tests + reglas
```

1. **Perfil del proyecto.** Cada sistema declara en su `CLAUDE.md` nombres, rutas, schema,
   conexiones, puertos y excepciones al estándar. Los agentes leen ese perfil y nunca asumen
   nombres. `/adoptar` lo completa.
2. **Contrato primero.** Si una feature cruza backend y frontend, el orquestador fija antes los
   DTOs, el endpoint, los códigos de error y el permiso, y se los pasa a los dos agentes.
3. **Trabajo en paralelo.** Las tareas independientes se lanzan a la vez: backend, frontend y
   Codex sobre archivos distintos. Las dependientes van en serie, y QA siempre al final.
4. **Verificación obligatoria.** El reporte de un agente no alcanza: el orquestador vuelve a
   correr build, tests, `ruff`, `mypy`, `import-linter` y `eslint`, y revisa contra las reglas.

## Qué incluye

| Carpeta | Contenido |
|---|---|
| `.claude/agents/` | `orquestador`, `backend-python`, `frontend-react`, `qa` |
| `.claude/rules/` | Arquitectura en 4 capas, API y envelope `ApiResponse`, datos y SQL, migraciones, React, estilos, testing, seguridad y CI. Las reglas con `paths:` se cargan solo al tocar esos archivos. |
| `.claude/commands/` | `/adoptar`, `/feature-backend`, `/feature-frontend`, `/migracion`, `/verificar`, `/revisar`, `/commit`, `/pr`, `/levantar`, `/qa`, `/jira`, `/codex-delegar`, `/codex-revisar` |
| `.claude/templates/` | Plantilla del perfil (`CLAUDE.proyecto.md`) y archivos base: workflow de CI, Dockerfile, `railway.toml`, `vercel.json`, dependabot, CODEOWNERS |
| `.claude/skills/` | Skills de terceros incluidos (ver [THIRD_PARTY.md](THIRD_PARTY.md)) |
| `.claude/settings.json` | Permisos: qué se permite, qué pregunta y qué se prohíbe |

## Límites incorporados

Están en las reglas **y** en los permisos de `settings.json`, no solo en los prompts:

- **Prohibido:** `git push`, `git reset --hard`, merge o aprobación de PRs, gestión de secretos
  de GitHub, y lectura de archivos `.env`.
- **Requiere confirmación:** `git commit`, `psql`, instalar dependencias.
- Ningún secreto se escribe en código, docs, logs ni permisos.

## Codex como segundo agente

- `/codex-delegar <tarea>`: Claude arma un prompt autosuficiente (perfil, reglas, contrato y
  límites), lo ejecuta con `codex exec -s workspace-write` y después **verifica** el resultado
  como el de cualquier otro agente.
- `/codex-revisar`: segunda opinión con `codex exec -s read-only`. Claude comprueba cada
  hallazgo en el código antes de reportarlo.

Codex nunca corre con `danger-full-access`, y el contexto viaja siempre dentro del prompt porque
Codex no lee `CLAUDE.md` por su cuenta.

## Uso

1. Copiá `.claude/` y `.mcp.json` a la raíz de tu proyecto, nuevo o existente.
2. Abrí Claude Code en el proyecto y corré `/adoptar`. Pregunta lo que falte, completa el perfil
   y reporta las diferencias entre el código existente y el estándar.
3. Pedí features en lenguaje natural. El orquestador decide qué hace cada agente.

Requisitos: Claude Code. Opcionales: Codex CLI (≥ 0.157), `gh` y el MCP de Atlassian para Jira.

## Lecciones que dejó usarlo en un proyecto real

El kit se probó construyendo un sistema de OCR + RAG sobre normativa aduanera centroamericana:

- **Medir antes de opinar.** Un set de evaluación con hit@k, MRR y porcentaje de citas completas
  convirtió "responde bien" en números. Un cambio de prompt llevó las citas completas del 6 % al
  81 %.
- **Verificar también lo que da el orquestador.** Dos números de contenedor "válidos" que le pasé
  a Codex tenían mal el dígito verificador. Una implementación correcta iba a "fallar" contra
  datos malos.
- **Los límites se escriben en permisos, no solo en prompts.** Un agente llegó a esquivar el
  bloqueo del `.env` con otro comando. Desde entonces la prohibición es explícita y está en
  `settings.json`.
- **Paralelizar exige contrato.** Tres agentes trabajando a la vez en backend, frontend y
  validaciones solo encajan si el contrato se fija antes y no se toca.

## Mantener el estándar

- Toda regla nueva debe ser **genérica**. Si menciona un nombre, tabla, puerto o ruta de un
  sistema concreto, va al perfil de ese proyecto.
- Cuando una mejora se prueba en un proyecto, se sube acá para que la hereden los demás.

## Licencia

El contenido propio del kit se publica bajo [MIT](LICENSE). Los skills de terceros conservan su
licencia original: ver [THIRD_PARTY.md](THIRD_PARTY.md).
