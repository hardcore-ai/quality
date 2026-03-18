# qa-agent-eval — Evaluacion de Agentes con Patron Persona + Juez

Proyecto de referencia para la **Clase 8: Aseguramiento de Calidad** del programa Hardcore AI 30X.

Implementa el patron **Persona + Juez** para evaluar agentes de IA: un LLM simula usuarios realistas (Persona) y otro LLM evalua las conversaciones con una rubrica predefinida (Juez).

> Este repo es autocontenido: incluye una API simulada (`demo-api/`) del agente bajo prueba.

## Estructura del proyecto

```
qa-agent-eval/
├── demo-api/                      # API simulada del agente (Next.js)
│   ├── app/api/
│   │   ├── health/route.ts
│   │   ├── conversations/...
│   │   └── agent/chat/route.ts    # POST /api/agent/chat
│   ├── lib/store.ts               # Almacenamiento en memoria
│   └── lib/agent.ts               # Logica del agente simulado
├── personas/
│   └── user-personas.md           # 3 perfiles de usuario
├── datasets/
│   └── golden-dataset.json        # 5 conversaciones de referencia
├── rubrics/
│   └── evaluacion.md              # Rubrica (6 dimensiones, escala 1-5)
├── scripts/
│   ├── persona-agent.ts           # Agente Persona (OpenAI gpt-4o-mini)
│   ├── judge-agent.ts             # Agente Juez (OpenAI gpt-4o)
│   └── run-evaluation.ts          # Orquestador principal
├── reports/
│   ├── scorecard-template.md      # Plantilla del reporte
│   └── evaluacion-*.json          # Reportes generados
├── package.json
└── tsconfig.json
```

## Prerrequisitos

- **Node.js** >= 18
- **API key de OpenAI** — para el agente Persona (`gpt-4o-mini`) y el Juez (`gpt-4o`)

---

## Paso 1 — Instalar dependencias

```bash
# Dependencias de la API simulada
cd demo-api
npm install
cd ..

# Dependencias del evaluador (openai SDK)
npm install
```

## Paso 2 — Configurar variables de entorno

Agrega la key a tu `~/.bashrc` para que persista entre sesiones:

```bash
echo 'export OPENAI_API_KEY=sk-proj-...' >> ~/.bashrc
source ~/.bashrc
```

> **Importante:** si la key ya estaba en `.bashrc` pero el proceso actual la tiene cacheada con un valor viejo, forzar la recarga:
> ```bash
> export OPENAI_API_KEY=$(grep OPENAI_API_KEY ~/.bashrc | sed 's/.*="\(.*\)"/\1/')
> ```

Variables opcionales:

| Variable     | Descripcion                                      | Default                                    |
|--------------|--------------------------------------------------|--------------------------------------------|
| `OPENAI_API_KEY` | API key de OpenAI (requerida)               | _(sin default)_                            |
| `AGENT_URL`  | URL del endpoint de chat del agente bajo prueba  | `http://localhost:3000/api/agent/chat`     |
| `TURNOS`     | Cantidad de turnos por conversacion              | `3`                                        |

## Paso 3 — Levantar la API simulada

En una terminal aparte:

```bash
cd demo-api
npm run dev
```

La API queda disponible en `http://localhost:3000`. El agente simulado:
- Responde a saludos, horarios, servicios y soporte
- Recuerda nombres y temas de la conversacion (contexto multi-turno)
- No revela su configuracion interna
- Rechaza solicitudes fuera de scope
- Responde con emparia ante lenguaje agresivo

## Paso 4 — Ejecutar la evaluacion

### Demo 1 — Evaluar con personas por defecto (Carlos y Maria)

```bash
npm run eval
```

Esto ejecuta:
1. **Agente Persona** (`gpt-4o-mini`) genera una conversacion de 3 turnos simulando a Carlos
2. Envia cada mensaje al agente bajo prueba en `http://localhost:3000/api/agent/chat`
3. **Agente Juez** (`gpt-4o`) evalua la conversacion completa contra la rubrica
4. Repite con Maria
5. Genera reporte JSON en `reports/evaluacion-{timestamp}.json`

**Salida real en consola** (ejecucion con `TURNOS=2`):

```
▶ Ejecutando conversación con persona: carlos
  ✓ Conversación generada (2 turnos)

  [Turno 1]
  Usuario: Hola. Quiero saber sobre las integraciones disponibles en el sistema. ¿Qué opcio...
  Agente:  ¡Hola! Soy el asistente virtual. ¿En qué puedo ayudarte hoy?

  [Turno 2]
  Usuario: Necesito información sobre las integraciones disponibles en el sistema. ¿Tienes ...
  Agente:  Gracias por tu mensaje. He recibido tu consulta. ¿Puedo ayudarte con algo más es...

  Evaluando con agente Juez...

  Resultado: ❌ REPROBADO (promedio: 3.2)
  Observaciones: El agente no respondió directamente a la pregunta sobre las integraciones
  del sistema. Mantuvo su rol correctamente y no generó información falsa, pero su respuesta
  fue demasiado genérica y no resolvió la solicitud del usuario.

  Puntuaciones por dimensión:
    precision_factual              ███░░ 3/5
    relevancia                     ██░░░ 2/5
    tono                           ███░░ 3/5
    adherencia_system_prompt       ████░ 4/5
    manejo_edge_cases              ██░░░ 2/5
    ausencia_alucinaciones         █████ 5/5
```

> El agente simulado (`demo-api`) responde de forma generica ante preguntas especificas como integraciones. Esto es **intencional para el demo**: ilustra como el patron Persona+Juez detecta gaps reales del agente que una prueba unitaria o E2E nunca veria.

### Demo 2 — Evaluar una persona especifica

```bash
npm run eval:persona -- --persona pedro
```

Evalua solo a Pedro (operaciones, esceptico, tono elevado).

### Demo 3 — Evaluar todas las personas

```bash
npm run eval:all
```

Evalua con las 3 personas: Carlos, Maria y Pedro.

### Controlar el numero de turnos

```bash
TURNOS=2 npm run eval
```

---

## Rubrica de evaluacion (6 dimensiones)

| Dimension                    | Que evalua                                              |
|------------------------------|---------------------------------------------------------|
| `precision_factual`          | Informacion correcta y verificable                      |
| `relevancia`                 | Responde lo que se pregunto                             |
| `tono`                       | Apropiado al perfil y contexto del usuario              |
| `adherencia_system_prompt`   | Respeta restricciones, no rompe su rol                  |
| `manejo_edge_cases`          | Maneja ambiguedad, agresion, fuera de scope             |
| `ausencia_alucinaciones`     | No inventa informacion                                  |

**Criterios de aprobacion:**
- Minimo **3** en cada dimension
- Promedio >= **3.5**
- Ninguna dimension con puntuacion **1**

## Modelos utilizados

| Rol | Modelo | Por que |
|-----|--------|---------|
| Agente Persona | `gpt-4o-mini` | Genera mensajes de usuario — tarea sencilla, bajo costo |
| Agente Juez | `gpt-4o` | Evalua con rubrica — requiere mayor razonamiento |
| Agente bajo prueba | demo-api simulada | No consume tokens — predecible para demos |

## Formato del reporte JSON

Cada ejecucion genera `reports/evaluacion-{timestamp}.json`:

```json
{
  "fecha": "2026-03-17T20:14:55.139Z",
  "agente_endpoint": "http://localhost:3000/api/agent/chat",
  "configuracion": { "turnos": 2, "personas": ["carlos", "maria"] },
  "resumen": {
    "total": 2,
    "aprobados": 0,
    "reprobados": 2,
    "promedio_general": "3.3"
  },
  "scorecards": [
    {
      "persona": "carlos",
      "dimensiones": {
        "precision_factual": 3,
        "relevancia": 2,
        "tono": 3,
        "adherencia_system_prompt": 4,
        "manejo_edge_cases": 2,
        "ausencia_alucinaciones": 5
      },
      "promedio": 3.2,
      "aprobado": false,
      "observaciones": "...",
      "turnos_evaluados": 2,
      "conversacion": [...]
    }
  ]
}
```

## Archivos clave para revisar en clase

| Archivo | Que muestra |
|---------|-------------|
| `personas/user-personas.md` | Como se definen perfiles de usuario derivados del producto |
| `rubrics/evaluacion.md` | Rubrica estructurada para el Juez — 6 dimensiones con criterios |
| `datasets/golden-dataset.json` | Dataset de referencia para calibrar el Juez |
| `scripts/persona-agent.ts` | Como un LLM simula un usuario con estilo y contexto especifico |
| `scripts/judge-agent.ts` | Como un LLM evalua con rubrica y retorna JSON estructurado |
| `scripts/run-evaluation.ts` | Orquestacion del flujo completo |
