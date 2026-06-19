# Constitution - System OMR

## Propósito

System OMR existe para procesar hojas ópticas previamente scaneadas en formato pdf de forma confiable, garantizando que los resultados obtenidos representen fielmente las marcaciones realizadas por los usuarios y permitiendo la trazabilidad completa del proceso.

---

# Principios Fundamentales

## 1. Exactitud sobre Rendimiento

La exactitud de las marcaciones detectadas tiene prioridad sobre el rendimiento del sistema.

Ninguna optimización, mejora de velocidad o reducción de recursos podrá implementarse si compromete la confiabilidad de los resultados obtenidos.

---

## 2. Validación Temprana

Toda hoja deberá superar las validaciones mínimas requeridas antes de iniciar su procesamiento.

Las hojas que no contengan los puntos de referencia necesarios para su alineación y lectura deberán ser descartadas inmediatamente, registrando la causa del descarte.

---

## 3. Trazabilidad Completa

Todo procesamiento deberá generar un registro que permita identificar el resultado de cada página evaluada.

El sistema deberá conservar información suficiente para determinar posteriormente:

* Qué páginas fueron procesadas.
* Qué páginas fueron descartadas.
* A qué procesamiento pertenecen.
* La razón de cualquier descarte realizado.

---

## 4. Respeto de las Reglas de la Plantilla

Las respuestas serán evaluadas exclusivamente según las reglas definidas por la plantilla correspondiente.

Cuando una marcación no cumpla las condiciones establecidas, deberá registrarse como inválida o incorrecta.

El sistema no realizará inferencias ni correcciones automáticas sobre las respuestas detectadas.

---

## 5. Simplicidad Orientada a Resultados

El objetivo principal del sistema es entregar resultados confiables.

La información presentada al usuario deberá priorizar:

* Resultados obtenidos.
* Páginas procesadas.
* Páginas descartadas.

Se evitará incorporar complejidad que no aporte valor directo al proceso de validación.

---

## 6. Auditabilidad Explicable

Toda decisión tomada durante el procesamiento deberá poder explicarse posteriormente.

Las páginas descartadas deberán registrar la causa específica del descarte para permitir revisiones y validaciones posteriores.

Ningún descarte deberá producirse sin una razón identificable.

---

## 7. Integridad de Resultados

La hoja óptica procesada constituye la fuente de verdad del sistema.

Una vez concluido el procesamiento, los resultados generados no deberán ser modificados manualmente.

El sistema debe reflejar las marcaciones detectadas y no interpretaciones posteriores realizadas por operadores o administradores.

---

## 8. Consistencia Entre Plantillas

Los principios definidos en esta constitución deberán aplicarse de manera uniforme a todas las plantillas soportadas por System OMR.

Las plantillas podrán definir diferentes zonas de captura, opciones de respuestas y orden, pero no modificarán las reglas fundamentales de validación, trazabilidad, auditoría e integridad.

---

## 9. Validación

Ante cualquier duda sobre la validez de una hoja, el sistema deberá priorizar la confiabilidad de los resultados.

Es preferible descartar una hoja para revisión que generar resultados potencialmente incorrectos.

La protección de la integridad de los resultados tiene prioridad sobre la maximización del número de páginas procesadas.

---

## 10. Independencia de Procesamiento

Cada página deberá evaluarse de manera independiente.

Los errores, descartes o fallos detectados en una página no deberán afectar el procesamiento de las demás páginas pertenecientes al mismo documento.

---

# Stack Tecnológico Aprobado

## Motor OMR

- Python + OpenCV (FIXED, no cambia sin aprobación del Tech Lead)
- Detección de puntos de referencia (anclas de esquina)
- Cálculos de orientación y rotación
- Análisis de imágenes: solo geometría clásica

## Pipeline

- Node.js + TypeScript
- Iteración de páginas PDF
- Integración con motor Python via subprocess

## Contenedores

- Docker para levantamiento de servicios
- Logs con timestamp por página
- Formato estructurado (JSON)

## Prohibido

- ML frameworks para rotación/detección (TensorFlow, PyTorch, etc.)
- Nuevas dependencias sin aprobación del Tech Lead
- Bases de datos adicionales
- Almacenamiento permanente de imágenes individuales

---

# Estándares de Código

## Patrones de Diseño (Obligatorios)

### Strategy

Para algoritmos de rotación.
Permite cambiar OpenCV por otra librería sin modificar el pipeline.
Ejemplo: DifferentRotationStrategy, RotationDetector.

### Pipeline

Para secuencia de procesamiento.
Claridad y extensibilidad en flujo: detección → orientación → lectura → reporte.
Ejemplo: OMRExecutionPipeline.

### Factory

Para crear procesadores según template.
Facilita soporte para versiones futuras (v4, v5) sin modificar código existente.
Ejemplo: TemplateProcessorFactory.

## Solid Principles

- SRP: Cada clase con una responsabilidad
- OCP: Abierto a extensión, cerrado a modificación
- LSP: Subtipos sustituibles
- ISP: Interfaces específicas
- DIP: Depender de abstracciones

## Manejo de Errores

- Códigos tipificados por tipo de fallo
- Excepciones controladas por página
- No detener procesamiento completo por error individual

---

# Restricciones de Rendimiento

## Por Página

- Detección puntos de referencia: < 500ms
- Orientación/rotación: < 500ms
- Total orientación: < 1s por hoja

## Por Lote

- Orientación no debe superar 20% del tiempo total
- El 80% restante es procesamiento OMR y reporte

## Medición

- Logs con timestamp por página en Docker
- Formato JSON: { "pagina": 1, "tiempo_orientacion_ms": 350, "rotacion": 90, "puntos_detectados": 4 }

---

# Restricciones de Seguridad

- No exponer credenciales en código
- No logs con datos sensibles de estudiantes
- Validación de inputs en cada capa
- Imágenes no se almacenan permanentemente (solo PDF original)

---

# Cumplimiento

Toda nueva funcionalidad, corrección o mejora deberá respetar los principios establecidos en esta constitución.

Ninguna especificación, plan de implementación o cambio futuro podrá contradecir estas reglas fundamentales.
