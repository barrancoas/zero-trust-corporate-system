# Hoja de Ruta Futura y Escalabilidad

**Proyecto:** Zero Trust Corporate System  
**Autor:** Asier Barranco  
**Fecha:** 12/05/2026  
**Versión:** 1.0  

---

## 1. Visión General

El Sistema Corporativo Zero Trust desplegado en este proyecto representa una base funcional y de calidad de producción. Todos los componentes principales — identidad federada, SSO, MFA, segmentación de red, protección perimetral y validación de seguridad — están operativos y documentados.

Este documento identifica la evolución natural de la arquitectura: mejoras, ampliaciones y componentes adicionales que se implementarían en un entorno de producción real o en una continuación de este proyecto. Cada elemento está fundamentado en limitaciones o restricciones observadas durante el despliegue actual.

La hoja de ruta se organiza en tres horizontes:

| Horizonte | Plazo | Enfoque |
|---|---|---|
| Corto plazo | 1–3 meses | Cerrar brechas conocidas y completar funcionalidades parcialmente implementadas |
| Medio plazo | 3–6 meses | Reforzar resiliencia, observabilidad y profundidad de seguridad |
| Largo plazo | 6–12 meses | Madurez Zero Trust completa y escalabilidad de nivel empresarial |

---

## 2. Mejoras a Corto Plazo

### 2.1 Integración SSO en Mattermost

**Estado actual:** Mattermost está desplegado y accesible en `https://abb-ztcs.com/mattermost/`. Funciona con cuentas locales, independientes del proveedor de identidad Keycloak.

**Brecha:** los usuarios deben mantener credenciales separadas para Mattermost, rompiendo el modelo de identidad única que es fundamental en Zero Trust.

**Siguiente paso:** configurar un cliente SAML 2.0 para Mattermost en Keycloak, siguiendo el mismo patrón utilizado para Nextcloud. Mattermost soporta SAML de forma nativa. La integración incluiría:

- Crear un cliente SAML dedicado `https://abb-ztcs.com/mattermost` en el realm `zerotrust`
- Configurar mappers de atributos para nombre de usuario, correo electrónico y nombre de pantalla
- Habilitar la autenticación SAML en la Consola del Sistema de Mattermost
- Aplicar la misma obligatoriedad de MFA (TOTP) ya activa para Nextcloud

**Estimación de esfuerzo:** 4–6 horas. Todos los prerrequisitos ya están en marcha.

---

### 2.2 LDAPS — LDAP Cifrado

**Estado actual:** Keycloak se comunica con Active Directory mediante LDAP sin cifrar en el puerto 389. El tráfico está protegido porque viaja a través del túnel VPN WireGuard, pero el propio LDAP no está cifrado.

**Brecha:** esto crea una dependencia entre la seguridad de LDAP y la disponibilidad de la VPN. Si el túnel WireGuard falla o el servicio se detiene, el tráfico LDAP dejaría de funcionar o, en un escenario mal configurado, viajaría sin cifrar.

**Siguiente paso:** migrar de LDAP (389) a LDAPS (636), que cifra el tráfico de directorio a nivel de protocolo mediante TLS, independientemente del túnel VPN.

La implementación requiere:
- Instalar el rol de Servicios de Certificados de Active Directory (AD CS) en DC01 para emitir un certificado interno para el controlador de dominio
- Configurar el proveedor LDAP de Keycloak para usar `ldaps://192.168.56.10:636` con el certificado de la CA interna en el almacén de confianza de Keycloak
- Abrir el puerto 636 en las reglas de enrutamiento de WireGuard

**Estimación de esfuerzo:** 3–4 horas.

---

### 2.3 Sistema de Copias de Seguridad Automatizado

**Estado actual:** los procedimientos de backup están documentados en el manual de administrador pero no están automatizados. La copia de seguridad de datos requiere la ejecución manual de comandos Docker y volcados de MariaDB.

**Brecha:** los backups manuales son propensos a errores y fácilmente olvidados. En un entorno de producción, la pérdida de datos por backups inexistentes es inaceptable.

**Siguiente paso:** implementar scripts de backup automatizados ejecutados mediante cron en `ztcs-services`:

```bash
# Ejemplo de entrada cron — se ejecuta diariamente a las 02:00
0 2 * * * /home/ubuntu/scripts/backup.sh >> /var/log/backup.log 2>&1
```

El script de backup realizaría:
- Volcado de todas las bases de datos MariaDB a un archivo SQL con marca de tiempo
- Archivo del volumen de datos de Nextcloud
- Exportación de la configuración del realm de Keycloak vía API
- Retención de los últimos 7 backups diarios, rotando los más antiguos
- Transferencia del archivo de backup a `ztcs-perimeter` o un bucket S3 para almacenamiento fuera de la instancia

**Estimación de esfuerzo:** 2–3 horas para scripting y pruebas.

---

### 2.4 Script de Monitorización de Logs

**Estado actual:** los logs de Nginx, Fail2ban, Keycloak y Nextcloud están disponibles en sus respectivas instancias pero deben revisarse manualmente.

**Brecha:** no existe una vista centralizada de eventos de seguridad ni alertas automáticas ante actividad sospechosa.

**Siguiente paso:** implementar un script de monitorización ligero que agregue e informe sobre eventos de seguridad clave:

- Intentos de autenticación fallidos (eventos LOGIN_ERROR de Keycloak vía journald)
- Eventos de baneo de Fail2ban
- Tasas de errores 4xx/5xx de Nginx
- Aviso de caducidad de certificado (< 30 días restantes)

El script se ejecutaría como tarea cron diaria, generando un informe de resumen commiteado al repositorio o enviado mediante notificación por correo electrónico o webhook.

**Estimación de esfuerzo:** 3–4 horas.

---

## 3. Mejoras a Medio Plazo

### 3.1 Alta Disponibilidad — Eliminación de Puntos Únicos de Fallo

**Estado actual:** cada componente se ejecuta en una única instancia. Si `ztcs-perimeter` falla, todos los servicios quedan inaccesibles. Si `ztcs-services` falla, todas las aplicaciones corporativas se detienen.

**Brecha:** la arquitectura no tiene redundancia. Cualquier fallo de instancia provoca una interrupción completa del servicio.

**Siguiente paso:** introducir redundancia en cada capa:

| Capa | Actual | Objetivo |
|---|---|---|
| Perímetro (Nginx) | 1× EC2 t3.small | 2× EC2 detrás de AWS Application Load Balancer |
| Identidad (Keycloak) | 1× contenedor (modo dev) | Clúster Keycloak (2+ nodos) con base de datos compartida |
| Servicios (Nextcloud) | 1× contenedor | 2× contenedores detrás de balanceador de carga interno |
| Base de datos (MariaDB) | 1× contenedor | MariaDB Galera Cluster (3 nodos) o AWS RDS |

Esta transición también requiere migrar Keycloak del modo desarrollo (base de datos H2 embebida) a una base de datos externa de nivel productivo, que es un prerrequisito para el clustering.

**Estimación de esfuerzo:** significativo — 2–3 semanas para un despliegue HA completo.

---

### 3.2 Gestión Centralizada de Logs (SIEM)

**Estado actual:** los logs están distribuidos entre `ztcs-perimeter` y `ztcs-services` sin una vista unificada.

**Siguiente paso:** desplegar una pila de logging centralizado. Una opción ligera para esta arquitectura sería:

- **Loki** (agregación de logs) + **Promtail** (agente de envío de logs) + **Grafana** (panel de visualización)
- Alternativamente, una pila **ELK completa** (Elasticsearch, Logstash, Kibana) para consultas más potentes

Todos los servicios enviarían logs a la pila central, permitiendo:
- Paneles en tiempo real para eventos de autenticación, intentos fallidos y actividad de baneo
- Correlación de eventos entre servicios
- Políticas de retención y trazas de auditoría para cumplimiento normativo

**Estimación de esfuerzo:** 1–2 semanas.

---

### 3.3 PKI Interna — Autoridad de Certificación

**Estado actual:** los certificados TLS son emitidos por Let's Encrypt para el dominio público `abb-ztcs.com`. Los servicios internos se comunican mediante HTTP dentro de la red Docker y la VPC.

**Brecha:** la comunicación interna entre servicios (Nginx → Nextcloud, Nginx → Keycloak) no está cifrada. En un modelo Zero Trust, todo el tráfico — incluido el interno — debería estar cifrado.

**Siguiente paso:** desplegar una Autoridad de Certificación interna mediante **Active Directory Certificate Services (AD CS)** en DC01, o una alternativa ligera como **step-ca** o **Vault PKI**. Esto permitiría:

- Certificados TLS para todos los endpoints de servicios internos
- TLS mutuo (mTLS) entre servicios — cada servicio presenta un certificado
- Certificados LDAPS para el controlador de dominio (prerrequisito para el punto 2.2)

**Estimación de esfuerzo:** 1–2 semanas para una PKI interna completa.

---

### 3.4 Infraestructura como Código (IaC)

**Estado actual:** toda la infraestructura AWS fue aprovisionada manualmente a través de la consola. La configuración está documentada en Markdown pero no puede reproducirse automáticamente.

**Brecha:** el aprovisionamiento manual es lento, propenso a errores y no repetible.

**Siguiente paso:** reescribir la infraestructura AWS como código utilizando **Terraform** o **AWS CloudFormation**:

```hcl
# Ejemplo de recurso Terraform
resource "aws_vpc" "ztcs_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "ztcs-vpc" }
}
```

Con IaC, todo el entorno puede destruirse y recrearse en minutos con un único comando. Esto también permite cambios de infraestructura con control de versiones y recuperación ante desastres.

**Estimación de esfuerzo:** 1–2 semanas para Terraform completo de la arquitectura existente.

---

## 4. Visión a Largo Plazo — Madurez Zero Trust Completa

### 4.1 Identidad de Dispositivo y Confianza en el Endpoint

**Estado actual:** Zero Trust se aplica en la capa de identidad — cada usuario debe autenticarse con credenciales y MFA. Sin embargo, la arquitectura no evalúa la fiabilidad del dispositivo desde el que se realiza la solicitud.

**Limitación:** un usuario válido autenticándose desde un dispositivo comprometido o no gestionado sigue obteniendo acceso.

**Siguiente paso:** integrar una solución de **Gestión de Dispositivos Móviles (MDM)** como **Jamf**, **Microsoft Intune** o **Fleet** (código abierto). Keycloak puede configurarse para exigir el cumplimiento del dispositivo como condición de acceso — solo los dispositivos registrados en el MDM y que cumplan las bases de seguridad definidas tendrían permitida la autenticación.

---

### 4.2 Políticas de Acceso Condicional

**Estado actual:** la autenticación es binaria — un usuario pasa el control de identidad o no. No existe control de acceso basado en contexto.

**Siguiente paso:** implementar políticas de acceso contextual en Keycloak mediante **Flujos de Autenticación** y **Condiciones**:

- Requerir autenticación escalonada (factor MFA adicional) para operaciones sensibles
- Bloquear acceso desde ubicaciones geográficas desconocidas o rangos de IP de alto riesgo
- Aplicar restricciones de acceso basadas en horario (sin acceso fuera del horario laboral para usuarios estándar)
- Requerir reautenticación tras un período de inactividad

---

### 4.3 Microsegmentación de Red

**Estado actual:** la subred privada trata todos los contenedores como igualmente confiables. Nextcloud, Mattermost y MariaDB comparten una única red bridge de Docker.

**Siguiente paso:** aplicar microsegmentación de red dentro de la subred privada:

- Redes Docker separadas por capa de servicio (red de base de datos, red de aplicación)
- Reglas de permiso explícitas entre capas — Nextcloud puede acceder a MariaDB, pero Mattermost no puede acceder a la base de datos de Nextcloud
- Reglas de Security Group de AWS que restringen el tráfico entre servicios a nivel de VPC

En un entorno de producción, esto se extendería a tecnología de **service mesh** (Istio, Linkerd) con mTLS entre todos los servicios.

---

### 4.4 Gestión de Secretos

**Estado actual:** las contraseñas de servicios y credenciales se almacenan en archivos `docker-compose.yml` en la instancia. Las versiones del repositorio están redactadas, pero los archivos de configuración activos contienen secretos en texto plano.

**Siguiente paso:** integrar una solución dedicada de gestión de secretos como **HashiCorp Vault** o **AWS Secrets Manager**:

- Todas las credenciales de servicio almacenadas en Vault, nunca en archivos de configuración
- Credenciales de base de datos dinámicas — contraseñas de MariaDB rotadas automáticamente por Vault
- Registro de auditoría de cada acceso a secretos
- Rotación automática de secretos según un calendario definido

---

### 4.5 Automatización de Seguridad y SOAR

**Estado actual:** la respuesta a incidentes es manual — un baneo de Fail2ban requiere que un administrador investigue, decida acciones adicionales y documente el evento.

**Siguiente paso:** implementar un flujo de trabajo de **Orquestación, Automatización y Respuesta de Seguridad (SOAR)**:

- Enriquecimiento automático de eventos de baneo de Fail2ban con inteligencia de amenazas (consulta de reputación de IP)
- Escalado automático de eventos de baneo repetidos a un canal de notificación (correo electrónico, webhook de Mattermost)
- Playbooks de auto-recuperación — si Keycloak no responde, reiniciar automáticamente el contenedor y alertar al administrador

---

## 5. Tabla Resumen

| Elemento | Horizonte | Esfuerzo | Impacto |
|---|---|---|---|
| SSO Mattermost | Corto | Bajo | Completa la unificación de identidad |
| LDAPS | Corto | Bajo | Elimina dependencia de VPN para seguridad LDAP |
| Backups automatizados | Corto | Bajo | Previene pérdida de datos |
| Script monitorización logs | Corto | Bajo | Visibilidad básica de seguridad |
| Alta disponibilidad | Medio | Alto | Elimina puntos únicos de fallo |
| Logging centralizado (SIEM) | Medio | Medio | Observabilidad de seguridad completa |
| PKI interna | Medio | Medio | Cifra todo el tráfico interno |
| Infraestructura como Código | Medio | Medio | Infraestructura reproducible y versionada |
| Confianza en dispositivo (MDM) | Largo | Alto | Zero Trust real — identidad + dispositivo |
| Acceso condicional | Largo | Medio | Políticas de seguridad contextuales |
| Microsegmentación de red | Largo | Alto | Elimina la confianza interna implícita |
| Gestión de secretos | Largo | Medio | Elimina credenciales estáticas |
| Automatización de seguridad (SOAR) | Largo | Alto | Respuesta a incidentes automatizada |

---

## 6. Conclusión

El Sistema Corporativo Zero Trust, tal como está desplegado, implementa los pilares fundamentales del modelo Zero Trust: identidad verificada, MFA obligatorio, segmentación de red, comunicaciones cifradas y prevención activa de intrusiones. Demuestra que una arquitectura Zero Trust de nivel productivo es alcanzable con herramientas de código abierto y recursos cloud modestos.

Los elementos de esta hoja de ruta representan la progresión natural de madurez de la arquitectura — desde un despliegue de entorno único bien funcional hacia una infraestructura de seguridad resiliente, completamente automatizada y de nivel empresarial. Cada paso se construye directamente sobre lo ya implementado, sin requerir cambios arquitectónicos en el diseño central.

Los principios permanecen constantes a través de todos los horizontes: **nunca confiar, siempre verificar** — aplicado progresivamente a cada capa del sistema.