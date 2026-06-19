# Test Cases - Motor de Orientacion v3

## Resumen ejecutivo

Como QA, valido el motor v3 desde sus resultados observables: los 4 puntos de referencia deben localizar y orientar la hoja, la barra fisica debe resolver la orientacion, y cualquier falta de evidencia debe terminar en descarte explicable. La suite cubre B1, B2, rotaciones 0/90/180/270, contraste critico, contrato `template_version: 3`, pass-through de `orientationBar`, paginas mixtas y compatibilidad de `discarded_reason`. Queda fuera validar cada pixel del algoritmo OpenCV; se valida que el comportamiento final cumpla la spec, el plan y la constitucion.

---

## 1. Evidencia revisada

La feature se viene resolviendo en OMR-system con commits reales. Estos son los hitos usados para disenar las pruebas:

| Commit | Fecha | Evidencia QA |
| --- | --- | --- |
| `d4b7851` | 2026-06-08 | Planificacion SDD del rediseno de deteccion Puntos de Referencia |
| `6274adb` | 2026-06-08 | Red de seguridad para redisenar Puntos de Referencia con fixtures 90/270/missing marker |
| `1202abe` | 2026-06-08 | Rediseno de deteccion Puntos de Referencia a una sola pasada |
| `d7d10b0` | 2026-06-09 | Contrato `template-v3-orientation-bar` y retiro del fast-path anterior |
| `09d30ef` | 2026-06-09 | Motor v3: Puntos de Referencia localizan/cortan, barra orienta |
| `b0649c3` | 2026-06-09 | Persistencia de `orientation_bar` en template v3 |
| `ce48223` | 2026-06-09 | Editor v3 con barra de orientacion y persistencia |
| `8436cd2` | 2026-06-10 | Orientacion por barra robusta para hojas invertidas 180 grados |
| `917f919` | 2026-06-10 | Orientacion robusta para 0/90/180/270 |
| `b4f0598` | 2026-06-13 | Exposicion de razon de paginas descartadas en resultados |

---

## 2. Datos de prueba

| ID | Descripcion |
| --- | --- |
| TPL-V3-OK | Template con `template_version: 3`, 4 `corner_positions`, `orientation_bar` valida y N zonas OMR |
| TPL-V3-NO-BAR | Template v3 sin `orientation_bar` |
| TPL-V3-BAR-0 | Template v3 con `orientation_bar.width_pct <= 0` o `height_pct <= 0` |
| TPL-V2 | Template con `template_version: 2` |
| PDF-0 | Hoja v3 correcta, sin rotacion |
| PDF-90 | Misma hoja v3 rotada 90 grados |
| PDF-180 | Misma hoja v3 rotada 180 grados |
| PDF-270 | Misma hoja v3 rotada 270 grados |
| PDF-MISSING-REF | Hoja donde se detectan menos de 4 Puntos de Referencias |
| PDF-MISSING-BAR | Hoja con 4 Puntos de Referencias detectables, pero sin barra donde el template la declara |
| PDF-MIX-5 | p1 normal, p2 rotada 90, p3 sin Puntos de Referencias suficientes, p4 rotada 180, p5 sin barra |

---

## 3. Casos de prueba

### TC-001 (AC-2.1, AC-2.2): Rotacion 180 grados por barra

Datos: `TPL-V3-OK` + `PDF-180`.

Pasos:
1. Procesar el PDF con motor v3.
2. Revisar la medicion de masa de tinta por borde.
3. Consultar el resultado de la pagina 1.

Esperado:
- El borde inferior se identifica como candidato top antes de normalizar.
- `best_fill` supera `V3_BAR_MIN_MASS = 0.12`.
- El contraste contra el borde opuesto es `>= 2.0`.
- `rotation_label` o metadata equivalente indica 180 grados.
- La imagen final queda con la barra arriba.

### TC-002 (AC-2.2, NFR-2): Contraste critico falla cerrado

Datos: escaneo claro donde la barra tiene masa `0.13` y el borde opuesto `0.08`.

Pasos:
1. Procesar con motor v3.
2. Calcular contraste `0.13 / 0.08`.
3. Revisar resultado de la pagina.

Esperado:
- El contraste calculado es `1.625`.
- Como `1.625 < 2.0`, la pagina se descarta.
- `discarded_reason` es `"missing_orientation_bar"`.
- No se leen zonas de respuesta.

### TC-003 (AC-1.1): B1 elige esquinas y descarta mancha central

Datos: imagen con 4 Puntos de Referencias en esquinas y una mancha cuadrada en el centro.

Pasos:
1. Ejecutar localizacion B1.
2. Revisar candidatos detectados por contorno.

Esperado:
- El sistema filtra candidatos cuadrados con aspect ratio entre `0.7` y `1.3`.
- Elige los 4 candidatos mas cercanos a las esquinas.
- Descarta la mancha central.
- Continua a B2 con 4 puntos validos.

### TC-004 (AC-1.1, AC-4.1): Sin Puntos de Referencias suficientes

Datos: `TPL-V3-OK` + `PDF-MISSING-REF`.

Pasos:
1. Procesar el PDF.
2. Consultar resultado de pagina 1.

Esperado:
- `status` es `"discarded"`.
- `discarded_reason` es `"missing_reference_markers"`.
- No se intenta resolver orientacion.
- `zones` es `[]`.

### TC-005 (AC-1.1, AC-2.1, AC-3.1): PDF mixto v3

Datos: `TPL-V3-OK` + `PDF-MIX-5`.

Pasos:
1. Procesar el PDF completo.
2. Revisar el resultado pagina por pagina.

Esperado:
- p1 procesa con orientacion 0 grados.
- p2 procesa con orientacion 90 grados.
- p3 se descarta por `"missing_reference_markers"`.
- p4 procesa con orientacion 180 grados.
- p5 se descarta por `"missing_orientation_bar"`.
- Las paginas descartadas no desplazan ni eliminan los resultados validos.

### TC-006 (NFR-1): Validacion post-warp

Datos: imagen que tras el warp queda desplazada 10 px.

Pasos:
1. Procesar con motor v3.
2. Ejecutar validacion post-warp.

Esperado:
- El sistema detecta desplazamiento mayor a 5 px.
- Falla con error de normalizacion geometrica o descarte controlado.
- No entrega respuestas OMR como si la hoja estuviera alineada.

### TC-007 (AC-1.1): Rotacion 90 grados

Datos: `TPL-V3-OK` + `PDF-90`.

Pasos:
1. Procesar el PDF.
2. Consultar resultado y metadata de pagina 1.

Esperado:
- La pagina no se descarta.
- La barra se usa para resolver orientacion.
- `rotation_label` o metadata equivalente indica 90 grados.
- Las zonas se leen en coordenadas canonicas.

### TC-008 (AC-2.1): Rotacion 270 grados

Datos: `TPL-V3-OK` + `PDF-270`.

Pasos:
1. Procesar el PDF.
2. Consultar resultado y metadata de pagina 1.

Esperado:
- La pagina no se descarta.
- La orientacion final permite leer las zonas correctamente.
- `rotation_label` o metadata equivalente indica 270 grados.

### TC-009 (AC-2.1, Art. 4): No usar ID Puntos de Referencia como fuente de orientacion

Datos: hoja v3 con 4 Puntos de Referencias detectables y barra visible.

Pasos:
1. Procesar la hoja.
2. Revisar logs o ruta de ejecucion.

Esperado:
- Los Puntos de Referencias se usan para localizar y recortar.
- La barra se usa para orientar.
- No se ejecuta el slow-path v2 de rotar y re-detectar Puntos de Referencias para inferir orientacion.

### TC-010 (AC-4.1, AC-4.2): Barra ausente con Puntos de Referencias presentes

Datos: `TPL-V3-OK` + `PDF-MISSING-BAR`.

Pasos:
1. Procesar el PDF.
2. Consultar resultado de pagina 1.

Esperado:
- Los 4 Puntos de Referencias se detectan.
- La pagina se descarta porque no hay barra confiable.
- `discarded_reason` es `"missing_orientation_bar"`.
- No se generan zonas de respuesta.

### TC-011 (AC-3.1): Auditoria v3 por pagina

Datos: `TPL-V3-OK` + `PDF-90`.

Pasos:
1. Procesar el PDF.
2. Consultar JSON/logs de salida.

Esperado:
- La salida incluye `rotation_label` o campo equivalente.
- La salida incluye tiempos o evidencia separada de B1 y B2.
- La evidencia esta asociada a la pagina fisica correcta.

### TC-012 (Contrato): Backend rechaza template que no sea v3

Datos: `TPL-V2`.

Pasos:
1. Solicitar procesamiento con `template_version: 2`.
2. Revisar respuesta del backend.

Esperado:
- El backend rechaza antes de encolar.
- El error indica que solo se soporta `template_version: 3`.
- No se lanza el subproceso Python.

### TC-013 (Contrato): Backend rechaza v3 sin `orientation_bar`

Datos: `TPL-V3-NO-BAR`.

Pasos:
1. Solicitar procesamiento con template v3 sin barra.
2. Revisar respuesta del backend.

Esperado:
- El backend rechaza antes de encolar.
- El error indica que falta la barra de orientacion.
- No se inventa un `orientationBar` en el manifest.

### TC-014 (Contrato): Backend rechaza barra degenerada

Datos: `TPL-V3-BAR-0`.

Pasos:
1. Solicitar procesamiento con barra de ancho o alto cero.
2. Revisar respuesta del backend.

Esperado:
- El backend rechaza antes de encolar.
- El error indica que la barra requiere dimensiones positivas.
- No se lanza Python.

### TC-015 (Contrato): Manifest propaga `orientationBar`

Datos: `TPL-V3-OK`.

Pasos:
1. Construir el manifest Node -> Python.
2. Inspeccionar campos enviados.

Esperado:
- El manifest incluye `template_version: 3`.
- El manifest incluye `orientationBar` con `x_pct`, `y_pct`, `width_pct`, `height_pct`.
- `orientationBar` es hermano de `templateCornerPositions`.
- `orientationBar` no aparece dentro de `zones`.

### TC-016 (Contrato): `orientationBar` no produce `ZoneResult`

Datos: `TPL-V3-OK` con N zonas de respuesta + `PDF-0`.

Pasos:
1. Procesar el PDF.
2. Contar zonas de respuesta.

Esperado:
- El resultado contiene exactamente N zonas de respuesta.
- No existe una zona adicional para la barra de orientacion.

### TC-017 (Compatibilidad): `discarded_reason` sigue siendo string libre

Datos: resultado real o simulado con `discarded_reason: "missing_orientation_bar"`.

Pasos:
1. Consumir el resultado desde API/UI.
2. Validar serializacion.

Esperado:
- `discarded_reason` se recibe como string.
- No falla por enum estricto.
- Valores existentes o futuros no rompen compatibilidad.

### TC-018 (NFR-3): Aislamiento entre paginas

Datos: `TPL-V3-OK` + `PDF-MIX-5`.

Pasos:
1. Procesar el PDF.
2. Revisar estado y resultados por pagina.

Esperado:
- El descarte de p3 y p5 no cambia el resultado de p1, p2 y p4.
- La numeracion de paginas no se reindexa tras descartes.
- El lote termina de forma controlada.

### TC-019 (NFR-1): Normalizacion no altera respuestas marcadas

Datos: `TPL-V3-OK` + `PDF-0`, `PDF-90`, `PDF-180`, `PDF-270` con las mismas respuestas.

Pasos:
1. Procesar los cuatro PDFs.
2. Comparar respuestas finales por zona.

Esperado:
- Las respuestas finales coinciden entre las cuatro orientaciones.
- La correccion geometrica no transforma una respuesta marcada en otra.

### TC-020 (NFR-5): Cumplimiento de constitution

Datos: `constitution.md` + cualquier escenario anterior.

Pasos:
1. Revisar que la solucion prioriza exactitud sobre rendimiento.
2. Revisar validacion temprana y trazabilidad de descartes.
3. Confirmar que los casos de prueba cubren fail-closed.

Esperado:
- Si falta evidencia geometrica, se descarta.
- Ninguna pagina queda sin estado explicable.
- La feature cumple separacion B1/B2 y fail-closed.

### TC-021 (NFR-6): Rendimiento orientación < 1s

Datos: PDF con 10 páginas mixtas (0°, 90°, 180°, 270°).

Pasos:
1. Procesar el PDF completo.
2. Medir tiempo de orientación por página (detección + rotación).
3. Registrar en logs con timestamp.

Esperado:
- Cada página: orientación < 1000ms.
- Total orientación ≤ 20% del tiempo total de procesamiento.
- Logs incluyen: { "pagina": N, "tiempo_orientacion_ms": X, "rotacion": Y, "puntos_detectados": Z }.

### TC-022 (NFR-7): Simplicidad verificable

Datos: Revisar archivos creados/modificados en el PR de la feature.

Pasos:
1. Contar archivos nuevos vs modificados.
2. Verificar que no se crearon abstracciones prematuras.

Esperado:
- Máximo 2 archivos nuevos (OMRImageAligner.ts).
- El resto son modificaciones de archivos existentes (PDFPageIterator.ts, OMRBatchReport.ts).

### TC-023 (NFR-8): Consistencia entre templates

Datos: Templates v3, v3 sin barra, v2.

Pasos:
1. Procesar con template v3 válido.
2. Intentar procesar con template v3 sin barra.
3. Intentar procesar con template v2.

Esperado:
- v3 válido: procesa correctamente.
- v3 sin barra: rechaza antes de encolar, error indica "falta barra de orientación".
- v2: rechaza antes de encolar, error indica "solo se soporta template_version: 3".

---

## 4. Matriz QA

| Requisito | Casos | Estado |
| --- | --- | --- |
| US-1 / AC-1.1 Localizacion B1 | TC-003, TC-004 | Cubierto |
| US-2 / AC-2.1 Masa de tinta por borde | TC-001, TC-005, TC-007, TC-008 | Cubierto |
| US-2 / AC-2.2 Contraste >= 2.0 | TC-001, TC-002, TC-010 | Cubierto |
| US-3 / AC-3.1 Auditoria v3 | TC-005, TC-011 | Cubierto |
| US-4 / AC-4.1 Descarte por referencias/barra | TC-004, TC-010 | Cubierto |
| US-4 / AC-4.2 Motivo de descarte visible | TC-005, TC-017 | Cubierto |
| NFR-1 Alineacion post-warp < 5px | TC-006, TC-019 | Cubierto |
| NFR-2 Masa minima 0.12 | TC-001, TC-002 | Cubierto |
| NFR-3 Aislamiento por pagina | TC-005, TC-018 | Cubierto |
| NFR-6 Rendimiento < 1s | TC-021 | Cubierto |
| NFR-7 Simplicidad | TC-022 | Cubierto |
| NFR-8 Consistencia templates | TC-023 | Cubierto |
| Contrato backend v3 | TC-012, TC-013, TC-014 | Cubierto |
| Manifest Node -> Python | TC-015, TC-016 | Cubierto |
| Constitution Art. 4 / Art. 7 | TC-009, TC-020 | Cubierto |
