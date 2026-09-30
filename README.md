# Ecosistema de Automatización IA — Gestión de Frees

**Entrega Final · Curso de AI Automation · Coderhouse**
Tomás Etiennot Sardo — KEMET / VIBRATE / PALMA

Sistema autónomo de intake, evaluación y respuesta de solicitudes de entradas de
cortesía para tres marcas de eventos. Orquestación en Make, memoria relacional en
Airtable, razonamiento con Claude y salida por Gmail, con validación humana antes
de toda comunicación externa.

---

## Documentación

**[`documentacion-entrega-final.pdf`](documentacion-entrega-final.pdf)** — documento
principal. Cubre los cinco criterios de evaluación:

| Sección | Criterio | Peso |
|---|---|---|
| 1 | Mapa de arquitectura del sistema | 20 % |
| 2 | Manual operativo de estructuras de datos | 20 % |
| 3 | Estrategia de optimización de costos y recursos | 20 % |
| 4 | Malla de seguridad, privacidad y resiliencia | 20 % |
| 5 | Dashboard de control ejecutivo | 20 % |

Anexo A: prueba de estrés. Anexo B: repositorio y enlaces.

---

## Enlaces públicos

| Recurso | Enlace |
|---|---|
| Formulario de solicitud | https://tally.so/r/1Aeqyb |
| Dashboard — Solicitudes | https://airtable.com/appIVK7AvA6qOrnQA/shrxQ5B6QKmLZdJVY |
| Dashboard — Log de errores | https://airtable.com/appIVK7AvA6qOrnQA/shr3Exf9iuTwaZ3S9 |

---

## Stack

| Capa | Herramienta | Rol |
|---|---|---|
| Entrada | Tally | Formulario público con aviso de IA |
| Orquestador | Make | Dos escenarios encadenados, 15 nodos |
| Base de datos | Airtable | 5 tablas relacionadas · memoria y registro |
| Procesamiento IA | Claude Sonnet 4.6 | Scoring y redacción con voz de marca |
| Canal de salida | Gmail API | Notificación interna y respuesta al solicitante |

---

## Cómo funciona

```
Formulario  →  Webhook  →  Normalización  →  Buscar evento
                                                  │
                                            ┌─────┴─────┐
                                           SÍ           NO
                                            │            │
                          Base de conocimiento    Log de errores
                          Historial del handle    (ejecución detenida
                          Evaluación con Claude ⚠   sin consumir IA)
                          Guardar solicitud
                          Notificar aprobación
                                            │
                              ═══ HUMAN-IN-THE-LOOP ═══
                                            │
                          Detectar aprobación  →  Filtro anti-bucle
                          →  Email al solicitante  →  Estado: Notificado

⚠ El módulo de evaluación tiene un error handler: si la API falla, registra la
  excepción en el log y la directiva Break preserva el bundle para reintentarlo.
```

El **escenario 1** corre sin intervención humana desde que llega la solicitud hasta
que queda evaluada, persistida y notificada. El **escenario 2** sólo puede arrancar
si una persona tildó la casilla de aprobación en Airtable.

### Modelo de scoring

| Variable | Peso | Fuente |
|---|---|---|
| Seguidores | 55 | Campo del formulario |
| Contraprestación ofrecida | 25 | Pitch — lo interpreta el LLM |
| Historial con la casa | 15 | Airtable |
| Tamaño del grupo | 5 | Campo del formulario |

**Tiers:** A (75–100) · B (50–74) · C (30–49) · Rechazado (<30)

**Reglas duras** evaluadas antes del score: cupo agotado, dos o más no-shows
confirmados, y seis o más invitados (topea el tier en C).

---

## Estructura

```
├── README.md
├── documentacion-entrega-final.pdf
├── blueprints/
│   ├── intake-y-scoring.json
│   └── notificacion-post-aprobacion.json
└── evidencias/
    ├── historial-ejecuciones-make.png.png
    ├── escenario1-ejecucion-completa.png
    ├── caso1-airtable.png
    ├── caso2-airtable.png
    ├── caso3-airtable.png
    ├── caso4-error-make.png
    ├── caso4-error-log.png
    ├── caso5-airtable.png
    ├── antibucle-filtro-bloquea.png
    ├── antibucle-filtro-pasa.png
    ├── hitl-mail-aprobacion.png
    └── dashboard-panel-control.png
```

### Qué muestra cada evidencia

| Archivo | Contenido |
|---|---|
| `historial-ejecuciones-make.png.png` | Corridas del escenario 1 con fecha, hora y operaciones |
| `escenario1-ejecucion-completa.png` | Canvas del escenario 1 con los módulos en verde |
| `caso1-airtable.png` … `caso5-airtable.png` | Resultado de cada caso de prueba: score, tier y estado |
| `caso4-error-make.png` · `caso4-error-log.png` | Camino infeliz: el router desvía y registra el error |
| `antibucle-filtro-bloquea.png` · `antibucle-filtro-pasa.png` | El filtro anti-bucle probado en las dos direcciones |
| `hitl-mail-aprobacion.png` | Notificación interna de validación humana |
| `dashboard-panel-control.png` | Panel de control con KPIs y tasa de errores |

---

## Resiliencia

Dos mecanismos distintos, para dos clases de fallo:

- **Router** — cuando el dato de entrada no resuelve (evento inexistente). Desvía
  antes de consumir la API de IA y registra el caso en `Log de ejecuciones`.
- **Error handler** — cuando la API de IA devuelve una excepción (límite de tasa,
  sobrecarga, timeout). Captura `error.message`, lo persiste, y la directiva
  **Break** preserva el bundle con tres reintentos espaciados.
- **Filtro anti-bucle** — en el escenario 2, la condición `Estado ≠ Notificado`
  impide que la actualización del registro vuelva a disparar el trigger.

---

## Notas de privacidad

El sistema **no recolecta** documento de identidad, teléfono ni fecha de nacimiento.
El único dato de contacto es el correo, necesario para responder. Las direcciones
con token de acceso que llegan en el payload del formulario se descartan
deliberadamente y no se persisten.

El *system prompt* incluye una barrera explícita contra la evaluación de género,
apariencia, edad o nacionalidad. Las cuatro variables del scoring miden
comportamiento y resultado, no atributos de la persona.

Clasificación bajo la IA Act: **riesgo limitado**. La obligación de transparencia se
cumple con el aviso visible en el formulario.
