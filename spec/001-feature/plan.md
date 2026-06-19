# Plan - Autoorientación y Normalización de Escaneos OMR

## Resumen ejecutivo

Este plan establece el flujo de ingeniería para orientar páginas OMR de forma atómica y determinística, antes de la lectura de marcas. La decisión técnica principal es ejecutar el proceso en **dos fases consecutivas y desacopladas**: primero la corrección geométrica (detección de un cuadrilátero por anclas + giro ortogonal sin recortes en memoria); segundo, la carga de la plantilla OMR sobre la matriz ya alineada. La orientación se resuelve con **geometría, no con Machine Learning**, para que cada corrección sea auditable y explicable (Constitución Art. 6 y NFR-4). No hay límite de descarte: si un PDF de 100 páginas tiene una sola válida, esa única se procesa (fail-closed, Art. 9).

Duda abierta crítica: **AC-1.4 pide corregir inclinación (skew sub-90°), pero el Scope y esta arquitectura solo cubren rotación ortogonal** — corregir skew exige deformar la imagen, lo que el ADR-1 prohíbe. Requiere decisión del PO antes de codear. También queda abierto el almacenamiento de las capturas por página (hoy Learnex solo preserva el PDF original).

---

## 1. Enfoque técnico (alto nivel)

El sistema aísla cada página del PDF en un búfer de imagen. Con OpenCV detecta las 4 anclas de esquina para formar un cuadrilátero y aplica giros ortogonales (sentido horario) hasta dejar la barra de orientación recta (horizontal o vertical según la plantilla). Una vez la imagen está derecha y estabilizada, **sin recortes ni distorsiones**, se carga la plantilla de lectura para mapear los casilleros OMR con precisión. Cada página se evalúa de forma independiente: si falta evidencia geométrica (anclas o barra), se descarta con motivo tipificado y el bucle continúa con la siguiente.

---

## 2. Componentes / archivos afectados

- `src/services/omr/OMRImageAligner.ts` **(NUEVO)**: lógica OpenCV. Detecta el cuadrilátero de anclas (B1), calcula el sentido del giro horario y rota la imagen en memoria manteniendo las dimensiones intactas (sin recortar). Resuelve la orientación con la barra física (B2).
- `src/pipelines/PDFPageIterator.ts` **(MODIFICADO)**: divide el PDF hoja por hoja y evalúa cada página de forma independiente. Si es válida invoca a `OMRImageAligner` y luego al motor de plantillas; si falla, acumula el motivo y continúa sin detener el lote.
- `src/models/OMRBatchReport.ts` **(MODIFICADO)**: persiste contadores y listas explícitas: `total_paginas`, `paginas_procesadas`, `paginas_descartadas` con su `discarded_reason`, y la metadata de corrección (`rotation_label`) por página para auditoría.
- **Lector OMR / motor de plantillas** (existente): NO se modifica; recibe siempre una imagen ya normalizada.

---

## 3. Decisiones de arquitectura (mini-ADR)

### ADR-1 · Rotar limpio vs. recortar/reescalar para encajar

- **DECISIÓN:** Pipeline secuencial desacoplado — Fase 1 (giro geométrico puro sin deformación) → Fase 2 (inyección de la plantilla OMR sobre los píxeles recolocados).
- **POR QUÉ:** Modificar el tamaño o recortar bordes altera las coordenadas relativas de las burbujas configuradas en la plantilla original → lecturas erróneas (NFR-1). Es más seguro rotar la hoja limpiamente y dejar que la plantilla mapee.
- **ALTERNATIVA DESCARTADA:** Recortar (*crop*) o reescalar la hoja al detectar las anclas para encajarla a la fuerza en la plantilla. Se descartó por el riesgo de corromper las coordenadas de las marcas.

### ADR-2 · Geometría determinística vs. clasificador ML

- **DECISIÓN:** Estimar la orientación con geometría sobre anclas + barra (giro ortogonal calculado), no con un modelo entrenado.
- **POR QUÉ:** Cada corrección queda justificada por la posición de las marcas: reproducible, explicable y sin dataset (Art. 6 Auditabilidad, Art. 4 No inferir, NFR-4).
- **ALTERNATIVA DESCARTADA:** Un clasificador de ML que prediga la rotación de la imagen. Se descartó por ser caja negra no auditable, introducir inferencia probabilística donde se exige evidencia, y requerir datos etiquetados inexistentes.

### ADR-3 · Fail-closed ante ambigüedad

- **DECISIÓN:** Sin evidencia suficiente (faltan anclas o barra) → descartar con motivo, nunca adivinar.
- **POR QUÉ:** Prioriza la integridad del resultado sobre el volumen procesado (Art. 9, NFR-4). Un acierto incorrecto produce resultados corruptos indetectables; un descarte es trazable.
- **ALTERNATIVA DESCARTADA:** Best-effort (aplicar la orientación más probable). Descartada por riesgo de lecturas silenciosamente erróneas.

---

## 4. Riesgos y dependencias

1. **Detección de la matriz (Tracer Bullet):** OpenCV podría fallar al calcular el cuadrilátero por ruido del escaneo. *Mitigación:* la lógica se desarrolla y prueba en aislamiento al inicio de la Fase 1 (TC-003, TC-006).
2. **Almacenamiento de imágenes:** Learnex guarda el PDF completo pero **no** las capturas corregidas por página; la auditoría depende solo de logs de texto. Si un tutor reclama un falso descarte, no hay imagen para verificarlo visualmente — tensión con Art. 6.
3. **Inconsistencia AC-1.4 (inclinación):** el plan corrige rotación ortogonal, no skew fino; AC-1.4 pide corregir inclinación. *Acción:* el PO debe acotar AC-1.4 a rotación ortogonal (moviendo el deskew al FUERA) o abrirlo como feature aparte. Sin esto, AC-1.4 queda sin plan ni test.
4. **Plantillas sin barra de orientación:** las 4 esquinas son simétricas; sin la barra, 0° vs. 180° es indistinguible. *Mitigación:* fail-closed (ADR-3) y exigir barra en la plantilla v3.
5. **Dependencia de OpenCV** para la transformación geométrica.

---

## 5. Trazabilidad: cada US del spec → dónde se implementa

- **US-1 — AC-1.1/1.2/1.3** (rotación 90/180/270) → `OMRImageAligner.ts` Fase 1 (ADR-1, ADR-2). Pruebas: TC-007, TC-001, TC-008.
- **US-1 — AC-1.4** (inclinación) → ⚠️ **sin implementación**; pendiente de decisión del PO (ver Riesgo 3).
- **US-2 — AC-2.1/2.2** (aislamiento por página) → `PDFPageIterator.ts` (NFR-3). Pruebas: TC-005, TC-018.
- **US-3 — AC-3.1/3.2** (auditoría de correcciones) → `OMRBatchReport.ts` + vista de detalle del lote. Pruebas: TC-011, TC-005.
- **US-4 — AC-4.1/4.2** (descartes y su motivo) → `PDFPageIterator.ts` + `OMRBatchReport.ts` (ADR-3). Pruebas: TC-004, TC-010, TC-017.

---

## 6. Cronograma de tareas (checklist de implementación)

### Fase 1 · El Tracer Bullet (la incertidumbre primero)
- [ ] **1.1** Instalar y configurar los bindings de procesamiento de imágenes (OpenCV) en el entorno.
- [ ] **1.2** Script aislado para detectar las 4 anclas de esquina y formar el cuadrilátero en memoria (B1).
- [ ] **1.3** Algoritmo de rotación horaria (0/90/180/270) dejando la barra recta, sin recortar (B2).
- [ ] **1.4** Validar que las coordenadas post-rotación de una hoja de muestra coincidan con lo esperado (post-warp < 5 px).

### Fase 2 · Aislamiento del pipeline e integración
- [ ] **2.1** Modificar `PDFPageIterator.ts` para extraer páginas de forma independiente con control de excepciones por página.
- [ ] **2.2** Integrar `OMRImageAligner`: si la página se orienta con éxito, cargar la plantilla y leer.
- [ ] **2.3** Si una página falla, registrar el descarte y continuar (PDFs mixtos o con 99% inválidas).

### Fase 3 · Reportes y auditoría
- [ ] **3.1** Actualizar `OMRBatchReport` con páginas procesadas y descartadas (motivo tipificado, `rotation_label`).
- [ ] **3.2** Conectar la salida del pipeline al reporte final del lote.
- [ ] **3.3** Pruebas end-to-end con PDFs reales simulando errores de escaneo.
