---
name: kommo-n8n-prompt-testing
description: Use when testing an n8n AI agent that receives Kommo CRM webhooks and responds via salesbot. Covers multi-turn conversation testing, Supabase memory polling, webhook payload construction, reset protocol, and response validation. Use when debugging agent behavior, validating prompt changes, or running a regression battery before deploying a new prompt version.
---

# Kommo + n8n Prompt Testing

## Requirements

Everything that must exist before activating this skill or requesting tests.

### n8n Side

| Requisito | Detalle |
|-----------|---------|
| Workflow publicado | El workflow debe estar en estado **Active** y con el webhook en modo Production |
| Webhook URL accesible | `https://<host>/webhook/<path>` — debe responder 200 a un POST vacío |
| Nodo `Postgres Chat Memory` | Credencial PostgreSQL adjunta y conexión probada (`Connection tested successfully`) |
| Nodo `Get a document` | Credencial Google Docs adjunta; el Doc debe ser accesible por la cuenta |
| Credenciales OAuth Kommo | Nodo `Credenciales_Kommo` y todos los httpRequest que usan `oAuth2Api` con credencial activa |
| `If_Pipeline correcto` | El filtro debe incluir la etapa del lead de prueba (por defecto `102560443` IA Atendiendo; agregar `102627227` CONTACTO_INICIAL si se usa `/reset`) |
| LLM configurado | Nodo de AI Agent con modelo válido (Gemini, GPT, Claude) y credencial activa |
| Backup del workflow exportado | Archivo `.json` del workflow exportado desde n8n UI — necesario para restaurar credenciales después de cada PUT por API |

### Kommo CRM Side

| Requisito | Detalle |
|-----------|---------|
| Lead de prueba dedicado | Un lead real en el pipeline correcto; se usa para tests sin afectar leads de producción |
| `entity_id` del lead | ID numérico del lead (ej. `58392030`) — va como `session_id` en Supabase y como `entity_id` en el webhook |
| `contact_id` del lead | ID del contacto asociado al lead (ej. `99455352`) |
| Token de larga duración | JWT Kommo con scope `crm` + `files` + `notifications`, expira ~2030. Obtener en: Kommo → Configuración → Integraciones → token |
| Etapas del pipeline mapeadas | Conocer los `status_id` de las etapas: IA Atendiendo, Taquilla, Atención Manual, r2-3dias, etc. |
| Bot de salesbot activo | `bot_id` del salesbot que envía mensajes al cliente; debe estar publicado |
| Campo `Detener IA` limpio | Campo `field_id: 3161418` del lead de prueba debe estar vacío/false; si está activo, el workflow para antes del agente |

### Supabase Side

| Requisito | Detalle |
|-----------|---------|
| Tabla `n8n_chat_histories` | Debe existir con columnas: `id` (serial), `session_id` (text), `message` (jsonb o text) |
| Anon key con acceso a la tabla | RLS desactivado para la tabla o política que permita SELECT/DELETE con anon key |
| URL del proyecto | `https://<project_ref>.supabase.co` |

### Credenciales del Agente (Claude Code)

Archivo `.claude/secrets/<proyecto>.env` debe contener:

```bash
KOMMO_LONG_LIVED_TOKEN=eyJ...
KOMMO_DOMAIN=https://<subdomain>.kommo.com
SUPABASE_URL=https://<ref>.supabase.co
SUPABASE_ANON_KEY=eyJ...
N8N_WEBHOOK_URL=https://<host>/webhook/<path>
N8N_API_TOKEN_90D=eyJ...           # Para PUTs al workflow vía API
TEST_LEAD_ID=<entity_id>
TEST_CONTACT_ID=<contact_id>
TEST_CHAT_ID=test_chat_<lead_id>   # Puede ser cualquier string único
```

### Checklist de verificación rápida

Antes de activar el skill o pedir pruebas, confirmar:

```
[ ] Workflow en n8n está Active
[ ] Postgres Chat Memory tiene credencial adjunta y testeada
[ ] Lead de prueba existe en Kommo en el pipeline correcto
[ ] Tengo el entity_id y contact_id del lead de prueba
[ ] Tengo el token de larga duración de Kommo
[ ] Tengo la anon key de Supabase
[ ] La tabla n8n_chat_histories existe y permite SELECT/DELETE
[ ] Tengo el backup exportado del workflow (.json) para restaurar credenciales
[ ] El bot de salesbot está publicado y activo
```

Si algún ítem falta, el agente de Claude puede ayudar a obtenerlo — pero necesita saber cuáles credenciales ya están disponibles.

---

## Overview

Technique for end-to-end testing of n8n AI agents integrated with Kommo CRM. The agent receives webhook messages, processes them with an LLM, updates Kommo lead fields, and stores conversation history in Supabase. Tests simulate real WhatsApp conversations by sending webhook payloads and polling Supabase for AI responses.

## Architecture

```
Test script
  → POST /webhook/JeLo_v1 (n8n webhook)
    → n8n workflow (AI agent + Kommo API calls)
      → Supabase n8n_chat_histories (session_id = entity_id)
      → Kommo lead fields (status_id, custom_fields_values)
Test script
  ← polls Supabase until new AI row appears
  ← reads parte1/parte2/parte3/acciones from AI content
  ← validates etapa, notificar_equipo, campos guardados
```

## Required Credentials

Store in `.claude/secrets/<project>.env`:

```bash
KOMMO_LONG_LIVED_TOKEN=eyJ...          # Long-lived JWT, expires ~2030
SUPABASE_ANON_KEY=eyJ...               # Anon key for REST API
SUPABASE_URL=https://<ref>.supabase.co
N8N_WEBHOOK_URL=https://<host>/webhook/<path>
KOMMO_DOMAIN=https://<subdomain>.kommo.com
TEST_LEAD_ID=58392030                  # Dedicated test lead
TEST_CONTACT_ID=99455352
TEST_CHAT_ID=test_chat_<lead_id>
```

## Reset Protocol

**CRITICAL — run before every test.** Two steps, always both:

```bash
# 1. Clear Supabase chat memory for this lead
curl -s -X DELETE \
  "$SUPABASE_URL/rest/v1/n8n_chat_histories?session_id=eq.$TEST_LEAD_ID" \
  -H "apikey: $SUPABASE_ANON_KEY" \
  -H "Authorization: Bearer $SUPABASE_ANON_KEY"

# 2. Reset Kommo lead stage + clear AI custom fields
curl -s -X PATCH "$KOMMO_DOMAIN/api/v4/leads/$TEST_LEAD_ID" \
  -H "Authorization: Bearer $KOMMO_LONG_LIVED_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "status_id": 102560443,
    "pipeline_id": 13299663,
    "custom_fields_values": [
      {"field_id": 3184142, "values": [{"value": ""}]},
      {"field_id": 3184144, "values": [{"value": ""}]},
      {"field_id": 3161412, "values": [{"enum_id": 7949472}]},
      {"field_id": 3161552, "values": [{"value": ""}]},
      {"field_id": 3161420, "values": [{"value": ""}]}
    ]
  }'
```

**Never use `/reset` webhook text for reset** — it changes the lead to CONTACTO_INICIAL which may not be in the If_Pipeline filter, causing all subsequent messages to be silently dropped in < 1 second.

## Sending a Test Message

```bash
send_message() {
  local TEXT="$1"
  local MSG_ID="test_$(date +%s%N)"
  curl -s -X POST "$N8N_WEBHOOK_URL" \
    -H "Content-Type: application/json" \
    -d "{
      \"account[id]\": \"36179743\",
      \"account[subdomain]\": \"<subdomain>\",
      \"account[_links][self]\": \"$KOMMO_DOMAIN\",
      \"message[add][0][id]\": \"$MSG_ID\",
      \"message[add][0][entity_id]\": \"$TEST_LEAD_ID\",
      \"message[add][0][contact_id]\": \"$TEST_CONTACT_ID\",
      \"message[add][0][chat_id]\": \"$TEST_CHAT_ID\",
      \"message[add][0][entity_type]\": \"lead\",
      \"message[add][0][text]\": \"$TEXT\",
      \"message[add][0][type]\": \"incoming_chat_message\",
      \"message[add][0][created_by]\": \"0\",
      \"message[add][0][created_at]\": \"$(date +%s)\"
    }"
}
```

Webhook returns 200 immediately — processing happens asynchronously (~20s for AI turns).

## Waiting for AI Response

Poll Supabase until a new row appears for the session:

```python
import subprocess, json, time

def wait_for_new_row(prev_id, supabase_url, key, lead_id, timeout=90):
    """Returns new last_id when AI responds, or prev_id on timeout."""
    start = time.time()
    while time.time() - start < timeout:
        r = subprocess.run([
            "curl", "-s",
            f"{supabase_url}/rest/v1/n8n_chat_histories"
            f"?session_id=eq.{lead_id}&select=id&order=id.desc&limit=1",
            "-H", f"apikey: {key}", "-H", f"Authorization: Bearer {key}"
        ], capture_output=True, text=True)
        rows = json.loads(r.stdout)
        if rows and rows[0]['id'] > prev_id:
            return rows[0]['id']
        time.sleep(3)
    return prev_id

def get_last_ai_content(supabase_url, key, lead_id):
    r = subprocess.run([
        "curl", "-s",
        f"{supabase_url}/rest/v1/n8n_chat_histories"
        f"?session_id=eq.{lead_id}&order=id.asc",
        "-H", f"apikey: {key}", "-H", f"Authorization: Bearer {key}"
    ], capture_output=True, text=True)
    rows = json.loads(r.stdout)
    for row in reversed(rows):
        msg = row['message']
        if isinstance(msg, str): msg = json.loads(msg)
        if msg.get('type') == 'ai':
            return msg.get('content', '')
    return ''
```

**Key invariant:** `session_id` = `entity_id` of the lead (as string). AI and human rows alternate. Wait for new row, then read the latest AI row.

## Parsing AI Response

The AI returns JSON inside markdown backticks inside the Supabase `content` field:

```python
import re

def extract(content, field):
    m = re.search(rf'"{field}":\s*([^\n,}}]+)', content)
    return m.group(1).strip() if m else None

def extract_str(content, field):
    m = re.search(rf'"{field}":\s*"([^"\\]+(?:\\.[^"\\]*)*)', content)
    return m.group(1) if m else None

# Usage:
etapa     = extract(c, 'cambiar_etapa_id')   # e.g. "102560447"
notificar = extract(c, 'notificar_equipo')    # "true" or "false"
parte1    = extract_str(c, 'parte1')          # message to client
parte3    = extract_str(c, 'parte3')          # follow-up question
asientos  = extract_str(c, 'asientos_id_3161420')
correo    = extract(c, 'correo_contacto')
```

## Multi-Turn Test Pattern

```python
def run_test(name, turns):
    """
    turns = list of (message_text, assertion_fn)
    assertion_fn receives last AI content string, returns (pass: bool, detail: str)
    """
    reset()
    prev_id = 0
    results = []
    for text, assert_fn in turns:
        send(text)
        new_id = wait_for_new_row(prev_id, ...)
        if new_id == prev_id:
            results.append((False, "TIMEOUT — no AI response"))
            break
        prev_id = new_id
        content = get_last_ai_content(...)
        ok, detail = assert_fn(content)
        results.append((ok, detail))
    return all(r[0] for r in results), results
```

## Common Test Scenarios

| Test | Input | Expected |
|------|-------|----------|
| Primer turno | "Hola, quiero X boletos Zona Y función Z" | Link mapa en parte2, pregunta asientos |
| Asientos flexibles | "c 1 2 3 4" | Parsea C1,C2,C3,C4 — pregunta confirmación |
| Ambigüedad ×1 | Respuesta sin código de asiento | Pide aclaración SIN reenviar mapa |
| Ambigüedad ×2 | Segunda respuesta vaga | etapa=102560451 (Atención Manual) |
| Sin correo | "no tengo correo" | correo=correo_de_respaldo, avanza a método de pago |
| Flujo completo | Zona + asientos + nombre + correo + método | etapa=102560447, notificar=true, interes=ALTO |
| Indeciso | "necesito pensarlo" | etapa=102606731 (r2-3dias) |
| Queja/reembolso | "quiero devolución" | etapa=102560451, parte1 menciona derivar |
| Post-cierre duda | "¿cuándo llega el QR?" | etapa=null, responde sobre QR |
| Post-cierre cambio | "quiero cambiar asientos" | etapa=102560451 |

## Diagnosing Silent Failures (< 2s executions)

If n8n execution completes in < 2 seconds with no Supabase output:

1. **Lead en etapa no permitida** — Check `If_Pipeline correcto` filter. Status `102560443` (IA Atendiendo) must be in the array. `102627227` (CONTACTO_INICIAL) must also be included if `/reset` is used. Fix: add missing status_id to the array.

2. **`Detener IA` field activo** — Field `3161418` on the lead has a truthy value. Fix: PATCH to clear it.

3. **Mensaje duplicado** — Same `message[add][0][id]` sent twice. n8n deduplicates. Fix: always use a unique `id` per message.

4. **Webhook path incorrecto** — Check the URL path matches the published webhook in n8n.

## Backup Before Every Workflow Change

**Run this before touching the workflow — via API or UI.** The n8n export from the UI is the only source that includes real credential IDs.

```bash
# Descargar workflow completo con credenciales desde n8n UI export
# (En n8n: Editor → ··· → Download — NO desde la API, que omite credentials)
# Guardar en: n8n-<proyecto>/Respaldos n8n/<nombre>(fecha).json

# Si usas la API para leer el estado actual (credentials vendrán vacías):
N8N_API_TOKEN="eyJ..."
WORKFLOW_ID="OhOYFy2Za4hjppbv"
curl -s \
  -H "X-N8N-API-KEY: $N8N_API_TOKEN" \
  "https://<host>/api/v1/workflows/$WORKFLOW_ID" \
  > /tmp/workflow_current.json
# ⚠️ Este archivo tendrá credentials:{} — NO sirve como backup de credenciales
```

**Estructura de carpeta de respaldos:**
```
n8n-<proyecto>/
  Respaldos n8n/
    <NombreWorkflow>(Apr 30 at 18_00_00).json   ← exportado desde UI
    <NombreWorkflow>(May 2 at 20_04_20).json
    <NombreWorkflow>(May 3 at 09_15_00).json     ← siempre el más reciente
```

Nombrar con fecha y hora del export. El archivo más reciente es el backup activo.

## n8n Workflow Update Safety — PATCH por nodo, no PUT completo

**Regla de oro:** el endpoint `PUT /api/v1/workflows/{id}` reemplaza el workflow completo. Si el payload tiene `credentials: {}` en algún nodo (que es lo que devuelve el GET de la API), n8n borra esa credencial del nodo en la base de datos. **Esto rompe todos los nodos OAuth en producción.**

### Opción A — PATCH selectivo (preferida)

Modifica solo el campo específico del nodo sin tocar el resto del workflow:

```python
import json, subprocess, re

def patch_node_field(workflow_json_path, node_name, param_path, new_value,
                     api_token, host, workflow_id):
    """
    Carga el workflow del backup (con credentials reales),
    modifica SOLO el campo indicado en el nodo, y hace PUT preservando todo.
    
    param_path: lista de keys para navegar en parameters, ej. ['jsonBody']
                o ['conditions', 'conditions', 0, 'leftValue']
    """
    with open(workflow_json_path) as f:
        wf = json.load(f)

    # Modificar solo el nodo objetivo
    modified = False
    for node in wf.get('nodes', []):
        if node['name'] == node_name:
            target = node['parameters']
            for key in param_path[:-1]:
                target = target[key] if isinstance(key, str) else target[key]
            target[param_path[-1]] = new_value
            modified = True
            break

    if not modified:
        raise ValueError(f"Nodo '{node_name}' no encontrado")

    # PUT con credentials del backup (nunca del GET de la API)
    payload = {
        "name": wf["name"],
        "nodes": wf["nodes"],           # incluye credentials reales
        "connections": wf["connections"],
        "settings": wf.get("settings", {}),
        "staticData": wf.get("staticData"),
    }

    with open('/tmp/wf_patch.json', 'w') as f:
        json.dump(payload, f)

    r = subprocess.run([
        "curl", "-s", "-o", "/dev/null", "-w", "%{http_code}",
        "-X", "PUT",
        "-H", f"X-N8N-API-KEY: {api_token}",
        "-H", "Content-Type: application/json",
        "-d", f"@/tmp/wf_patch.json",
        f"https://{host}/api/v1/workflows/{workflow_id}"
    ], capture_output=True, text=True)

    return r.stdout.strip()  # "200" si OK
```

### Opción B — PUT con restauración de credenciales (cuando no hay backup reciente)

```python
import json

def safe_put_workflow(backup_path, current_api_json, api_token, host, workflow_id):
    """
    Usa el backup exportado desde UI (con credentials reales) como fuente de verdad
    para las credenciales. Aplica los cambios del current_api_json (sin credentials)
    sobre la estructura del backup.
    """
    with open(backup_path) as f:
        backup = json.load(f)

    # Mapa nombre → credentials del backup (fuente de verdad)
    cred_map = {n['name']: n['credentials']
                for n in backup['nodes'] if n.get('credentials')}

    # Restaurar credentials en el JSON modificado
    for node in current_api_json.get('nodes', []):
        if node['name'] in cred_map:
            node['credentials'] = cred_map[node['name']]
        # Si credentials sigue siendo {} después, ese nodo no tenía credential en backup
        # (normal para nodos sin autenticación)

    # PUT seguro
    # ...
```

### Flujo completo ante cada cambio al workflow

```
1. Exportar workflow desde n8n UI → guardar en Respaldos n8n/ con fecha
2. Editar el JSON del backup (tiene credentials reales)
3. Construir payload PUT: {name, nodes, connections, settings, staticData}
4. PUT a la API
5. Verificar HTTP 200
6. Hacer una ejecución de prueba para confirmar que las credenciales siguen activas
```

**Señales de credenciales rotas post-PUT:**
- Ejecuciones fallan en nodos httpRequest con error 401 o "credential not found"
- `Postgres Chat Memory` no guarda filas en Supabase
- `Get a document` no carga el prompt

## Supabase Direct Access

```bash
# List recent rows for a session
curl -s "$SUPABASE_URL/rest/v1/n8n_chat_histories?session_id=eq.$LEAD&order=id.asc" \
  -H "apikey: $SUPABASE_ANON_KEY" -H "Authorization: Bearer $SUPABASE_ANON_KEY"

# Clear session memory
curl -s -X DELETE \
  "$SUPABASE_URL/rest/v1/n8n_chat_histories?session_id=eq.$LEAD" \
  -H "apikey: $SUPABASE_ANON_KEY" -H "Authorization: Bearer $SUPABASE_ANON_KEY"
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `/reset` webhook text for reset | Use direct PATCH to Kommo + DELETE to Supabase |
| Reusing the same `message[add][0][id]` | Generate unique ID per message (timestamp-based) |
| Parsing AI content before new row appears | Always `wait_for_new_row` before reading content |
| Checking only `parte1` for pass/fail | Also check `cambiar_etapa_id`, `notificar_equipo`, and custom field values |
| Forgetting to reset Kommo stage | Prompts may behave differently depending on current stage |
| PUT workflow without restoring credentials | Load backup exportado desde UI, restaurar mapa de credentials antes de cada PUT |
| Usar GET de la API como backup de credenciales | El GET devuelve `credentials:{}` — solo el export de la UI tiene los IDs reales |
| Editar el workflow desde la UI sin exportar antes | Exportar siempre antes de cambiar cualquier cosa |
| Poner header de versión en el Google Doc | El header `# v2.9` se inyecta en el system prompt y cambia el comportamiento del agente |
| Sobreescribir el archivo de prompt activo | Crear siempre un archivo nuevo con número de versión mayor |
| Sending next message before previous AI response | Wait for new Supabase row before each `send()` |

## Prompt Version Control

### Estructura de carpeta

```
n8n-<proyecto>/
  PROMPT-n8n-<Nombre>/
    prompt-v2.6.md    ← versión histórica
    prompt-v2.7.md
    prompt-v2.8.md
    prompt-v2.9.md    ← versión activa en Google Doc
```

**Una archivo por versión, nunca sobreescribir.** Cada cambio — aunque sea de una línea — crea un archivo nuevo con número de versión mayor.

### Formato del archivo de versión

```markdown
# v2.9

> **Cambios vs v2.8:**
> - PASO 7: link del mapa solo se envía una vez (Paso 6B)
> - PASO 7: respuesta sin asientos → pide aclaración sin reenviar mapa (Intento 1)

> **Cambios vs v2.7 (heredados):**
> - Links de Google Maps en puntos de entrega

[contenido real del prompt desde aquí...]
```

El header de versión y changelog van SOLO en el archivo local `.md`. **No copiar al Google Doc** — se inyectan en el system prompt del agente y afectan el comportamiento.

### Flujo de cambio de prompt

```
1. Copiar el archivo activo → prompt-vX.Y+1.md
2. Aplicar cambios en el nuevo archivo
3. Actualizar el header de versión y changelog
4. Correr la batería de tests completa contra el Google Doc ACTUAL (sin cambios)
   → Confirmar que todos los tests previos siguen pasando (baseline)
5. Actualizar el Google Doc con el contenido del nuevo archivo
   (sin el header # vX.Y ni el bloque > Cambios vs...)
6. Correr la batería de tests completa de nuevo
   → Confirmar que los nuevos tests también pasan
7. Solo si pasa todo: el nuevo archivo queda como versión activa
```

### Relación Google Doc ↔ archivo local

| Google Doc | Archivo local `.md` |
|------------|---------------------|
| Solo el contenido del prompt (sin header ni changelog) | Todo: header, changelog + contenido |
| Actualizado manualmente después de validar | Versionado en el repo |
| Cargado en tiempo real por n8n en cada ejecución | Referencia histórica y punto de restauración |
| Si se corrompe → restaurar desde el `.md` activo | Fuente de verdad |

### Cuándo crear nueva versión

| Cambio | Acción |
|--------|--------|
| Nuevo comportamiento (nueva regla, nuevo paso) | Versión mayor: v2.9 → v2.10 |
| Fix de bug en regla existente | Versión menor: v2.9 → v2.9.1 |
| Corrección tipográfica sin impacto en comportamiento | Opcional: puede ser el mismo archivo con nota |
| Cambio que requiere nueva batería de tests | Siempre versión nueva |

## Full Battery Template

```python
BATTERY = [
    ("T1",    [("Hola quiero 2 boletos Bronce sábado 5pm", check_bienvenida)]),
    ("T2",    [("Diamante 4 boletos sáb 5pm", skip),
               ("c 1 2 3 4",                  check_confirms_c1234)]),
    ("T2b",   [("Diamante 2 boletos sáb 5pm", skip),
               ("los de en medio",             check_no_map_resent),
               ("los que queden libres",       check_atencion_manual)]),
    ("T4b",   [("Bronce 4 sáb 5pm",           skip),
               ("Juan Pérez",                  skip),
               ("no tengo correo",             check_correo_respaldo)]),
    ("T5",    [("Diamante 4 sáb 5pm",          skip),
               ("C1 C2 C3 C4",                 skip),
               ("sí exacto",                   skip),
               ("María González",              skip),
               ("maria@gmail.com",             skip),
               ("tarjeta de crédito",          check_taquilla_alto)]),
    ("T_ind", [("me interesa pero no sé",      skip),
               ("necesito pensarlo",           check_r2_3dias)]),
    ("T_queja", [("quiero reembolso",          check_atencion_manual)]),
    ("T_post1", [("¿cuándo llega el QR?",      check_no_etapa_change)]),
    ("T_post2", [("quiero cambiar asientos",   check_atencion_manual)]),
]

for name, turns in BATTERY:
    ok, results = run_test(name, turns)
    print(f"{'✅' if ok else '❌'} {name}")
```

---

## Live Telegram Audit

Optional feature: stream test progress to a Telegram chat (private or group) in real time. Sends 1 message per turn, 1 summary per test, and 1 summary per battery — exactly mirroring what the test runner is doing.

### When to enable

- New battery being run for first time → enable to watch real-time
- User explicitly approves a passed test and wants to inform stakeholders
- Regression battery before deploying a prompt change
- Asks to enable **once per battery run** (not silent default)

### Setup (one-time per project)

1. **Create Telegram bot** via `@BotFather` → save `BOT_TOKEN`.
2. **Get the chat_id**:
   - Private: send any message to the bot → `https://api.telegram.org/bot<TOKEN>/getUpdates` → take `chat.id`
   - Group: create group, add bot, send `/start@<bot_username>` → poll updates → take negative `chat.id`
3. **Store in env**:
   ```bash
   TELEGRAM_BOT_TOKEN=<token>
   TELEGRAM_AUDIT_CHAT_ID=<chat_id>
   ```
4. **Deploy n8n workflow** `cami-audit-telegram-push` (template: receives webhook → formats Markdown → posts to Telegram via Bot API).

### Webhook contract — `cami-audit-push`

Endpoint your test runner calls: `POST https://<n8n-host>/webhook/cami-audit-push`

Three event types, distinguished by `kind`:

#### `kind: "turn"` — fired after each user↔AI exchange
```json
{
  "kind": "turn",
  "battery_label": "v3.10 post-fix-concatena",
  "test_id": "T01",
  "turn_idx": 4,
  "turn_total": 5,
  "lead_name": "Fran de emprendeespina",
  "lead_url": "https://camialegre.kommo.com/leads/detail/39283259",
  "user_message": "fui a nutris pero me dan dietas genericas",
  "ai_response": "Te entiendo, Fran...",
  "latency_ms": 2100,
  "asserts": [
    {"name": "response_pregunta_count == 1", "passed": true},
    {"name": "no_concatena_dos_preguntas", "passed": true, "expected": "...", "actual": "..."}
  ],
  "crm_diff": {
    "stage": {"from": {"name": "LEAD NUEVO"}, "to": {"name": "iA Agente iG, Tik, Fb"}},
    "tags_added": ["objecion_nutricionistas"],
    "tags_removed": [],
    "fields_changed": [
      {"field": "objetivo", "from": "", "to": "personalizar plan"}
    ],
    "notes_added": [{"text": "Cliente rebota de nutris convencionales"}],
    "tasks_added": [{"text": "Seguimiento mañana", "complete_till": "2026-05-19T13:00:00Z"}]
  }
}
```

#### `kind: "test_summary"` — fired at end of each test
```json
{
  "kind": "test_summary",
  "battery_label": "v3.10 post-fix-concatena",
  "test_id": "T01",
  "description": "P4 no concatena dos preguntas",
  "passed_count": 5,
  "total_count": 5,
  "turns": [
    {"turn_idx": 1, "latency_ms": 1800, "asserts": [{"passed": true}, {"passed": true}]}
  ],
  "notes": "FIX VALIDADO"
}
```

#### `kind: "battery_summary"` — fired at end of battery
```json
{
  "kind": "battery_summary",
  "battery_label": "v3.10 post-fix-concatena",
  "tests_count": 4,
  "passed_asserts": 18,
  "total_asserts": 18,
  "duration_s": 47,
  "report_path": "tests/reports/2026-05-18_v3.10_post-fix-concatena.md",
  "tests": [
    {"test_id": "T01", "passed_count": 5, "total_count": 5}
  ]
}
```

### Trigger flow (per battery)

Before running the battery, **always ask the user**:
1. **chat_id** target (suggest last used, allow override)
2. **mode** (full report — default — or executive summary)
3. **label** of the battery (free text, ej. "v3.10 post-fix-concatena")

Then:
1. POST `{kind: "battery_start", ...}` → bot announces start in chat.
2. For each test, for each turn: POST `{kind: "turn", ...}`.
3. At end of test: POST `{kind: "test_summary", ...}`.
4. At end of battery: POST `{kind: "battery_summary", ...}` with `report_path` pointing to the generated `.md`.

### Folder convention

Reports persist in `<project>/tests/`:

```
<project>/tests/
├── batteries/                                      # YAML definitions
│   └── v3.10_post-fix-concatena.yaml
├── reports/                                        # one .md per battery run
│   ├── 2026-05-18_14-23_v3.10_post-fix-concatena.md   # human-readable
│   └── 2026-05-18_14-23_v3.10_post-fix-concatena.json # raw data
└── archive/                                        # compressed old reports
```

The `.md` report mirrors what was sent to Telegram, in long form (no message size limit), plus prompt diff, environment snapshot, and notes added by the user manually post-run.

### Battery YAML format

```yaml
battery_label: v3.10 post-fix-concatena
prompt_version: "3.10"
lead_id: 39283259
chat_id_target: -5244213309   # Telegram group
tests:
  - test_id: T01
    description: "P4 no concatena dos preguntas"
    turns:
      - user: "Holaa"
        asserts:
          - name: response_pregunta_count == 1
            kind: pregunta_count
            expected: 1
      - user: "peso 78, quiero 65"
        asserts:
          - name: response_pregunta_count == 1
            kind: pregunta_count
            expected: 1
          - name: tag_added contains "calificando"
            kind: tag_present
            expected: "calificando"
      - user: "fui a nutris, dietas genéricas, las dejo"
        asserts:
          - name: response_pregunta_count == 1
            kind: pregunta_count
            expected: 1
          - name: no_concatena_dos_preguntas
            kind: response_not_contains
            expected: "y te funcionó? no? porque"
```

### Built-in assertion kinds

| Kind | Description |
|------|-------------|
| `pregunta_count` | `expected` is N; count of `?` in AI response must equal N |
| `response_contains` | response must include the substring |
| `response_not_contains` | response must NOT include the substring |
| `tag_present` | lead must have this tag after the turn |
| `tag_absent` | lead must NOT have this tag |
| `stage_id` | lead status_id must equal expected |
| `field_value` | `field` key + `expected` value; checks current value |
| `note_added` | a note with this substring was created in this turn |
| `task_added` | a task with this substring was created |
| `latency_under_ms` | turn must complete under N ms |

### Security note

Never commit `BOT_TOKEN` to the repo. Always read from `~/.claude/secrets/<project>.env`. If a token appears in git history, immediately revoke via `@BotFather` → Bot Settings → API Token → Revoke.

