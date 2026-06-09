# worldcup_bets_agent

Agente LangGraph que cada día a las 08:00 busca los partidos del Mundial 2026, genera un análisis de apuestas por partido y lo envía por email como TXT, con revisión humana antes del envío.

---

## Stack

| Componente | Uso |
|---|---|
| Azure OpenAI (gpt-4o-mini) | LLM del agente y del subgrafo |
| LangGraph | Orquestación del grafo principal y subgrafo |
| Tavily | Búsqueda web en tiempo real |
| ThreadPoolExecutor | Paralelización de tool calls y búsquedas de forma |
| Gmail SMTP | Envío del email |

---

## Flujo general

```
        08:00 (scheduler)
               │
               ▼
         ┌─────────────┐     - - - - - - - - - - - - - - -
         │  node_agent │ --> │ tool: get_matches_today    │
         │             │ <-- │ tool: get_next_matches     │
         └─────────────┘     - - - - - - - - - - - - - - -
               │ (sin tool_calls)
               ▼
       ┌──────────────────┐
       │ node_parse_fixtures│   extrae JSON con la lista de partidos
       └──────────────────┘
               │
        ¿hay partidos?
        /             \
      sí               no
      │                 │
      ▼                 ▼
┌──────────────┐    [no_fixtures]
│node_analyze  │         │
│  _matches    │        END
│              │
│ por cada partido invoca el SUBGRAFO:
│   [fetch_form] → [generate_analysis]
└──────────────┘
      │
      ▼
┌──────────────┐
│node_write    │   une todos los bloques en partidos.txt
│  _report     │
└──────────────┘
      │
      ▼
┌──────────────────┐
│ node_human_review│  <- PAUSA (interrupt_before)
│   (interrupt)    │     muestra partidos.txt por pantalla
└──────────────────┘
      │
 aprueba / rechaza
      │
      ▼
┌───────────┐
│ node_send │   envia email si approved=True
└───────────┘
      │
     END
```

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
| `EMAIL_PASSWORD` | Contraseña de aplicación Gmail (requiere verificación en 2 pasos) → https://myaccount.google.com/apppasswords |
| `EMAIL_TO` | Destinatario del email diario |

---

## Tools (celda 3)

### `get_matches_today`
Busca en Tavily los partidos del Mundial 2026 programados para hoy. Query con la fecha actual en inglés.

### `get_next_matches`
Lanza tres búsquedas en Tavily, una por cada uno de los próximos 3 días. Devuelve resultados agrupados por fecha con cabecera `=== YYYY-MM-DD ===`. Usa `max_results=8` por día.

### `get_team_form`
Recibe el nombre de un equipo y busca su estado de forma reciente: últimos resultados, jugadores clave y momento actual. La invoca el subgrafo en paralelo para local y visitante.

### `write_matches_txt`
Recibe el texto completo del análisis y el asunto. Escribe `partidos.txt` (sobreescribe si ya existe) y guarda el asunto en `email_subject.txt`.

### `send_email_with_file`
Lee `partidos.txt` y `email_subject.txt`, construye el mensaje con el análisis en el cuerpo y `partidos.txt` como adjunto, y lo envía via SMTP SSL. Siempre debe llamarse desde `node_send`, nunca directamente por el agente.

---

## Subgrafo — análisis por partido (celda 4)

El subgrafo es un grafo LangGraph independiente con su propio estado (`MatchState`) que no comparte nada con el grafo principal. El grafo principal lo invoca una vez por cada partido encontrado.

### Estado del subgrafo

```python
class MatchState(TypedDict):
    home:         str   # equipo local
    away:         str   # equipo visitante
    fixture_info: str   # fecha, hora, estadio
    home_form:    str   # resultado de get_team_form(home)
    away_form:    str   # resultado de get_team_form(away)
    analysis:     str   # bloque final generado por el LLM
```

### `subnode_fetch_form`
Lanza las dos llamadas a `get_team_form` (local y visitante) en paralelo con `ThreadPoolExecutor(max_workers=2)`. El tiempo total es el de la llamada más lenta, no la suma de ambas.

### `subnode_generate_analysis`
Con `home_form` y `away_form` ya rellenos en el estado, llama al LLM con un prompt estructurado que incluye la información de forma de ambos equipos. Devuelve el bloque de análisis de apuestas en texto plano.

### Flujo del subgrafo

```
[fetch_form] ──> [generate_analysis] ──> END
  (paralelo)       (LLM con contexto)
```

### Por qué un subgrafo y no una tool

Con una tool el LLM decide cuándo y cómo llamarla, lo que genera comportamiento impredecible (puede saltarse equipos, no respetar el orden, llamar a get_team_form de forma asíncrona con el análisis). Con el subgrafo el flujo es determinista: primero siempre se busca la forma de ambos equipos, luego siempre se genera el análisis. Es código, no instrucciones en un prompt.

---

## Nodos del grafo principal (celda 5)

### `node_agent`
Prepende el system prompt y llama al LLM con solo dos tools disponibles: `get_matches_today` y `get_next_matches`. El agente no tiene acceso a `get_team_form`, `write_matches_txt` ni `send_email_with_file` — esas las gestionan los nodos dedicados. Imprime `[nodo: agent]` en cada ejecución.

### `node_tools`
Ejecuta las tool calls del LLM en paralelo con `ThreadPoolExecutor`. Preserva el orden original de los resultados por índice. Si solo hay una tool call evita el overhead del pool y la ejecuta directamente.

### `node_parse_fixtures`
Extrae el JSON de la respuesta del LLM con regex y lo parsea. Rellena el campo `fixtures` del estado con la lista de partidos `[{home, away, fixture_info}]`. Si el parseo falla devuelve lista vacía.

### `node_analyze_matches`
Itera sobre `state["fixtures"]` e invoca el subgrafo para cada partido. Los subgrafos se ejecutan secuencialmente (uno por partido), pero dentro de cada subgrafo las dos búsquedas de forma van en paralelo. Acumula los bloques de análisis en `state["analyses"]`.

### `node_write_report`
Une todos los bloques de `state["analyses"]` con separadores, añade la cabecera con la fecha y llama a `write_matches_txt` para escribir el fichero. Genera el asunto dinámico según el número de partidos.

### `node_human_review`
Abre `partidos.txt` y lo imprime por pantalla. El grafo no ejecuta este nodo — el `interrupt_before=["human_review"]` lo congela antes de entrar, preservando todo el estado en `MemorySaver`.

### `node_send`
Lee `state["approved"]`. Si es `True` llama a `send_email_with_file`. Si es `False` cancela.

### `node_no_fixtures`
Fallback cuando `parse_fixtures` devuelve lista vacía. Imprime un aviso y termina en `END`.

---

## Funciones de routing

### `should_continue`
Después de `node_agent`: si el último mensaje tiene `tool_calls` va a `tools`, si no va a `parse_fixtures`.

### `has_fixtures`
Después de `node_parse_fixtures`: si `state["fixtures"]` no está vacío va a `analyze_matches`, si está vacío va a `no_fixtures`.

---

## Estado del grafo principal

```python
class AgentState(TypedDict):
    messages:  Annotated[list[BaseMessage], add_messages]
    fixtures:  list[dict]   # partidos extraidos por el agente
    analyses:  list[str]    # bloques generados por el subgrafo
    approved:  bool         # decision del humano
```

`messages` es acumulativo — `add_messages` hace que cada nodo añada al historial. El LLM ve todo el historial en cada vuelta.

`fixtures` y `analyses` son listas simples que se sobreescriben en cada nodo que las modifica.

`approved` se inyecta en la segunda invocación del grafo para que `node_send` sepa si enviar o cancelar.

---

## Paralelización

Hay dos niveles de paralelización:

**Nivel 1 — `node_tools`**: cuando el agente lanza múltiples tool calls en el mismo mensaje se ejecutan todas a la vez con `ThreadPoolExecutor`. Aplica a `get_matches_today` y `get_next_matches` si el LLM las lanza juntas.

**Nivel 2 — `subnode_fetch_form`**: dentro del subgrafo, las dos llamadas a `get_team_form` (local y visitante) se lanzan en paralelo con `ThreadPoolExecutor(max_workers=2)`. Con 6 partidos esto ahorra ~6 llamadas HTTP secuenciales (de 12 a 6 en términos de tiempo de espera).

---

## Human in the Loop (celdas 6 y 7)

Está entre `node_write_report` y `node_send`. El motivo es evitar enviar un análisis con datos incorrectos — el agente puede alucinar partidos o equipos.

`MemorySaver` serializa el estado completo del grafo identificado por `thread_id`. Cuando llega al `interrupt_before`, congela el grafo y devuelve el control al notebook.

La celda 7 reanuda con `graph.invoke({"approved": True/False}, config=config)`. LangGraph recupera el checkpoint por `thread_id`, inyecta `approved` y continúa desde `node_human_review`.

```
Celda 6                                  Celda 7
   │                                        │
graph.invoke(messages, fixtures=[], ...)    graph.invoke({"approved": True})
   │                                        │
[agent]→[tools]→[parse]→[analyze]→[write]  [human_review]→[send]→END
   │
PAUSA (interrupt_before human_review)
```

El `thread_id` en el scheduler cambia cada día (`mundial_YYYY-MM-DD`) para que cada ejecución tenga su propio checkpoint independiente.

---

## Salida esperada

**`partidos.txt`** — análisis completo, un bloque por partido, texto plano. Cabecera con fecha, separadores entre partidos, claves de apuesta estructuradas.

**Email** — asunto dinámico con equipos y fecha. Cuerpo = contenido del TXT. Adjunto = `partidos.txt`.

---

## Debug (celda 8)

Muestra el número de partidos y análisis generados, el estado de `approved`, y el historial completo de mensajes del grafo con tipo y preview de contenido.

---

## Scheduler (celda 9)

Usa `schedule` para ejecutar el agente cada día a las 08:00. El `thread_id` incluye la fecha del día para aislar cada ejecución. Para producción se recomienda sustituirlo por un cron del sistema o Azure Functions con timer trigger.
