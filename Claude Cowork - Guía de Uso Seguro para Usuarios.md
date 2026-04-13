---
title: "Claude Cowork — Guía de Uso Seguro para Usuarios"
type: guide
tags: [claude, cowork, usuario-final, pii, seguridad, buenas-practicas]
related: ["[[Guides/Claude Cowork - Evidencias a Solicitar para Auditoría]]", "[[Guides/Claude Cowork - Diagramas para Cliente]]"]
sources: ["Harmonic Security - Securing Claude Cowork.md", "Uso de Cowork en entornos empresariales.docx"]
updated: 2026-04-13
---

# Claude Cowork — Guía de Uso Seguro para Usuarios

**Para:** Empleados y usuarios que utilizan Claude Cowork en su trabajo diario
**No requiere conocimientos técnicos**

---

## Lo primero que tenés que saber

Claude Cowork no es como buscar en Google o usar un chatbot. Cuando le das una instrucción, puede:

- Abrir y modificar archivos de tu computadora
- Navegar sitios web en tu nombre
- Conectarse a sistemas internos de la empresa
- Ejecutar tareas mientras vos no estás mirando

Esto lo hace muy útil — pero también significa que **lo que le decís y lo que le mostrás importa más de lo que parece.**

---

## Las 5 reglas de uso

### Regla 1 — Nunca le pases estos datos directamente

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

---

### Regla 2 — Sanitizá antes de pegar

"Sanitizar" significa reemplazar los datos sensibles por un placeholder antes de pegarlo en Cowork. Claude puede hacer el trabajo igual — no necesita el valor real, necesita la estructura.

**Ejemplo práctico:**

Tenés que analizar un listado de empleados. En vez de pegar esto:

```
Juan Pérez | DNI 30.123.456 | CUIL 20-30123456-7 | Sueldo $850.000
María López | DNI 27.890.123 | CUIL 27-27890123-4 | Sueldo $920.000
```

Pegás esto:

```
Empleado_1 | DNI [REDACTED] | CUIL [REDACTED] | Sueldo [MONTO_1]
Empleado_2 | DNI [REDACTED] | CUIL [REDACTED] | Sueldo [MONTO_2]
```

Claude puede analizar el formato, detectar errores, generar fórmulas o hacer lo que necesitás — sin que los datos reales salgan de tu control.

**Otra opción:** describir el dato en lugar de pegarlo.

> "Tengo una planilla con 50 empleados que tiene columnas nombre, DNI, CUIL y sueldo. Quiero una fórmula para calcular..."

---

### Regla 3 — Controlá a qué carpetas le das acceso

Cuando Cowork te pide acceso a carpetas, dale acceso solo a lo que necesita para esa tarea. No le des acceso a:

- Tu carpeta de Documentos completa si solo necesita un archivo
- Carpetas con contratos, legajos o información de clientes si la tarea no los requiere
- La raíz del disco (C:\) — nunca es necesario

**Cómo hacerlo:** cuando Cowork te pide confirmar el scope de trabajo, elegí la carpeta específica del proyecto, no la carpeta padre.

---

### Regla 4 — No confirmes acciones que no entendés

Cowork a veces te pedirá confirmar antes de hacer algo: modificar un archivo, enviar un mail, acceder a un sistema. Si el pedido de confirmación dice algo que no reconocés o no solicitaste:

1. No confirmes
2. Cancelá la sesión
3. Avisale a tu referente IT

Esto puede ser un caso de **prompt injection**: una instrucción oculta en un documento o página web que intentó secuestrar a Claude para que haga algo que vos no pediste.

> **Señal de alerta:** Cowork quiere hacer algo que no tiene nada que ver con lo que le pediste, o actúa "por su cuenta".

---

### Regla 5 — Las tareas programadas necesitan más cuidado

Si configurás una tarea para que Cowork la haga automáticamente (por ejemplo, todas las mañanas generar un resumen de emails), tené en cuenta:

- Esa tarea corre sola, sin que vos estés mirando
- Si en algún momento Claude recibe instrucciones maliciosas (vía un mail o documento envenenado), puede actuar sobre ellas sin que te enteres
- Revisá periódicamente qué tareas programadas tenés activas y si siguen haciendo lo que esperás

**Recomendación:** que las tareas programadas solo lean información — no envíen mails, no modifiquen archivos, no publiquen contenido.

---

## Qué podés usar sin preocuparte

Estas tareas son de bajo riesgo y son el punto fuerte de Cowork:

| Tarea | Por qué es segura |
|---|---|
| Resumir un documento propio | No involucra datos de terceros ni sistemas externos |
| Redactar borradores de texto | Solo genera contenido, no accede a sistemas |
| Analizar una planilla con datos sanitizados | Los datos reales no salen de tu control |
| Investigar un tema en la web | Actividad acotada, sin acceso a sistemas internos |
| Formatear o reorganizar documentos | Tarea local, sin envío de datos |
| Generar código o fórmulas | No involucra datos sensibles |

---

## Señales de que algo no está bien

Prestá atención si Cowork:

- Quiere acceder a carpetas que no mencionaste en la tarea
- Intenta enviar información fuera de la empresa sin que vos lo hayas pedido
- Hace preguntas raras o actúa de forma inconsistente con lo que le pediste
- Pide credenciales, contraseñas o accesos a sistemas
- Genera texto que parece de un tercero intentando darte instrucciones

En cualquiera de estos casos: **cerrá la sesión y avisale a IT.**

---

## Preguntas frecuentes

**¿Cowork guarda todo lo que le muestro?**
Sí. El historial de conversaciones queda guardado en tu computadora. Si alguien accede a tu máquina, puede ver todo lo que hablaste con Cowork.

**¿Anthropic (la empresa que hace Claude) puede ver mis conversaciones?**
Sí, salvo que tu empresa tenga un acuerdo especial llamado ZDR. Si no sabés si lo tienen, consultá con IT antes de trabajar con información sensible.

**¿Puedo usarlo para procesar datos de clientes?**
Depende del tipo de dato y del acuerdo de privacidad con ese cliente. La regla general: si los datos están sujetos a la Ley 25.326 (datos personales en Argentina), sanitizá primero o consultá con IT.

**¿Qué hago si no sé si un dato es sensible o no?**
Si tenés duda, sanitizá. Reemplazar un valor por `[REDACTED]` no arruina el trabajo — evita un problema potencial.

**¿Puedo darle a Cowork acceso a mi mail?**
Técnicamente sí si el conector está habilitado. Pero pensá dos veces: tu mail puede contener información de clientes, datos de salud, información legal. Si lo hacés, configurá la tarea para que solo lea, nunca envíe.

---

## Resumen en una página

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

*Guía preparada por el equipo IT | Abril 2026 | Para dudas: consultá con tu referente de seguridad*
*Ver también: [[Guides/Claude Cowork - Diagramas para Cliente]]*
