# worldcup_bets_agent

Agente LangGraph que busca los partidos del Mundial 2026, genera un análisis de apuestas por partido y lo envía por email como TXT.

---

## Flujo general
 
```
             start
               │
               ▼
         ┌─────────────┐        - - - - - - - - - - - -
         │   AI AGENT  │ - - -> │ tool: get_matches    │
         │             │ <----- │ tool: get_next       │
         └─────────────┘        │ tool: get_team_form  │
               │                 - - - - - - - - - - - -
               ▼
          ¿hay partidos?
          /            \
        sí              no (skip)
        │                    │
        ▼                    ▼
  ┌───────────┐           [END]
  │escribir   │
  │   TXT     │
  └───────────┘
        │
        ▼
  ┌───────────────┐
  │generar fichero│
  │ + enviar email│
  └───────────────┘
        │
        ▼
      [END]
```

El grafo arranca en `agent`, el LLM decide qué tool llamar, `tools` la ejecuta y devuelve el resultado, y el ciclo se repite hasta que el LLM termina sin llamar a ninguna tool.

---

## Claves necesarias (celda 2)

| Variable | Dónde conseguirla |
|---|---|
| `AZURE_OPENAI_ENDPOINT` | Foundry → Overview → endpoint |
| `AZURE_OPENAI_API_KEY` | Foundry → Overview → Keys → Key 1 |
| `AZURE_OPENAI_DEPLOYMENT` | Foundry → Models + Endpoints → nombre del deployment |
| `AZURE_OPENAI_API_VERSION` | Usar `2024-02-15-preview` |
| `TAVILY_API_KEY` | https://app.tavily.com → API Keys |
| `EMAIL_FROM` | Tu cuenta Gmail |
| `EMAIL_PASSWORD` | Contraseña de aplicación Gmail (requiere verificación en 2 pasos activada) → https://myaccount.google.com/apppasswords |
| `EMAIL_TO` | Destinatario del email diario |

---

## Tools (celda 3)

### `get_matches_today`
Busca en Tavily los partidos del Mundial 2026 programados para hoy. Lanza una query con la fecha actual en inglés. Si no hay partidos, el agente pasa al flujo de próximos partidos.

### `get_next_matches`
Lanza tres búsquedas en Tavily, una por cada uno de los próximos 3 días. Devuelve los resultados agrupados por fecha con cabecera `=== YYYY-MM-DD ===`. Usa `max_results=8` por día para no perder partidos.

### `get_team_form`
Recibe el nombre de un equipo y busca su estado de forma reciente: últimos resultados, jugadores clave y momento actual. El agente la llama dos veces por partido, una para cada equipo.

### `write_matches_txt`
Recibe el texto completo del análisis y el asunto del email. Escribe `partidos.txt` (sobreescribe si ya existe) y guarda el asunto en `email_subject.txt`. Devuelve confirmación con el número de caracteres escritos.

### `send_email_with_file`
Lee `partidos.txt` y `email_subject.txt`, construye el mensaje con el análisis en el cuerpo y `partidos.txt` como adjunto, y lo envía via SMTP SSL a Gmail. Incluye manejo explícito de error de autenticación. Siempre debe llamarse después de `write_matches_txt`.

---

## Nodos del grafo (celda 4)

### `node_agent`
Prepende el system prompt a los mensajes del estado y llama al LLM con las tools vinculadas (`llm.bind_tools`). Devuelve el mensaje del LLM al estado. Imprime `[nodo: agent]` en cada ejecución.

### `node_tools`
Lee los `tool_calls` del último mensaje del LLM, ejecuta cada tool contra `TOOLS_BY_NAME` e imprime `[nodo: tools] → nombre(args)` por cada llamada. Devuelve los `ToolMessage` al estado para que el LLM los vea en la siguiente vuelta.

### `should_continue`
Función de routing. Si el último mensaje del LLM tiene `tool_calls` devuelve `"tools"`, si no devuelve `END`. Es el único punto de decisión del grafo.

---

## Estado del grafo

```python
class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
```

Lista acumulativa de mensajes. `add_messages` hace que cada nodo añada al historial en lugar de sobreescribirlo. El LLM ve todo el historial en cada vuelta.

---

## System prompt

Define el flujo exacto que debe seguir el agente:

- Si hay partidos hoy → analiza → escribe → envía
- Si no hay partidos → busca próximos 3 días → analiza → escribe → envía

Formato de salida: texto plano con bloque por partido. Cada bloque incluye contexto del partido y cinco claves de apuesta (1X2, más/menos 2.5, ambos marcan, handicap, apuesta recomendada). Horas siempre en hora española CEST (UTC+2).

---

## Salida esperada

**`partidos.txt`** — análisis completo, un bloque por partido, texto plano.

**Email** — asunto dinámico con los equipos y la fecha. Cuerpo = contenido del TXT. Adjunto = `partidos.txt`.

---
