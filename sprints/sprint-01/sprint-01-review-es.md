# Sprint 01 — Review y Retrospectiva

**Período:** 13/04/2026 → 24/04/2026  
**Fecha:** 27/04/2026  
**Hora:** 15:20  
**Lugar:** Aula 209  
**Asistentes:** Asier Barranco  

---

## 1. Objetivo del Sprint — Revisión

El objetivo del Sprint 1 era completar toda la documentación fundacional e iniciar el despliegue técnico de la infraestructura on-premise (Windows Server) y la infraestructura AWS.

El sprint tuvo un carácter predominantemente documental. Todos los documentos fundacionales fueron producidos y commiteados al repositorio. Las tareas de despliegue técnico se iniciaron pero no se completaron dentro del sprint.

---

## 2. Tablero al Final del Sprint

![Sprint 01 Tablero final](../../media/sprint-01-end-board.png)

### 2.1 Tareas Completadas (Done)

Se ha completado con éxito el 100% de la documentación bilingüe planificada (Inglés/Español). Con todas las dependencias de la rúbrica resueltas y subidas al repositorio, el proyecto queda ahora posicionado para centrarse exclusivamente en el despliegue técnico de la infraestructura

### 2.2 Tareas Incompletas — Trasladadas al Sprint 2

| # | Tarea | Motivo |
|---|---|---|
| 11 | Despliegue de infraestructura AWS | No iniciada — tiempo del sprint consumido por la documentación |
| 12 | Despliegue servidor Windows Server on-premise | Iniciada parcialmente — instalación de AD DS en curso |
| 13 | Acta de Sprint 01 Review y Retrospectiva | Producida el 27/04, primer día del Sprint 2 |

---

## 3. Cambio de Calendario — Actualización de la Estructura de Sprints

Durante esta revisión de sprint se formalizó un cambio en el calendario del proyecto debido al calendario escolar.

**Estructura original:**

| Sprint | Período |
|---|---|
| Sprint 1 | 13/04/2026 → 24/04/2026 |
| Sprint 2 | 27/04/2026 → 08/05/2026 |
| Sprint 3 | 11/05/2026 → 15/05/2026 |

**Estructura actualizada:**

| Sprint | Período |
|---|---|
| Sprint 1 | 13/04/2026 → 24/04/2026 |
| Sprint 2 | 27/04/2026 → 12/05/2026 |

El Sprint 3 queda suprimido y absorbido por el Sprint 2, que se amplía en consecuencia. La defensa del proyecto ha sido confirmada para el **20/05/2026**, lo que proporciona tiempo adicional tras el cierre del Sprint 2 (12/05) para pruebas finales y preparación de la presentación de defensa. Estas actividades no requieren un sprint formal y se gestionarán de forma informal en esa ventana.

Todos los documentos del proyecto que hacían referencia a una estructura de tres sprints han sido actualizados para reflejar este cambio.

---

## 4. Retrospectiva

### Qué ha ido bien
- Toda la documentación fundacional se completó con un nivel de calidad alto dentro del sprint
- El stack tecnológico fue decidido y justificado completamente mediante el análisis de mercado
- El diagrama de arquitectura alcanzó un estado final y validado
- La estructura del repositorio quedó establecida y los commits se realizaron de forma consistente

### Qué no ha ido bien
- El sprint tuvo un carácter más documental de lo previsto, dejando tiempo insuficiente para el despliegue técnico
- El despliegue del Windows Server fue subestimado en tiempo — la configuración del entorno (red VirtualBox, instalación del SO) tomó más tiempo del planificado

### Qué cambiará en el Sprint 2
- Las tareas técnicas tienen prioridad desde el primer día
- La documentación se produce en paralelo al despliegue, no antes
- El despliegue de infraestructura AWS y la configuración de AD on-premise son las dos primeras tareas a cerrar

---

## 5. Previsión del Sprint 2

El Sprint 2 transcurre del **27/04/2026 al 12/05/2026**. Hereda las dos tareas incompletas del Sprint 1 y cubre la implementación técnica completa de las tres capas de la arquitectura, incluyendo federación de identidad, configuración perimetral, servicios corporativos y la fase de seguridad Purple Team.