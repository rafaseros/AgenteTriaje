## 🎬 Prompts S1–S7 (cada uno finaliza con build + verificación + entregables)

> **Recuerda**: **NO hacer commits**. Al final de _cada_ prompt: **`npm run build`**.

---

## S1 — Observe: carga, validación/normalización, inicio del flujo

```markdown
[HEADER GLOBAL — pegar arriba]

Objetivo:
Cargar un `.txt|.json` (AGENT_INPUT v1), validar con Zod, sanitizar `symptoms.text`, normalizar y **emitir `FLOW_STARTED`** (crear `runId`, `startedAt`). Usar `vitals_reference.json` para marcar `warning/danger` (sin bloquear).

Implementa:

- `src/services/InputSchema.ts`: Zod + `parseAndNormalize(raw) -> { input, issues[] }`.
- `src/services/Guards.ts` (entrada): `validateVitals(input, vitalsRef) -> { issues[], severity }` usando `plausible_limits` y `severity_thresholds`.
- `src/ui/FilePicker.tsx`: carga de archivo, parseo/validación, issues visibles (alto contraste solo si warning/danger), botón **Iniciar ejecución**.
- `src/agent/AgentController.ts`: `startRun(input)` → inicializa `AGENT_OUTPUT`, emite `FLOW_STARTED`, marca **Observe = ok** en Panel.
- `src/ui/PanelFlujoAgente.tsx`: badges de nodos (pendiente/activo/ok/error).
- `src/pages/AgentStudioPage.tsx`: layout base (FilePicker | PanelFlujoAgente | Timeline | Métricas | JSON placeholders).

Visibilidad inmediata:

- **Observe = ok** y issues (si hay) visibles en FilePicker.

Ejecución:

- `npm run build`

Qué revisar:

- ✅ Compila sin errores.
- ✅ Input válido → sin errores (o advertencias visibles).
- ✅ `AGENT_OUTPUT.input` poblado; `FLOW_STARTED` registrado.

Entregables:

- `InputSchema.ts`, `Guards.ts` (entrada), `AgentController.ts` (inicio), `FilePicker.tsx`, `PanelFlujoAgente.tsx`, wiring en `AgentStudioPage.tsx`.
```

---

## S2 — Think: contexto, retrieval y timeline

```markdown
[HEADER GLOBAL — pegar arriba]

Objetivo:
Construir **contexto** desde el input; ejecutar **Retrieval** contra `src/data/protocols.json`; correr **PromptChaining** determinista; llenar Timeline y emitir eventos.

Implementa:

- `src/services/RetrievalService.ts`: `searchProtocols(ctx) -> [{id, snippet}]` por keywords (vitals/síntomas).
- `src/services/PromptChaining.ts`:
  - Pasos: `evaluateVitals` → `analyzeSymptoms` → `assessRisk` → `finalizeContext`.
  - Cada paso: `NODE_STARTED/FINISHED`, `EDGE_TRAVERSED`, añadir a `AGENT_OUTPUT.steps` (nota ≤160 chars, `severity:"neutral"`).
- `src/ui/TimelineDecisiones.tsx`: render de pasos (texto breve + hora relativa + fuentes si aplica).
- `src/agent/AgentController.ts`:
  - `think()` orquesta chaining + retrieval; calcula **complexityLevel** (con `vitals_reference.json`) → `"simple"|"medio"|"complejo"`; marca **Think = ok**.

Visibilidad inmediata:

- **Think = ok** y 3–4 pasos nuevos en Timeline.

Ejecución:

- `npm run build`

Qué revisar:

- ✅ Compila sin errores.
- ✅ Timeline ordenada y clara.
- ✅ `complexityLevel` persistido (chip/etiqueta).

Entregables:

- `RetrievalService.ts`, `PromptChaining.ts`, `TimelineDecisiones.tsx`, actualización `AgentController.ts`.
```

---

## S3 — Plan: ruta y selección condicional de herramientas

```markdown
[HEADER GLOBAL — pegar arriba]

Objetivo:
Decidir **ruta** usando `routing_rules` (de `vitals_reference.json`) y **seleccionar herramientas** según `complexityLevel` (ahorra recursos: saltar LLM/EO en casos simples).

Implementa:

- `src/services/Router.ts`: `routeCase(ctx) -> { route: "rutinario"|"complejo"|"emergencia_inmediata", triageClass }` + eventos de Plan; añade “Routing” a Timeline.
- `src/agent/ToolRegistry.ts`: `selectTools(level, route)`:
  - simple → `RiskRulesTool` (solo reglas).
  - medio → `RetrievalTool` + LLM (Gemini), sin Evaluator.
  - complejo/emergencia → Retrieval + LLM + Evaluator.
- `src/ui/RouteBadge.tsx`: chip con ruta (alto contraste si emergencia).
- `src/agent/AgentController.ts`: persistir ruta y herramientas; **Plan = ok**.

Visibilidad inmediata:

- RouteBadge visible; **Plan = ok**; lista corta de herramientas elegidas.

Ejecución:

- `npm run build`

Qué revisar:

- ✅ Compila sin errores.
- ✅ Inputs simples → solo reglas.
- ✅ Inputs complejos → LLM (+ Evaluator si corresponde).

Entregables:

- `Router.ts`, `ToolRegistry.ts`, `RouteBadge.tsx`, actualización `AgentController.ts`.
```

---

## S4 — Act: decisión (reglas o Gemini) en JSON estricto con fuentes

```markdown
[HEADER GLOBAL — pegar arriba]

Objetivo:
Generar **decisión inicial**:

- **Simple** → atajo determinista (reglas, sin LLM), citando p.ej. `prot-rangos-normales`.
- **Medio/Complejo** → **Gemini** con plantilla determinista (JSON estricto + `fuentes_usadas` válidas).

Implementa:

- `src/services/AIClient.ts` (interfaz), `src/services/GeminiClient.ts` y `src/services/StubIAClient.ts`:
  - Leer `.env` → **API_GEMINI**.
  - Config determinista: `{ temperature: 0, topK: 1, topP: 0.1, candidateCount: 1, maxOutputTokens: 512 }` y MIME `application/json` si aplica.
- **Plantilla GEMINI — Clasificación** (ver sección “Plantillas GEMINI”).
- `src/services/TriageService.ts`: `classifyPatient(ctx, {useLLM})` → reglas o Gemini + añadir “Act” a Timeline y `AGENT_OUTPUT.decision`.
- `src/agent/AgentController.ts`: decidir `useLLM` con `selectTools`; marcar **Act = ok**.

Visibilidad inmediata:

- **Act = ok**; Timeline añade “Act”; JSON parcial muestra `decision`.

Ejecución:

- `npm run build`

Qué revisar:

- ✅ Compila sin errores.
- ✅ Caso simple: sin LLM, rápido, fuente válida.
- ✅ Medio/Complejo: LLM activo, JSON estricto y `fuentes_usadas` ⊂ `protocols.json`.

Entregables:

- `AIClient.ts`, `GeminiClient.ts`, `StubIAClient.ts`, `TriageService.ts`, actualización `AgentController.ts`.
```

---

## S5 — Reflect: Evaluator–Optimizer (límite de iteraciones y timeout)

```markdown
[HEADER GLOBAL — pegar arriba]

Objetivo:
Aplicar **segunda opinión** solo si procede. Evitar bucles con `maxIterations` (2–3), `confidenceThreshold` (~0.75), `timeoutMs` (~7000).

Implementa:

- **Plantilla GEMINI — Evaluación** (ver sección “Plantillas GEMINI”).
- `src/services/Evaluator.ts`:
  - `evaluateDecision(decision, ctx) -> { score, critiques[], needsMoreInfo }`
  - `refineDecision(decision, critiques)` cuando aplique.
  - Limitar por `maxIterations`, threshold y timeout; emitir eventos (EO); registrar en `AGENT_OUTPUT.evaluator`.
- `src/ui/InspectorEO.tsx`: panel colapsable con iteración/score/críticas/cambios; botón **Escalar a humano** (marca en salida y detiene).
- `src/agent/AgentController.ts`: activar EO **solo si** `useEvaluator` o `score<threshold`; **Reflect/EO = ok** (o escalado).

Visibilidad inmediata:

- **Reflect/EO = ok**; Timeline agrega iteraciones (si hubo).

Ejecución:

- `npm run build`

Qué revisar:

- ✅ Compila sin errores.
- ✅ EO se ejecuta solo si corresponde; termina ≤ `maxIterations` o por `timeout`.
- ✅ “Escalar a humano” se refleja en la salida.

Entregables:

- `Evaluator.ts`, `InspectorEO.tsx`, actualización `AgentController.ts`.
```

---

## S6 — Guards: verificador de salida + auditoría + severidades

```markdown
[HEADER GLOBAL — pegar arriba]

Objetivo:
Verificar **salida** con `vitals_reference.json` y `protocols.json`, y registrar **auditLog**. Señalizar con alto contraste (solo aquí).

Implementa:

- `src/services/Guards.ts` (salida): `verifyOutput(decision, ctx, sources)`:
  - Si `emergencias`: requiere evidencia fuerte/combinada + fuente válida.
  - Verificar que `fuentes_usadas` existan en `protocols.json`.
  - Si falla: `outputIssues.push(...)`, emitir `FLOW_ERROR` (`severity:"danger"`).
- `src/services/AuditLog.ts`: `append(runId, event, detail)` → agrega a `AGENT_OUTPUT.auditLog`.
- `src/ui/Alerta.tsx`: banner `warning|danger`.

Integración:

- Añadir “Guards” a Timeline; **Guards = ok** o **error**; mostrar `Alerta` y actualizar JSON.

Visibilidad inmediata:

- Banner de alerta si hay issues; estado de **Guards** reflejado.

Ejecución:

- `npm run build`

Qué revisar:

- ✅ Compila sin errores.
- ✅ Errores → `FLOW_ERROR` + `Alerta` visible.
- ✅ `guards` y `auditLog` actualizados.

Entregables:

- `Guards.ts` (salida), `AuditLog.ts`, `Alerta.tsx`, wiring controlador/timeline.
```

---

## S7 — Result: métricas, JSON final y replay sin red

```markdown
[HEADER GLOBAL — pegar arriba]

Objetivo:
Completar la **visualización** (badges conectados), calcular **métricas**, mostrar el **JSON final** y ofrecer **replay** sin red.

Implementa:

- `src/services/Metrics.ts`: medir `byNode[]`, `totalMs`, `iterationsEO`, `guardsTriggered`.
- `src/ui/PanelMetricas.tsx`: chips con métricas.
- `src/ui/VisorJSON.tsx`: render del **AGENT_OUTPUT** + “Copiar” / “Descargar .json”.
- `src/ui/BarraDemo.tsx`: toggles Presenter/Focus/Dark (opcional) + **Reproducir flujo** (eventos simulados, JSON demo).
- `src/agent/AgentController.ts`: `replayDemo()` para reproducir un run típico (sin depender de red).
- `src/ui/PanelFlujoAgente.tsx` (final): badges conectados (Observe → … → Result) con estado en tiempo real.
- `src/pages/AgentStudioPage.tsx`: layout final con todas las zonas.

Cierre:

- Completar `endedAt`, `metrics`, `auditLog`, `steps`; emitir `FLOW_COMPLETED`; habilitar copiar/descargar.

Visibilidad inmediata:

- **Result = ok**, métricas visibles, JSON final listo y **Replay** funcionando.

Ejecución:

- `npm run build`

Qué revisar:

- ✅ Compila sin errores.
- ✅ Flujo extremo a extremo visible.
- ✅ Métricas correctas y JSON descargable.
- ✅ Replay operativo sin conexión.

Entregables:

- `Metrics.ts`, `PanelMetricas.tsx`, `VisorJSON.tsx`, `BarraDemo.tsx`, `PanelFlujoAgente.tsx` (final), `AgentStudioPage.tsx` consolidada, `AgentController.ts` con `replayDemo`.
```
