---
name: kommo-n8n-prompt-testing
description: Use when testing an n8n AI agent that receives Kommo CRM webhooks and responds via salesbot. Covers multi-turn conversation testing, Supabase memory polling, webhook payload construction, reset protocol, and response validation. Use when debugging agent behavior, validating prompt changes, or running a regression battery before deploying a new prompt version.
---

# Kommo + n8n Prompt Testing

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

## n8n Workflow Update Safety

**CRITICAL:** Updating the workflow via PUT API clears all OAuth credential references from nodes that have `credentials: {}` in the API response. The GET endpoint omits credentials for security — the stored object differs from what is returned.

**Safe update procedure:**

```python
import json

# 1. Load the latest exported backup (contains real credential IDs)
with open('backup.json') as f:
    backup = json.load(f)

cred_map = {n['name']: n['credentials']
            for n in backup['nodes'] if n.get('credentials')}

# 2. Load current workflow (credentials will be {})
with open('current.json') as f:
    current = json.load(f)

# 3. Restore credentials before PUT
for node in current['nodes']:
    if node['name'] in cred_map:
        node['credentials'] = cred_map[node['name']]

# 4. PUT with credentials restored
```

Keep an exported backup in the repo. Re-export after any UI change that modifies credentials.

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
| PUT workflow without restoring credentials | Load backup, restore `credentials` map before any PUT |
| Sending next message before previous AI response | Wait for new Supabase row before each `send()` |

## Prompt Version Control

- Keep prompt versions as local `.md` files (e.g., `prompt-v2.9.md`)
- Prompt lives in a Google Doc fetched at runtime by the `Get a document` n8n node
- Update the Google Doc manually after validating the new version passes the full battery
- Do NOT include changelog headers in the Google Doc — they get injected into the system prompt
- Changelog belongs only in the local `.md` file

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
