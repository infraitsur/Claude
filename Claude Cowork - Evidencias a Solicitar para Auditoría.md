---
title: "Claude Cowork — Evidencias a Solicitar para Auditoría Empresarial"
type: guide
tags: [claude, cowork, seguridad, auditoria, enterprise, opentelemetry, mcp, pii, sanitizado]
related: ["[[Sources/Harmonic Security - Securing Claude Cowork]]"]
sources: ["Harmonic Security - Securing Claude Cowork.md", "Uso de Cowork en entornos empresariales.docx"]
updated: 2026-04-13
---

# Claude Cowork — Evidencias a Solicitar para Auditoría Empresarial

Guía práctica para auditar el uso de Claude Cowork en entornos empresariales. Define qué solicitar al cliente, por qué, y qué brechas estructurales impiden visibilidad completa.

> **Contexto crítico:** La Compliance API y los Audit Logs de Anthropic **excluyen Cowork completamente**. La única fuente de telemetría disponible es OpenTelemetry. Sin OTel ni SIEM, la auditoría centralizada es imposible — pero hay riesgos igualmente verificables a nivel endpoint.

---

## 1. Archivos de configuración (por endpoint)

| Archivo | Ruta en Windows | Por qué importa |
|---|---|---|
| `settings.json` | `%USERPROFILE%\.claude\settings.json` | Configuración local de Claude; vector de ataque documentado (CVE-2025-59536: RCE via hooks maliciosos) |
| `claude_desktop_config.json` | `%APPDATA%\Claude\claude_desktop_config.json` | Contiene credenciales de MCP servers hardcodeadas en campo `env` — API keys, tokens, conexiones a BD |
| `*.jsonl` (historial) | `%APPDATA%\Claude\` | Todo lo que Claude leyó, escribió y ejecutó — texto plano, sin cifrado |
| `.claude.json` | `%USERPROFILE%\.claude.json` | Repos marcados como "confiables" — revela activos críticos |
| `managed-mcp.json` | Desplegado por MDM/GPO | Inventario centralizado de MCP servers aprobados |
| `.mcp.json` | Por repo clonado | Puede contener hooks maliciosos en repos externos |

---

## 2. Telemetría: con y sin SIEM

### Si tienen OTel + SIEM configurado
Solicitar:
- Endpoint de exportación: variable `OTEL_EXPORTER_OTLP_ENDPOINT`
- Variable `OTEL_LOG_USER_PROMPTS` — si = `1`, los prompts completos se loguean (incluido PII)
- Dashboards activos y alertas configuradas en SIEM
- Logs de ejecución de herramientas (comandos bash, paths, valores que pueden ser sensibles)

### Si NO tienen SIEM (caso frecuente en PyMEs)

La ausencia de SIEM no elimina el riesgo — lo hace **invisible**. Sin telemetría centralizada:

- No hay forma de saber qué datos procesó Cowork ni cuándo
- El historial de sesiones existe **solo en cada endpoint** en archivos `.jsonl` en texto plano
- Una brecha en el endpoint expone todo el historial sin posibilidad de detección temprana

**Qué verificar en ese escenario:**

| Control compensatorio | Cómo verificarlo | Sin esto el riesgo es |
|---|---|---|
| BitLocker activo en endpoint | `manage-bde -status C:` en PowerShell | El historial `.jsonl` queda expuesto ante acceso físico o malware |
| EDR instalado y activo | Panel del agente EDR (CrowdStrike, Defender for Endpoint, etc.) | Sin detección de acceso a `%APPDATA%\Claude\` por procesos externos |
| Directorio Claude incluido en backup cifrado | Política de backup — verificar scope | Pérdida de historial ante ransomware; o backup sin cifrar = exposición |
| Cowork restringido a carpetas dedicadas | `settings.json` → campo `deny` | Acceso irrestricto al filesystem del usuario |
| `.gitignore` global con archivos Claude | `~/.gitconfig` → `core.excludesfile` | Credenciales hardcodeadas en `claude_desktop_config.json` pusheadas a repos |

> **Conclusión para clientes sin SIEM:** la superficie de riesgo es equivalente, pero la detección es nula. El foco de la auditoría se desplaza a controles de endpoint (cifrado, EDR, permisos de carpeta) y a prevención en origen (sanitizado de PII antes de usar Cowork).

---

## 3. PII sin sanitizar — qué significa y cómo verificarlo

**PII** (*Personally Identifiable Information*) son datos que identifican a una persona: DNI, CUIL/CUIT, CBU/CVU, datos médicos, salarios, emails y nombres de clientes. En Argentina aplica la **Ley 25.326**.

**"PII sin sanitizar"** significa que el usuario le pasó esos datos reales a Cowork sin anonimizarlos antes. El problema: Claude no necesita los valores exactos para trabajar — puede operar con estructura y forma. Si el usuario pasa un listado de clientes con DNIs reales cuando solo necesitaba un análisis de formato, generó exposición innecesaria.

**Dónde queda registrada esa PII:**
1. Archivos `.jsonl` de historial en texto plano en la máquina del usuario
2. En los logs de OTel si `OTEL_LOG_USER_PROMPTS=1` (van al SIEM en crudo)
3. En servidores de Anthropic durante el procesamiento (salvo acuerdo ZDR activo)

**Cómo verificarlo en auditoría:**
- Revisar muestras de archivos `.jsonl` en `%APPDATA%\Claude\` buscando patrones de DNI, CUIL, CBU (regex: `\b\d{8}\b`, `\b\d{2}-\d{8}-\d{1}\b`, `\b\d{22}\b`)
- Preguntar al cliente si tienen política documentada de qué datos pueden pasarle a Cowork
- Verificar si existe un MCP Gateway de filtrado (Zscaler, Netskope, Palo Alto Prisma) con reglas de detección de PII

**Sanitizado correcto — ejemplo:**

| Sin sanitizar (riesgo) | Sanitizado (correcto) |
|---|---|
| "Juan Pérez, DNI 30.123.456, sueldo $850.000" | "Empleado_1, DNI [REDACTED], sueldo [MONTO]" |
| CBU 0070123412345678901234 | CBU [REDACTED] |
| Historial clínico con diagnóstico real | Registro médico con campos [DIAGNÓSTICO] |

---

## 4. Configuración del Admin Panel

Para planes Enterprise o Team:

- **Toggle de Cowork:** ON / OFF
- **Chrome:** habilitado / deshabilitado / allowlist de dominios
- **Conectores activos:** email, Slack, calendar, Google Drive, etc.
- **Marketplace de plugins:** lista de plugins aprobados
- **Tenant restrictions** (solo Enterprise): header `anthropic-allowed-org-ids` en proxy — bloquea cuentas personales
- **Instrucciones globales:** system prompts defensivos contra prompt injection

---

## 5. Inventario de tareas programadas

Las tareas programadas corren **sin supervisión humana** — son la superficie de ataque más opaca de Cowork.

Solicitar:
- Lista de tareas programadas activas (por usuario o global)
- Permisos de cada tarea: read-only vs. write/execute
- Historial de ejecuciones (en OTel si está configurado; en `.jsonl` local si no)

---

## 6. Configuración de red y egress

- Allowlist/blocklist de dominios en proxy o firewall
- Confirmar si `anthropic.com` está en allowlists — es canal de exfiltración potencial
- Logs de tráfico hacia dominios hardcodeados: `api.anthropic.com`, `pypi.org`, `github.com`, `npmjs.org`
- Categorías bloqueadas a nivel browser (por defecto: finanzas, crypto, adulto — **no incluye** portales de salud, consolas cloud, SSO admin, wikis internas)

---

## 7. Brechas estructurales — lo que no podrás obtener

| Dato buscado | Disponibilidad | Motivo |
|---|---|---|
| Historial de conversaciones centralizado | **No disponible** | Local en cada endpoint; fuera de políticas de Anthropic |
| Cowork en Compliance API | **No disponible** | Excluido por Anthropic — sin fecha de cierre pública |
| Cowork en Data Exports | **No disponible** | Ídem |
| Granularidad por usuario en controles admin | **No disponible** | Controles son todos-o-nadie para toda la organización |
| Garantías de residencia de datos regional | Solo via Bedrock / Vertex / Azure Foundry | Por defecto: US/Europa/Asia |

---

## 8. Evaluación de riesgo por postura

| Postura detectada | Nivel de riesgo | Acción recomendada |
|---|---|---|
| Sin OTel ni SIEM | Crítico | Auditoría centralizada imposible; foco en controles de endpoint |
| PII sin sanitizar en historial `.jsonl` | Alto | Política de sanitizado + revisar si BitLocker activo |
| Sin BitLocker en endpoints con Cowork | Alto | Historial en texto plano expuesto ante compromiso físico |
| Chrome habilitado sin allowlist | Alto | Superficie de prompt injection amplia |
| Tareas programadas sin restricción | Alto | Ejecución no supervisada, potencial loop de ataque |
| Sin tenant restrictions (plan Team) | Medio-Alto | Cuentas personales pueden acceder a datos org |
| MCP servers sin allowlist central | Alto | CVEs documentados; riesgo supply chain |
| Sin instrucciones globales defensivas | Medio | Sin mitigación de prompt injection |
| Sin política de clasificación de datos | Alto | Usuarios no saben qué pueden pasarle a Cowork |

---

## 9. Conclusión para industrias reguladas

Si el cliente opera en **salud, finanzas o legal**: la brecha de auditoría de Cowork es razón técnica suficiente para recomendar postura **Lockdown** (Cowork OFF) hasta que Anthropic cierre el gap en Compliance API. La ausencia de SIEM agrava esta recomendación — sin visibilidad centralizada, cualquier incidente de PII es indetectable hasta que hay daño concreto.

Para clientes **sin SIEM y con datos regulados**: el mínimo aceptable antes de habilitar Cowork es BitLocker + EDR + política de sanitizado documentada y comunicada a usuarios.

---

*Basado en: [[Sources/Harmonic Security - Securing Claude Cowork]] (Ed Merrett, marzo 2026) y documento interno "Uso de Cowork en entornos empresariales" (Gustavo Marcos, abril 2026)*
