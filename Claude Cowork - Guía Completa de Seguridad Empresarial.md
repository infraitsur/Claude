---
title: "Claude Cowork — Guía Completa de Seguridad Empresarial"
type: guide
tags: [claude, cowork, seguridad, auditoria, enterprise, pii, eset, gpo, active-directory, usuario-final]
related: ["[[Sources/Harmonic Security - Securing Claude Cowork]]"]
sources: ["Harmonic Security - Securing Claude Cowork.md", "Uso de Cowork en entornos empresariales.docx"]
updated: 2026-04-13
---

# Claude Cowork — Guía de Seguridad Empresarial

> Guía completa para equipos IT y usuarios finales sobre el uso seguro de Claude Cowork en entornos empresariales.
> Preparada por Gustavo Marcos | Abril 2026

**Entorno de referencia:**
- Endpoints: Windows 10/11
- Directorio: Active Directory local (on-premise)
- Plan Claude: Pro (cuentas personales de usuarios)
- Despliegue: no gestionado — usuarios instalaron Claude por su cuenta
- Protección endpoint: ESET PROTECT Entry cloud (Endpoint & Server Security)
- Telemetría centralizada: ninguna (sin OTel, sin SIEM)

---

## Índice

1. [¿Qué es Claude Cowork y por qué es diferente?](#1-qué-es-claude-cowork-y-por-qué-es-diferente)
2. [¿A dónde van los datos?](#2-a-dónde-van-los-datos)
3. [El problema del PII sin sanitizar](#3-el-problema-del-pii-sin-sanitizar)
4. [Estado actual del entorno — diagnóstico](#4-estado-actual-del-entorno--diagnóstico)
5. [Visibilidad real con ESET — qué cubre y qué no](#5-visibilidad-real-con-eset--qué-cubre-y-qué-no)
6. [Controles disponibles en este entorno](#6-controles-disponibles-en-este-entorno)
7. [Hoja de ruta para recuperar control](#7-hoja-de-ruta-para-recuperar-control)
8. [Evidencias a relevar en auditoría (equipo IT)](#8-evidencias-a-relevar-en-auditoría-equipo-it)
9. [Guía de uso seguro para usuarios](#9-guía-de-uso-seguro-para-usuarios)

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

    USER -->|prompt con datos| CLAUDE[🤖 Claude Cowork\ncuenta Pro personal]

    CLAUDE -->|procesamiento| ANTHRO["☁️ Servidores Anthropic\n(US / Europa / Asia)\n⚠️ Sin ZDR en plan Pro\nAntropic puede usar datos\npara entrenamiento*"]
    CLAUDE -->|historial de sesión| JSONL["📄 Archivos .jsonl\n%APPDATA%\\Claude\\\n⚠️ Texto plano, sin cifrado"]
    CLAUDE -->|credenciales MCP| CONFIG["🔑 claude_desktop_config.json\n%APPDATA%\\Claude\\\n⚠️ API keys en texto plano"]

    JSONL -->|ESET puede detectar\nacceso malicioso| ESET["🛡️ ESET PROTECT\ndetecta malware que\nintenta leer estos archivos"]
    JSONL -->|si BitLocker OFF| RIESGO1["🚨 Exposición ante\nacceso físico o robo"]
    CONFIG -->|si en repo git| RIESGO2["🚨 Credenciales\nen GitHub"]
    ANTHRO -->|plan Pro| RIESGO3["⚠️ Sin opt-out automático\nde entrenamiento\n(debe activarse manualmente)"]

    style ANTHRO fill:#e3f2fd,stroke:#1e88e5
    style JSONL fill:#fff3e0,stroke:#fb8c00
    style CONFIG fill:#fff3e0,stroke:#fb8c00
    style ESET fill:#e8f5e9,stroke:#43a047
    style RIESGO1 fill:#ffebee,stroke:#e53935
    style RIESGO2 fill:#ffebee,stroke:#e53935
    style RIESGO3 fill:#fff3e0,stroke:#fb8c00
```

**\* Plan Pro — dato importante:** a diferencia de los planes Team y Enterprise, en Pro el opt-out del uso de conversaciones para entrenamiento **no es automático**. Cada usuario debe desactivarlo manualmente en su configuración de cuenta en claude.ai.

---

## 3. El problema del PII sin sanitizar

**PII** (*Personally Identifiable Information*) son datos que identifican a una persona: DNI, CUIL/CUIT, CBU/CVU, datos médicos, salarios, emails y nombres de clientes. En Argentina aplica la **Ley 25.326**.

**"PII sin sanitizar"** significa pasarle esos datos reales a Cowork sin anonimizarlos antes. ESET no puede detectar ni prevenir este tipo de exposición — ocurre dentro de una sesión legítima del usuario.

```mermaid
flowchart LR
    subgraph MAL["❌ Sin sanitizar — RIESGO"]
        direction TB
        D1["Usuario pega:\n'Juan Pérez\nDNI 30.123.456\nCBU 007012341234...\nSueldo $850.000'"]
        D1 --> C1[Claude procesa]
        C1 --> J1["📄 .jsonl guarda\ntodo en texto plano"]
        C1 --> A1["☁️ Anthropic recibe\ndatos reales\n⚠️ posible uso en entrenamiento\nsi opt-out no está activo"]
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

## 4. Estado actual del entorno — diagnóstico

```mermaid
flowchart TD
    subgraph ACTUAL["Estado actual — despliegue no gestionado"]
        A1["❌ Sin admin panel organizacional\n(plan Pro no lo incluye)"]
        A2["❌ Sin toggle de Cowork\npor parte de IT"]
        A3["❌ Sin tenant restrictions\n(requiere Enterprise)"]
        A4["❌ Sin instrucciones globales\ndefensivas"]
        A5["❌ Sin MCP allowlist\ncentralizado"]
        A6["❌ Sin opt-out automático\nde entrenamiento"]
        A7["❌ Sin telemetría\n(OTel / SIEM)"]
        A8["⚠️ Usuarios con cuentas Pro\npersonales — IT sin visibilidad\nni control sobre las cuentas"]
    end

    subgraph DISPONIBLE["Lo que IT SÍ puede hacer hoy"]
        B1["✅ GPO para restringir\ninstalación o ejecución\nde Claude Desktop"]
        B2["✅ ESET HIPS para alertar\nsobre acceso a archivos Cowork"]
        B3["✅ ESET Web Control\npara filtrar dominios"]
        B4["✅ BitLocker para cifrar\nhistorial local"]
        B5["✅ GPO para bloquear\n%APPDATA%\\Claude\\\npor procesos no autorizados"]
        B6["✅ Política interna documentada\ny comunicada a usuarios"]
    end

    style ACTUAL fill:#ffebee,stroke:#e53935
    style DISPONIBLE fill:#e8f5e9,stroke:#43a047
```

**Conclusión del diagnóstico:** este es el escenario de mayor exposición posible para Cowork. La organización no tiene ningún control sobre cómo los usuarios usan la herramienta, qué datos le pasan, ni qué hace con ellos. La única palanca de control inmediata es GPO + ESET + política de uso comunicada a los usuarios.

---

## 5. Visibilidad real con ESET — qué cubre y qué no

```mermaid
flowchart TD
    subgraph CUBRE["✅ Lo que ESET SÍ cubre"]
        E1["Malware que intenta leer\n%APPDATA%\\Claude\\ (HIPS)"]
        E2["Acceso web a dominios\nmaliciosos conocidos"]
        E3["Archivos maliciosos\ndescargados o ejecutados"]
        E4["Alertas en consola ESET PROTECT\npor comportamiento anómalo"]
        E5["Protección del endpoint\ncontra ransomware"]
    end

    subgraph NOCUBRE["❌ Lo que ESET NO cubre"]
        N1["Qué datos le pasó el usuario\na Cowork en sus prompts"]
        N2["Qué archivos leyó o modificó\nCowork en sesiones legítimas"]
        N3["PII enviada a servidores\nde Anthropic"]
        N4["Prompt injection desde\ndocumentos o web"]
        N5["Tareas programadas\nejecutándose sin supervisión"]
        N6["Actividad de Cowork\nregistrada o auditable"]
    end

    style CUBRE fill:#e8f5e9,stroke:#43a047
    style NOCUBRE fill:#ffebee,stroke:#e53935
```

### Qué configurar en ESET para Cowork

| Configuración | Dónde | Qué hace |
|---|---|---|
| Regla HIPS: alerta si proceso externo accede a `%APPDATA%\Claude\` | ESET PROTECT cloud → Políticas → HIPS | Detecta malware intentando robar historial |
| Web Control: bloquear categorías de riesgo | ESET PROTECT cloud → Políticas → Web Control | Limita navegación de Chrome de Cowork |
| Device Control: bloquear USB | ESET PROTECT cloud → Políticas → Device Control | Previene exfiltración física del historial |
| Reporte semanal de amenazas por endpoint | ESET PROTECT cloud → Informes | Visibilidad mínima de actividad |

---

## 6. Controles disponibles en este entorno

Con AD local, Windows 10/11 y plan Pro, los controles posibles son los siguientes:

### Via GPO (Active Directory)

| Control | Mecanismo GPO | Efecto |
|---|---|---|
| Bloquear instalación de Claude Desktop | Software Restriction Policies o AppLocker | Impide nuevas instalaciones en equipos del dominio |
| Restringir ejecución de `claude.exe` | AppLocker → Executable Rules | Solo usuarios autorizados pueden correr Cowork |
| Proteger `%APPDATA%\Claude\` de acceso externo | ACLs via GPO Preferences | Endurece el acceso al historial local |
| Forzar BitLocker en todos los endpoints | `Computer Configuration → Administrative Templates → BitLocker` | Cifra el historial local automáticamente |

### Via ESET PROTECT cloud

| Control | Configuración | Efecto |
|---|---|---|
| HIPS: alerta en `%APPDATA%\Claude\` | Regla personalizada en política HIPS | Detecta acceso malicioso al historial |
| Web Control | Bloquear categorías: finanzas, salud, cloud consoles | Reduce superficie de prompt injection |
| Device Control | Bloquear USB en equipos con Cowork | Previene exfiltración física del historial |

### Via política interna

- Política de uso aceptable documentada y firmada por usuarios
- Checklist de sanitizado de PII antes de usar Cowork
- Procedimiento de reporte ante comportamiento anómalo
- Verificación manual periódica de tareas programadas activas

---

## 7. Hoja de ruta para recuperar control

### Inmediato — sin cambios de infraestructura

- [ ] Comunicar a usuarios la política de uso aceptable y las reglas de sanitizado
- [ ] Verificar que BitLocker esté activo en todos los endpoints con Cowork (`manage-bde -status C:`)
- [ ] Crear regla HIPS en ESET PROTECT cloud para `%APPDATA%\Claude\`
- [ ] Solicitar a cada usuario activar el opt-out de entrenamiento (claude.ai → Settings → Privacy)
- [ ] Relevar qué usuarios tienen Cowork instalado y qué conectores/MCP tienen activos

### Corto plazo — con GPO

- [ ] Implementar AppLocker para controlar qué usuarios pueden ejecutar Claude Desktop
- [ ] Forzar BitLocker via GPO en todos los endpoints del dominio
- [ ] Aplicar Web Control en ESET para limitar dominios accesibles desde Cowork

### Mediano plazo — decisión estratégica

- [ ] Evaluar migración a plan **Team o Enterprise**
  - Team agrega: toggle de Cowork, opt-out automático de entrenamiento
  - Enterprise agrega: tenant restrictions, SAML/SCIM, Chrome admin, red egress controlada
- [ ] Si se mantiene Pro: formalizar que Cowork **no está aprobado para datos regulados**

---

## 8. Evidencias a relevar en auditoría (equipo IT)

> **Contexto:** no hay políticas de Cowork desplegadas. El objetivo es relevar el estado real de cada endpoint y establecer una línea de base.

### 8.1 Relevamiento por endpoint

```powershell
# Verificar si Claude Desktop está instalado
Get-ItemProperty "HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*" |
  Where-Object DisplayName -like "*Claude*" | Select-Object DisplayName, DisplayVersion

# Verificar si BitLocker está activo
manage-bde -status C:

# Listar archivos de configuración presentes
Get-ChildItem "$env:APPDATA\Claude\" -ErrorAction SilentlyContinue
Get-ChildItem "$env:USERPROFILE\.claude\" -ErrorAction SilentlyContinue

# Ver tamaño del historial (indica nivel de uso)
(Get-ChildItem "$env:APPDATA\Claude\*.jsonl" -ErrorAction SilentlyContinue |
  Measure-Object -Property Length -Sum).Sum / 1MB
```

### 8.2 Archivos críticos a inspeccionar

| Archivo | Ruta | Qué verificar |
|---|---|---|
| `claude_desktop_config.json` | `%APPDATA%\Claude\` | Que no tenga credenciales hardcodeadas en campo `env` |
| `settings.json` | `%USERPROFILE%\.claude\` | Si tiene campo `deny` con rutas sensibles |
| `*.jsonl` (historial) | `%APPDATA%\Claude\` | Presencia de PII con búsqueda de patrones |
| `.claude.json` | `%USERPROFILE%\` | Repos marcados como confiables |

**Búsqueda de PII en historial:**

```powershell
Select-String -Path "$env:APPDATA\Claude\*.jsonl" `
  -Pattern "\b\d{7,8}\b|\b\d{2}-\d{7,8}-\d{1}\b|\b\d{22}\b" |
  Select-Object -First 20
```

### 8.3 Estado de ESET por endpoint

Verificar desde consola ESET PROTECT cloud:

- Endpoints con detecciones sin resolver
- Versión de firmas actualizada
- Política HIPS aplicada correctamente
- Web Control activo

### 8.4 Opt-out de entrenamiento

En plan Pro debe verificarse cuenta por cuenta. Pedirle a cada usuario:

> claude.ai → Settings → Privacy → **"Improve Claude for everyone"** → desactivar

### 8.5 Brechas que no pueden cerrarse sin cambiar de plan

| Control necesario | Plan mínimo | Situación actual |
|---|---|---|
| Toggle de Cowork (ON/OFF) para toda la org | Team | No disponible en Pro |
| Opt-out automático de entrenamiento | Team | Manual por usuario en Pro |
| Tenant restrictions | Enterprise | No disponible en Pro |
| Admin de Chrome / dominios | Enterprise | No disponible en Pro |
| SAML / SSO / SCIM | Enterprise | No disponible en Pro |
| Instrucciones globales defensivas | Enterprise | No disponible en Pro |

### 8.6 Evaluación de riesgo

| Hallazgo | Riesgo | Acción recomendada |
|---|---|---|
| BitLocker inactivo en endpoints con Cowork | Crítico | Activar de inmediato via GPO |
| Opt-out de entrenamiento no activo | Alto | Solicitar activación a cada usuario |
| Credenciales en `claude_desktop_config.json` | Alto | Rotar y eliminar hardcoding |
| PII encontrada en archivos `.jsonl` | Alto | Implementar política de sanitizado |
| Claude instalado sin inventario IT | Alto | Relevar todos los endpoints del dominio |
| Tareas programadas activas sin supervisión | Alto | Auditar y restringir a solo lectura |
| Sin regla HIPS en ESET para `%APPDATA%\Claude\` | Medio | Crear regla en ESET PROTECT cloud |
| Sin política de uso documentada | Alto | Redactar, aprobar y comunicar |

### 8.7 Conclusión de auditoría

El despliegue actual es **shadow IT no gestionado con plan Pro**. Acciones mínimas antes de aceptar el riesgo residual:

1. **BitLocker activo** en todos los endpoints con Cowork
2. **Opt-out de entrenamiento** activado en todas las cuentas Pro
3. **Política de uso documentada y comunicada** formalmente
4. **Regla HIPS en ESET** para `%APPDATA%\Claude\`
5. **Inventario completo** de endpoints con Claude instalado y conectores activos
6. **Decisión formal** sobre datos regulados — recomendación: no aprobar hasta tener plan Team o Enterprise

---

## 9. Guía de uso seguro para usuarios

> Para empleados que usan Claude Cowork en su trabajo diario. No requiere conocimientos técnicos.

### Lo primero que tenés que saber

Claude Cowork no es como buscar en Google o usar un chatbot. Cuando le das una instrucción, puede:

- Abrir y modificar archivos de tu computadora
- Navegar sitios web en tu nombre
- Conectarse a sistemas internos de la empresa
- Ejecutar tareas mientras vos no estás mirando

Esto lo hace muy útil — pero también significa que **lo que le decís y lo que le mostrás importa más de lo que parece.**

> Usás Cowork con una cuenta personal Pro. El área IT no tiene visibilidad sobre lo que hacés en Cowork. La responsabilidad del uso seguro recae directamente en vos.

---

### Antes de empezar — activá el opt-out de entrenamiento

En el plan Pro, Anthropic puede usar tus conversaciones para entrenar sus modelos futuros **salvo que lo desactives manualmente**:

1. Entrá a claude.ai
2. Configuración → Privacy
3. Desactivar **"Improve Claude for everyone"**

---

### Las 5 reglas de uso

#### Regla 1 — Nunca le pases estos datos directamente

| Tipo de dato | Ejemplos concretos |
|---|---|
| Documentos de identidad | DNI, CUIL, CUIT, pasaporte |
| Datos bancarios | CBU, CVU, número de tarjeta, claves |
| Datos de salud | Diagnósticos, historiales clínicos, resultados de laboratorio |
| Contraseñas y claves | Contraseñas de sistemas, API keys, tokens |
| Información salarial individual | Recibos de sueldo con montos, contratos con cifras |
| Datos de clientes identificables | Nombre + teléfono + email en conjunto |

#### Regla 2 — Sanitizá antes de pegar

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

O describí el dato en lugar de pegarlo:

> *"Tengo una planilla con 50 empleados con columnas nombre, DNI, CUIL y sueldo. Quiero una fórmula para calcular..."*

#### Regla 3 — Controlá a qué carpetas le das acceso

Dale acceso solo a la carpeta específica del proyecto. Nunca a:
- Tu carpeta de Documentos completa
- Carpetas con contratos, legajos o información de clientes
- La raíz del disco (`C:\`)

#### Regla 4 — No confirmes acciones que no entendés

Si Cowork te pide confirmar algo que no reconocés:
1. No confirmes
2. Cancelá la sesión
3. Avisale a IT

> **Señal de alerta:** Cowork quiere hacer algo que no tiene nada que ver con lo que le pediste.

#### Regla 5 — Las tareas programadas necesitan más cuidado

- Corren solas, sin que vos estés mirando
- El antivirus no detecta instrucciones maliciosas dentro de una tarea
- Revisalas periódicamente

**Recomendación:** solo lectura — no enviar mails, no modificar archivos, no publicar contenido.

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

Prestá atención si Cowork quiere acceder a carpetas que no mencionaste, intenta enviar información fuera de la empresa, pide credenciales o actúa de forma inconsistente con lo que le pediste.

En cualquiera de estos casos: **cerrá la sesión y avisale a IT.**

---

### Preguntas frecuentes

**¿El antivirus me protege de lo que hago en Cowork?**
Parcialmente. ESET detecta malware y bloquea sitios maliciosos, pero no puede ver qué datos le pasás a Claude en una sesión normal. La prevención depende de vos.

**¿Anthropic puede usar mis conversaciones para entrenar IA?**
Sí, en plan Pro, salvo que hayas desactivado el opt-out. Es el primer paso antes de usar Cowork con información de trabajo.

**¿Cowork guarda todo lo que le muestro?**
Sí, en texto plano en `%APPDATA%\Claude\`. Si alguien accede a tu máquina, puede ver todo el historial.

**¿IT puede ver lo que hago en Cowork?**
No directamente. IT puede ver actividad de red y alertas de antivirus, pero no el contenido de tus conversaciones.

**¿Puedo procesar datos de clientes?**
Solo con datos sanitizados. Si aplica Ley 25.326, sanitizá primero o consultá con IT.

**¿Qué hago si no sé si un dato es sensible?**
Sanitizá. Reemplazar por `[REDACTED]` no arruina el trabajo — evita un problema potencial.

---

### Resumen rápido

```
ANTES DE EMPEZAR:
→ Activar opt-out en claude.ai → Settings → Privacy → desactivar "Improve Claude for everyone"

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
