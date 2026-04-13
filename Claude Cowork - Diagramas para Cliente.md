---
title: "Claude Cowork — Diagramas para Presentación a Cliente"
type: guide
tags: [claude, cowork, seguridad, auditoria, diagrama, cliente, pii]
related: ["[[Guides/Claude Cowork - Evidencias a Solicitar para Auditoría]]", "[[Sources/Harmonic Security - Securing Claude Cowork]]"]
sources: ["Harmonic Security - Securing Claude Cowork.md", "Uso de Cowork en entornos empresariales.docx"]
updated: 2026-04-13
---

# Claude Cowork — Diagramas para Presentación a Cliente

---

## Diagrama 1 — ¿Qué hace Cowork que un chatbot no hace?

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

> **Mensaje clave:** Cowork no es un chatbot — toma acciones reales en el sistema. El riesgo es radicalmente distinto.

---

## Diagrama 2 — ¿A dónde van los datos que le paso a Cowork?

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

---

## Diagrama 3 — El problema del PII sin sanitizar

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

## Diagrama 4 — Visibilidad según configuración

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

---

## Diagrama 5 — ¿Puedo habilitar Cowork? Árbol de decisión

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

## Diagrama 6 — Mapa de controles por área

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

*Basado en: [[Sources/Harmonic Security - Securing Claude Cowork]] y documento interno "Uso de Cowork en entornos empresariales" (Gustavo Marcos, abril 2026)*
*Ver guía de auditoría completa: [[Guides/Claude Cowork - Evidencias a Solicitar para Auditoría]]*
