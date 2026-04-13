# Claude Cowork — Guía de Seguridad Empresarial

> Guía completa para equipos IT y usuarios finales sobre el uso seguro de Claude Cowork en entornos empresariales.
> Preparada por Gustavo Marcos | Abril 2026

---

## Índice

1. [¿Qué es Claude Cowork y por qué es diferente?](#1-qué-es-claude-cowork-y-por-qué-es-diferente)
2. [¿A dónde van los datos?](#2-a-dónde-van-los-datos)
3. [El problema del PII sin sanitizar](#3-el-problema-del-pii-sin-sanitizar)
4. [Visibilidad y auditoría: con y sin SIEM](#4-visibilidad-y-auditoría-con-y-sin-siem)
5. [¿Puedo habilitar Cowork?](#5-puedo-habilitar-cowork)
6. [Mapa de controles de seguridad](#6-mapa-de-controles-de-seguridad)
7. [Evidencias a solicitar para auditoría (equipo IT)](#7-evidencias-a-solicitar-para-auditoría-equipo-it)
8. [Guía de uso seguro para usuarios](#8-guía-de-uso-seguro-para-usuarios)

---

## 1. ¿Qué es Claude Cowork y por qué es diferente?

Claude Cowork no es un chatbot. A diferencia del chat web, Cowork puede tomar acciones directas sobre el sistema.

```mermaid
flowchart LR
    subgraph CHATBOT["💬 Claude Chat (chatbot clásico)"]
        U1([Usuario]) -->|escribe texto| C1[Claude]
        C1 -->|responde texto| U1
    end

    subgraph COWORK["🤖 Claude Cowork (agente)"]
        U2([Usuario]) -->|da instrucción| C2[Claude Cowork]
        C2 --> FS[📁 Lee y escribe\narchivos locales]
        C2 --> WEB[🌐 Navega la web\ncon Chrome real]
        C2 --> MCP[🔌 Ejecuta MCP servers\nconexiones a BD, APIs]
        C2 --> SCHED[⏰ Tareas programadas\nsin supervisión]
        C2 --> CODE[⚙️ Ejecuta código\nen VM local]
    end

    style CHATBOT fill:#e8f5e9,stroke:#43a047
    style COWORK fill:#fff3e0,stroke:#fb8c00
```

> **Mensaje clave:** Cowork toma acciones reales en el sistema. El riesgo es radicalmente distinto al de un chatbot.

---

## 2. ¿A dónde van los datos?

```mermaid
flowchart TD
    USER([👤 Usuario\npega datos en Cowork])

    USER -->|prompt con datos| CLAUDE[🤖 Claude Cowork]

    CLAUDE -->|procesamiento| ANTHRO["☁️ Servidores Anthropic\n(US / Europa / Asia)\n⚠️ Salvo acuerdo ZDR"]
    CLAUDE -->|historial de sesión| JSONL["📄 Archivos .jsonl\n%APPDATA%\\Claude\\\n⚠️ Texto plano, sin cifrado"]
    CLAUDE -->|si OTel activo| OTEL["📡 OpenTelemetry\n→ SIEM"]
    CLAUDE -->|credenciales MCP| CONFIG["🔑 claude_desktop_config.json\n%APPDATA%\\Claude\\\n⚠️ API keys en texto plano"]

    OTEL -->|si OTEL_LOG_USER_PROMPTS=1| SIEM["🖥️ SIEM\n(prompts completos\nincluido PII)"]
    OTEL -->|si no hay SIEM| NOSIEM["❌ Sin visibilidad\ncentralizada"]

    JSONL -->|si BitLocker OFF| RIESGO1["🚨 Exposición ante\nacceso físico o malware"]
    CONFIG -->|si en repo| RIESGO2["🚨 Credenciales\nen GitHub"]

    style ANTHRO fill:#e3f2fd,stroke:#1e88e5
    style JSONL fill:#fff3e0,stroke:#fb8c00
    style CONFIG fill:#fff3e0,stroke:#fb8c00
    style RIESGO1 fill:#ffebee,stroke:#e53935
    style RIESGO2 fill:#ffebee,stroke:#e53935
    style NOSIEM fill:#ffebee,stroke:#e53935
    style SIEM fill:#e8f5e9,stroke:#43a047
```

**Puntos críticos:**

- El historial de conversaciones queda en cada endpoint en **texto plano** — no está cifrado por Cowork
- La Compliance API y los Audit Logs de Anthropic **excluyen Cowork completamente**
- Las credenciales de MCP servers se guardan en `claude_desktop_config.json` — nunca deben ir a un repositorio git

---

## 3. El problema del PII sin sanitizar

**PII** (*Personally Identifiable Information*) son datos que identifican a una persona: DNI, CUIL/CUIT, CBU/CVU, datos médicos, salarios, emails y nombres de clientes. En Argentina aplica la **Ley 25.326**.

**"PII sin sanitizar"** significa pasarle esos datos reales a Cowork sin anonimizarlos antes.

```mermaid
flowchart LR
    subgraph MAL["❌ Sin sanitizar — RIESGO"]
        direction TB
        D1["Usuario pega:\n'Juan Pérez\nDNI 30.123.456\nCBU 007012341234...\nSueldo $850.000'"]
        D1 --> C1[Claude procesa]
        C1 --> J1["📄 .jsonl guarda\ntodo en texto plano"]
        C1 --> A1["☁️ Anthropic recibe\ndatos reales"]
    end

    subgraph BIEN["✅ Sanitizado — CORRECTO"]
        direction TB
        D2["Usuario transforma primero:\n'Empleado_1\nDNI [REDACTED]\nCBU [REDACTED]\nSueldo [MONTO]'"]
        D2 --> C2[Claude procesa]
        C2 --> J2["📄 .jsonl guarda\ndatos anónimos"]
        C2 --> A2["☁️ Anthropic recibe\nestructura sin PII"]
    end

    style MAL fill:#ffebee,stroke:#e53935
    style BIEN fill:#e8f5e9,stroke:#43a047
```

> **Regla práctica:** Claude no necesita el valor real para hacer el trabajo — necesita la estructura. Reemplazar antes de pegar no cambia la calidad del resultado, pero elimina la exposición.

---

## 4. Visibilidad y auditoría: con y sin SIEM

```mermaid
flowchart TD
    START([¿Tienen OTel configurado?])

    START -->|Sí| OTEL_SI[OTel activo]
    START -->|No| OTEL_NO["❌ Sin visibilidad centralizada\nAuditoría imposible"]

    OTEL_SI --> SIEM_Q[¿Conectado a SIEM?]
    SIEM_Q -->|Sí| SIEM_SI["✅ Visibilidad parcial\n(Cowork excluido de\nCompliance API igual)"]
    SIEM_Q -->|No| SIEM_NO["⚠️ Logs locales solo\nsin alertas ni dashboards"]

    SIEM_SI --> PROMPTS_Q[¿OTEL_LOG_USER_PROMPTS=1?]
    PROMPTS_Q -->|Sí| PROM_SI["✅ Prompts completos logueados\n⚠️ PII también va al SIEM"]
    PROMPTS_Q -->|No| PROM_NO["⚠️ Solo metadatos\nno contenido de prompts"]

    OTEL_NO --> COMP[Controles compensatorios\nmínimos requeridos]
    COMP --> BL["🔒 BitLocker en endpoints"]
    COMP --> EDR["🛡️ EDR activo"]
    COMP --> POL["📋 Política de sanitizado\ndocumentada"]

    style OTEL_NO fill:#ffebee,stroke:#e53935
    style SIEM_NO fill:#fff3e0,stroke:#fb8c00
    style SIEM_SI fill:#e8f5e9,stroke:#43a047
    style PROM_SI fill:#e8f5e9,stroke:#43a047
    style PROM_NO fill:#fff3e0,stroke:#fb8c00
    style BL fill:#e3f2fd,stroke:#1e88e5
    style EDR fill:#e3f2fd,stroke:#1e88e5
    style POL fill:#e3f2fd,stroke:#1e88e5
```

**Sin SIEM:** la superficie de riesgo es equivalente pero la detección es nula. El foco se desplaza a controles de endpoint y prevención en origen.

---

## 5. ¿Puedo habilitar Cowork?

```mermaid
flowchart TD
    START([¿Habilitar Cowork?])

    START --> REG[¿Manejo datos regulados?\nHIPAA / PCI-DSS / SOX\nLey 25.326 Argentina]

    REG -->|Sí| ZDR[¿Tienen acuerdo ZDR\ncon Anthropic?]
    REG -->|No| PLAN[¿Qué plan tienen?]

    ZDR -->|No| LOCK["🔴 LOCKDOWN\nNo habilitar Cowork\nhasta tener ZDR +\ncontroles compensatorios"]
    ZDR -->|Sí| AUDIT[¿Pueden implementar\nOTel + SIEM?]

    AUDIT -->|No| LOCK2["🔴 LOCKDOWN\nSin auditoría no hay\ncumplimiento regulatorio"]
    AUDIT -->|Sí| CTRL["🟡 CONTROLLED\nCowork ON con\nguardrails estrictos"]

    PLAN -->|Pro o Max| LOCK3["🔴 LOCKDOWN\nSin controles admin\norganizacionales"]
    PLAN -->|Team| TEAM[¿Pueden restringir\nChrome y MCP?]
    PLAN -->|Enterprise| ENT["🟢 CONTROLLED u OPEN\nsegún madurez de seguridad"]

    TEAM -->|No| RISK["🟡 CONTROLLED mínimo\nRiesgo medio-alto\n(sin tenant restrictions)"]
    TEAM -->|Sí| RISK2["🟡 CONTROLLED\nHabilitar con guardrails"]

    style LOCK fill:#ffebee,stroke:#e53935
    style LOCK2 fill:#ffebee,stroke:#e53935
    style LOCK3 fill:#ffebee,stroke:#e53935
    style CTRL fill:#fff3e0,stroke:#fb8c00
    style RISK fill:#fff3e0,stroke:#fb8c00
    style RISK2 fill:#fff3e0,stroke:#fb8c00
    style ENT fill:#e8f5e9,stroke:#43a047
```

---

## 6. Mapa de controles de seguridad

```mermaid
flowchart LR
    subgraph PREV["🛡️ Prevención"]
        P1["Sanitizado de PII\nantes de usar Cowork"]
        P2["Chrome deshabilitado\no con allowlist"]
        P3["MCP servers\nallowlisteados"]
        P4["Instrucciones globales\nantiinyección"]
        P5["Tenant restrictions\n(Enterprise)"]
    end

    subgraph DET["👁️ Detección"]
        D1["OpenTelemetry\n→ SIEM"]
        D2["EDR con alerta en\n%APPDATA%\\Claude\\"]
        D3["Revisión semanal\nde tareas programadas"]
    end

    subgraph RESP["🚨 Respuesta"]
        R1["Kill-switch:\nCowork OFF desde admin"]
        R2["Playbook de IR\npara incidentes Cowork"]
        R3["Rotación de credenciales\nMCP servers"]
    end

    subgraph BASE["🏗️ Base"]
        B1["BitLocker en endpoints"]
        B2["SSO + SCIM"]
        B3["Política de clasificación\nde datos documentada"]
        B4["Training obligatorio\na usuarios"]
    end

    BASE --> PREV
    BASE --> DET
    PREV --> DET
    DET --> RESP
```

---

## 7. Evidencias a solicitar para auditoría (equipo IT)

> **Contexto crítico:** La Compliance API y los Audit Logs de Anthropic excluyen Cowork completamente. La única fuente de telemetría disponible es OpenTelemetry.

### 7.1 Archivos de configuración (por endpoint)

| Archivo | Ruta en Windows | Por qué importa |
|---|---|---|
| `settings.json` | `%USERPROFILE%\.claude\settings.json` | Configuración local; vector de ataque CVE-2025-59536 |
| `claude_desktop_config.json` | `%APPDATA%\Claude\claude_desktop_config.json` | Credenciales de MCP servers hardcodeadas |
| `*.jsonl` (historial) | `%APPDATA%\Claude\` | Todo lo que Claude leyó, escribió y ejecutó — texto plano |
| `.claude.json` | `%USERPROFILE%\.claude.json` | Repos marcados como "confiables" |
| `managed-mcp.json` | Desplegado por MDM/GPO | Inventario centralizado de MCP servers aprobados |
| `.mcp.json` | Por repo clonado | Puede contener hooks maliciosos |

### 7.2 Telemetría

**Con OTel + SIEM:**
- Variable `OTEL_EXPORTER_OTLP_ENDPOINT` configurada
- Variable `OTEL_LOG_USER_PROMPTS` — si = `1`, prompts completos se loguean (incluido PII)
- Dashboards y alertas activas en SIEM

**Sin SIEM — controles compensatorios mínimos:**

| Control | Cómo verificarlo | Sin esto el riesgo es |
|---|---|---|
| BitLocker activo | `manage-bde -status C:` | Historial `.jsonl` expuesto ante acceso físico |
| EDR instalado | Panel del agente EDR | Sin detección de acceso a `%APPDATA%\Claude\` |
| Directorio Claude en backup cifrado | Política de backup | Pérdida o exposición del historial |
| Cowork restringido a carpetas dedicadas | `settings.json` → campo `deny` | Acceso irrestricto al filesystem |
| `.gitignore` global con archivos Claude | `~/.gitconfig` → `core.excludesfile` | Credenciales pusheadas a repos |

### 7.3 Configuración del Admin Panel

- Toggle de Cowork: ON / OFF
- Chrome: habilitado / deshabilitado / allowlist de dominios
- Conectores activos: email, Slack, calendar, etc.
- Tenant restrictions (solo Enterprise): header `anthropic-allowed-org-ids`
- Instrucciones globales: system prompts defensivos

### 7.4 Inventario de tareas programadas

Las tareas corren sin supervisión — son la superficie de ataque más opaca.

- Lista de tareas activas por usuario
- Permisos: read-only vs. write/execute
- Historial de ejecuciones (OTel o `.jsonl` local)

### 7.5 Configuración de red

- Confirmar si `anthropic.com` está en allowlists del proxy — es canal de exfiltración potencial
- Dominios hardcodeados por Anthropic: `api.anthropic.com`, `pypi.org`, `github.com`, `npmjs.org`
- Categorías bloqueadas en browser (por defecto NO incluye: portales de salud, consolas cloud, SSO admin)

### 7.6 Brechas estructurales — lo que no podrás obtener

| Dato buscado | Disponibilidad | Motivo |
|---|---|---|
| Historial de conversaciones centralizado | **No disponible** | Local en cada endpoint |
| Cowork en Compliance API | **No disponible** | Excluido por Anthropic — sin fecha de cierre |
| Cowork en Data Exports | **No disponible** | Ídem |
| Granularidad por usuario en controles | **No disponible** | Controles son todos-o-nadie |
| Residencia de datos regional | Solo via Bedrock / Vertex / Azure | Por defecto: US/Europa/Asia |

### 7.7 Evaluación de riesgo por postura

| Postura detectada | Riesgo | Acción recomendada |
|---|---|---|
| Sin OTel ni SIEM | Crítico | Foco en controles de endpoint |
| PII sin sanitizar en historial `.jsonl` | Alto | Política de sanitizado + BitLocker |
| Sin BitLocker en endpoints con Cowork | Alto | Historial en texto plano expuesto |
| Chrome habilitado sin allowlist | Alto | Superficie de prompt injection amplia |
| Tareas programadas sin restricción | Alto | Ejecución no supervisada |
| Sin tenant restrictions (plan Team) | Medio-Alto | Cuentas personales acceden a datos org |
| MCP servers sin allowlist central | Alto | CVEs documentados; riesgo supply chain |
| Sin política de clasificación de datos | Alto | Usuarios no saben qué pasarle a Cowork |

### 7.8 Conclusión para industrias reguladas

Para organizaciones en **salud, finanzas o legal**: la brecha de auditoría es razón técnica suficiente para recomendar postura **Lockdown** (Cowork OFF) hasta que Anthropic cierre el gap en Compliance API.

Para clientes **sin SIEM y con datos regulados**: el mínimo aceptable antes de habilitar Cowork es **BitLocker + EDR + política de sanitizado documentada y comunicada a usuarios**.

---

## 8. Guía de uso seguro para usuarios

> Para empleados que usan Claude Cowork en su trabajo diario. No requiere conocimientos técnicos.

### Lo primero que tenés que saber

Claude Cowork no es como buscar en Google o usar un chatbot. Cuando le das una instrucción, puede:

- Abrir y modificar archivos de tu computadora
- Navegar sitios web en tu nombre
- Conectarse a sistemas internos de la empresa
- Ejecutar tareas mientras vos no estás mirando

Esto lo hace muy útil — pero también significa que **lo que le decís y lo que le mostrás importa más de lo que parece.**

---

### Las 5 reglas de uso

#### Regla 1 — Nunca le pases estos datos directamente

Antes de pegar cualquier cosa en Cowork, revisá que no contenga:

| Tipo de dato | Ejemplos concretos |
|---|---|
| Documentos de identidad | DNI, CUIL, CUIT, pasaporte |
| Datos bancarios | CBU, CVU, número de tarjeta, claves |
| Datos de salud | Diagnósticos, historiales clínicos, resultados de laboratorio |
| Contraseñas y claves | Contraseñas de sistemas, API keys, tokens |
| Información salarial individual | Recibos de sueldo con montos, contratos con cifras |
| Datos de clientes identificables | Nombre + teléfono + email en conjunto |

> Si el trabajo lo requiere, hay una forma de hacerlo igual — ver Regla 2.

#### Regla 2 — Sanitizá antes de pegar

"Sanitizar" significa reemplazar los datos sensibles por un placeholder antes de pegarlo en Cowork. Claude puede hacer el trabajo igual — no necesita el valor real, necesita la estructura.

**Ejemplo práctico:**

En vez de pegar esto:

```
Juan Pérez | DNI 30.123.456 | CUIL 20-30123456-7 | Sueldo $850.000
María López | DNI 27.890.123 | CUIL 27-27890123-4 | Sueldo $920.000
```

Pegás esto:

```
Empleado_1 | DNI [REDACTED] | CUIL [REDACTED] | Sueldo [MONTO_1]
Empleado_2 | DNI [REDACTED] | CUIL [REDACTED] | Sueldo [MONTO_2]
```

**Otra opción:** describir el dato en lugar de pegarlo.

> *"Tengo una planilla con 50 empleados que tiene columnas nombre, DNI, CUIL y sueldo. Quiero una fórmula para calcular..."*

#### Regla 3 — Controlá a qué carpetas le das acceso

Cuando Cowork te pide acceso a carpetas, dale acceso solo a lo que necesita para esa tarea. No le des acceso a:

- Tu carpeta de Documentos completa si solo necesita un archivo
- Carpetas con contratos, legajos o información de clientes si la tarea no los requiere
- La raíz del disco (`C:\`) — nunca es necesario

#### Regla 4 — No confirmes acciones que no entendés

Si Cowork te pide confirmar algo que no reconocés o no solicitaste:

1. No confirmes
2. Cancelá la sesión
3. Avisale a tu referente IT

Esto puede ser un caso de **prompt injection**: una instrucción oculta en un documento o página web que intentó redirigir a Claude.

> **Señal de alerta:** Cowork quiere hacer algo que no tiene nada que ver con lo que le pediste, o actúa "por su cuenta".

#### Regla 5 — Las tareas programadas necesitan más cuidado

Si configurás una tarea automática, tené en cuenta:

- Corre sola, sin que vos estés mirando
- Si Claude recibe instrucciones maliciosas vía un mail o documento, puede actuar sobre ellas sin que te enteres
- Revisá periódicamente qué tareas tenés activas

**Recomendación:** las tareas programadas solo deben leer información — no enviar mails, no modificar archivos, no publicar contenido.

---

### Qué podés usar sin preocuparte

| Tarea | Por qué es segura |
|---|---|
| Resumir un documento propio | No involucra datos de terceros ni sistemas externos |
| Redactar borradores de texto | Solo genera contenido, no accede a sistemas |
| Analizar una planilla con datos sanitizados | Los datos reales no salen de tu control |
| Investigar un tema en la web | Actividad acotada, sin acceso a sistemas internos |
| Formatear o reorganizar documentos | Tarea local, sin envío de datos |
| Generar código o fórmulas | No involucra datos sensibles |

---

### Señales de que algo no está bien

Prestá atención si Cowork:

- Quiere acceder a carpetas que no mencionaste
- Intenta enviar información fuera de la empresa sin que vos lo hayas pedido
- Hace preguntas raras o actúa de forma inconsistente
- Pide credenciales, contraseñas o accesos a sistemas
- Genera texto que parece de un tercero intentando darte instrucciones

En cualquiera de estos casos: **cerrá la sesión y avisale a IT.**

---

### Preguntas frecuentes

**¿Cowork guarda todo lo que le muestro?**
Sí. El historial queda guardado en tu computadora en texto plano. Si alguien accede a tu máquina, puede ver todo lo que hablaste con Cowork.

**¿Anthropic puede ver mis conversaciones?**
Sí, salvo que tu empresa tenga un acuerdo especial llamado ZDR. Si no sabés si lo tienen, consultá con IT antes de trabajar con información sensible.

**¿Puedo usarlo para procesar datos de clientes?**
Depende del tipo de dato. La regla general: si los datos están sujetos a la Ley 25.326 (datos personales en Argentina), sanitizá primero o consultá con IT.

**¿Qué hago si no sé si un dato es sensible o no?**
Si tenés duda, sanitizá. Reemplazar un valor por `[REDACTED]` no arruina el trabajo — evita un problema potencial.

**¿Puedo darle acceso a mi mail?**
Técnicamente sí si el conector está habilitado. Pero tu mail puede contener información de clientes, datos de salud, información legal. Si lo hacés, configurá la tarea para que solo lea, nunca envíe.

---

### Resumen rápido

```
LO QUE NUNCA HACÉS:
✗ Pegar DNI, CUIL, CBU, contraseñas, datos médicos o salariales sin sanitizar
✗ Darle acceso a carpetas completas cuando solo necesita un archivo
✗ Confirmar acciones que no reconocés o no pediste
✗ Dejar tareas programadas con permisos de escritura o envío

LO QUE SIEMPRE HACÉS:
✓ Reemplazar datos sensibles por [REDACTED] o [MONTO] antes de pegar
✓ Darle acceso solo a la carpeta del proyecto
✓ Revisar periódicamente las tareas programadas activas
✓ Avisar a IT ante cualquier comportamiento raro

ANTE LA DUDA:
→ Sanitizá primero
→ Consultá con IT
→ Cerrá la sesión si algo no cierra
```

---

*Gustavo Marcos | Abril 2026 | Basado en Harmonic Security — Securing Claude Cowork (Ed Merrett, marzo 2026)*
