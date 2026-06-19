# Equipo R1 - Feature Autoorientacion OMR v3

## Integrantes

- Jhulino Ventura Ramirez - Product (PO), dueno de `spec.md`.
- Martin Alonso Centeno Leon - Tech Lead, dueno de `plan.md`.
- Chen Yang Lin Chiu - QA Tester, dueno de `test-cases.md`.

## Feature

La feature utiliza los 4 puntos de referencia para detectar y validar la ubicación de la hoja, mientras que la barra de orientación física determina su posición correcta. Con esta información, el sistema orienta automáticamente la página antes de continuar con el procesamiento. La feature prioriza exactitud, descarte conservador y trazabilidad por página.

## Artefactos

- `constitution.md` - Fase 0: principios no negociables.
- `spec.md` - Fase 1: que se construye y por que.
- `plan.md` - Fase 2: enfoque tecnico y decisiones.
- `test-cases.md` - Fase 3: casos QA, error, borde y contrato.

---

## Coverage Matrix (Gate de consistencia · §2.7)

Mapea cada requisito de la spec a su lugar en el plan y a su(s) caso(s) de prueba. Los huecos quedan visibles a propósito: una fila sin plan o sin test es una señal, no un error que ocultar.

| Requisito (US / AC / NFR) | ¿En el plan? | ¿Tiene caso de prueba? | Estado |
| --- | --- | --- | --- |
| US-1 / AC-1.1 (rotación 90°) | Sí → `OMRImageAligner` Fase 1 | TC-007 | ✅ |
| US-1 / AC-1.2 (rotación 180°) | Sí → `OMRImageAligner` Fase 1 | TC-001 | ✅ |
| US-1 / AC-1.3 (rotación 270°) | Sí → `OMRImageAligner` Fase 1 | TC-008 | ✅ |
| **US-1 / AC-1.4 (inclinación/skew)** | **No** (ADR-1 prohíbe deformar) | **No hay TC** | 🔴 **HUECO** |
| US-2 / AC-2.1 (evaluación por página) | Sí → `PDFPageIterator` | TC-005, TC-018 | ✅ |
| US-2 / AC-2.2 (válidas conservan resultado) | Sí → `PDFPageIterator` | TC-005, TC-018 | ✅ |
| US-3 / AC-3.1 (registrar corrección) | Sí → `OMRBatchReport` | TC-011, TC-005 | ✅ |
| US-3 / AC-3.2 (identificar páginas corregidas en el lote) | Sí → `OMRBatchReport` / vista de lote | TC-011 | 🟡 parcial |
| US-4 / AC-4.1 (descarte sin evidencia) | Sí → `PDFPageIterator` (ADR-3) | TC-004, TC-010 | ✅ |
| US-4 / AC-4.2 (motivo de descarte visible) | Sí → `OMRBatchReport` | TC-005, TC-010, TC-017 | ✅ |
| NFR-1 (no alterar marcas) | Sí → validación post-warp | TC-006, TC-019 | ✅ |
| NFR-2 (registrar corrección para auditoría) | Sí → `OMRBatchReport` | TC-011 | ✅ |
| NFR-3 (aislamiento entre páginas) | Sí → `PDFPageIterator` | TC-018 | ✅ |
| NFR-4 (no inferir sin evidencia) | Sí → ADR-3 fail-closed | TC-002, TC-020 | ✅ |
| NFR-5 (respeta la constitución) | Sí → todo el plan | TC-020 | ✅ |
| _Contrato backend v3 (template_version)_ | Implícito en pipeline | TC-012, TC-013, TC-014, TC-015, TC-016 | ⚠️ no pedido por la spec |
| _NFR-6 rendimiento <1s_ | No declarado en spec | TC-021 | ⚠️ scope creep (test sin requisito) |
| _NFR-7 simplicidad_ | No declarado en spec | TC-022 | ⚠️ scope creep (test sin requisito) |
| _NFR-8 consistencia templates_ | No declarado en spec | TC-023 | ⚠️ scope creep (test sin requisito) |

**Lectura de la matriz:**
- 🔴 **AC-1.4 es un hueco real:** la spec lo pide, pero ni el plan ni los tests lo cubren, y contradice el ADR-1 (rotar sin deformar). Decisión pendiente del PO: acotarlo a rotación ortogonal (moverlo al FUERA) o abrirlo como feature aparte.
- ⚠️ **Scope creep en pruebas:** TC-021/022/023 validan NFR-6/7/8 que la spec no declara (solo llega a NFR-5). O se suben esos NFR a la spec, o se quitan los tests.
- 🟧 **Drift de terminología:** la matriz QA renumera los AC (su "AC-2.1" = masa de tinta) frente a la spec (AC-2.1 = evaluación por página). Hay que unificar la numeración spec ↔ test.

---

## Gate de claridad (§2.8) · Veredicto

Revisión cruzada de la spec por las 4 categorías. _(Intercambio con el grupo: `<COMPLETAR — grupo revisor>`.)_

| Categoría | Resultado | Hallazgo |
| --- | --- | --- |
| 🟦 Completitud | 🟡 | Cada US tiene criterios y hay casos borde; falta cubrir AC-1.4 (inclinación) con plan y test. |
| 🟨 Claridad | 🟢 | Criterios observables con datos concretos (umbral de masa 0.12, contraste ≥ 2.0, post-warp < 5 px). |
| 🟧 Consistencia | 🔴 | Drift de numeración de AC entre spec y test; NFR-6/7/8 aparecen en tests pero no en la spec. |
| 🟩 Testabilidad | 🟢 | Cada AC con plan tiene TC con entrada → resultado esperado; los huecos están marcados, no maquillados. |

**Veredicto global: 🟡 — pasa con avisos.** Bloqueante a resolver antes de codear: la decisión sobre **AC-1.4** y unificar la numeración de AC spec ↔ test. El resto son avisos no bloqueantes.
