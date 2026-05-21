# kommo-n8n-prompt-testing

Skill de Claude Code para probar agentes de IA construidos en **n8n** que reciben webhooks de **Kommo CRM** y responden vía salesbot. Está diseñado para correr baterías de regresión multi-turno contra agentes de ventas tipo "Cami" que combinan Kommo + ManyChat + Supabase como memoria.

## ¿Qué hace?

Cuando se activa, el skill le da al agente un protocolo completo para:

- Construir payloads de webhook de Kommo y dispararlos contra n8n.
- Esperar y validar la respuesta del AI agent leyendo la memoria de chat en **Supabase** (`Postgres Chat Memory`).
- Ejecutar conversaciones **multi-turno** con reset entre escenarios (`/reset` → etapa CONTACTO_INICIAL).
- Parsear el contenido del AI (`mensaje`, `notif`, campos calculados) y validar contra expectativas.
- Hacer **backup obligatorio** del workflow antes de cada cambio (`GET → cp a Respaldos n8n/ → PATCH/PUT`).
- Aplicar cambios al workflow con **PATCH por nodo** en lugar de PUT completo, para no romper credenciales.
- Restaurar credenciales después de un PUT cuando es inevitable.
- (Opcional) Hacer **streaming en vivo** del audit a un canal de Telegram para revisión humana.
- Diagnosticar fallas silenciosas (`executions < 2s`, filtros de pipeline mal configurados, credenciales caídas).
- Mantener control de versiones de prompts (`prompt-vN.N.md`) con changelog.

## ¿Cuándo se activa?

El skill se invoca automáticamente cuando el usuario dice frases como:

- "correr batería" / "probar prompt v3.10" / "regression test Cami"
- "validar fix" / "live audit" / "audit live Telegram"
- "test runner Kommo" / "probar el agente"

## Requisitos previos

Antes de usar el skill debe existir:

**n8n**
- Workflow `Active` con webhook en modo Production.
- Nodos `Postgres Chat Memory`, `Get a document` y `Credenciales_Kommo` con credenciales válidas.
- `If_Pipeline` que incluya la(s) etapa(s) del lead de prueba.
- LLM configurado (Gemini / GPT / Claude) con credencial activa.
- Backup `.json` del workflow exportado desde la UI.

**Kommo CRM**
- Lead de prueba con `lead_id` conocido.
- Pipeline y etapas mapeadas (ej. `102560443` IA Atendiendo, `102627227` CONTACTO_INICIAL).
- Token OAuth válido.

**Supabase**
- Tabla de memoria de chat accesible vía REST con `service_role` key.
- Columna `session_id` correspondiente al `lead_id`.

**Claude Code**
- Credenciales en `.env` o variables de sesión: `KOMMO_TOKEN`, `N8N_API_TOKEN`, `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, `N8N_HOST`, `WEBHOOK_URL`.
- (Opcional) `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` para audit en vivo.

Ver `SKILL.md` para la checklist completa de verificación.

## Estructura

```
kommo-n8n-prompt-testing/
└── SKILL.md     # Protocolo completo del skill (723 líneas)
```

Todo el conocimiento operativo vive en `SKILL.md` con secciones para:

1. Requirements & Architecture
2. Reset Protocol
3. Sending a Test Message
4. Waiting for / Parsing AI Response
5. Multi-Turn Test Pattern
6. Diagnosing Silent Failures
7. Backup Before Every Workflow Change
8. n8n Workflow Update Safety (PATCH vs PUT)
9. Supabase Direct Access
10. Common Mistakes
11. Prompt Version Control

## Instalación

Clonar el repo dentro del directorio de skills de Claude Code:

```bash
cd ~/.claude/skills
git clone https://github.com/integralmarketingmx/kommo-n8n-prompt-testing.git
```

Claude Code detecta el skill automáticamente al iniciar la siguiente sesión.

## Uso

Una vez instalado, basta con pedirle al agente algo como:

```
corre la batería de regresión del prompt v3.12 contra el lead de prueba
```

```
valida el fix del nodo FormatDirector con audit en vivo a Telegram
```

El skill se activará, leerá `SKILL.md` y ejecutará el protocolo.

## Convenciones del proyecto donde nació

Este skill se desarrolló en el contexto del proyecto **JesusEsLaOnda — Venta de Boletos** (n8n + Kommo + ManyChat + Supabase). Algunas convenciones del SKILL.md asumen ese setup:

- Carpeta `Respaldos n8n/` para backups manuales del workflow.
- Carpeta `PROMPT-n8n-Jesuseslaonda/` con `prompt-vN.N.md` versionados.
- `notificar_equipo` como nodo de notificaciones a Diana (operadora humana).
- Etapas de pipeline `IA Atendiendo` y `CONTACTO_INICIAL`.

Si lo adaptas a otro proyecto, revisa esas referencias.

## Licencia

Uso interno de Integral Marketing MX.
