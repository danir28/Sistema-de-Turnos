# Skill Observation Log

Observations captured during task-oriented work. Each entry identifies a
potential skill improvement or new skill opportunity.

**Status key:** OPEN = not yet actioned | ACTIONED = skill updated/created |
DECLINED = user decided not to pursue

---

## 2026-09-14

### Observation 1: Matriz RF↔CU no detecta contradicciones con secciones de "fuera de alcance"

**Status:** OPEN
**Date:** 2026-09-14
**Session context:** El usuario pidió revisar los 17 Casos de Uso de Sistema-de-Turnos contra Requerimientos.md y reportar diferencias en ambos sentidos. Ya existía `Docs/Business/Trazabilidad-RF-CU.md` con una matriz RF→CU previa.
**Skill:** New skill candidate: "matching-documentacion-vs-especificacion" (o extender una guía existente de trazabilidad)
**Type:** open-source
**Phase/Area:** Verificación de consistencia entre documentos de negocio y especificaciones técnicas derivadas

**Issue:** La matriz de trazabilidad RF↔CU existente cruza cada requerimiento funcional contra el CU que lo implementa, y detecta bien los huecos de cobertura (RF sin CU, CU sin RF). Pero no cruza los CU contra la sección "Limitaciones y exclusiones (fuera de alcance)" del documento de requerimientos. Al releer Requerimientos.md línea por línea junto con los 17 CU, encontré que CU-17 (Enviar Recordatorio de Turno) usa "WhatsApp o email" como canal de ejemplo, mientras que la sección 1.2 de Requerimientos.md excluye explícitamente "Notificaciones automáticas por WhatsApp, ya que requieren integración con WhatsApp" para esta versión. Una matriz RF→CU tabular no captura este tipo de contradicción porque el ítem excluido no tiene un ID (RF/RN) contra el cual cruzarse — solo aparece como texto libre en la sección de alcance.

**Suggested improvement:** Cuando se pida "hacer matching" o "verificar consistencia" entre un documento de requerimientos/alcance y especificaciones derivadas (CU, historias de usuario, tickets), el proceso debe incluir un paso explícito de cruzar cada CU/especificación contra la sección de exclusiones/fuera-de-alcance del documento fuente, no solo contra los requerimientos positivos (RF/RN) que tienen ID. Las exclusiones son texto libre y requieren lectura completa, no solo grep de IDs.

**Principle:** Un documento de trazabilidad tabular (ID↔ID) es ciego a las contradicciones con texto libre (como una sección de "fuera de alcance"). La verificación de consistencia entre documentos siempre debe incluir una relectura completa de ambos documentos, no solo cruzar los identificadores formales — las contradicciones más costosas de encontrar en producción suelen esconderse justo en las partes sin ID (notas al margen, exclusiones, aclaraciones pendientes tipo "ver con fulano").
