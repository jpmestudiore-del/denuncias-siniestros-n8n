# Denuncias de Siniestros — Análisis con IA + Revisión Humana (HITL)

Trabajo final de automatización de procesos con **n8n**. Automatiza la gestión de
denuncias de siniestros automotores presentadas por un PAS (Productor Asesor de
Seguros), incorporando análisis con Inteligencia Artificial y un punto de control
humano (Human-In-The-Loop) antes de la decisión final.

---

## Descripción del caso

Cuando un PAS carga una nueva denuncia de siniestro en Airtable, el flujo:

1. **Detecta** automáticamente la nueva denuncia (estado "Pendiente").
2. **Analiza** con IA (GPT-4o-mini) si existe un tercero responsable del accidente
   y si hubo daños en el/los vehículo(s).
3. **Registra** el análisis de la IA en Airtable.
4. **Solicita revisión humana** por correo: un responsable aprueba o rechaza.
5. Según la decisión humana:
   - **Aprobado → Atender cliente:** se envía un email al asegurado y se marca
     la denuncia para su gestión.
   - **Rechazado → Descartado:** se marca la denuncia como descartada (sin
     responsable).
6. Si la IA falla, la denuncia queda marcada para **revisión manual**.

El objetivo es acelerar el triage de siniestros sin quitar el control final a una
persona.

---

## Arquitectura del flujo

**Stack:** n8n (orquestación) · Airtable (base de datos) · OpenAI GPT-4o-mini
(análisis) · Gmail (notificaciones y aprobación HITL).

### Diagrama del flujo

┌──────────────────────────────┐
│ Nueva denuncia (Airtable) │ Trigger: nueva fila con Estado = "Pendiente"
│ Trigger — poll cada minuto │
└───────────────┬──────────────┘
│
▼
┌──────────────────────────────┐
│ Analizar siniestro con IA │ GPT-4o-mini · salida JSON estructurada
│ (OpenAI) │ (responsable, daños, nivel, resumen)
└───────┬───────────────┬──────┘
│ éxito │ error
▼ ▼
┌───────────────┐ ┌──────────────────────────┐
│ Guardar │ │ Registrar error de IA │
│ análisis │ │ Estado = "Error IA - │
│ Estado = │ │ Revisar" │
│ "Procesado │ └──────────────────────────┘
│ por IA" │
└───────┬───────┘
▼
┌──────────────────────────────┐
│ Revisión humana (HITL) │ Gmail: Send & Wait for approval
│ Botones: Atender / Descartar │ Espera hasta 2 días
└───────────────┬──────────────┘
▼
┌──────────────────────────────┐
│ ¿Aprobado por humano? (IF) │
└───────┬───────────────┬──────┘
TRUE │ FALSE │
▼ ▼
┌───────────────┐ ┌──────────────────────────┐
│ Atender │ │ Estado: Descartado │
│ cliente (Gmail)│ │ "Descartado (sin │
│ email al │ │ responsable)" │
│ asegurado │ └──────────────────────────┘
└───────┬───────┘
▼
┌───────────────┐
│ Estado: │
│ Atender cliente│ "Aprobado por Humano -
│ (Airtable) │ Atender Cliente"
└───────────────┘


### Estados de la denuncia

| Estado                                  | Cuándo se asigna                          |
|-----------------------------------------|-------------------------------------------|
| `Pendiente`                             | Al crear la denuncia (dispara el flujo)   |
| `Procesado por IA`                      | Tras el análisis exitoso de la IA         |
| `Error IA - Revisar`                    | Si la IA falla (revisión manual)          |
| `Aprobado por Humano - Atender Cliente` | El revisor aprueba (hay responsable)      |
| `Descartado (sin responsable)`          | El revisor rechaza (sin responsable)      |

> El trigger filtra por `Estado = "Pendiente"`, lo que evita reprocesar denuncias
> ya gestionadas y previene bucles infinitos.

---

https://airtable.com/appraIGAQu0s7ghJX/shrzQLKsoyS4LLC6w
