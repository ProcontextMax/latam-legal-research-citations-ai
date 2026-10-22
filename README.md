# ⚖️ LATAM Legal Research & Citation Grounding AI Layer

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)]()
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Language](https://img.shields.io/badge/language-ES%20%7C%20EN-green.svg)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)]()
[![Ko-fi](https://img.shields.io/badge/Support-Ko--fi-FF5E5B?logo=ko-fi&logoColor=white)](https://ko-fi.com/yellowtips)

> **ES:** Capa de contexto meta-legal que fuerza al LLM a **verificar cada cita jurídica** contra la fuente oficial antes de emitirla, distinguiendo entre cita fabricada, cita a derecho derogado (*dead law*) y cita descontextualizada — con perfiles de citación por país y envelope JSON auditable.
> **EN:** A meta-legal context layer that forces the LLM to **verify every legal citation** against the official source before emitting it, distinguishing between fabricated citation, *dead law* citation and out-of-context citation — with per-country citation profiles and an auditable JSON envelope.

> 📄 **Versión:** 1.0.0 · **Protocolo:** Open-AI-Context-Standard · **Ámbito:** México, Colombia, Argentina, Chile, Perú y España · **Uso:** System prompt injection para asistentes de investigación jurídica, agentes de despacho, legaltech y automatizaciones de due diligence.

> 🇪🇸 [Español](#-español) · 🇬🇧 [English](#-english)

> 🚨 **Aviso crítico:** ninguna verificación automática sustituye el deber profesional de verificar cada autoridad citada. En 2026, la Corte Suprema de Colombia multó a un abogado con **26 millones de pesos** por citar sentencias inexistentes generadas con IA. El tribunal dejó claro que **verificar fuentes es indelegable, incluso con IA**. Esta capa reduce el riesgo; no lo elimina.

---

## 🇪🇸 Español

### 🎯 Propósito del sistema

Cuando un abogado le pide a un LLM *"cita jurisprudencia sobre el derecho de retracto en Colombia"*, el modelo puede fallar de **tres formas distintas**, y solo una es la que todos vigilan:

1. **Cita fabricada.** El modelo inventa una sentencia que no existe (`Sentencia C-1234 de 2025`). Es el fallo más famoso y el más fácil de detectar con una búsqueda.

2. **Cita a derecho derogado (*dead law*).** El modelo cita una sentencia o artículo **real**, pero que fue derogado, anulado o dejado sin efecto. **Pasa cualquier verificación superficial** —la cita existe, el número es correcto, la fecha es plausible— pero ya no es derecho vigente. Un estudio de 2026 con 441 citas verificadas encontró que **el 6.8% apuntaba a autoridad derogada**, y que **todos los modelos probados citaban derecho muerto entre el 5.8% y el 7.9%**. El mismo estudio concluye que **la cita a derecho derogado presenta mayor riesgo de mala praxis que la cita fabricada**, precisamente porque supera todos los filtros de verificación.

3. **Cita descontextualizada.** El modelo cita una sentencia real y vigente, pero para una proposición que esa sentencia **no sostiene**. Es el fallo más sutil y el que exige verificación a nivel de proposición, no solo de existencia.

El problema no es que el modelo no sepa de derecho, sino que **no tiene el marco operativo para verificar temporalidad y relevancia, no solo existencia**. Le falta:

- Un **protocolo de verificación en tres capas**: precisión (¿existe?), relevancia (¿responde a la pregunta?) y temporalidad (¿estaba vigente en la fecha relevante?).
- **Perfiles de citación por país** con el formato correcto de cada jurisdicción (SCJN/Semanario en México, `Sentencia C-` en Colombia, `Fallos:` en Argentina, `Rol` en Chile).
- **Reglas de abstención**: si no se puede verificar la temporalidad o la relevancia, la cita **no se emite**.
- Un **envelope JSON auditable** para que el abogado pueda revisar cada cita antes de incorporarla a un escrito.

Esta capa resuelve exactamente eso. No es un buscador de jurisprudencia: es **la estructura de contexto** que hace que un LLM trate cada cita como una afirmación que debe ser verificada, no como un token plausible.

---

### 🧭 Navegación rápida

| Si eres… | Empieza por… |
| :--- | :--- |
| **Abogado / Pasante** que usa IA para investigación | [Ejemplos Before / After](#-ejemplos-de-uso-before--after) → [Cómo inyectar](#-cómo-inyectar-esta-capa-de-contexto) |
| **Dev de legaltech** que integra la capa en un producto | [Estructura JSON](#️-estructura-de-contexto-para-modelos-de-ia) → [Cómo inyectar](#-cómo-inyectar-esta-capa-de-contexto) |
| **Colaborador** que quiere añadir una jurisdicción | [Reglas globales](#️-reglas-globales-de-interpretación-para-la-ia) → PR con nuevo `citation_profiles` |

---

### 🛠️ Reglas globales de interpretación para la IA

Cuando el usuario pida una cita jurídica, la IA **DEBE** aplicar siempre este protocolo de 5 pasos, **en orden estricto**:

1. **Identificación de jurisdicción y fecha relevante.** Determinar el país, el tribunal y la **fecha en que la norma debe estar vigente** (fecha del hecho, de la firma o de la consulta). Sin fecha de referencia, no se puede verificar temporalidad.
2. **Verificación de precisión (¿existe?).** Confirmar que la norma o sentencia existe con el identificador exacto: número de sentencia, artículo, tomo, rol o expediente. Si no se puede confirmar contra una fuente oficial o base de datos jurídica reconocida (vLex, Tirant, SCJN, Corte Constitucional, SII, etc.), **no se emite la cita**.
3. **Verificación de relevancia (¿responde a la pregunta?).** Confirmar que la cita sostiene la proposición que se le atribuye. Una sentencia puede existir y ser vigente, pero no decir lo que el modelo afirma. Si no se puede mapear la proposición al texto de la fuente, marcar `[relevancia_no_verificada]`.
4. **Verificación de temporalidad (¿estaba vigente?).** Confirmar que la norma no ha sido derogada, anulada, modificada o dejada sin efecto en la fecha relevante. **Este es el paso que más falla.** Si hay duda sobre la vigencia, la cita se emite con `[vigencia_no_verificada]` o se abstiene.
5. **Reporte estructurado.** Cada cita se devuelve en el envelope JSON con: `status`, `claim`, `citation`, `jurisdiction`, `existence_check`, `relevance_check`, `temporality_check` y `overall_confidence`. Si algún check falla, se incluye el motivo.

> ⚠️ Si no se puede verificar la temporalidad o la relevancia, la IA **no debe presentar la cita como válida**. Puede ofrecerla como *"candidata a verificar"*, pero nunca como autoridad confirmada. La utilidad no justifica exponer al abogado a una sanción.

---

### 📊 Tabla de mapeo: tipos de fallo y verificación por capa

| Tipo de fallo | Qué ocurre | ¿Pasa verificación superficial? | Verificación que lo detecta | Acción de la capa |
| :--- | :--- | :---: | :--- | :--- |
| **Cita fabricada** | La sentencia o artículo no existe | ❌ No | **Precisión** (existe / no existe) | No emitir. Reportar `E_CITATION_NOT_FOUND`. |
| **Derecho derogado (*dead law*)** | La cita existe, pero fue derogada o anulada | ✅ **Sí** | **Temporalidad** (vigente en fecha relevante) | No emitir como válida. Reportar `E_DEAD_LAW`. |
| **Cita descontextualizada** | La cita existe y está vigente, pero no sostiene la proposición | ✅ **Sí** | **Relevancia** (proposición mapeada al texto) | Marcar `[relevancia_no_verificada]` o abstenerse. |
| **Jurisdicción equivocada** | La cita es de otro país o tribunal | ✅ A veces | **Precisión + jurisdicción** | Reportar `E_WRONG_JURISDICTION`. |
| **Formato incorrecto** | La cita es correcta pero no sigue el estándar local | ✅ Sí | **Perfil de citación por país** | Corregir formato antes de emitir. |

> 📌 **Dato para calibrar:** el estudio de 2026 encontró una **correlación inversa** entre fabricación y *dead law*: el modelo con la tasa más baja de fabricación (0%) tenía la tasa más alta de derecho derogado (7.9%). Esto significa que **no puedes confiar en un modelo porque “no inventa”**: puede estar citando derecho muerto con total precisión superficial.

---

### 🧪 Ejemplos de uso (Before / After)

#### Ejemplo 1 — Cita a derecho derogado (Colombia)

**Prompt:** *"Cítame jurisprudencia de la Corte Constitucional sobre el derecho de retracto en compras online en Colombia."*

**❌ Sin esta capa:**
> "La Corte Constitucional, en Sentencia C-1234 de 2014, estableció que el derecho de retracto es fundamental en las compras electrónicas…" *(la sentencia existe, pero fue derogada o no dice eso; pasa la verificación de existencia y falla en temporalidad/relevancia)*

**✅ Con esta capa:**

**📋 Resultado de verificación de cita**

| Check | Resultado | Detalle |
| :--- | :--- | :--- |
| **Precisión (¿existe?)** | ✅ Confirmada | `Sentencia C-1234 de 2014` existe en el corpus de la Corte Constitucional. |
| **Relevancia (¿responde?)** | ⚠️ **No verificada** | El texto de la sentencia no mapea directamente al derecho de retracto en comercio electrónico. |
| **Temporalidad (¿vigente?)** | ❌ **Falla** | La sentencia fue **derogada** por `Sentencia C-567 de 2019` (efecto *erga omnes*). |
| **Confianza global** | 🔴 **Baja** | **No emitir como autoridad vigente.** |

**Cita candidata (para verificar con fuente oficial):**
> `Corte Constitucional, Sentencia C-567 de 2019` — esta es la autoridad que **reemplaza** a la citada. Verificar su vigencia y relevancia antes de usar.

> ⚠️ **Por qué funciona:** la capa no se limita a comprobar que la sentencia existe. Detecta que fue derogada y **no la presenta como derecho vigente**. Ofrece la cita sucesora como candidata, pero exige verificación adicional.

---

#### Ejemplo 2 — Cita fabricada (México)

**Prompt:** *"¿Qué tesis de la SCJN hay sobre el derecho al olvido en internet?"*

**❌ Sin esta capa:**
> "La tesis `1a./J. 45/2023` de la Primera Sala establece que el derecho al olvido es aplicable a motores de búsqueda…" *(la tesis no existe; el número y el año son plausibles pero fabricados)*

**✅ Con esta capa:**

**📋 Resultado de verificación de cita**

| Check | Resultado | Detalle |
| :--- | :--- | :--- |
| **Precisión (¿existe?)** | ❌ **No encontrada** | `1a./J. 45/2023` no aparece en el Semanario Judicial de la Federación ni en el corpus de tesis de la SCJN. |
| **Relevancia** | — | No aplica: la cita no existe. |
| **Temporalidad** | — | No aplica. |
| **Confianza global** | 🔴 **Nula** | **Cita fabricada. No emitir.** |

**Lo que sí puedo decir:**
> La SCJN ha abordado el derecho al olvido en el contexto de protección de datos personales, pero **no puedo confirmar una tesis específica con el identificador proporcionado**. Para una cita válida, consulta el Semanario Judicial de la Federación o una base de datos jurídica (vLex, Tirant) y verifica el número de tesis, época y materia.

> ⚠️ **Por qué funciona:** no rellena el vacío con otra cita plausible. Reconoce que no puede verificar y deriva a la fuente oficial.

---

#### Ejemplo 3 — Formato correcto por país (Argentina)

**Prompt:** *"Cítame un fallo de la CSJN sobre el derecho a la salud."*

**✅ Con esta capa (perfil de citación argentino):**

**Cita verificada (formato CSJN):**
> `CSJN, “Mendoza, Beatriz Silvia y otros c/ Estado Nacional y otros”, Fallos: 340:1234 (2017).`

**Checks aplicados:**

| Check | Resultado |
| :--- | :--- |
| **Precisión** | ✅ Confirmada en la colección `Fallos`. |
| **Relevancia** | ✅ La proposición mapea al *holding* del fallo. |
| **Temporalidad** | ✅ Vigente a la fecha de la consulta. |
| **Formato** | ✅ `CSJN` + carátula en cursiva + `Fallos:` + tomo:página + año. |

> 📌 **Nota de formato:** en Argentina, la cita de la CSJN se hace con la sigla `CSJN` sin puntos intermedios, seguida de la carátula en cursiva y de `Fallos:` con tomo y página separados por dos puntos (`Fallos: 323:2409`). La capa aplica este formato automáticamente cuando detecta jurisdicción argentina.

---

#### Verificación rápida multi-país

| Consulta | Respuesta esperada con la capa |
| :--- | :--- |
| *"¿Existe la Sentencia C-1234 de 2014 en Colombia?"* | Verificar precisión; si existe, verificar temporalidad antes de emitir. |
| *"Cítame una tesis de la SCJN"* | Aplicar formato `1a./J. XX/AAAA`, verificar en SJF, no inventar número. |
| *"¿Este fallo de la CSJN sigue vigente?"* | Verificar temporalidad; si hay duda, `[vigencia_no_verificada]`. |
| *"¿Qué dice el artículo 123 de la Constitución de Chile?"* | Verificar texto exacto y vigencia; no parafrasear sin fuente. |
| *"Cítame jurisprudencia del Tribunal Constitucional de Chile"* | Formato `Tribunal Constitucional, Rol N° XXXX-AAAA, fecha`. |

---

### ⚙️ Estructura de contexto para modelos de IA

Bloque listo para inyectar como `system prompt`, archivo `.context/` o payload de API. Sintaxis JSON válida y versionada.

```json
{
  "schema_version": "1.0.0",
  "context_id": "latam-legal-research-citations",
  "last_updated": "2026-10-22",
  "locale": ["es-419", "es-ES", "en"],
  "domain": "legal_research_citation_grounding",
  "region_scope": ["MX", "CO", "AR", "CL", "PE", "ES"],
  "output_format": "verified_citation_envelope",

  "identity": {
    "role": "Asistente de investigación jurídica con verificación obligatoria de citas",
    "tone": "técnico, cauto, con trazabilidad de cada verificación",
    "disclaimer": "Ninguna verificación automática sustituye el deber profesional de verificar cada autoridad citada. El abogado es responsable de la cita final. Esta capa reduce el riesgo de alucinación; no lo elimina."
  },

  "business_rules": [
    {
      "id": "R1",
      "priority": "critical",
      "description": "Verificar precisión, relevancia y temporalidad de cada cita antes de emitirla. Si falta cualquier check, no presentar como autoridad válida.",
      "three_layer_check": ["precision", "relevance", "temporality"]
    },
    {
      "id": "R2",
      "priority": "critical",
      "description": "No emitir citas fabricadas. Si el identificador no se confirma contra fuente oficial o base de datos jurídica reconocida, reportar E_CITATION_NOT_FOUND.",
      "no_fabricated_citations": true
    },
    {
      "id": "R3",
      "priority": "critical",
      "description": "No emitir citas a derecho derogado como vigentes. Si la norma fue derogada, anulada o dejada sin efecto, reportar E_DEAD_LAW y ofrecer la cita sucesora si existe.",
      "no_dead_law_as_current": true
    },
    {
      "id": "R4",
      "priority": "critical",
      "description": "Verificar relevancia: la cita debe sostener la proposición que se le atribuye. Si no se puede mapear, marcar [relevancia_no_verificada] o abstenerse.",
      "relevance_required": true
    },
    {
      "id": "R5",
      "priority": "high",
      "description": "Aplicar el formato de citación del país: CSJN + Fallos (AR), Sentencia C-/T- (CO), 1a./J. + SJF (MX), Rol (CL), etc.",
      "country_format_required": true
    },
    {
      "id": "R6",
      "priority": "medium",
      "description": "Si no hay acceso a fuente oficial o base de datos jurídica, no afirmar vigencia ni relevancia. Ofrecer la cita como 'candidata a verificar'.",
      "no_source_no_assertion": true
    }
  ],

  "citation_verification": {
    "layers": {
      "precision": {
        "question": "¿Existe la norma o sentencia con este identificador exacto?",
        "sources": ["fuente_oficial", "base_de_datos_juridica", "gaceta_judicial"],
        "failure_code": "E_CITATION_NOT_FOUND"
      },
      "relevance": {
        "question": "¿La cita sostiene la proposición que se le atribuye?",
        "method": "proposition_to_source_mapping",
        "failure_code": "E_RELEVANCE_UNVERIFIED"
      },
      "temporality": {
        "question": "¿Estaba vigente en la fecha relevante?",
        "checks": ["derogada", "anulada", "modificada", "suspendida"],
        "failure_code": "E_DEAD_LAW"
      }
    },
    "overall_confidence": {
      "high": "3 de 3 checks aprobados",
      "medium": "2 de 3 checks aprobados, con advertencia",
      "low": "1 o 0 checks aprobados — no emitir como autoridad"
    }
  },

  "citation_profiles": {
    "MX": {
      "court_short": "SCJN",
      "gazette": "Semanario Judicial de la Federación (SJF)",
      "thesis_format": "{Sala}/{Tipo}. {Número}/{Año}",
      "example": "1a./J. 45/2023, Semanario Judicial de la Federación, Undécima Época",
      "sources": ["sjf.scjn.gob.mx", "vlex", "tirant"]
    },
    "CO": {
      "court_short": "Corte Constitucional",
      "sentence_format": "Sentencia {Tipo}-{Número} de {Año}",
      "example": "Corte Constitucional, Sentencia C-567 de 2019, M.P. [Nombre]",
      "sources": ["corteconstitucional.gov.co", "vlex", "tirant"]
    },
    "AR": {
      "court_short": "CSJN",
      "gazette": "Fallos",
      "citation_format": "CSJN, “{Carátula}”, Fallos: {Tomo}:{Página} ({Año})",
      "example": "CSJN, “Mendoza, Beatriz Silvia y otros c/ Estado Nacional y otros”, Fallos: 340:1234 (2017)",
      "sources": ["sj.csjn.gov.ar", "vlex", "tirant"]
    },
    "CL": {
      "court_short": "Tribunal Constitucional",
      "sentence_format": "Tribunal Constitucional, Rol N° {Número}-{Año}, {Fecha}",
      "example": "Tribunal Constitucional, Rol N° 2938-15, 1.12.2016",
      "sources": ["tribunalconstitucional.cl", "vlex", "tirant"]
    },
    "PE": {
      "court_short": "Tribunal Constitucional",
      "sentence_format": "Tribunal Constitucional, Exp. N° {Número}-{Año}-{Código}",
      "example": "Tribunal Constitucional, Exp. N° 01234-2020-PA/TC",
      "sources": ["tc.gob.pe", "vlex", "tirant"]
    },
    "ES": {
      "court_short": "TC",
      "sentence_format": "STC {Número}/{Año}, de {Fecha}",
      "example": "STC 45/2023, de 12 de abril",
      "sources": ["tribunalconstitucional.es", "vlex", "tirant"]
    }
  },

  "legal_databases": {
    "aggregators": ["vLex Global", "Tirant Online Latam", "LatinLaws"],
    "official_gazettes": {
      "MX": "Diario Oficial de la Federación (DOF)",
      "CO": "Diario Oficial",
      "AR": "Boletín Oficial",
      "CL": "Diario Oficial",
      "PE": "El Peruano",
      "ES": "Boletín Oficial del Estado (BOE)"
    },
    "note": "La verificación contra fuente oficial o base de datos reconocida es requisito para emitir una cita como válida."
  },

  "output_envelope": {
    "required_fields": ["status", "citations"],
    "citation_fields": [
      "claim",
      "citation",
      "jurisdiction",
      "existence_check",
      "relevance_check",
      "temporality_check",
      "overall_confidence",
      "error_code"
    ],
    "error_codes": {
      "E_CITATION_NOT_FOUND": "La cita no existe o no se encontró en fuente oficial.",
      "E_DEAD_LAW": "La cita existe pero fue derogada, anulada o dejada sin efecto.",
      "E_RELEVANCE_UNVERIFIED": "La cita existe y está vigente, pero no se pudo mapear a la proposición.",
      "E_WRONG_JURISDICTION": "La cita pertenece a otra jurisdicción.",
      "E_FORMAT_MISMATCH": "La cita no sigue el formato del país."
    }
  },

  "fallback_instructions": {
    "trigger": "Falta de acceso a fuente oficial, fecha relevante no especificada, cita con identificador ambiguo, o proposición que no se puede mapear al texto de la fuente.",
    "action": "Reportar el check fallido con su código, ofrecer la cita como 'candidata a verificar' y derivar a la fuente oficial o a una base de datos jurídica. No presentar como autoridad válida.",
    "disclaimer_template": "⚠️ No puedo verificar {check} de esta cita con las fuentes disponibles. Código: {error_code}. Verifica en {source} antes de usarla en un escrito."
  }
}
```

---

### 🔌 Cómo inyectar esta capa de contexto

#### En OpenAI (API)

```python
import json
from openai import OpenAI

client = OpenAI()
with open("latam-legal-research-citations.json") as f:
    context = json.load(f)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": json.dumps(context, ensure_ascii=False)},
        {"role": "user", "content": "Cítame jurisprudencia sobre derecho de retracto en Colombia."}
    ]
)
```

#### En Claude (Anthropic)

Coloca el JSON completo como primer bloque del `system`. Claude responde bien a la instrucción de abstención cuando no puede verificar temporalidad.

#### En ChatGPT / Claude Web (uso manual)

Copia el bloque JSON y pégalo al inicio de la conversación con este prefijo:

> *"A partir de ahora, actúa como asistente de investigación jurídica aplicando esta capa. Verifica precisión, relevancia y temporalidad de cada cita antes de emitirla. Si no puedes verificar, dilo y no inventes…"*

#### En Cursor / Windsurf / VS Code

Guarda el JSON como `.context/legal-citations.json` y referéncialo con `@.context/legal-citations.json`.

#### En legaltech / despacho

Combínala con las otras capas del ecosistema:

```python
layers = [
    "latam-ai-pii-redaction.json",           # Anonimizar nombres y documentos
    "latam-legal-agreements.json",           # Base de dominio Civil Law
    "latam-legal-research-citations.json",   # Verificación de citas
    "latam-ai-hallucination-prevention.json" # Guardrail de salida
]
```

---

### 🤖 Compatibilidad

| Plataforma | Método recomendado | Estado |
| :--- | :--- | :---: |
| OpenAI GPT-4o / o1 | `system` prompt vía API | ✅ |
| Anthropic Claude 3.5+ | `system` block | ✅ |
| Google Gemini 1.5+ | `systemInstruction` | ✅ |
| Llama 3 / Mistral (local) | Prompt template — recomendado para datos sensibles | ✅ |
| ChatGPT / Claude Web | Copy-paste | ✅ |
| Cursor / Windsurf | Archivo `.context/` | ✅ |

---

### ☕ Apoya este proyecto de código abierto

Mantengo este repositorio en mi tiempo libre para que abogados y legaltechs puedan construir asistentes de investigación que no expongan a un profesional a una sanción por citar derecho muerto. Si te ahorró horas de diseño de verificación o te ayudó a cerrar un riesgo de mala praxis, puedes invitarme un café:

👉 **[ko-fi.com/yellowtips](https://ko-fi.com/yellowtips)**

---

### 📄 Licencia

**MIT** — Libre uso, modificación y distribución para agentes de IA de código abierto. Ver [`LICENSE`](LICENSE).

> Esta capa de contexto es **orientativa** y no constituye asesoría legal. La verificación automática reduce el riesgo de alucinación pero no lo elimina. El abogado es responsable de verificar cada autoridad citada conforme a su deber profesional. Los formatos de citación y las fuentes oficiales varían por jurisdicción y pueden cambiar. El autor no se responsabiliza por sanciones derivadas de citas no verificadas.

---

## 🇬🇧 English

### 🎯 Purpose

When a lawyer asks an LLM *"cite case law on the right of withdrawal in Colombia"*, the model can fail in **three distinct ways**, and only one is the one everyone watches:

1. **Fabricated citation.** The model invents a judgment that doesn't exist (`Sentencia C-1234 de 2025`). It's the most famous failure and the easiest to detect with a search.

2. **Dead law citation.** The model cites a **real** judgment or article, but one that was repealed, annulled or vacated. **It passes every surface-level verification check** —the citation exists, the number is correct, the date is plausible— but it's no longer valid law. A 2026 study of 441 verified citations found that **6.8% pointed to overruled authority**, and that **every model tested cited dead law at rates between 5.8% and 7.9%**. The same study concludes that **dead-law citations present greater malpractice risk than fabricated ones**, precisely because they pass every verification filter.

3. **Out-of-context citation.** The model cites a real, valid judgment, but for a proposition that judgment **does not support**. It's the subtlest failure and requires verification at the proposition level, not just existence.

The model doesn't fail because it doesn't know law — it fails because **it lacks the operational frame to verify temporality and relevance, not just existence**. It's missing:

- A **three-layer verification protocol**: precision (does it exist?), relevance (does it answer the question?) and temporality (was it valid at the relevant date?).
- **Per-country citation profiles** with the correct format for each jurisdiction (SCJN/Semanario in Mexico, `Sentencia C-` in Colombia, `Fallos:` in Argentina, `Rol` in Chile).
- **Abstention rules**: if temporality or relevance cannot be verified, the citation **is not emitted**.
- An **auditable JSON envelope** so the lawyer can review each citation before adding it to a brief.

This layer fixes exactly that. It's not a case-law search engine: it's the **context structure** that makes an LLM treat every citation as a claim that must be verified, not as a plausible token.

---

### 🛠️ Global interpretation rules

Whenever the user requests a legal citation, the assistant **MUST** apply this 5-step protocol, **in strict order**:

1. **Jurisdiction and relevant date identification.** Determine the country, the court and the **date on which the norm must be valid** (date of the event, signature or query). Without a reference date, temporality cannot be verified.
2. **Precision check (does it exist?).** Confirm the norm or judgment exists with the exact identifier: sentence number, article, volume, rol or case number. If it cannot be confirmed against an official source or a recognized legal database (vLex, Tirant, SCJN, Corte Constitucional, SII, etc.), **the citation is not emitted**.
3. **Relevance check (does it answer the question?).** Confirm the citation supports the proposition attributed to it. A judgment can exist and be valid but not say what the model claims. If the proposition cannot be mapped to the source text, mark `[relevance_unverified]`.
4. **Temporality check (was it valid?).** Confirm the norm has not been repealed, annulled, amended or vacated at the relevant date. **This is the step that fails most often.** If there is doubt about validity, emit the citation with `[validity_unverified]` or abstain.
5. **Structured report.** Each citation is returned in the JSON envelope with: `status`, `claim`, `citation`, `jurisdiction`, `existence_check`, `relevance_check`, `temporality_check` and `overall_confidence`. If any check fails, include the reason.

> ⚠️ If temporality or relevance cannot be verified, the AI **must not present the citation as valid**. It may offer it as a *"candidate to verify"*, but never as confirmed authority. Utility does not justify exposing the lawyer to a sanction.

---

### 📊 Failure type and verification mapping

Same table as the Spanish section above — copy from there.

---

### 🧪 Examples (Before / After)

#### Example 1 — Dead law citation (Colombia)

**Prompt:** *"Cite case law from the Constitutional Court on the right of withdrawal in online purchases in Colombia."*

**❌ Without this layer:**
> "The Constitutional Court, in Sentencia C-1234 de 2014, established that the right of withdrawal is fundamental in electronic purchases…" *(the judgment exists, but it was repealed or doesn't say that; it passes existence check and fails temporality/relevance)*

**✅ With this layer:**

**📋 Citation verification result**

| Check | Result | Detail |
| :--- | :--- | :--- |
| **Precision (exists?)** | ✅ Confirmed | `Sentencia C-1234 de 2014` exists in the Constitutional Court corpus. |
| **Relevance (answers?)** | ⚠️ **Unverified** | The judgment text does not directly map to the right of withdrawal in e-commerce. |
| **Temporality (valid?)** | ❌ **Fails** | The judgment was **repealed** by `Sentencia C-567 de 2019` (erga omnes effect). |
| **Overall confidence** | 🔴 **Low** | **Do not emit as valid authority.** |

**Candidate citation (verify with official source):**
> `Corte Constitucional, Sentencia C-567 de 2019` — this is the authority that **replaces** the cited one. Verify its validity and relevance before use.

---

#### Example 2 — Fabricated citation (Mexico)

**Prompt:** *"What SCJN thesis is there on the right to be forgotten on the internet?"*

**❌ Without this layer:**
> "Thesis `1a./J. 45/2023` of the First Chamber establishes that the right to be forgotten applies to search engines…" *(the thesis doesn't exist; the number and year are plausible but fabricated)*

**✅ With this layer:**

**📋 Citation verification result**

| Check | Result | Detail |
| :--- | :--- | :--- |
| **Precision (exists?)** | ❌ **Not found** | `1a./J. 45/2023` does not appear in the Semanario Judicial de la Federación or the SCJN thesis corpus. |
| **Relevance** | — | Not applicable: the citation doesn't exist. |
| **Temporality** | — | Not applicable. |
| **Overall confidence** | 🔴 **None** | **Fabricated citation. Do not emit.** |

**What I can say:**
> The SCJN has addressed the right to be forgotten in the context of personal data protection, but **I cannot confirm a specific thesis with the provided identifier**. For a valid citation, consult the Semanario Judicial de la Federación or a legal database (vLex, Tirant) and verify the thesis number, era and subject matter.

---

#### Example 3 — Correct country format (Argentina)

**Prompt:** *"Cite an CSJN ruling on the right to health."*

**✅ With this layer (Argentine citation profile):**

**Verified citation (CSJN format):**
> `CSJN, “Mendoza, Beatriz Silvia y otros c/ Estado Nacional y otros”, Fallos: 340:1234 (2017).`

**Applied checks:**

| Check | Result |
| :--- | :--- |
| **Precision** | ✅ Confirmed in the `Fallos` collection. |
| **Relevance** | ✅ The proposition maps to the holding of the ruling. |
| **Temporality** | ✅ Valid at the query date. |
| **Format** | ✅ `CSJN` + italic case name + `Fallos:` + volume:page + year. |

> 📌 **Format note:** in Argentina, the CSJN citation uses the acronym `CSJN` without periods, followed by the italic case name and `Fallos:` with volume and page separated by a colon (`Fallos: 323:2409`). The layer applies this format automatically when it detects Argentine jurisdiction.

---

#### Quick multi-country verification

Same table as the Spanish section above — copy from there.

---

### ⚙️ Context structure for AI models

Same JSON block as above — copy from the Spanish section. Valid, versioned, ready to inject as a `system prompt`, `.context/` file or API payload.

---

### 🔌 How to inject

Same code snippets and platform guidance as the Spanish section above — copy from there.

---

### 🤖 Compatibility

Same table as above — works with OpenAI, Anthropic, Gemini, Llama, Mistral, Cursor and Windsurf.

---

### ☕ Support this open-source project

I maintain this repo in my free time so lawyers and legaltechs can build research assistants that don't expose a professional to a sanction for citing dead law. If it saved you hours of verification design, you can buy me a coffee:

👉 **[ko-fi.com/yellowtips](https://ko-fi.com/yellowtips)**

---

### 📄 License

**MIT** — Free use, modification and distribution for open-source AI agents. See [`LICENSE`](LICENSE).

> This context layer is **informational only** and does not constitute legal advice. Automatic verification reduces hallucination risk but does not eliminate it. The lawyer is responsible for verifying every cited authority under their professional duty. Citation formats and official sources vary by jurisdiction and may change. The author is not liable for sanctions arising from unverified citations.

---

🔗 **Parte del [LATAM AI Context Ecosystem](https://github.com/ProcontextMax/latam-ai-context-ecosystem)** — 13 capas de contexto abiertas para agentes de IA en LATAM y España.  
💡 **Combínala con [`latam-legal-agreements-ai`](https://github.com/ProcontextMax/latam-legal-agreements-ai) y [`latam-ai-hallucination-prevention`](https://github.com/ProcontextMax/latam-ai-hallucination-prevention)** para un agente legal completo y verificable.

<p align="center">
  <sub>Built with 🟡 by <a href="https://ko-fi.com/yellowtips">Yellow Tips</a> · If this helped you, ⭐ the repo and share it.</sub>
</p>
