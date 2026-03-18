# Orquestación de Agentes y Solución de Problemas

---

## 1. Evolución de los Entornos de Desarrollo: Hacia el AI-First IDE

Desde 2022, el entorno de desarrollo integrado (IDE) ha dejado de ser una simple superficie de escritura para transformarse en un socio cognitivo. No estamos ante una mejora incremental, sino ante un cambio en la naturaleza de la interacción hombre-máquina.

### Cronología de la Transformación (2022–2026)

| Año | Fase | Ejemplos |
|-----|------|----------|
| 2022 | **IDEs Clásicos** — Autocompletado básico de sintaxis | IntelliJ, VS Code |
| 2023 | **Chat-based coding** — Interfaces de chat para consultas | V0, Warp |
| 2024 | **Copilots** — Sugerencias de bloques de código en tiempo real | Cursor, Codex |
| 2025 | **AI Agents** — Autonomía para ejecutar tareas complejas | Claude Code, Zed, Trae, Kiro |
| 2026 | **AI-First IDEs** — La IA como núcleo arquitectónico | Lovable, Antigravity |

### Las 4 Características Fundamentales de un IDE AI-First

1. **Contexto completo**: Visibilidad total del codebase, no solo del archivo activo.
2. **Edición multi-archivo**: Coordinación de refactorizaciones en múltiples ficheros simultáneamente.
3. **Ejecución real**: Capacidad para lanzar comandos, ejecutar tests y gestionar procesos de build o deploy.
4. **Iteración autónoma**: Detección proactiva de errores y corrección automática mediante bucles de retroalimentación.

> En este paradigma, el desarrollador asume el rol de **Director u orquestador**, validando intenciones estratégicas en lugar de corregir líneas de código aisladas.

---

## 2. El Salto Cualitativo: Chatbots vs. Agentes Autónomos

Para un profesional, el uso de chats convencionales es una ineficiencia estratégica. La transición a flujos agénticos permite operar en un nivel de abstracción superior.

| Característica | Chatbot (Turno único) | Agente (Flujo Agéntico) |
|---|---|---|
| Interacción | Pregunta → Respuesta | Bucle autónomo iterativo |
| Acceso al sistema | Bloqueado / Sin acceso | Lectura/Escritura de archivos |
| Verificación | Requiere copy-paste manual | Ejecuta y verifica resultados |
| Rol del Humano | Corrección de sintaxis | Validación de intenciones |

### El Bucle del Agente

```
Entiende → Planifica → Actúa → Evalúa → Ajusta → (Repite)
```

---

## 3. Control Estratégico y Reglas del Proyecto

### Plan Mode (Modo Plan)

Funcionalidad crítica donde el agente detalla una estrategia **antes de tocar el código**. Es el mecanismo principal para evitar cambios destructivos.

- ✅ **Cuándo usarlo**: Tareas de gran envergadura, refactorizaciones complejas y supervisión estratégica.
- ❌ **Cuándo NO usarlo**: Tareas atómicas, correcciones obvias o prototipado rápido.

### Archivos de Reglas y AGENT.md

Son archivos Markdown que funcionan como **system prompts persistentes**. Dependiendo del IDE, adoptan nombres específicos:

| IDE | Nombre del archivo |
|-----|--------------------|
| Genérico | `AGENT.md` |
| Cursor | `.cursorrules` |
| Claude Code | `CLAUDE.md` |
| Copilot | `copilot-instructions.md` |

**Contenido obligatorio:**
- Stack tecnológico
- Convenciones de código
- Patrones arquitectónicos
- Prohibiciones explícitas
- Estructuras de carpetas
- Estilo de commits y PRs

> ⚠️ **Restricción técnica**: No debe exceder las **500 líneas**. Superar este límite genera ruido, aumenta la latencia y provoca la "lobotomía" o amnesia del modelo.

---

## 4. Conectividad y Conocimiento: MCP y Skills

### Model Context Protocol (MCP)

Estándar abierto diseñado para conectar la IA con herramientas externas mediante una API estándar (JSON-RPC) para un flujo de datos bidireccional y seguro.

**Conectividad disponible:**
- **Bases de datos**: PostgreSQL, Supabase
- **Herramientas de desarrollo**: GitHub, Sentry
- **Productividad**: Notion, Slack

**Ventajas:** seguridad por diseño, evita el vendor lock-in y permite la reutilización total de conectores entre diferentes IDEs.

### Skills (Habilidades)

A diferencia del MCP (servicios externos), las Skills representan el **conocimiento interno** y las mejores prácticas del proyecto.

- **Estructura**: Definidas mediante archivos `SKILL.md` y scripts ejecutables.
- **Skills Registry**: Funciona como un Router liviano con **Lazy Loading**, inyectando solo el conocimiento necesario para la tarea actual y evitando saturar la ventana de contexto.

---

## 5. Arquitectura de Memoria: El Problema del Contexto y Engram

### El Problema de la Ventana de Contexto

Incluso con millones de tokens, los modelos sufren de:

1. **Ruido**: Información irrelevante que degrada la precisión.
2. **Amnesia**: Pérdida de contexto entre sesiones de trabajo.
3. **Compactación / Lobotomía**: El sistema resume forzosamente el historial, perdiendo el "porqué" de las decisiones técnicas.

### Engram (Memoria Persistente)

Solución de memoria persistente basada en **SQLite**, diseñada para ser agent-agnostic.

- **Funcionamiento**: Registra observaciones de arquitectura, resúmenes de sesión y patrones detectados.
- **Estructura de datos**: Para cada decisión registra el **Qué**, el **Porqué**, el **Dónde** y qué aprendimos, permitiendo recuperar el razonamiento técnico de sesiones pasadas de forma determinista.

---

## 6. Orquestación Multi-Agente: De la Tarea Unitaria a la Legión

La orquestación profesional se divide en tres niveles de madurez:

| Nivel | Nombre | Descripción |
|-------|--------|-------------|
| **1** | Subagents | Uso de herramientas nativas (ej. `task_tool` en Claude Code) para generar agentes efímeros que resuelven subtareas paralelas sin contaminar el contexto principal. |
| **2** | Agent Teams Lite | Orquestación declarativa en Markdown puro. Open Source, Zero dependencies, colaboración entre agentes sin lógica de programación compleja. |
| **3** | Full Agent Teams | Flujos basados en DAG (Directed Acyclic Graph) con fases secuenciales/paralelas, persistencia total con Engram y puntos de aprobación humana (Approval Gates). |

### Herramientas Clave

- **Claude Code**: CLI y sistema agéntico de Anthropic con soporte nativo para asincronía.
- **Open Code**: Alternativa Open Source con soporte multi-provider (varios modelos simultáneos) para evitar la dependencia de un solo proveedor de IA.

---

## 7. SDD Orchestrator: El Pipeline de Desarrollo Profesional

El **SDD Orchestrator** (Spec Driven Development) automatiza el ciclo de vida del software mediante agentes especializados que generan **Contratos de Resultado** entre fases:

```
Exploratorio → Proposer → Spec Writer & Designer → Task Planner → Implementer → Verifier
```

| Agente | Rol |
|--------|-----|
| **Exploratorio** | Investiga el codebase para entender el estado actual. |
| **Proposer** | Emite una propuesta técnica preliminar. |
| **Spec Writer & Designer** | Redactan especificaciones técnicas y definen la arquitectura, creando un contrato de requisitos. |
| **Task Planner** | Descompone el plan en tareas atómicas y ejecutables. |
| **Implementer** | Ejecuta la codificación física de los componentes. |
| **Verifier** | Valida que la implementación cumpla con los Contratos de Resultado y las pruebas de calidad. |

---

## 8. El Nuevo Stack Cognitivo y Conclusiones

### Stack Cognitivo Completo para 2026

```
Engram (Persistencia)
  + Agent Teams Lite (Orquestación)
  + Skills Registry (Conocimiento)
  + Human in the Loop (Control)
```

### Competencias Críticas del Desarrollador Senior

1. **Dominio de Fundamentos Técnicos**: Sin bases sólidas, es imposible auditar la producción de la "Legión" de agentes.
2. **Maestría en Orquestación**: Habilidad para configurar flujos deterministas mediante `AGENT.md`, protocolos MCP y Skills.
3. **Control Crítico (Human in the Loop — HITL)**: Supervisión de la autonomía gradual, manteniendo el juicio sobre la estrategia para asegurar software sostenible y escalable.
