# Plan de Gestión de Riesgos

**Proyecto:** Zero Trust Corporate System  
**Autor:** Asier Barranco  
**Fecha:** 14/04/2026  
**Versión:** 1.0

---

## 1. Introducción

Este documento identifica los riesgos laborales y técnicos asociados al desarrollo de este proyecto, junto con las medidas preventivas definidas para minimizar su impacto. Se ha elaborado conforme a los requisitos del ciclo formativo y es de aplicación a todo el trabajo realizado durante el periodo del proyecto, tanto en el aula como en el entorno cloud.

---

## 2. Entorno de Trabajo

El proyecto se desarrolla en los siguientes contextos físicos y digitales:

- **Aula:** puesto de trabajo estándar con monitor, teclado y ratón. Trabajo en posición sedente durante períodos prolongados.
- **Entorno cloud (AWS):** administración remota mediante navegador y terminal SSH. No se manipula infraestructura física más allá del puesto local.
- **Virtualización on-premise (VirtualBox):** máquinas virtuales ejecutadas sobre el equipo local. No se requiere hardware adicional.

---

## 3. Identificación de Riesgos y Medidas Preventivas

### 3.1 Riesgos Laborales y Ergonómicos

| # | Riesgo | Probabilidad | Gravedad | Medidas Preventivas |
|---|--------|-------------|----------|---------------------|
| R01 | Trastornos musculoesqueléticos por postura sedente prolongada | Media | Media | Mantener postura correcta. Realizar pausas cortas cada 45–60 minutos. Ajustar la altura de la silla y la distancia al monitor. |
| R02 | Fatiga visual por exposición prolongada a la pantalla | Media | Baja | Aplicar la regla 20-20-20: cada 20 minutos, fijar la vista en un punto a 6 metros durante 20 segundos. Ajustar el brillo de pantalla a la iluminación del aula. |
| R03 | Lesiones por movimientos repetitivos en muñeca y mano | Baja | Media | Mantener posicionamiento ergonómico. Evitar apoyar las muñecas sobre superficies duras al teclear. |
| R04 | Fatiga y estrés por plazos del proyecto | Media | Media | Planificar el trabajo en tareas de sprint manejables. Evitar sobrecargar sesiones individuales. Usar ProofHub para mantener visibilidad clara del progreso. |

---

### 3.2 Riesgos Eléctricos y de Equipamiento

| # | Riesgo | Probabilidad | Gravedad | Medidas Preventivas |
|---|--------|-------------|----------|---------------------|
| R05 | Descarga eléctrica por equipo defectuoso en el aula | Baja | Alta | No manipular cables ni hardware interno. Notificar al profesor cualquier equipo dañado de inmediato. |
| R06 | Sobrecalentamiento del equipo por carga elevada de máquinas virtuales | Baja | Media | Monitorizar el uso de CPU y RAM. No ejecutar más máquinas virtuales simultáneas de las que el hardware puede soportar. Cerrar aplicaciones no necesarias. |
| R07 | Pérdida accidental de datos por fallo de hardware | Baja | Alta | Hacer commit al repositorio de GitHub con regularidad. Nunca mantener la única copia de un documento en local. |

---

### 3.3 Riesgos Digitales y de Ciberseguridad

| # | Riesgo | Probabilidad | Gravedad | Medidas Preventivas |
|---|--------|-------------|----------|---------------------|
| R08 | Exposición accidental de credenciales de AWS | Media | Alta | No subir claves de acceso ni secretos a GitHub. Usar variables de entorno o roles IAM. Revocar y rotar las credenciales inmediatamente si se exponen. |
| R09 | Costes no previstos en AWS por recursos mal configurados | Media | Alta | Configurar una alerta de facturación al 80% del presupuesto disponible. Revisar las instancias activas diariamente. Terminar los recursos no utilizados al finalizar cada sesión. |
| R10 | Acceso involuntario a sistemas reales durante los ejercicios de Red Team | Baja | Alta | Todas las simulaciones de ataque se realizan exclusivamente dentro del entorno de laboratorio aislado. No se ejecuta ningún test contra sistemas externos reales. |
| R11 | Pérdida de datos del proyecto por borrado accidental | Baja | Alta | Trabajar con ramas en GitHub. No hacer force-push a la rama principal sin revisión previa. |
| R12 | Dependencia de una única región cloud ante una interrupción del servicio | Baja | Media | Documentar todas las configuraciones para que el entorno pueda reproducirse. Mantener la documentación de infraestructura actualizada. |

---

### 3.4 Riesgos Organizativos

| # | Riesgo | Probabilidad | Gravedad | Medidas Preventivas |
|---|--------|-------------|----------|---------------------|
| R13 | Expansión del alcance — añadir funcionalidades fuera del backlog | Media | Media | Seguir estrictamente el backlog definido. Cualquier cambio de alcance debe evaluarse frente al tiempo disponible antes de incorporarse a un sprint. |
| R14 | Estimación de tiempo incorrecta que provoca sobrecarga del sprint | Media | Media | Incluir un margen de tiempo en cada sprint. Si una tarea no se completa, trasladarla al siguiente sprint y documentar el motivo en la retrospectiva. |
| R15 | Documentación insuficiente durante la implementación | Media | Alta | Documentar cada componente a medida que se despliega, no al final del sprint. Seguir la definición de hecho: ninguna tarea está completa sin su documentación commiteada. |

---

## 4. Procedimientos de Emergencia

- **Emergencia médica en el aula:** notificar al profesor de inmediato y seguir el protocolo de emergencias del centro.
- **Incidente eléctrico:** no tocar el equipo afectado. Desconectar la alimentación en el cuadro eléctrico si es seguro hacerlo. Notificar al profesor.
- **Filtración de credenciales AWS:** revocar inmediatamente las credenciales expuestas desde la consola IAM de AWS, rotar todas las claves relacionadas y revisar los logs de CloudTrail para evaluar si ha habido actividad no autorizada.
- **Pérdida accidental de datos:** revisar el historial de commits y las copias de ramas en GitHub antes de intentar cualquier recuperación.