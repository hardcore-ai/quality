# Hardcore AI 30X — Clase 8: Aseguramiento de Calidad

Repositorio de material y ejercicios prácticos para la **Clase 8: Aseguramiento de Calidad — Pruebas Web, API y Agentes con IA Generativa** del programa Hardcore AI 30X.

Esta sesión cubre los fundamentos del testing moderno — desde la pirámide de pruebas hasta BDD con Gherkin — y aborda tres dimensiones de pruebas: **E2E web** con Playwright, **pruebas de API** para servicios y abstracciones de agentes, y **evaluación de agentes inteligentes** usando el patrón Persona + Juez. El diferencial: cada paso se acelera con coding agents, MCPs de Playwright y skills especializadas en IDEs agénticos.

## Estructura del repositorio

```
repos/
├── qa-e2e/            # Pruebas E2E web y BDD
├── qa-api/            # Pruebas de API REST
└── qa-agent-eval/     # Evaluación de agentes (Persona + Juez)
```

Cada carpeta contiene su propia `demo-api/` o `demo-app/` con una aplicación Next.js de ejemplo (un asistente virtual de soporte al cliente) que sirve como sujeto de pruebas.

---

## qa-e2e — Pruebas E2E Web y BDD

Pruebas de interfaz web de punta a punta usando Playwright, con tres niveles de testing:

- **Unit tests** con Vitest — validación de lógica pura del agente
- **E2E tests** con Playwright — flujos completos en el navegador (enviar mensajes, verificar respuestas, historial)
- **BDD tests** con Gherkin + playwright-bdd — escenarios en español con sintaxis Given/When/Then

Patrones aplicados: **Page Object Model** (`HomePage`, `ConversacionPage`), step definitions reutilizables.

```bash
cd repos/qa-e2e
npm install

# Levantar la app demo
cd demo-app && npm install && npm run dev &

# Ejecutar tests
cd ..
npx playwright test              # Todos los tests
npx playwright test tests/e2e/   # Solo E2E
npm run test:bdd                 # Solo BDD (Gherkin)
npx playwright test --ui         # Modo interactivo
npm run report                   # Ver reporte HTML
```

---

## qa-api — Pruebas de API REST

Suite de pruebas de API usando el `request` context de Playwright para validar endpoints REST sin navegador:

- **endpoints.spec.ts** — CRUD de conversaciones y mensajes (health, create, list, get, delete)
- **contratos.spec.ts** — Validación de contratos de respuesta (estructura, tipos, tiempos)
- **agente-api.spec.ts** — Comportamiento semántico del agente vía API (contexto, seguridad, edge cases)

Patrón aplicado: **Request Builders** (`fixtures/request-builders.ts`) para abstraer las llamadas HTTP.

```bash
cd repos/qa-api
npm install

# Levantar la API demo
cd demo-api && npm install && npm run dev &

# Ejecutar tests
cd ..
npx playwright test                              # Todos
npx playwright test tests/api/endpoints.spec.ts  # Solo endpoints
npx playwright test tests/api/contratos.spec.ts  # Solo contratos
npx playwright test tests/api/agente-api.spec.ts # Solo agente
npm run report                                   # Ver reporte HTML
```

---

## qa-agent-eval — Evaluación de Agentes (Persona + Juez)

Implementación del patrón **Persona + Juez** para evaluar agentes inteligentes de forma automatizada:

1. **Agente Persona** (`scripts/persona-agent.ts`) — Un LLM (GPT-4o-mini) simula un usuario real basado en personas predefinidas e interactua con el agente bajo prueba via API
2. **Agente Juez** (`scripts/judge-agent.ts`) — Un LLM (GPT-4o) evalua la conversacion completa usando un rubric de 6 dimensiones (escala 1-5)
3. **Orquestador** (`scripts/run-evaluation.ts`) — Ejecuta el flujo completo y genera scorecards

Recursos incluidos:
- `personas/user-personas.md` — 3 personas de usuario (Carlos: tecnico, Maria: no tecnica, Pedro: frustrado)
- `rubrics/evaluacion.md` — Rubric de evaluacion con 6 dimensiones
- `datasets/golden-dataset.json` — 5 conversaciones de referencia
- `reports/` — Scorecards generados en JSON

```bash
cd repos/qa-agent-eval
npm install

# Levantar la API demo
cd demo-api && npm install && npm run dev &

# Configurar tu API key de OpenAI (NO la incluyas en el repo)
export OPENAI_API_KEY="tu-api-key"

# Ejecutar evaluacion
cd ..
npm run eval                        # Evaluacion con persona por defecto
npm run eval:persona -- --persona carlos  # Persona especifica
npm run eval:all                    # Todas las personas
```

---

## Prerrequisitos

- **Node.js** >= 18
- **IDE agéntico** configurado (Kiro, Cursor o Claude Code con MCP habilitado)
- **API key de OpenAI** (solo para `qa-agent-eval`) — configurada como variable de entorno, nunca en el codigo
- Familiaridad basica con terminal y TypeScript/JavaScript

## Conceptos cubiertos

| Concepto | Proyecto |
|---|---|
| Piramide de pruebas (Unit, Integration, E2E) | qa-e2e |
| Page Object Model (POM) | qa-e2e |
| BDD con Gherkin (Given/When/Then) | qa-e2e |
| Testing de API con Playwright request context | qa-api |
| Request Builders y Contract Testing | qa-api |
| Patron Persona + Juez para agentes IA | qa-agent-eval |
| Evaluacion semantica con rubrics | qa-agent-eval |
| Playwright MCP Server | Todos |
| Skills y steering files para testing | qa-e2e |

## Herramientas de IA utilizadas

| Herramienta | Uso |
|---|---|
| Playwright MCP Server | Navegar, inspeccionar DOM, screenshots y ejecutar acciones en el browser |
| Playwright API Testing | `request` context para pruebas de endpoints REST y agentes |
| Coding Agent (Claude Code / Kiro / Cursor) | Generar tests, page objects, step definitions y corregir fallos |
| LLM como Agente Persona | Simular usuarios reales interactuando con el agente bajo prueba |
| LLM como Agente Juez | Evaluar conversaciones con rubric y generar scorecards |
| Skills / Steering Files | Convenciones de testing consistentes para el agente |

---

**Autor:** Andres Caicedo | **Programa:** Hardcore AI 30X
