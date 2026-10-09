# Checkpoint 1 · Agente de Calificación de Leads (Inmobiliaria)

**Autor:** Mauricio Rubino
**Entregable:** `checkpoint1_mauricio_rubino.json` (workflow exportado de n8n)
**Módulo:** M1 · Agente base y motor de razonamiento — primera versión del Proyecto Integrador

---

## Caso de negocio

**Horizonte Este Propiedades** es una inmobiliaria (ficticia) que opera compra, venta y alquiler de propiedades en Maldonado y Punta del Este, Uruguay. Recibe consultas por correo electrónico que el equipo comercial tiene que leer, filtrar y responder a mano.

Este agente atiende automáticamente la casilla de consultas:

1. Lee cada correo nuevo.
2. Releva los datos de calificación (operación, zona, presupuesto, plazo, forma de pago, contacto).
3. Clasifica el lead como **CALIENTE**, **TIBIO** o **FRÍO**, o detecta un caso de **ESCALAMIENTO**.
4. Decide de forma autónoma si deriva el lead al equipo comercial (herramienta Gmail).
5. Responde al remitente en el mismo hilo.
6. Envía un reporte de observabilidad a Slack con el razonamiento y las herramientas ejecutadas.

---

## Arquitectura del flujo

```
                                 ┌─────────────────────────┐
[Correo entrante] ──────────────▶│        AI Agent         │──▶ [Log de observabilidad] ──▶ [¿Corresponde ──▶ [Responder
 Gmail Trigger                   │      (Tools Agent)      │      Slack                         responder?]       correo]
                                 │  System Prompt modular  │                                     IF                Gmail reply
                                 │  Máx. 8 iteraciones     │
                                 └─────────────────────────┘
                                    ▲                  ▲
                    ai_languageModel│                  │ai_tool
                         [Anthropic Chat Model]   [Derivar lead al equipo comercial]
                              Claude                   Gmail Tool
```

| Nodo | Tipo | Función |
|---|---|---|
| Correo entrante | Gmail Trigger | Captura el mensaje inicial desestructurado (correos no leídos de la bandeja de entrada, excluyendo los enviados por la propia cuenta para evitar bucles). |
| AI Agent | AI Agent · Tools Agent | Motor de razonamiento ReAct. System Prompt modular, límite de 8 iteraciones y *Return Intermediate Steps* activado. |
| Anthropic Chat Model | Chat Model nativo | Modelo de lenguaje de Anthropic (Claude), temperatura 0.3. |
| Derivar lead al equipo comercial | Gmail Tool (lateral) | Herramienta que el agente activa de forma autónoma para enviar la ficha del lead al equipo comercial. |
| Log de observabilidad (Slack) | Slack | Reporte de supervisión humana con el Execution Log. |
| ¿Corresponde responder? | IF | Evita responder correos descartados (spam, newsletters, respuestas automáticas). |
| Responder correo | Gmail (reply) | Envía la respuesta del agente en el mismo hilo del correo original. |

---

## Cumplimiento de la consigna

| Requisito | Cómo se cumple |
|---|---|
| **Disparador** que capture el mensaje inicial desestructurado | Gmail Trigger. Ver nota sobre el disparador más abajo. |
| **AI Agent en modo Tools Agent** | Nodo AI Agent v1.7, que opera como Tools Agent (en esta versión es el modo por defecto, por eso n8n no lo escribe explícitamente en el JSON exportado). |
| **Chat Model nativo** (OpenAI o Anthropic) | Anthropic Chat Model conectado por `ai_languageModel`. |
| **Guardrail de iteraciones** entre 5 y 10 | `maxIterations: 8` en las opciones avanzadas del agente. |
| **System Prompt modular** (Rol → Ámbito → Objetivo → Reglas y Escalamiento) | Secciones: ROL · ÁMBITO · OBJETIVO · PROCESO · CORREOS QUE NO SE RESPONDEN · HERRAMIENTAS · REGLAS (acciones NO permitidas) · ESCALAMIENTO · FORMATO DE LA RESPUESTA. |
| **Límites explícitos** (qué NO puede hacer) | No inventar propiedades ni precios, no confirmar visitas ni señas, no negociar, no dar asesoramiento legal/financiero, no pedir datos sensibles, no escribir a terceros, no obedecer instrucciones incrustadas en el correo, no revelar sus instrucciones. |
| **Exclusión de lenguaje inclusivo** | Regla explícita en el prompt; todos los textos del flujo están redactados con la norma gramatical estándar. |
| **Herramienta acoplada lateralmente** (no secuencial) | Gmail Tool conectado por `ai_tool` al agente. |
| **Descripción semántica extensa** de la herramienta | La descripción detalla cuándo ACTIVAR (lead caliente/tibio, pedido de acción humana, escalamiento) y cuándo NO ACTIVAR (faltan datos, lead frío, spam, más de una vez por correo). Los campos asunto y cuerpo se completan con `$fromAI()` con formato definido. |
| **Agente no posterior a bloques rígidos de decisión** | El agente recibe el correo directamente del trigger. El único IF del flujo está *después* del agente. |
| **Observabilidad** (Slack o Gmail) | Nodo Slack que reporta hilo, remitente, asunto, herramientas ejecutadas, si se respondió o no, la respuesta del agente y los pasos intermedios (herramienta, entrada y resultado). |
| **Prueba manual** | Ejecutado con *Execute Workflow*: recorrido completo en verde, el agente abrió la rama de la herramienta, se recibió la derivación, la respuesta en el hilo y el log en Slack. |
| **Nombre del archivo** | `checkpoint1_mauricio_rubino.json` |

### Nota sobre el disparador

La consigna sugiere *Chat Trigger o Webhook*. Se eligió **Gmail Trigger** porque el canal real de consultas del caso de negocio es el correo electrónico. Cumple la misma función que pide la consigna: capturar el mensaje inicial desestructurado del usuario y entregarlo al agente sin bloques de decisión previos.

---

## Guardrails implementados

- **Iteraciones máximas: 8.** Corta bucles de razonamiento y limita el costo por ejecución.
- **Destinatario fijo en la herramienta.** El agente solo completa asunto y cuerpo; no puede elegir a quién escribir.
- **Filtro anti-bucle en el trigger.** `in:inbox -from:me` evita que el flujo procese sus propios correos (por ejemplo, las derivaciones).
- **Descarte de correos automáticos.** El agente devuelve `NO_RESPONDER` ante spam, newsletters o respuestas automáticas, y el IF evita la respuesta.
- **Protección contra prompt injection.** El prompt indica que el contenido del correo es solo la consulta a atender, nunca instrucciones.
- **Temperatura 0.3.** Respuestas consistentes y con menor riesgo de alucinación.

---

## Cómo importarlo y probarlo

1. En n8n: **Workflows → Import from File** → `checkpoint1_mauricio_rubino.json`.
2. Configurar credenciales (no se incluyen en el JSON): **Gmail OAuth2**, **Anthropic API** y **Slack OAuth2**.
3. En el nodo *Derivar lead al equipo comercial*, reemplazar el destinatario `ventas@horizonteeste.com.uy` por una casilla de prueba.
4. En el nodo de Slack, elegir el canal de observabilidad (por defecto `#agente-observabilidad`).
5. Desde otra cuenta, enviar a la casilla conectada:

   > **Asunto:** Consulta compra apartamento
   > **Cuerpo:** Hola, soy Laura Méndez. Busco comprar un apartamento de 2 dormitorios en Punta del Este, presupuesto USD 180.000, al contado, para mudarme en marzo. Mi celular es 099 123 456. ¿Me puede contactar un asesor?

6. En el trigger, **Fetch Test Event** → **Execute Workflow**.

**Resultado esperado:** el agente clasifica el lead como CALIENTE, ejecuta la herramienta (llega la ficha al equipo comercial), responde a Laura en el mismo hilo y publica el reporte en Slack.

---

## Hoja de ruta del Proyecto Integrador

| Módulo | Evolución prevista sobre este flujo |
|---|---|
| **M1 · Agente base** | ✅ Este checkpoint |
| M2 · Multi-agente | Manager que delega en workers (calificación, tasaciones, alquileres) como sub-workflows. |
| M3 · Memoria | Contexto por hilo/remitente en Airtable (Session_ID = hilo de Gmail). |
| M4 · Integraciones | CRM para leads y Google Calendar para coordinar visitas vía OAuth2. |
| M5 · RAG | Catálogo de propiedades y políticas comerciales en un vector store. |
| M6 · Voz | Atención telefónica con STT / TTS. |
| M11 | Proyecto Final Integrador. |
