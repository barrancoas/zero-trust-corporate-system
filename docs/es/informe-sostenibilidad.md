# Informe de Sostenibilidad

**Proyecto:** Zero Trust Corporate System
**Autor:** Asier Barranco
**Fecha:** 20/04/2026
**Versión:** 1.0

---

## 1. Introducción

Este informe analiza las dimensiones ambiental, social y de gobernanza (ASG) del proyecto Zero Trust Corporate System. Evalúa las implicaciones de sostenibilidad de las decisiones arquitectónicas adoptadas, con especial atención al modelo de nube híbrida y su impacto respecto a enfoques de infraestructura tradicionales.

El análisis sigue el marco ASG utilizado en los estándares internacionales de sostenibilidad, evaluando cada dimensión de forma independiente antes de extraer conclusiones globales.

---

## 2. Dimensión Ambiental (A)

### 2.1 Consumo Energético y Eficiencia Cloud

El proyecto adopta una arquitectura híbrida que combina un componente on-premise mínimo (una única máquina virtual VirtualBox que actúa como controlador de dominio Active Directory) con servicios alojados en AWS. Esta decisión tiene implicaciones ambientales directas.

La infraestructura corporativa tradicional depende de servidores físicos permanentemente encendidos en centros de datos propios, consumiendo energía independientemente del uso real. El modelo AWS opera bajo un principio de infraestructura compartida: el hardware físico se comparte entre miles de clientes y los recursos se asignan de forma dinámica. Los centros de datos de AWS reportan consistentemente valores de eficiencia energética (PUE) significativamente por debajo de la media del sector, y la compañía se ha comprometido a alimentar sus operaciones globales con energía 100% renovable.

La naturaleza bajo demanda de las instancias EC2 permite detenerlas cuando no están en uso, eliminando el consumo energético en reposo. En el contexto de este proyecto, las instancias pueden apagarse fuera del horario de trabajo, reduciendo la huella energética efectiva a las horas de uso activo.

### 2.2 Huella de Hardware

Al desplegar los servicios en infraestructura cloud en lugar de hardware físico dedicado, el proyecto evita el impacto de fabricación y eliminación de servidores adicionales. El único hardware físico involucrado es el equipo de trabajo existente utilizado para VirtualBox, que ya estaba en uso y no requiere ninguna adquisición adicional.

La contenedorización con Docker mejora adicionalmente la eficiencia de recursos: múltiples servicios (Nextcloud, Mattermost, MariaDB) se ejecutan en una única instancia EC2 en lugar de requerir máquinas físicas o virtuales dedicadas para cada servicio, reduciendo tanto el consumo energético como el desperdicio de recursos.

### 2.3 Alineación con la Economía Circular

El uso exclusivo de software open source elimina los ciclos de reemplazo de hardware impulsados por licencias, una fuente habitual de eliminación innecesaria de equipos en entornos de software propietario. La arquitectura está diseñada para ser reproducible y portable — si cambia el proveedor cloud o el tipo de instancia, la misma configuración Docker Compose puede redesplegar sin cambios de infraestructura, extendiendo la vida útil del sistema y reduciendo el desperdicio.

---

## 3. Dimensión Social (S)

### 3.1 Seguridad de Datos y Privacidad de Usuarios

El principal impacto social de este proyecto es la protección de los datos corporativos de los usuarios. El modelo Zero Trust garantiza que ningún usuario pueda acceder a los recursos corporativos sin una autenticación explícita y verificada, reduciendo el riesgo de brechas de datos causadas por credenciales comprometidas o accesos internos no autorizados.

Todo el tráfico entre los usuarios y los servicios corporativos se cifra de extremo a extremo mediante TLS/HTTPS. La autenticación está protegida por Autenticación de Múltiples Factores obligatoria, lo que reduce significativamente el riesgo de toma de control de cuentas incluso cuando las contraseñas están comprometidas.

### 3.2 Cumplimiento Normativo

La arquitectura está diseñada con principios de protección de datos alineados con la LOPDGDD (Ley Orgánica de Protección de Datos y Garantía de los Derechos Digitales), la implementación española del marco RGPD. Las medidas de cumplimiento integradas en la arquitectura incluyen:

- Control de acceso basado en el principio de mínimo privilegio, aplicado mediante políticas de grupo de Active Directory y asignaciones de roles en Keycloak.
- Registro de auditoría de todos los eventos de autenticación e intentos de acceso, permitiendo la trazabilidad y la respuesta ante incidentes.
- Datos almacenados en subredes privadas sin exposición directa al exterior, minimizando la superficie de ataque.
- Cifrado en tránsito para todos los servicios orientados al usuario y la comunicación entre servicios a través del túnel VPN.

### 3.3 Desarrollo de Competencias Digitales

El proyecto contribuye a la dimensión social de la sostenibilidad mediante el desarrollo de competencias digitales avanzadas en ciberseguridad, infraestructura cloud y administración de sistemas. Las habilidades adquiridas — diseño de arquitecturas Zero Trust, federación de identidades, pruebas de seguridad ofensiva y defensiva — están directamente alineadas con la creciente demanda del mercado laboral en el sector de la ciberseguridad, contribuyendo a la empleabilidad individual y al objetivo más amplio de reducir la brecha de competencias en ciberseguridad en el mercado europeo.

---

## 4. Dimensión de Gobernanza (G)

### 4.1 Open Source y Soberanía Tecnológica

Todo el stack de servicios está construido sobre software open source: Keycloak, Nginx, Nextcloud, Mattermost, MariaDB y WireGuard. Esta elección tiene implicaciones significativas de gobernanza. El software open source elimina la dependencia de un único proveedor — si algún componente necesita ser reemplazado, existe una alternativa open source equivalente sin barreras contractuales ni de licencia. También garantiza que el software pueda ser auditado, lo que significa que las vulnerabilidades de seguridad pueden ser identificadas y corregidas por la comunidad en lugar de depender del ciclo de parches de un proveedor.

Esto se alinea con el impulso más amplio de la Comisión Europea hacia la soberanía digital y la adopción de software open source en instituciones públicas y educativas.

### 4.2 Viabilidad Económica

El coste total de infraestructura de este proyecto está acotado por el crédito AWS disponible (~40$). El desglose de costes estimados es el siguiente:

| Recurso | Coste mensual estimado |
|---------|----------------------|
| EC2 t3.small (subred pública — proxy + Keycloak) | ~15$ |
| EC2 t3.medium (subred privada — servicios Docker) | ~20$ |
| Transferencia de datos y almacenamiento | ~3$ |
| **Total estimado** | **~38$/mes** |

Este modelo de costes demuestra la viabilidad económica de una arquitectura Zero Trust de grado productivo a coste mínimo cuando se construye sobre componentes open source. La solución comercial equivalente (Okta + WAF gestionado + gestión documental SaaS) costaría varios cientos de dólares al mes a la misma escala.

### 4.3 Escalabilidad y Sostenibilidad a Largo Plazo

La arquitectura está diseñada para escalar horizontalmente. Se pueden añadir instancias EC2 adicionales a la subred privada para gestionar mayor carga en Nextcloud o Mattermost sin cambios arquitectónicos. Keycloak soporta clustering para alta disponibilidad. Las configuraciones Docker Compose pueden migrarse a plataformas de orquestación de contenedores si la organización crece.

El uso de Infraestructura-como-Documentación — todos los pasos de despliegue están documentados en el repositorio — garantiza que el entorno pueda ser reproducido, transferido o ampliado por cualquier administrador cualificado, reduciendo la dependencia de una única persona y mejorando la resiliencia organizativa.

### 4.4 Metodología Ágil y Gobernanza del Proyecto

El proyecto se gestiona mediante el marco SCRUM con sprints definidos, sesiones de planificación y retrospectivas. Esta metodología garantiza la entrega continua de valor, la detección temprana de bloqueos técnicos y el seguimiento transparente del progreso a través de ProofHub y GitHub. El enfoque documentación-primero — commiteando actas de sprint, diagramas de arquitectura y guías de configuración al repositorio — crea un rastro auditable de decisiones y progreso que apoya la responsabilidad y la transferencia de conocimiento.

---

## 5. Conclusiones

El proyecto Zero Trust Corporate System demuestra que una infraestructura de seguridad robusta y de grado productivo puede construirse de forma sostenible — ambiental, social y económicamente — utilizando tecnologías open source sobre infraestructura cloud.

El modelo de nube híbrida reduce los requisitos de hardware físico y el consumo energético respecto a los despliegues on-premise tradicionales. La arquitectura Zero Trust aplica principios sólidos de protección de datos alineados con los marcos regulatorios vigentes. El stack open source elimina la dependencia de proveedores, reduce costes y apoya la soberanía tecnológica. La metodología ágil garantiza una gobernanza del proyecto transparente y responsable.

En conjunto, estas características hacen que el proyecto no solo sea técnicamente sólido, sino también alineado con los principios de infraestructura digital responsable y sostenible.