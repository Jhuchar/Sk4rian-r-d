# Plan - Autoorientación y Normalización de Escaneos OMR

## Resumen ejecutivo
Este plan establece el flujo de ingeniería para la detección geométrica y orientación horaria de páginas OMR de forma atómica. La decisión técnica principal es ejecutar el proceso en dos fases consecutivas e independientes: primero, la detección de un cuadrilátero mediante anclas y rotación sin recortes en memoria; segundo, la carga de la plantilla OMR sobre la matriz ya alineada. No habrá límite de descarte de páginas (si un PDF de 100 páginas solo tiene una válida, esa única se procesa). Las dudas abiertas se centran en el almacenamiento futuro de las capturas individuales, ya que actualmente solo se preserva el PDF original por parte de Learnex.

## 1. Enfoque técnico (alto nivel)
El sistema aislará cada página del PDF en un búfer de imagen. Utilizando OpenCV, detectará las 4 anclas para formar un cuadrilátero y aplicará giros ortogonales (en el sentido de las agujas del reloj) hasta alinear la barra de orientación de forma recta (horizontal o vertical según la plantilla). Una vez estabilizada y derecha la imagen sin sufrir recortes ni distorsiones, se cargará la plantilla de lectura seleccionada para mapear los casilleros de marcas OMR de forma precisa.

## 2. Componentes / archivos afectados

* **DECISIÓN:** Crear un módulo de pre-procesamiento de imagen `OMRImageAligner` y modificar el pipeline de iteración de páginas `PDFPageIterator`.
* **POR QUÉ:** Al separar la corrección geométrica de la carga de la plantilla de lectura, garantizamos que el motor OMR trabaje siempre sobre una imagen normalizada estándar, eliminando la duplicidad de lógica.

### Componentes a crear o modificar:
1. `src/services/omr/OMRImageAligner.ts` (NUEVO): Contiene la lógica OpenCV. Detecta el cuadrilátero de anclas, calcula el sentido del giro horario y rota la imagen en memoria manteniendo las dimensiones intactas (sin recortar).
2. `src/pipelines/PDFPageIterator.ts` (MODIFICADO): Pipeline que divide el PDF hoja por hoja. Evaluará cada página de manera independiente. Si la página es válida, invoca a `OMRImageAligner` y luego al motor de plantillas; si falla, la salta de inmediato, acumula el error y continúa con la siguiente.
3. `src/models/OMRBatchReport.ts` (MODIFICADO): Actualización del esquema de datos y reportes para persistir contadores y listas explícitas: `total_paginas`, `paginas_procesadas` (IDs/números de página exitosos) y `paginas_descartadas` junto con su `motivo_descarte`.

## 3. Decisiones de arquitectura (mini-ADR)

* **DECISIÓN SELECCIONADA:** Pipeline secuencial desacoplado: Fase 1 (Giro geométrico puro sin deformación) -> Fase 2 (Inyección de plantilla OMR de Learnex).
* **ALTERNATIVA DESCARTADA:** Intentar recortar (*crop*) o reescalar la hoja al mismo tiempo que se detectan las anclas para encajarla a la fuerza en la plantilla.
* **POR QUÉ SE DESCARTÓ:** Modificar el tamaño o recortar los bordes puede alterar las coordenadas relativas de las burbujas configuradas en la plantilla original de Learnex, generando lecturas erróneas. Es más seguro rotar la hoja limpiamente en sentido horario de manera recta y dejar que la plantilla mapee sobre los píxeles recolocados.

## 4. Riesgos y dependencias
1. **Riesgo en la Detección de la Matriz (Tracer Bullet):** Que la biblioteca de OpenCV falle en calcular correctamente el cuadrilátero de las anclas debido a ruido en el escaneo, impidiendo determinar si la barra quedó en posición recta horizontal o vertical. *Mitigación:* Esta lógica se desarrollará y probará en aislamiento completo al inicio de la Fase 1.
2. **Dependencia de almacenamiento de imágenes:** Como Learnex guarda el PDF completo pero **no** las capturas o capturas corregidas individualmente por página, la auditoría dependerá exclusivamente de los logs de texto en base de datos. Si un tutor reclama un falso descarte, no habrá imagen guardada de la hoja corregida para comprobarlo visualmente.

## 5. Trazabilidad
* **US-1 (Corrección Geométrica)** → Se implementa en la Fase 1 del desarrollo dentro de `OMRImageAligner.ts` mediante transformaciones de rotación horaria.
* **US-2 y US-3 (Aislamiento y Reporte sin Límites)** → Se implementa en `PDFPageIterator.ts` y se refleja en `OMRBatchReport.ts`, asegurando la separación estricta entre páginas procesadas con éxito y páginas descartadas (sin importar si es 1 o son 100).
* **US-4 (Clasificación de Descartes)** → Capturado en el flujo individual por página del pipeline, mapeando códigos de descarte legibles al reporte final del lote.

## 6. Cronograma de Tareas (Checklist de Implementación)

### Fase 1: El Tracer Bullet (Incertidumbre Primero)
* [ ] **Tarea 1.1:** Instalar y configurar bindings de procesamiento de imágenes en el entorno del proyecto.
* [ ] **Tarea 1.2:** Crear script aislado de prueba para detectar las 4 anclas de esquina y formar el cuadrilátero en memoria.
* [ ] **Tarea 1.3:** Implementar el algoritmo de rotación horaria (0°, 90°, 180°, 270°) buscando dejar la barra de orientación de forma recta (horizontal/vertical) sin recortar la imagen.
* [ ] **Tarea 1.4:** Validar matemáticamente que las coordenadas post-rotación de una hoja de muestra coincidan con la orientación esperada.

### Fase 2: Aislamiento del Pipeline e Integración
* [ ] **Tarea 2.1:** Modificar `PDFPageIterator.ts` para extraer páginas de manera totalmente independiente y envolver cada una en bloques de control de excepciones.
* [ ] **Tarea 2.2:** Integrar el servicio de la Fase 1 en el flujo: si la página es orientada con éxito, proceder inmediatamente a cargar la plantilla seleccionada para la lectura de marcas.
* [ ] **Tarea 2.3:** Asegurar que si una página falla, el bucle registre el descarte y pase a la siguiente sin detener el procesamiento del PDF (incluso en PDFs mixtos o con 99% de hojas inválidas).

### Fase 3: Reportes y Auditoría de Datos
* [ ] **Tarea 3.1:** Actualizar el modelo/tabla de reportes de lotes (`OMRBatchReport`) para incluir los campos estructurados de páginas procesadas y páginas descartadas.
* [ ] **Tarea 3.2:** Conectar la salida del pipeline para que guarde el conteo exacto y los motivos tipificados en el reporte final del lote.
* [ ] **Tarea 3.3:** Ejecutar pruebas de extremo a extremo con PDFs reales simulando errores de escaneo para validar la consistencia del reporte.