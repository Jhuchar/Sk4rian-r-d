# Equipo R1 - Feature Autoorientacion OMR v3

## Integrantes

- Jhulino Ventura Ramirez - Product (PO), dueno de `spec.md`.
- Martin Alonso Centeno Leon - Tech Lead, dueno de `plan.md`.
- Chen Yang Lin Chiu - QA Tester, dueno de `test-cases.md`.

## Feature

Motor de autoorientacion v3 para OMR: los 4 ArUcos localizan y recortan la hoja
(B1), y la barra de orientacion fisica resuelve la rotacion (B2). La feature
prioriza exactitud, descarte conservador y trazabilidad por pagina.

## Artefactos

- `constitution.md` - Fase 0: principios no negociables.
- `spec.md` - Fase 1: que se construye y por que.
- `plan.md` - Fase 2: enfoque tecnico y decisiones.
- `test-cases.md` - Fase 3: casos QA, error, borde y contrato.

---

## Gate de consistencia

| Requisito | En el plan | Tiene caso de prueba | Estado |
| --- | --- | --- | --- |
| US-1 / AC-1.1 Localizacion B1 por contornos | Si, seccion 1 y 5: `_locate_markers_by_contours` | TC-003, TC-004 | OK |
| US-2 / AC-2.1 Masa de tinta por borde | Si, seccion 1: `_resolve_orientation_by_bar` | TC-001, TC-005, TC-007, TC-008 | OK |
| US-2 / AC-2.2 Contraste relativo >= 2.0 | Si, seccion 3: barra relativa vs umbral fijo | TC-001, TC-002, TC-010 | OK |
| US-3 / AC-3.1 Auditoria v3 | Si, seccion 5: `rotation_label` | TC-005, TC-011 | OK |
| Descarte `missing_reference_markers` | Si, B1 fail-closed y constitution Art. 7 | TC-004, TC-005 | OK |
| Descarte `missing_orientation_bar` | Si, B2 fail-closed y constitution Art. 4 | TC-002, TC-005, TC-010 | OK |
| NFR-1 Alineacion post-warp < 5px | Si, seccion de validacion post-warp | TC-006, TC-019 | OK |
| NFR-2 Masa minima de barra 0.12 | Si, `V3_BAR_MIN_MASS` | TC-001, TC-002 | OK |
| Contrato `template_version: 3` | Si, dependencias del motor v3 | TC-012, TC-013, TC-014 | OK |
| `orientationBar` no es zona de respuesta | Si, separacion B1/B2 y contrato manifest | TC-015, TC-016 | OK |
| Constitution Art. 4 / Art. 7 | Si, mini-ADR y limites fail-closed | TC-009, TC-020 | OK |

No se detectan requisitos de la spec sin plan o sin caso QA. No se detecta scope
creep bloqueante; los casos de contrato agregados por QA protegen la frontera
Node -> Python que el plan depende de forma implicita.

---

## Gate de claridad

Grupo revisor: pendiente de completar antes de entregar.

| Categoria | Estado | Nota |
| --- | --- | --- |
| Completitud | Pendiente | Revisar que cada AC mantenga dato observable |
| Claridad | Pendiente | Revisar terminos B1, B2, barra, ArUco y descarte |
| Consistencia | Pendiente | Confirmar que `rotation_label` y `discarded_reason` usen nombres finales |
| Testabilidad | Pendiente | Casos QA definidos con datos y esperado verificable |

Veredicto final: pendiente de grupo revisor.
