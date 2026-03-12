# Guía Completa: Claude Code, Skills, Agentes y MCP
> Documento de referencia para NotebookLM — Actualizado a marzo 2026

---

## Índice

1. [¿Qué es Claude Code?](#1-qué-es-claude-code)
2. [Cómo instalar Claude Code](#2-cómo-instalar-claude-code)
3. [CLAUDE.md — La memoria del proyecto](#3-claudemd--la-memoria-del-proyecto)
4. [Agent Skills (Habilidades)](#4-agent-skills-habilidades)
5. [MCP — Model Context Protocol](#5-mcp--model-context-protocol)
6. [Subagentes](#6-subagentes)
7. [Hooks](#7-hooks)
8. [Slash Commands](#8-slash-commands)
9. [Plugins](#9-plugins)
10. [Agent SDK](#10-agent-sdk)
11. [Skills vs. MCP vs. Subagentes — ¿Cuándo usar qué?](#11-skills-vs-mcp-vs-subagentes--cuándo-usar-qué)
12. [Ejemplo de workflow completo](#12-ejemplo-de-workflow-completo)
13. [Disponibilidad por producto](#13-disponibilidad-por-producto)
14. [Fuentes oficiales](#14-fuentes-oficiales)

---

## 1. ¿Qué es Claude Code?

Claude Code es una **herramienta de programación agentica** creada por Anthropic. No es solo un asistente de chat: es un agente general capaz de leer tu base de código, editar archivos, ejecutar comandos en terminal, y conectarse a tus herramientas de desarrollo de forma autónoma.

### Características principales

- **Lee y entiende tu codebase completo**, incluyendo múltiples archivos
- **Edita archivos directamente** sin que tengas que copiar y pegar
- **Ejecuta comandos** en terminal (tests, builds, git, etc.)
- **Trabaja en múltiples entornos**: terminal, IDE, app de escritorio, navegador, y móvil
- Sigue la **filosofía Unix**: es composable y puede encadenarse con otras herramientas

### Entornos soportados

| Entorno | Descripción |
|---|---|
| **Terminal CLI** | La interfaz completa por línea de comandos |
| **VS Code** | Extensión integrada en el editor |
| **JetBrains** | Plugin para IDEs de JetBrains |
| **App de escritorio** | Aplicación nativa |
| **Navegador web** | Acceso desde cualquier browser |
| **Móvil** | Para continuar trabajo en movimiento |

> Todos los entornos comparten el mismo motor subyacente: los archivos CLAUDE.md, configuraciones y servidores MCP funcionan en todos ellos.

### Uso en modo Unix / CI

Claude Code puede usarse de forma no interactiva, encadenado con otras herramientas:

```bash
# Monitorizar logs y recibir alertas
tail -f app.log | claude -p "Avísame por Slack si ves anomalías"

# Automatizar traducciones en CI
claude -p "traduce los nuevos strings al francés y abre un PR"

# Revisar seguridad en archivos cambiados
git diff main --name-only | claude -p "revisa estos archivos por problemas de seguridad"
```

---

## 2. Cómo instalar Claude Code

### Requisitos
- Node.js instalado
- En Windows: Git for Windows (instalarlo primero)
- Una suscripción de Claude o cuenta de Anthropic Console

### Instalación

```bash
npm install -g @anthropic-ai/claude-code
```

El paquete oficial en npm es `@anthropic-ai/claude-code`.

> Los entornos Terminal y VS Code también soportan proveedores de terceros (third-party providers).

---

## 3. CLAUDE.md — La memoria del proyecto

`CLAUDE.md` es un **archivo Markdown que se coloca en la raíz del proyecto**. Claude Code lo lee al inicio de cada sesión de trabajo.

### ¿Para qué sirve?

- Definir **estándares de código** del proyecto
- Documentar **decisiones de arquitectura**
- Especificar **librerías preferidas**
- Incluir **checklists de revisión**
- Guardar comandos de build o debugging importantes

### Auto-memoria

Claude Code también construye **auto-memoria** mientras trabaja: guarda aprendizajes como comandos de build y soluciones de debugging a través de sesiones, sin que el usuario tenga que escribir nada manualmente.

### Ejemplo de CLAUDE.md

```markdown
# Proyecto: Mi App

## Estándares de código
- Usar TypeScript estricto
- Tests con Jest, cobertura mínima 80%
- No commitear hasta que el usuario apruebe

## Arquitectura
- Backend: Express + PostgreSQL
- Frontend: React + Tailwind

## Comandos útiles
- `npm run dev` — servidor de desarrollo
- `npm run test` — ejecutar tests
```

---

## 4. Agent Skills (Habilidades)

Las **Agent Skills** (introducidas el 16 de octubre de 2025) son el sistema de extensión más poderoso de Claude. Representan una arquitectura de **"meta-herramienta basada en prompts"**: en lugar de ejecutar código directamente, preparan a Claude para resolver un problema de forma especializada.

### Definición

> Una Skill es una carpeta que contiene instrucciones, scripts y recursos que los agentes descubren y cargan dinámicamente para mejorar su rendimiento en tareas específicas.

La fórmula es:

```
Skill = Plantilla de Prompt 
      + Inyección de Contexto en Conversación 
      + Modificación del Contexto de Ejecución 
      + Scripts/Archivos opcionales
```

### Analogía clave

Crear una Skill es como preparar una **guía de onboarding para un nuevo empleado**: le explicas qué hacer, cómo hacerlo, y le das las herramientas necesarias.

### Estructura de una Skill

```
mi-skill/
├── SKILL.md       # Archivo principal con instrucciones (obligatorio)
├── script.py      # Scripts ejecutables opcionales
├── examples.txt   # Ejemplos de uso
└── /assets        # Recursos adicionales
```

El archivo `SKILL.md` debe comenzar con **frontmatter YAML**:

```markdown
---
name: calculadora
description: Realiza cálculos matemáticos con alta precisión
---

# Skill: Calculadora

## Cuándo usar esta skill
Cuando el usuario pida cálculos complejos o necesite verificar operaciones matemáticas.

## Instrucciones
...
```

### Divulgación progresiva (Progressive Disclosure)

Este es el mecanismo central de eficiencia de las Skills:

1. **Nivel 1 — Sistema**: Solo el nombre y la descripción se cargan en el system prompt siempre. Esto permite a Claude saber qué Skills existen sin consumir tokens.
2. **Nivel 2 — Bajo demanda**: El contenido completo de `SKILL.md` se carga solo cuando la tarea es relevante.
3. **Nivel 3 — Específico**: Scripts y recursos adicionales se cargan solo cuando son necesarios.

### Skills pre-construidas (de Anthropic)

Anthropic provee Skills oficiales para tareas comunes con documentos:

| Skill ID | Función |
|---|---|
| `pptx` | Crear y editar presentaciones PowerPoint |
| `xlsx` | Crear y analizar hojas de cálculo Excel |
| `docx` | Crear y editar documentos Word |
| `pdf` | Generar y manipular archivos PDF |

Estas Skills están disponibles en claude.ai y vía API.

### Dónde se almacenan las Skills

Claude Code escanea estas ubicaciones automáticamente:

```
~/.config/claude/skills/     # Configuración global del usuario
.claude/skills/              # Configuración del proyecto actual
```

### Diferencia entre Skill y Herramienta (Tool)

| Aspecto | Skill | Tool (Herramienta) |
|---|---|---|
| **Acción** | Prepara a Claude para resolver el problema | Ejecuta y devuelve resultados |
| **Naturaleza** | Enriquece el contexto/instrucciones | Ejecuta código/funciones |
| **Portabilidad** | Muy alta (funciona en Claude Code, API, claude.ai) | Depende de la implementación |
| **Ejemplo** | "Aquí está la guía para hacer PDFs" | Llamada a función `crear_pdf()` |

### Diferencia entre Skill y Prompt

| Aspecto | Skill | Prompt |
|---|---|---|
| **Alcance** | Reutilizable en múltiples conversaciones | Instrucción para una sola conversación |
| **Carga** | Bajo demanda, solo cuando es relevante | Siempre presente en el contexto |
| **Gestión** | Archivos en el filesystem, versionables | Escrito manualmente cada vez |

### Disponibilidad por plataforma

| Plataforma | Skills pre-construidas | Skills personalizadas |
|---|---|---|
| **claude.ai** (Pro/Max/Team/Enterprise) | ✅ Automáticas | ✅ Vía Settings > Features (zip) |
| **Claude API** | ✅ Con `skill_id` | ✅ Vía `/v1/skills` (compartidas en org) |
| **Claude Code** | ❌ Solo custom | ✅ Filesystem-based |
| **Agent SDK** | ❌ Solo custom | ✅ En `.claude/skills/` |

> Importante: Las Skills no se sincronizan entre plataformas. Una Skill subida a claude.ai no está disponible automáticamente en la API, y viceversa.

---

## 5. MCP — Model Context Protocol

El **Model Context Protocol (MCP)** es un estándar abierto (lanzado en noviembre 2024, transferido a la Linux Foundation en diciembre 2025) para conectar asistentes de IA con sistemas y datos externos.

### Definición

MCP actúa como una **capa de conexión universal**: en lugar de construir integraciones personalizadas para cada fuente de datos, se construye contra un único protocolo.

> Si las Skills son el "libro de jugadas interno" del agente, MCP es el "sistema nervioso" que lo conecta con el mundo exterior.

### Cómo funciona

```
Claude Code (MCP Client)
        │
        ├──► MCP Server (GitHub)    → acceso a repos, PRs, issues
        ├──► MCP Server (Postgres)  → consultas a base de datos
        ├──► MCP Server (Slack)     → leer/enviar mensajes
        └──► MCP Server (Jira)      → gestionar tickets
```

1. Los usuarios configuran su agente con un archivo `.mcp.json` que lista los servidores MCP disponibles.
2. Cuando Claude necesita usar una herramienta (consultar datos, ejecutar una acción), emite una solicitud estándar MCP.
3. El cliente enruta la solicitud al servidor MCP correspondiente.
4. El servidor ejecuta la acción (llamada a API, consulta BD, etc.) y devuelve resultados estructurados.

### Transporte

Desde marzo 2025 (spec 2025-03-26), el transporte estándar es **Streamable HTTP**, que reemplazó al anterior HTTP+SSE. Soporta:
- Transferencia por chunks (chunked transfer encoding)
- Entrega progresiva de mensajes
- Despliegue en entornos serverless

### Casos de uso típicos

- Leer documentos de diseño en **Google Drive**
- Actualizar tickets en **Jira**
- Obtener datos de **Slack**
- Consultar **bases de datos** PostgreSQL
- Interactuar con **GitHub** (código, PRs, issues)
- Publicar en redes sociales (Twitter/X, Shopify, etc.)

### MCP es agnóstico

Al estar bajo la Linux Foundation, MCP no está atado a Claude. Clientes como ChatGPT, Gemini, y otros IDEs con integración MCP también pueden usarlo.

### Limitaciones de MCP

- Cargar un servidor MCP puede consumir **miles de tokens** de contexto (el servidor de GitHub oficial puede consumir decenas de miles de tokens solo en definiciones de herramientas).
- No es una solución "llave en mano" para flujos complejos: tiene dificultades con secuencias largas, modelos de datos específicos de apps, triggers en tiempo real, y escenarios con intervención humana.

---

## 6. Subagentes

Los **subagentes** son asistentes de IA especializados con su propio contexto, system prompt personalizado, y permisos de herramientas específicos. Permiten paralelizar y aislar el trabajo.

### Cómo funcionan

- Cada subagente opera con **su propia ventana de contexto** (context window)
- El agente principal puede **delegar tareas** a subagentes automáticamente o de forma explícita
- Los subagentes reportan sus resultados al agente principal
- Trabajan de forma **simultánea** en diferentes partes de una tarea

### Ejemplo práctico

```
Agente principal: "Implementar autenticación OAuth 2.0"
    │
    ├── Subagente 1: Investigar mejores prácticas de OAuth 2.0
    ├── Subagente 2: Documentar el flujo de auth actual
    └── Subagente 3: Escribir tests para la nueva implementación
```

Los tres trabajan en paralelo y luego reportan al agente principal.

### Caso de uso: Subagente revisor de código

```
Crear subagente "code-reviewer" con acceso a:
  ✅ Read, Grep, Glob  (puede leer código)
  ❌ Write, Edit       (NO puede modificar)
```

Así Claude delega automáticamente las revisiones de calidad y seguridad a este subagente sin riesgo de cambios no deseados.

### Características importantes

- **Solo disponibles en Claude Code y Claude Agent SDK** (no en claude.ai ni API directa)
- Tienen las mismas capacidades que el agente padre (no poderes especiales)
- **Aislamiento de contexto**: los subagentes no pueden compartir información directamente entre sí
- Cada subagente cuenta hacia los límites de uso

### Skill vs. Subagente

| Uso | Skill | Subagente |
|---|---|---|
| Compartir expertise entre agentes | ✅ Ideal | ❌ No aplica |
| Ejecutar tareas de forma paralela | ❌ | ✅ Ideal |
| Aislar permisos de herramientas | ❌ | ✅ Ideal |
| Portabilidad entre plataformas | ✅ Alta | ❌ Solo Claude Code/SDK |

---

## 7. Hooks

Los **Hooks** son comandos de shell que se ejecutan automáticamente **antes o después de acciones de Claude Code**.

### Casos de uso comunes

- Auto-formatear código después de cada edición de archivo
- Ejecutar linter antes de cada commit
- Requerir aprobación manual antes de ejecutar comandos bash
- Validar escrituras en archivos
- Inicializar sesiones (por ejemplo, cargar variables de entorno)
- Enviar notificaciones de escritorio cuando Claude termina una tarea

### Cuándo usar Hooks

Los Hooks son perfectos para **reglas y automatizaciones repetitivas** que siempre deben ocurrir en puntos específicos del flujo de trabajo, independientemente del task.

---

## 8. Slash Commands

Los **Slash Commands** (`/nombre`) son atajos para **workflows repetibles** que los equipos pueden compartir.

### Ejemplos

```
/review-pr          → Ejecuta el flujo completo de revisión de PR
/deploy-staging     → Despliega al entorno de staging
/load-context       → Inicializa un nuevo chat con el estado del proyecto
```

### Diferencia con Skills

| | Slash Command | Skill |
|---|---|---|
| **Activación** | El usuario la dispara explícitamente escribiendo `/comando` | Claude la activa automáticamente según el contexto de la tarea |
| **Uso** | Workflows que quieres controlar manualmente | Expertise que Claude aplica cuando lo necesita |

---

## 9. Plugins

Los **Plugins** son **bundles distribuibles** que empaquetan comandos, hooks, skills y metadatos en una configuración compartible.

### Para qué sirven

- Compartir la configuración completa de un equipo con nuevos miembros
- Distribuir workflows de dominio específico
- Instalar configuraciones pre-construidas de la comunidad

### Ejemplo

```bash
# Instalar marketplace de skills (setup único)
/plugin marketplace add anthropics/skills
```

Los Plugins son la capa de distribución: empaquetan todo lo anterior para que otros puedan usarlo fácilmente.

---

## 10. Agent SDK

El **Claude Agent SDK** es para construir **agentes desplegables y autónomos** con bucles de decisión completos.

### Cuándo usar el Agent SDK

- Necesitas **control total** sobre la orquestación
- Quieres gestionar tú mismo los permisos de herramientas
- Estás construyendo workflows de agentes a medida para producción
- Quieres agentes que se ejecuten de forma **completamente autónoma**

### Skills en el Agent SDK

```python
# Configuración de Skills en el SDK
allowed_tools = ["Skill", "Read", "Write", ...]
# Las Skills se descubren automáticamente en .claude/skills/
```

---

## 11. Skills vs. MCP vs. Subagentes — ¿Cuándo usar qué?

Esta es la pregunta clave. Aquí el resumen definitivo:

### Regla rápida

| Quiero... | Usar |
|---|---|
| Dar a Claude expertise en un dominio (cómo hacer PDFs, cómo seguir mis estándares) | **Skill** |
| Conectar Claude a un sistema externo (GitHub, Jira, BD) | **MCP** |
| Ejecutar tareas en paralelo o aislar permisos | **Subagente** |
| Recordar instrucciones del proyecto entre sesiones | **CLAUDE.md** |
| Disparar manualmente un workflow repetible | **Slash Command** |
| Automatizar acciones en puntos específicos del flujo | **Hook** |
| Distribuir mi configuración a otros | **Plugin** |

### Capas del stack agentic de Claude

```
┌─────────────────────────────────────────────────────┐
│                   CLAUDE CODE / Apps                │
│                                                     │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │  CLAUDE.md  │  │    Skills    │  │   Hooks   │  │
│  │  (memoria)  │  │  (expertise) │  │ (reglas)  │  │
│  └─────────────┘  └──────────────┘  └───────────┘  │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │              Subagentes                      │   │
│  │     (paralelismo + aislamiento)              │   │
│  └──────────────────────────────────────────────┘   │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │           MCP Client                         │   │
│  └────────┬────────────┬──────────┬─────────────┘   │
└───────────┼────────────┼──────────┼─────────────────┘
            │            │          │
     ┌──────▼───┐  ┌─────▼────┐  ┌─▼────────┐
     │  GitHub  │  │ Postgres │  │  Slack   │
     │  (MCP)   │  │  (MCP)   │  │  (MCP)   │
     └──────────┘  └──────────┘  └──────────┘
```

### Skills y MCP son complementarios, no competidores

- **MCP** conecta al agente con el mundo exterior
- **Skills** enseñan al agente cómo trabajar en ese mundo
- Juntos forman el stack completo de automatización

---

## 12. Ejemplo de workflow completo

### Escenario: Agente de análisis competitivo

Este ejemplo combina todas las piezas:

**CLAUDE.md** (memoria del proyecto):
```markdown
# Proyecto: Análisis Competitivo
Analiza competidores desde la perspectiva de nuestra estrategia de producto.
Enfócate en oportunidades de diferenciación y tendencias emergentes.
```

**Skill: Google Drive Navigation**:
```markdown
---
name: gdrive-navigation
description: Estrategia optimizada de búsqueda en Google Drive de Meridian Tech
---
Usa esta skill para localizar eficientemente documentos internos, investigación 
y materiales estratégicos...
```

**Subagentes** (trabajando en paralelo):
- Subagente 1: Recopila datos de competidores via MCP → GitHub
- Subagente 2: Busca documentos internos via Skill → GDrive
- Subagente 3: Analiza tendencias de mercado

**MCP** conectado a:
- Google Drive (documentos internos)
- Jira (seguimiento de tareas)
- Slack (compartir resultados)

**Hook** al finalizar:
- Notificación en Slack con el resumen del análisis

---

## 13. Disponibilidad por producto

| Característica | claude.ai | Claude API | Claude Code | Agent SDK |
|---|---|---|---|---|
| Skills pre-construidas (pptx, xlsx, docx, pdf) | ✅ | ✅ | ❌ | ❌ |
| Skills personalizadas | ✅ (Pro+) | ✅ | ✅ | ✅ |
| MCP | ✅ | ✅ | ✅ | ✅ |
| Subagentes | ❌ | ❌ | ✅ | ✅ |
| Hooks | ❌ | ❌ | ✅ | ❌ |
| CLAUDE.md | ❌ | ❌ | ✅ | ❌ |
| Slash Commands | ❌ | ❌ | ✅ | ❌ |
| Plugins | ❌ | ❌ | ✅ | ❌ |
| Agent SDK | ❌ | ❌ | ❌ | ✅ |

---

## 14. Fuentes oficiales

| Recurso | URL |
|---|---|
| Claude Code - Documentación oficial | https://docs.claude.com/en/docs/claude-code/overview |
| Agent Skills - Overview | https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview |
| Blog Anthropic: Agent Skills | https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills |
| Blog: Skills vs. Prompts vs. MCP | https://claude.com/blog/skills-explained |
| npm: Claude Code package | https://www.npmjs.com/package/@anthropic-ai/claude-code |
| Soporte claude.ai | https://support.claude.com |

---

*Documento generado con Claude Sonnet 4.6 — Marzo 2026*
